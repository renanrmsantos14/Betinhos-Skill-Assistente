# Financeiro — somente leitura

Metadata conferida em DEV em 02/10/2026. **Nunca grave** em `cr40f_financeiro`, `cr40f_pagantes`, `cr40f_recibos`, `cr40f_pagamentoaterceiro` nem em `cr40f_statusdefaturamento`/valores da reserva. Pedido de baixa, estorno, link ou recibo → explique que é feito no módulo financeiro e ofereça a consulta.

Ao responder, identifique sempre registro, valor e status (ex.: "PAG-295 — R$ 1.000,00 — Pendente"). Não some nem arredonde valores de registros com status diferentes sem dizer.

## Cadeia
Reserva (`cr40f_financeiro`) → Operação financeira `cr40f_financeiro` (OP-n) → Pagantes `cr40f_pagantes` (PAG-n, um por pagador).

## Faturamento de um serviço
```sql
SELECT cr40f_id, cr40f_statusdefaturamentoname, cr40f_cotao, cr40f_valor_a_receber, cr40f_formadepagamentoname,
       cr40f_financeiro, cr40f_financeironame, new_observacaodefaturamento
FROM cr40f_reservadeveculos
WHERE cr40f_id = 'OS-1234'
```

## Operação financeira (`cr40f_financeiro`)
Status: Aberta 202410000 · Pagamento Pendente 100000001 · Pagamento Parcial 202410006 · Paga 202410003 · Cancelada 202410004.
```sql
SELECT cr40f_financeiroid, cr40f_idfinanceiro, cr40f_statusname, cr40f_valor, new_clientedaop,
       cr40f_datado1servico, cr40f_datadoultimoservico, cr40f_trajetos
FROM cr40f_financeiro
WHERE cr40f_idfinanceiro = 'OP-58'
```
Em aberto: `WHERE cr40f_status <> 202410003 AND cr40f_status <> 202410004 ORDER BY cr40f_datado1servico` (TOP 20).

Serviços de uma operação:
```sql
SELECT TOP 20 cr40f_id, cr40f_dataehorriodesada, cr40f_trajeto, cr40f_cotao, cr40f_statusdefaturamentoname
FROM cr40f_reservadeveculos
WHERE cr40f_financeiro = '<guid-financeiro>'
ORDER BY cr40f_dataehorriodesada
```

## Pagantes (`cr40f_pagantes`)
Status: Pendente 202410001 · Pago 202410002 · Negado 202410003 · Expirado 202410004 · Cancelado 202410005 · Não Finalizado 202410006 · Autorizado 202410007 · Aguardando Facial 202410010.
```sql
SELECT TOP 20 cr40f_id, cr40f_statusname, cr40f_valor, cr40f_formadepagamentoname, cr40f_bancodedadosname,
       cr40f_linkdepagamento, cr40f_statusgeracaolinkname, cr40f_statusdoreciboname, cr40f_linkdorecibo,
       cr40f_datadoprimeiropagamento, cr40f_datadoultimolembrete
FROM cr40f_pagantes
WHERE cr40f_financeiro = '<guid-financeiro>'
```
Pendentes em geral: `WHERE cr40f_status = 202410001 ORDER BY createdon` (inclua `cr40f_financeironame`). Erros de link, e-mail ou recibo: `cr40f_errogeracaolink`, `cr40f_erroenvioemail`, `new_errogeracaorecibo`.

Só mostre link de pagamento ou de recibo quando o usuário pedir.

## Faturamento pendente por cliente
```sql
SELECT TOP 20 cr40f_id, cr40f_dataehorriodesada, cr40f_trajeto, cr40f_cotao, cr40f_statusdefaturamentoname
FROM cr40f_reservadeveculos
WHERE cr40f_cliente = '<guid-cliente>' AND cr40f_status = 202410008
  AND (cr40f_statusdefaturamento = 202410005 OR cr40f_statusdefaturamento = 100000001)
ORDER BY cr40f_dataehorriodesada
```
Status de faturamento: Pendente 202410005 · Composição Realizada 100000001 · Faturamento Mensal 202410008 · Pagamento Pendente 202410007 · Pagamento Em Atraso 202410012 · Pago 202410010 · Não Faturável 202410011 · Cortesia 202410003 · Permuta 202410004 · Pagante em Viagem 202410006 · Cancelado Sem Taxa 202410000 · Cancelado Com Taxa 202410002.

## Pagamento a terceiros (`cr40f_pagamentoaterceiro`)
```sql
SELECT TOP 20 cr40f_name, cr40f_terceirofavorecidoname, cr40f_statusname, cr40f_statuspagamentoname,
       cr40f_quantidadeservicos, cr40f_totalrepasse, cr40f_totalcobradocliente, cr40f_margemtotal, cr40f_pagoem
FROM cr40f_pagamentoaterceiro
WHERE cr40f_statuslote = 100000000 AND cr40f_statuspagamento = 100000000
ORDER BY createdon DESC
```
Status: Rascunho 100000000 · Falha no envio 100000001 · Aguardando pagamento 100000002 · Pago 100000003.
