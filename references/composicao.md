# Composição de preço (`cr40f_composicaodeprecos`)

Metadata conferida em DEV em 02/10/2026.

## Regras
- **Nunca crie nem exclua** composição: ela nasce junto com o serviço. Só `update_record`.
- O GUID da composição é o **mesmo GUID da reserva** (`cr40f_composicaodeprecosid` = `cr40f_reservadeveculosid`). Confirme pela consulta "composição do serviço"; se não existir, avise o usuário e pare.
- **Nunca grave** `new_subtotal`, `new_valortotal`, `new_status`, `cr40f_id`: o plugin `PriceComposition.Recalculate` recalcula os totais a cada item alterado; quem conclui é o financeiro.
- Composição com `new_status` = Concluido (100000001) → não altere sem pedido explícito e confirmação própria.
- Não altere `cr40f_cotao`, `cr40f_valor_a_receber` nem `cr40f_statusdefaturamento` na reserva.
- Se o `update_record` for recusado pelo servidor (plugin `PriceComposition.Protect`), mostre a mensagem de erro e não insista.

## Itens (todos MONEY, em R$)
| Campo | Item |
|---|---|
| `new_valorlocacao` | Locação — **valor base do trajeto** (é aqui que entra a tarifa) |
| `cr40f_valordadiaria` | Diária |
| `cr40f_valorhoramotorista` | Hora do motorista / hora extra |
| `cr40f_valortempodeespera` | Tempo de espera / hora parada |
| `cr40f_valorpassageirosadicionais` | Passageiros adicionais |
| `new_valorpedagios` | Pedágios |
| `new_valorestacionamento` | Estacionamento |
| `cr40f_valorcombustivel` | Combustível |
| `new_valorpassagens` | Passagens |
| `cr40f_valoradequacaoderota` | Adequação de rota |
| `cr40f_valoradequacaoderotaporkmrodado` | Adequação de rota por km rodado |
| `cr40f_valordesviosentrecidades` | Desvios entre cidades |
| `cr40f_valorrodoanel` | Rodoanel |
| `cr40f_valorarcometropolitano` | Arco Metropolitano |
| `cr40f_valorayrtonsenna` | Ayrton Senna |
| `new_ajuste` | Ajuste (desconto negativo ou acréscimo) |
| `cr40f_valorrepasseterceiro` | Repasse a terceiro — custo, **não entra no total**; só quando o motorista é terceiro |

## Fluxo
1. Localize o serviço (OS) e leia a composição atual.
2. Busque valores, nesta ordem, e **guarde a fonte de cada um**:
   1. **Histórico comparável** — composições com total > 0 do mesmo cliente, mesmo `cr40f_tipodeveiculo` e trajeto equivalente (consulta abaixo). Prefira as mais recentes e as Concluídas.
   2. **Tabela de tarifas** — `references/tarifas.md`, só para o cliente correspondente.
   3. Nada encontrado → deixe o item em branco e peça o valor.
3. Mostre o resumo e peça confirmação:
```
*Composição OS-xxxx* — confirma?
• Locação: R$ 620,49 — histórico OS-9120 (12/09/2026)
• Pedágios: R$ 38,00 — histórico OS-9120
• Hora parada: R$ 48,88 — tarifa de tabela (planilha 2023)
• Total estimado: R$ 707,37 (o sistema recalcula)
```
4. `update_record` em `cr40f_composicaodeprecos` só com os itens confirmados (número, sem "R$").
5. Releia a composição e informe o `new_valortotal` calculado pelo sistema. Se divergir da soma, avise.

Valor de histórico é **preço histórico**; de `tarifas.md` é **tarifa de tabela**; sem fonte é **estimativa**. Nunca apresente um como o outro.

## Mono ou bilíngue (Johnson e Kenvue)
A tarifa muda se o passageiro é visitante estrangeiro. Decida nesta ordem:
1. **Histórico do passageiro**: valor já cobrado dele no mesmo trajeto.
2. **Cadastro**: `cr40f_preferenciasdopassageiro`, `cr40f_classificacao` e DDI do telefone (diferente de +55 indica visitante).
3. **Nome**: apenas indício.

Sinais concordam → assuma e mostre a premissa no resumo ("Tarifa bilíngue: telefone +1"). Sinais conflitam ou só existe o nome → **pergunte** antes de compor.

## Consultas

Composição do serviço:
```sql
SELECT cr40f_composicaodeprecosid, cr40f_id, new_status, new_subtotal, new_valortotal, new_valorlocacao,
       cr40f_valordadiaria, cr40f_valorhoramotorista, cr40f_valortempodeespera, cr40f_valorpassageirosadicionais,
       new_valorpedagios, new_valorestacionamento, cr40f_valorcombustivel, new_ajuste, cr40f_valorrepasseterceiro
FROM cr40f_composicaodeprecos
WHERE cr40f_servicorelacionadogeral = '<guid-reserva>'
```
Colunas com valor nulo não aparecem no retorno. Para os demais itens (rodoanel, desvios etc.), consulte só quando forem citados.

Histórico comparável:
```sql
SELECT TOP 10 r.cr40f_id, r.cr40f_dataehorriodesada, r.cr40f_trajeto, r.cr40f_tipodeveiculoname,
       c.new_statusname, c.new_valortotal, c.new_valorlocacao, c.new_valorpedagios, c.new_valorestacionamento,
       c.cr40f_valortempodeespera, c.cr40f_valorhoramotorista, c.cr40f_valorpassageirosadicionais
FROM cr40f_composicaodeprecos c
JOIN cr40f_reservadeveculos r ON c.cr40f_servicorelacionadogeral = r.cr40f_reservadeveculosid
WHERE r.cr40f_cliente = '<guid-cliente>' AND r.cr40f_tipodeveiculo = <codigo>
  AND r.cr40f_trajeto LIKE '%<origem>%' AND r.cr40f_trajeto LIKE '%<destino>%'
  AND c.new_valortotal > 0
ORDER BY r.cr40f_dataehorriodesada DESC
```
Sem resultado: troque o filtro de trajeto por `r.cr40f_tipodoservico = <codigo>`; ainda sem resultado, retire o filtro de cliente e avise que o comparável é de outro cliente.
