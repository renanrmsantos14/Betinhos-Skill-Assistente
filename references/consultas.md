# Consultas prontas (`read_query`)

Regras do `read_query`: máximo `TOP 20`; sem subconsulta, `DISTINCT`, `HAVING` ou `CAST`. Colunas de lookup e choice têm versão texto com sufixo `name` (ex.: `cr40f_clientename`, `cr40f_statusname`).

**Fuso:** em `read_query`, tanto o retorno quanto o `WHERE` de data/hora usam **horário de Brasília**. Só na **gravação** (`create_record`/`update_record`) a data vai em UTC (+3h, sufixo `Z`).

## Cadastros

Cliente:
```sql
SELECT TOP 10 cr40f_clientes1id, cr40f_nomedocliente, cr40f_endereco
FROM cr40f_clientes1
WHERE cr40f_nomedocliente LIKE '%johnson%' AND statecode = 0
```

Passageiro/solicitante do cliente:
```sql
SELECT TOP 10 cr40f_bancodedadosid, cr40f_nomedopassageiro, cr40f_telefone, cr40f_email,
       cr40f_enderecodesaida, new_tipodoveiculoname, cr40f_preferenciasdopassageiro, cr40f_classificacaoname
FROM cr40f_bancodedados
WHERE cr40f_nomedopassageiro LIKE '%gabriela%' AND cr40f_cliente = '<guid-cliente>' AND cr40f_status = 202410000
```
Sem resultado: repita sem o filtro de cliente e confirme com o usuário antes de usar.

## Histórico (padrões do cliente/passageiro)
```sql
SELECT TOP 10 cr40f_id, cr40f_dataehorriodesada, cr40f_trajeto, cr40f_endereodesada, cr40f_destino,
       cr40f_tipodeveiculoname, cr40f_tipodoserviconame, cr40f_passageirosetelefonedecontato
FROM cr40f_reservadeveculos
WHERE cr40f_cliente = '<guid-cliente>' AND new_categoriadoitem = 100000000
ORDER BY createdon DESC
```
Para um passageiro sem lookup preenchido, troque o filtro de cliente por `cr40f_passageirosetelefonedecontato LIKE '%<nome>%'`.

## Duplicidade (antes de criar)
```sql
SELECT TOP 10 cr40f_id, cr40f_dataehorriodesada, cr40f_trajeto, cr40f_statusname, cr40f_passageirosetelefonedecontato
FROM cr40f_reservadeveculos
WHERE cr40f_dataehorriodesada >= '2026-10-02T00:00:00' AND cr40f_dataehorriodesada < '2026-10-03T00:00:00'
  AND cr40f_passageirosetelefonedecontato LIKE '%<nome>%'
  AND cr40f_status <> 202410002 AND cr40f_status <> 202410010
```

Chave de idempotência livre?
```sql
SELECT cr40f_reservadeveculosid FROM cr40f_reservadeveculos WHERE cr40f_iachaveidempotencia = '<chave>'
```

## Conferência (após criar)
```sql
SELECT cr40f_reservadeveculosid, cr40f_id, cr40f_idnovo, cr40f_statusname, cr40f_dataehorriodesada,
       cr40f_trajeto, cr40f_passageirosetelefonedecontato
FROM cr40f_reservadeveculos
WHERE cr40f_iachaveidempotencia = '<chave>'
```
OK somente se `cr40f_id` começar com `OS-` seguido de números.

## Agenda

Serviços de um dia (exclui cancelados):
```sql
SELECT TOP 20 cr40f_id, cr40f_dataehorriodesada, cr40f_clientename, cr40f_passageirosetelefonedecontato,
       cr40f_trajeto, cr40f_tipodeveiculoname, cr40f_statusname, cr40f_motoristaname, cr40f_veiculoname
FROM cr40f_reservadeveculos
WHERE cr40f_dataehorriodesada >= '2026-10-02T00:00:00' AND cr40f_dataehorriodesada < '2026-10-03T00:00:00'
  AND new_categoriadoitem = 100000000
  AND cr40f_status <> 202410002 AND cr40f_status <> 202410010
ORDER BY cr40f_dataehorriodesada
```
Dias cheios passam de 20: divida em faixas (00–12h, 12–18h, 18–24h) e junte.

Contagem por status no dia:
```sql
SELECT cr40f_status, COUNT(cr40f_reservadeveculosid) AS total
FROM cr40f_reservadeveculos
WHERE cr40f_dataehorriodesada >= '2026-10-02T00:00:00' AND cr40f_dataehorriodesada < '2026-10-03T00:00:00'
  AND new_categoriadoitem = 100000000
GROUP BY cr40f_status
```
Agrupe pelo código (`cr40f_status`); colunas `...name` não aceitam `GROUP BY`, mas o retorno já traz o rótulo. Use a contagem para saber se o dia passa de 20 serviços.

Serviços de um motorista: acrescente `AND cr40f_motoristaname LIKE '%<nome>%'` à consulta do dia.

Sem motorista/veículo (pendentes de programação): acrescente `AND cr40f_motorista IS NULL`.
