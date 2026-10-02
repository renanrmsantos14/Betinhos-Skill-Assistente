# Programação e disparo

Metadata conferida em DEV em 02/10/2026.

São duas ações separadas:
- **Programar** = gravar motorista e veículo na reserva. **Não muda o status.**
- **Disparar** = enviar o serviço ao motorista (status Programado). Só quando o usuário pedir "disparar".

## Programar

### 1. Candidatos
Liste os motoristas ativos (consulta "motoristas") e, para o dia do serviço, a agenda de cada um (consulta "agenda dos motoristas"). O veículo sugerido é o `cr40f_veiculoatual` do motorista.

### 2. Alertas (avisam, não bloqueiam — o usuário decide)
| Alerta | Como checar |
|---|---|
| Conflito de horário | outro serviço ativo do motorista com saída a menos de 30 min, ou serviço anterior com `cr40f_horrioprevistoderetorno` depois da nova saída |
| CNH vencida | `cr40f_validadedacnh` anterior à data do serviço |
| Descanso curto | última jornada (consulta "jornada"): menos de 11 h entre `cr40f_fimjornada` e a saída. A jornada é importada do ponto e pode estar defasada — informe a data do último registro |
| Sem veículo | motorista sem `cr40f_veiculoatual` → pergunte qual veículo usar |
| Veículo em manutenção | consulta "manutenções do veículo" com data igual à do serviço |
| Troca de carro na janela | consulta "trocas de carro" cobrindo o horário do serviço |
| Blindado | reserva Blindado/Van Blindado e veículo com `cr40f_blindado` falso |
| Terceiro | `cr40f_tipodevinculo` = Terceiro → lembre que há repasse na composição |

O veículo não tem campo de tipo (Executivo, Van…): mostre marca/modelo e deixe o usuário julgar.

Apresente até 5 candidatos, sem alertas primeiro:
```
*OS-xxxx* — 03/10 07:30 — SJCampos / Aerop. Guarulhos — Executivo
1. Carlos — Corolla ABC1D23 — livre (serviço anterior termina 05:40)
2. Danilo — Jetta XYZ4E56 — ⚠ conflito: OS-9701 às 07:15
```

### 3. Gravar
Mostre antes/depois e peça confirmação. `update_record` em `cr40f_reservadeveculos`:
```json
{"cr40f_motorista": {"relatedTable": "cr40f_funcionarios", "recordId": "<guid>"},
 "cr40f_veiculo":   {"relatedTable": "cr40f_veiculos", "recordId": "<guid>"}}
```
Não envie `cr40f_status`, `new_foiprogramado` nem `new_veiculoreal` (este é preenchido pelo rastreador). Releia a reserva e confirme motorista e veículo.

## Disparar
Equivale ao botão **Disparar** da grade (`new_programarRegistros.js`): grava `cr40f_status` = 202410005 (Programado) e `new_foiprogramado` = true. O envio ao motorista é feito pelo servidor a partir dessa gravação.

### Validações (antes de gravar)
Para cada serviço pedido, releia a reserva e exija:
1. `cr40f_motorista` e `cr40f_veiculo` preenchidos;
2. `cr40f_status` atual = Solicitado (202410004) ou Confirmado (202410001);
3. `cr40f_dataehorriodesada` no futuro;
4. ao menos um passageiro vinculado (consulta "vínculos da reserva" em `consultas.md`);
5. `new_categoriadoitem` = Serviço (100000000).

Quem falhar **fica fora do disparo**, com o motivo. Já Programado → informe "já disparado" e não regrave.

### Confirmação e gravação
```
*Disparar 3 serviços* — confirma?
• OS-9701 — 03/10 07:15 — Danilo — Jetta XYZ4E56
• OS-9702 — 03/10 09:00 — Carlos — Corolla ABC1D23
Fora do disparo: OS-9705 (sem motorista)
```
Após o "sim", `update_record` por serviço (ou `records` em lote, máx. 25):
```json
{"cr40f_status": 202410005, "new_foiprogramado": true}
```
Confira o resultado por registro, releia os status e responda: "Disparado: OS-xxxx, OS-yyyy". Falha em algum → informe qual e o erro; não repita às cegas.

"Dispara os de amanhã" → liste os serviços do dia (consulta "Agenda" em `consultas.md`), aplique as validações e peça confirmação da lista inteira.

## Consultas

Motoristas ativos:
```sql
SELECT TOP 20 cr40f_funcionariosid, new_apelido, cr40f_nomecompleto, cr40f_veiculoatual, cr40f_veiculoatualname,
       cr40f_validadedacnh, cr40f_tipodevinculoname
FROM cr40f_funcionarios
WHERE cr40f_funcao = 202410000 AND cr40f_status = 0 AND statecode = 0
ORDER BY new_apelido
```
Mais de 20 motoristas: repita com `AND new_apelido > '<último apelido>'`. Motorista pelo nome: `AND (new_apelido LIKE '%x%' OR cr40f_nomecompleto LIKE '%x%')`.

Agenda dos motoristas no dia:
```sql
SELECT TOP 20 cr40f_id, cr40f_dataehorriodesada, cr40f_horrioprevistoderetorno, cr40f_trajeto,
       cr40f_motorista, cr40f_motoristaname, cr40f_veiculoname, cr40f_statusname
FROM cr40f_reservadeveculos
WHERE cr40f_dataehorriodesada >= '2026-10-03T00:00:00' AND cr40f_dataehorriodesada < '2026-10-04T00:00:00'
  AND cr40f_motorista IS NOT NULL AND new_categoriadoitem = 100000000
  AND cr40f_status <> 202410002 AND cr40f_status <> 202410010
ORDER BY cr40f_dataehorriodesada
```
Dia cheio: divida em faixas de horário, como em `consultas.md`.

Jornada (último registro do motorista):
```sql
SELECT TOP 1 cr40f_iniciojornada, cr40f_fimjornada, cr40f_horastrabalhadasminutos, cr40f_descansoanteriorminutos
FROM cr40f_jornadacolaborador
WHERE cr40f_funcionario = '<guid-motorista>'
ORDER BY cr40f_fimjornada DESC
```

Veículo:
```sql
SELECT cr40f_veiculosid, cr40f_placa, cr40f_marca, cr40f_modelo, cr40f_cor, cr40f_blindado,
       cr40f_statusdoveiculoname, new_categoriadoveiculoname, cr40f_motoristaatualname
FROM cr40f_veiculos
WHERE cr40f_veiculosid = '<guid-veiculo>'
```

Manutenções do veículo, trocas de carro: ver `frota.md`.
