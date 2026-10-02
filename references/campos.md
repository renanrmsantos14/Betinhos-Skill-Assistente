# Campos — Reserva de Veículos (`cr40f_reservadeveculos`)

Metadata conferida em DEV e PROD em 01/10/2026.

## Proibidos na escrita
| Campo | Motivo |
|---|---|
| `cr40f_id` | Numeração automática `OS-{SEQNUM}`; nome principal e chave alternativa. **Fica fora do payload mesmo que a ferramenta peça o nome principal** — qualquer valor fora de `OS-<n>` é removido pelo plugin `Betinhos.AutoNumberGuard` antes de gravar |
| `cr40f_idnovo` | Numeração automática `{SEQNUM:5}` |
| `cr40f_statusdefaturamento`, `cr40f_cotao`, `cr40f_valor_a_receber`, `cr40f_financeiro` | Financeiro, fora do agendamento |
| `cr40f_motorista`, `cr40f_veiculo` | Fora do agendamento. Só com pedido explícito, pelo fluxo de `programacao.md` |
| `new_veiculoreal` | Preenchido pelo rastreador. Nunca grave |
| `new_foiprogramado` | Só no fluxo "Disparar" de `programacao.md` |
| `cr40f_passageiro1`…`cr40f_passageiro4` | Legado. Passageiro entra **apenas** por `cr40f_servicosporpassageiro` |
| `cr40f_passageirosetelefonedecontato` | Campo VIEW, recalculado automaticamente a partir de `cr40f_servicosporpassageiro`. Somente leitura |

## Payload de criação
| Campo | Tipo | Conteúdo | Exemplo |
|---|---|---|---|
| `cr40f_cliente` | lookup `cr40f_clientes1` | GUID do cliente | `{"relatedTable":"cr40f_clientes1","recordId":"<guid>"}` |
| `cr40f_solicitante` | lookup `cr40f_bancodedados` | GUID de quem pediu | idem, `relatedTable: cr40f_bancodedados` |
| `cr40f_dataehorriodesada` | data/hora | **UTC**: hora de Brasília + 3h, sufixo `Z` | 07:30 em Brasília → `2026-10-02T10:30:00Z` |
| `cr40f_horrioprevistoderetorno` | data/hora | opcional, mesma regra | |
| `cr40f_endereodesada` | texto | uma linha por passageiro: `1. Nome - endereço`. Saída no Aeroporto de Guarulhos → dados do voo na mesma linha (ver "Observações e dados de voo") | `1. Gabriela - Hotel Radisson Vila Olímpia` · `1. Alvadi - Aeroporto de Guarulhos - Voo LA 8127` |
| `cr40f_destino` | texto | destino; vários → `1 - …\n2 - …` | `Aerop. Guarulhos` |
| `cr40f_trajeto` | texto curto | `Origem / Destino` | `SJCampos / Aerop. Guarulhos` |
| `cr40f_tipodeveiculo` | choice | ver tabela | `202410001` |
| `cr40f_tipodoservico` | choice | ver tabela | `202410009` |
| `cr40f_status` | choice | **`202410004` (Solicitado)**; outro status só com pedido explícito | |
| `new_categoriadoitem` | choice | **sempre `100000000` (Serviço)** | |
| `cr40f_obsdeoperao` | texto | **visível aos motoristas**: só o que ajuda a executar o serviço. Sem preferências do passageiro e sem dados do voo de saída em GRU | `Alinhar endereço de destino com o pax` |
| `cr40f_observaointerna` | texto | **só a gestão vê**: `Criado via Claude (skill assistente-betinhos)` + preferências do passageiro + pendências da gestão | `Criado via Claude (skill assistente-betinhos). Pax prefere motorista Amadeu.` |
| `cr40f_iachaveidempotencia` | texto (180) | chave única do pedido | `claude:20261002:0730:gabriela:kq7m` |
| `cr40f_formadepagamento` | choice | só se informado | |
| `cr40f_cr` | texto | centro de custo, só se informado | |

## Payload do vínculo de passageiro (`cr40f_servicosporpassageiro`)

Única forma de inserir passageiro. Um registro por passageiro, criado depois da reserva. Metadata conferida em DEV e PROD em 02/10/2026.

| Campo | Tipo | Conteúdo | Exemplo |
|---|---|---|---|
| `cr40f_geral` | lookup `cr40f_reservadeveculos` | GUID da reserva criada | `{"relatedTable":"cr40f_reservadeveculos","recordId":"<guid>"}` |
| `cr40f_bancodedados` | lookup `cr40f_bancodedados` | GUID do passageiro no cadastro (obrigatório) | `{"relatedTable":"cr40f_bancodedados","recordId":"<guid>"}` |
| `cr40f_ordemdeselecao` | inteiro | ordem de embarque: 1, 2, 3… | `1` |
| `new_enderecodesaidacolunaservicosporpassageiro` | texto | endereço de saída desse passageiro (com os dados do voo se a saída for o Aeroporto de Guarulhos) | `Hotel Radisson Vila Olímpia` · `Aeroporto de Guarulhos - Voo LA 8127` |

**Não envie `cr40f_id`** do vínculo: é numeração automática (ex.: `37811`).

## Observações e dados de voo

**Quem vê cada campo**
- `cr40f_obsdeoperao` → motoristas e operação.
- `cr40f_observaointerna` → só a gestão.

**Onde vai cada informação**
- Preferências do passageiro (motorista preferido, `cr40f_preferenciasdopassageiro` do cadastro ou dito no pedido) → **só** observação interna. Nunca na obs. de operação.
- Pendência que o motorista pode resolver na hora (ex.: alinhar destino com o pax) → obs. de operação **e** observação interna (`Pendente: …`).
- Pendência só da gestão → observação interna.

**Dados de voo**
- **Saída no Aeroporto de Guarulhos**: os dados do voo vão no endereço de saída, ao lado do aeroporto (`cr40f_endereodesada` e `new_enderecodesaidacolunaservicosporpassageiro`), **não** na obs. de operação. Ex.: `1. Alvadi - Aeroporto de Guarulhos - Voo LA 8127`. Sem dados → `Aeroporto de Guarulhos - Dados do voo pendentes` e peça os dados no passo 1.
- **Aeroporto de Guarulhos como destino**: não registre nem peça dados do voo (horário é agendado).

## Choices

**`cr40f_tipodeveiculo`** — Básico 202410000 · Executivo 202410001 · Blindado 202410002 · Van 202410003 · Van Blindado 202410004 · Spin 202410005 · Somente Motorista 202410006 · Micro-ônibus 100000001 · Minivan 100000002

**`cr40f_tipodoservico`** (região/tarifa) — Guarulhos 202410000 · São Paulo 202410001 · Outras Cidades 202410002 · Vale do Paraíba 202410003 · Rio de Janeiro 202410004 · Pindamonhangaba 202410005 · São José dos Campos 202410006 · Congonhas 202410007 · Campinas 202410008 · Dentro de São Paulo 202410009 · Litoral 202410010 · Região dos Lagos 202410011 · Extrema 202410012 · Nova Odessa 202410013 · Baixada Santista 202410014 · Minas Gerais 202410015 · GPX 202410016

Dica: traslado de/para aeroporto de Guarulhos costuma usar "Guarulhos"; deslocamento dentro da capital, "Dentro de São Paulo". Na dúvida, siga o histórico do cliente e pergunte.

**`cr40f_status`** — Pré-reserva 202410000 · Solicitado 202410004 · Confirmado 202410001 · Programado 202410005 · Em Execução 202410006 · Concluído 202410008 · Requer Análise 100000001 · Cancelado 202410002 · Cancelado com Ressalvas 202410010

**`cr40f_formadepagamento`** — Cartão de crédito 202410000 · Pedido de compra 202410001 · Pix 202410002

## Tabelas relacionadas
- `cr40f_clientes1`: `cr40f_clientes1id`, `cr40f_nomedocliente`, `cr40f_endereco`
- `cr40f_bancodedados` (passageiros/solicitantes): `cr40f_bancodedadosid`, `cr40f_nomedopassageiro`, `cr40f_cliente`, `cr40f_telefone`, `cr40f_email`, `cr40f_enderecodesaida`, `cr40f_preferenciasdopassageiro`, `new_tipodoveiculo`, `cr40f_classificacao`, `cr40f_status` (Ativo 202410000)
