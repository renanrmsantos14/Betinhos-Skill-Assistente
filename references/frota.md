# Frota — somente leitura

Metadata conferida em DEV em 02/10/2026. **Nunca grave** nestas tabelas: manutenção, troca de carro e multa têm fluxo próprio no sistema. Pedido de alteração → explique que é feito no app e ofereça a consulta.

## Veículos (`cr40f_veiculos`)
```sql
SELECT TOP 20 cr40f_veiculosid, cr40f_placa, cr40f_marca, cr40f_modelo, cr40f_cor, cr40f_blindado,
       new_categoriadoveiculoname, cr40f_motoristaatualname
FROM cr40f_veiculos
WHERE cr40f_statusdoveiculo = 202410001 AND statecode = 0
ORDER BY cr40f_placa
```
Por placa: `AND cr40f_placa LIKE '%ABC%'`. Categoria: Próprio, Terceiro, Aluguel.

## Posse (quem está com qual carro) — `new_possedeveiculo`
```sql
SELECT TOP 20 new_veiculoname, new_motoristaname, new_iniciodaposse, new_fimdaposse
FROM new_possedeveiculo
WHERE new_veiculo = '<guid-veiculo>'
ORDER BY new_iniciodaposse DESC
```
Posse aberta = sem `new_fimdaposse`. Para o motorista, troque o filtro por `new_motorista = '<guid>'`.

## Manutenções (`cr40f_manutencoes`)
Status: Solicitado 0 · Aprovado 202410001 · Programado 202410005 · Realizado 202410002 · Reprovado 100000002 · Cancelado 100000001.

Manutenções do veículo (abertas):
```sql
SELECT TOP 10 cr40f_id, cr40f_statusname, cr40f_agendarpara, cr40f_graudamanutencaoname, cr40f_tipodoreparoname,
       cr40f_descricao, cr40f_estabelecimento
FROM cr40f_manutencoes
WHERE cr40f_placa_carro = '<guid-veiculo>'
  AND (cr40f_status = 0 OR cr40f_status = 202410001 OR cr40f_status = 202410005)
ORDER BY cr40f_agendarpara
```
Todas as abertas da frota: retire o filtro de veículo e inclua `cr40f_placa_carroname`. Histórico e custo: troque o filtro de status por `cr40f_status = 202410002` e inclua `cr40f_datamanutencao`, `cr40f_valor`, `cr40f_kmatual`.

## Trocas de carro (`cr40f_trocasdecarro`)
Status: Programada 202410000 · Confirmada 100000001 · Concluída 202410001 · Cancelada 202410002.
```sql
SELECT TOP 10 cr40f_id, cr40f_statusdatrocaname, new_tipodetrocaname, cr40f_iniciodajaneladetroca, cr40f_fimdajaneladetroca,
       cr40f_motorista1name, cr40f_veiculo1antesdatrocaname, cr40f_motorista2name, cr40f_veiculo2antesdatrocaname
FROM cr40f_trocasdecarro
WHERE cr40f_iniciodajaneladetroca < '2026-10-04T00:00:00' AND cr40f_fimdajaneladetroca >= '2026-10-03T00:00:00'
  AND cr40f_statusdatroca <> 202410002
ORDER BY cr40f_iniciodajaneladetroca
```
De um motorista: `AND (cr40f_motorista1 = '<guid>' OR cr40f_motorista2 = '<guid>')`.

## Multas (`cr40f_multas`)
```sql
SELECT TOP 10 cr40f_id_multa, cr40f_statusname, cr40f_dataehorario, cr40f_placaname, cr40f_motoristaname,
       cr40f_localdainfracao, cr40f_cidade, cr40f_dataparaindicar, cr40f_indicado
FROM cr40f_multas
WHERE cr40f_status <> 202410005 AND cr40f_status <> 202410006
ORDER BY cr40f_dataparaindicar
```
Destaque as que têm `cr40f_dataparaindicar` próxima e `cr40f_indicado` falso. Por motorista ou placa: filtre por `cr40f_motorista` / `cr40f_placa`.

## Disponibilidade de um veículo num dia
Combine: manutenções abertas na data + trocas na janela + serviços do dia com `cr40f_veiculo = '<guid>'` (consulta "Agenda" em `consultas.md`).
