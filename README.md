# Consolidated Replay no Oracle RAT

Guia em Markdown para GitHub, baseado no procedimento descrito no artigo sobre `Consolidated Replay` com Oracle Real Application Testing.

## Visao Geral

Este material descreve um fluxo de teste de performance usando `Database Replay` em ambiente Oracle multitenant, consolidando capturas de workload de multiplos PDBs em um unico replay.

Cenario de referencia:

- ambiente de origem em `Oracle RAC`
- banco `multitenant` com varios PDBs
- `standby fisico` criado no novo hardware
- conversao posterior do standby para `snapshot standby`
- execucao do replay nesse ambiente de destino

## Estrutura de Diretorios

As capturas devem ficar organizadas em um diretorio raiz com um subdiretorio por PDB:

```text
/u01/rat
|-- pdb1
|-- pdb2
|-- pdb3
`-- pdb4
```

Essa estrutura facilita o uso do `Consolidated Replay`, no qual varias capturas sao agrupadas e executadas simultaneamente.

## 1. Captura do Workload por PDB

Em ambiente multitenant, a captura precisa ser feita individualmente em cada PDB.

Exemplo:

```sql
ALTER SESSION SET CONTAINER=PDB1;
CREATE DIRECTORY PDB1_DIR AS '/u01/rat/pdb1';
EXEC DBMS_WORKLOAD_CAPTURE.START_CAPTURE(
  name => 'CAPTURA_PDB1',
  dir => 'PDB1_DIR',
  duration => 7200
);
```

Repita o procedimento para os demais PDBs, apontando cada um para seu proprio diretorio.

## 2. Exportacao do AWR das Capturas

Ao final da captura, exporte os dados de AWR de cada PDB para permitir a comparacao posterior com o replay:

```sql
ALTER SESSION SET CONTAINER=PDB1;
BEGIN
  DBMS_WORKLOAD_CAPTURE.EXPORT_AWR(capture_id => 4);
END;
/
```

Para descobrir o `capture_id`, consulte:

```sql
SELECT id capture_id,
       name,
       status,
       TO_CHAR(start_time,'dd/mm/yy hh24:mi') start_time,
       TO_CHAR(end_time,'dd/mm/yy hh24:mi') end_time
  FROM dba_workload_captures
 ORDER BY id DESC;
```

## 3. Preparacao do Snapshot Standby

Antes do replay:

- converter o `standby fisico` em `snapshot standby`
- montar o armazenamento compartilhado com os arquivos das capturas
- criar os diretorios no `CDB$ROOT`

Exemplo:

```sql
CREATE DIRECTORY RAT_DIR  AS '/u01/rat/';
CREATE DIRECTORY PDB1_DIR AS '/u01/rat/pdb1';
CREATE DIRECTORY PDB2_DIR AS '/u01/rat/pdb2';
CREATE DIRECTORY PDB3_DIR AS '/u01/rat/pdb3';
CREATE DIRECTORY PDB4_DIR AS '/u01/rat/pdb4';
```

## 4. Processamento das Capturas

Cada captura deve ser processada antes do replay:

```sql
EXEC DBMS_WORKLOAD_REPLAY.PROCESS_CAPTURE('PDB1_DIR');
EXEC DBMS_WORKLOAD_REPLAY.PROCESS_CAPTURE('PDB2_DIR');
EXEC DBMS_WORKLOAD_REPLAY.PROCESS_CAPTURE('PDB3_DIR');
EXEC DBMS_WORKLOAD_REPLAY.PROCESS_CAPTURE('PDB4_DIR');
```

Depois, defina o diretorio raiz do replay:

```sql
EXEC DBMS_WORKLOAD_REPLAY.SET_REPLAY_DIRECTORY('RAT_DIR');
```

## 5. Criacao do Replay Schedule

Como existem varias capturas, e necessario criar um `schedule` consolidado:

```sql
EXEC DBMS_WORKLOAD_REPLAY.BEGIN_REPLAY_SCHEDULE('MY_SCHEDULE');
SELECT DBMS_WORKLOAD_REPLAY.ADD_CAPTURE('PDB1_DIR') FROM DUAL;
SELECT DBMS_WORKLOAD_REPLAY.ADD_CAPTURE('PDB2_DIR') FROM DUAL;
SELECT DBMS_WORKLOAD_REPLAY.ADD_CAPTURE('PDB3_DIR') FROM DUAL;
SELECT DBMS_WORKLOAD_REPLAY.ADD_CAPTURE('PDB4_DIR') FROM DUAL;
EXEC DBMS_WORKLOAD_REPLAY.END_REPLAY_SCHEDULE;
```

Os `schedule_cap_id` gerados nessa etapa sao fundamentais para o remapeamento das conexoes:

```sql
SELECT schedule_cap_id, capture_dir
  FROM dba_workload_schedule_captures
 WHERE schedule_name = 'MY_SCHEDULE';
```

## 6. Inicializacao do Consolidated Replay

Com o schedule pronto:

```sql
EXEC DBMS_WORKLOAD_REPLAY.INITIALIZE_CONSOLIDATED_REPLAY(
  'MY_CONSOLIDATED_REPLAY',
  'MY_SCHEDULE'
);
```

## 7. Remapeamento das Conexoes

Como o replay e inicializado no `CDB$ROOT`, cada conexao precisa ser redirecionada ao servico correto do PDB correspondente.

Consulta de apoio:

```sql
SELECT conn_id, schedule_cap_id, replay_conn
  FROM dba_workload_connection_map
 WHERE replay_id = 1;
```

Exemplo de remapeamento manual:

```sql
EXEC DBMS_WORKLOAD_REPLAY.REMAP_CONNECTION(
  connection_id     => 1,
  schedule_cap_id   => 1,
  replay_connection => '(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=mydbhost.domain)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=PDB1_SERVICE)))'
);
```

Quando houver muitas conexoes, vale automatizar o remapeamento com PL/SQL usando `schedule_cap_id` para decidir qual `SERVICE_NAME` aplicar.

## 8. Preparacao do Replay

Depois do remapeamento:

```sql
EXEC DBMS_WORKLOAD_REPLAY.PREPARE_CONSOLIDATED_REPLAY(
  SYNCHRONIZATION => 'TIME'
);
```

## 9. Calibracao com WRC

A calibracao deve ser executada para cada diretorio de captura:

```bash
wrc system@my_cdb mode=calibrate replaydir=/u01/rat/PDB1
wrc system@my_cdb mode=calibrate replaydir=/u01/rat/PDB2
wrc system@my_cdb mode=calibrate replaydir=/u01/rat/PDB3
wrc system@my_cdb mode=calibrate replaydir=/u01/rat/PDB4
```

Some a recomendacao de clients de todos os diretorios para definir quantos processos `wrc` devem ser iniciados no replay consolidado.

Importante: no `Consolidated Replay`, os clients devem apontar para o diretorio raiz, e nao para os subdiretorios individuais.

```bash
nohup wrc system/senha@my_cdb mode=replay replaydir=/u01/rat > out1.log 2>&1 &
```

## 10. Inicio e Acompanhamento do Replay

Com os clients conectados:

```sql
EXEC DBMS_WORKLOAD_REPLAY.START_CONSOLIDATED_REPLAY;
```

Para acompanhar o status:

```sql
SELECT id, name, start_time, end_time, status
  FROM dba_workload_replays
 ORDER BY id DESC;
```

## 11. Importacao do AWR e Relatorio Comparativo

Crie um schema de staging para importar os dados de AWR exportados nas capturas:

```sql
CREATE USER C##RAT_STAGE IDENTIFIED BY RAT_STAGE DEFAULT TABLESPACE USERS;
GRANT DBA TO C##RAT_STAGE;
```

Importe o AWR de cada captura:

```sql
VAR ret NUMBER
EXEC :ret := DBMS_WORKLOAD_CAPTURE.IMPORT_AWR(
  capture_id     => 21,
  staging_schema => 'C##RAT_STAGE'
);
PRINT ret
```

Por fim, gere o relatorio comparativo entre captura e replay:

```sql
VAR comp_report CLOB;
EXEC DBMS_WORKLOAD_REPLAY.COMPARE_PERIOD_REPORT(
  replay_id1 => 1,
  replay_id2 => NULL,
  format     => 'HTML',
  result     => :comp_report
);
```

## Observacoes Praticas

- Em ambiente multitenant, cada PDB exige sua propria captura.
- O ponto central do `Consolidated Replay` e agrupar as capturas em um `schedule`.
- O remapeamento de conexoes e uma das etapas mais importantes do processo.
- O `snapshot standby` permite testar mudancas sem impacto no ambiente produtivo definitivo.
- O relatorio comparativo ajuda a identificar gargalos antes da migracao para o novo hardware.
