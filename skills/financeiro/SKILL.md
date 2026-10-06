---
name: financeiro
description: Use quando o usuário perguntar o que tem a receber ou a pagar, quem está devendo, pedir para cobrar um cliente por link ou boleto, juntar contas numa fatura, classificar contas no plano de contas ou mudar vencimento e valor em lote, gerar remessa ou processar retorno do banco, conciliar extrato ou vendas da maquininha, ou ver caixa, fluxo de caixa, DRE e indicadores.
---

# Financeiro

Se o usuário falar de outra empresa, use `list_companies` e informe `company_id` nas chamadas.

## Regras que valem para tudo

- Situação de conta em `list_receivables` e `list_payables`: `PENDENTE` (em aberto), `QUITADO` (paga) ou `CANCELADO`. Vencida é PENDENTE com vencimento antes de hoje.
- Datas aceitam só a data (AAAA-MM-DD; no fim do período vale o dia inteiro) ou data e hora com fuso. Vale para `due_from` e `due_to` dessas duas e para `dre_report`.
- `count` e `total` somam só o que voltou: 50 contas por padrão, até 500 com `limit`. Se `count` for igual ao limite, a soma está incompleta: avise ou use `financial_dashboard`. `amount` é o valor cheio da conta, sem descontar pagamento parcial. `overdue` e `total_overdue` contam só PENDENTE com vencimento passado.
- `query` aceita nome, CPF/CNPJ ou número do lançamento. Centro de custo, plano de contas e data de quitação vão em `filters`, com os campos de `list_filter_fields`. Intervalo em `filters`: operador BETWEEN com o valor `inicio|fim`.
- Não existe ferramenta para incluir, dar baixa manual ou cancelar conta a receber ou a pagar (fora as que nascem do boleto avulso, da fatura e do acerto do cartão), nem para aceitar fechamento de caixa. Não é falta de permissão: oriente fazer na tela do Financeiro. Alterar plano de contas, centro de custo, vencimento ou valor é com `update_receivables_batch` e `update_payables_batch`.
- O id de caixa ou conta bancária (`cash_box_id`, `cash_id`, `cash_register`) é o campo `caixa` da lista `caixas` de `cash_flow_dashboard` chamado sem `cash_register`. Caixa sem nenhum movimento não aparece ali.
- Arquivo do usuário (retorno, extrato) vai em base64 do jeito que veio: CNAB é posicional, e espaço ou quebra de linha a menos estraga a leitura. Se não conseguir enviar o arquivo inteiro assim, oriente fazer pela tela.

## Confirme antes de chamar

Mostre cliente, contas (número e valor), total, datas e canal, e só chame depois do sim:

- `send_receivable_payment_link`: a mensagem chega ao cliente.
- `create_receivable_invoice`: as contas juntadas ficam canceladas e só voltam se a fatura for cancelada na tela.
- `update_receivables_batch` e `update_payables_batch`: mudam várias contas de uma vez.
- `generate_receivable_boleto`: registra o boleto no banco.
- `generate_bank_slip_remittance`: os boletos saem da fila de envio; gerar de novo não traz os mesmos.
- `process_bank_slip_return`: dá baixa nas contas pagas.
- `cancel_bank_slip` e `delete_bank_slip`.
- `reconcile_bank_statement` com `account_id`: baixa o título, e `unreconcile_bank_statement` não desfaz a baixa.
- `settle_card_sales`: mexe no caixa do operador; `undo_card_settlement` só desfaz enquanto nenhuma conta do acerto foi baixada.

## Quanto tenho a receber e a pagar

1. Visão geral: `financial_dashboard` traz o que vence hoje, o que já venceu (com as pessoas de maior valor), os próximos 30 dias, os maiores atrasos, a previsão de 12 meses e o que já foi recebido ou pago hoje. Os valores já descontam pagamentos parciais.
2. Total rápido: `financial_summary`. Soma o valor cheio das contas pendentes, e `expected_balance` é receber menos pagar, não saldo em banco.
3. Contas de um cliente, fornecedor ou período: `list_receivables` ou `list_payables` com `status` PENDENTE, `query` e o período.
4. Quem está devendo: `list_receivables` com `status` PENDENTE, `due_to` com a data de ontem e `limit` 500. Some por pessoa e mostre do maior para o menor, com os dias de atraso.

## Cobrar pelo link de pagamento

1. `list_receivables` com `status` PENDENTE e o cliente em `query`.
2. Escolha o canal. E-mail é o padrão: `email` vazio usa o e-mail do cadastro de cada cliente (confira em `search_people` se ele tem). WhatsApp: `channel` whatsapp com `whatsapp_integration_id`, a conta de `list_integrations` com `channel` whatsapp; sem conta conectada, o envio por WhatsApp é feito pela tela, em Financeiro > Contas a Receber, botão Cobrança.
3. Confirme as contas, o total e o destino.
4. `send_receivable_payment_link` com `ids` e o canal escolhido.
5. `sent` traz as contas cobradas e o total; `not_sent` lista as que ficaram de fora, com o motivo (não é conta a receber pendente ou não tem link de pagamento). Cliente sem e-mail não recebe e o envio aparece como falha em `billing_dashboard` do dia.

- O cliente só consegue pagar se a empresa tiver integração de pagamento nas formas de pagamento. Sem ela, o link abre sem opção de pagamento, e o envio não avisa.

## Fatura juntando contas

1. `list_receivables` com `status` PENDENTE e o cliente. Só entram contas a receber do mesmo cliente, sem nenhum pagamento e que não sejam outra fatura.
2. Confirme os números dos lançamentos, o total e o novo vencimento.
3. `create_receivable_invoice` com `ids`, `due_date` e, se quiser, `description` (até 60 caracteres; vazio vira FATURA).
4. Informe número, valor e vencimento da fatura. Ela é uma conta pendente nova: cobre pelo link ou por boleto com o `id` dela.

## Plano de contas e alteração em lote

1. `list_chart_of_accounts` traz código, nome, tipo (c receita, d despesa) e se a conta recebe lançamento (analítica).
2. Conta sem plano de contas fica fora do DRE. `suggest_receivables_chart_of_account` e `suggest_payables_chart_of_account` sugerem plano de contas e centro de custo de até 40 contas por vez (sem `ids`, pegam as que estão sem plano), sem gravar. Usam IA e consomem créditos quando a empresa não tem integração de IA própria.
3. Mostre as sugestões e, com o sim, aplique com `update_receivables_batch` ou `update_payables_batch` (`ids`, `plano_conta_id`, `centro_custo_id`), juntando as contas que vão para a mesma classificação.
4. As mesmas ferramentas mudam `vencimento`, `valor` (só nas contas em aberto), `tipo_moeda` e `baixa_automatica` de várias contas. Só os campos informados mudam.
5. Boleto ou conta de consumo em imagem ou PDF: `read_bill_document` devolve a conta a pagar preenchida (fornecedor, valor, vencimento, linha digitável e plano de contas), sem gravar. O lançamento é feito na tela.

## Boleto

Com carteira de cobrança ativa (`list_bank_slip_wallets` com `active` true), o boleto sai na conta da empresa e depende de remessa e retorno; se a empresa tiver os dois caminhos, pergunte qual usar. Sem carteira, `generate_receivable_boleto` registra o boleto da conta pela integração bancária da forma de pagamento de boleto e devolve `boleto_url`; repetir a chamada devolve o mesmo boleto, e a baixa chega sozinha.

Pela carteira:

1. Emitir: `generate_bank_slips` com `account_ids` (de `list_receivables`) e `wallet_id`. Sai um boleto por conta; `errors` lista as recusadas (já com boleto em aberto, quitada, cancelada, sem cliente ou sem vencimento).
2. Sem conta lançada: `create_standalone_bank_slip` com `wallet_id`, `person_id` (de `search_people`), `amount` e `due_date` AAAA-MM-DD a partir de hoje. A conta a receber é criada junto.
3. O boleto nasce EMITIDO e só vale no banco depois da remessa: rode `generate_bank_slip_remittance` no mesmo dia. Sai um arquivo por carteira; entregue cada um ao usuário com o nome `file_name` (conteúdo em `file_base64`) para ele enviar no internet banking. Se não puder entregar o arquivo nesta conversa, não gere: oriente gerar em Financeiro > Boletos > Boletos emitidos, botão Gerar remessa.
4. Para o cliente pagar, passe a linha digitável (`digitable_line`), o valor e o vencimento de `list_bank_slips` ou `get_bank_slip`. Nenhuma ferramenta envia o PDF: `print_bank_slip` devolve só os dados do layout, e o PDF sai na tela Boletos emitidos, botão Imprimir.
5. Retorno: peça o arquivo do banco e chame `process_bank_slip_return` com `file_content`. Informe quantos foram pagos e o valor, os rejeitados com o motivo e os `not_found` (boletos de fora do sistema). Repetir o mesmo arquivo não duplica a baixa.
6. Cancelar: `cancel_bank_slip` com `reason`. EMITIDO cancela na hora; REMESSA_GERADA ou REGISTRADO fica com baixa pendente e precisa de nova remessa. Boleto pago não se cancela. Cancelar o boleto não cancela a conta a receber, que continua pendente. `delete_bank_slip` só para EMITIDO, CANCELADO ou REJEITADO.
7. Cliente pagou por fora: não cancele o boleto. Com a baixa da conta feita na tela, a próxima remessa pede a baixa ao banco.
8. Rejeitado: o motivo está no histórico de `get_bank_slip` (CEP, endereço ou CPF/CNPJ do cliente, carteira). Corrija, cancele e emita outro.

Situações: EMITIDO, REMESSA_GERADA, REGISTRADO, PAGO, BAIXADO, CANCELADO e REJEITADO. Panorama: `report_bank_slips` com `report` dashboard_overview ou dashboard_receipts; vem em HTML, então resuma os números.

Carteira nova: `list_bank_slip_banks` (código e layouts aceitos), os dados do contrato com o usuário (agência, conta, dígitos, carteira e, quando o banco usa, convênio, modalidade e código do beneficiário), `cash_id` do caixa que recebe os pagamentos e `create_bank_slip_wallet`. O sistema testa a carteira ao salvar, e o erro aponta o campo. Carteira com boletos não sai com `delete_bank_slip_wallet`: desative com `update_bank_slip_wallet` e `active` false.

## Conciliação bancária

1. Ache o `cash_box_id` da conta bancária (veja as regras do início).
2. `import_bank_statement` com o arquivo (OFX, CNAB 240 ou 400, CSV) em `file_content` e `validate_only` true; sem erros, repita sem ele. Lançamento já importado não entra de novo. O retorno da cobrança de boletos vai em `process_bank_slip_return`, não aqui.
3. `list_bank_statement_entries` com `status` PENDENTE mostra o que falta. `suggest_bank_reconciliation` no período propõe o movimento de caixa ou o título de cada lançamento, com `confidence`.
4. Mostre as sugestões ao usuário. CONFIANCA_MEDIA e CONFIANCA_BAIXA não conferem o documento do lançamento: confirme essas uma a uma.
5. `reconcile_bank_statement` com `items`: `entry_id` e `cash_movement_id` só vinculam; `entry_id`, `account_id` e `payment_method_id` (de `list_payment_methods`) dão baixa no título pelo valor do extrato (crédito em conta a receber, débito em conta a pagar).
6. O que não tem contrapartida e o usuário não vai lançar: `ignore_bank_statement_entries` com `reason`. `unreconcile_bank_statement` volta conciliado ou ignorado para pendente, sem desfazer baixa.
7. Feche com `bank_reconciliation_summary`: pendentes, conciliados, ignorados e saldo do período.

## Maquininha e acerto do cartão

1. As vendas chegam pelo aviso da adquirente (`list_card_sale_webhooks` dá o endereço a cadastrar nela) ou por arquivo em `import_card_sales`: planilha exige `acquirer_cnpj` e `acquirer_name`, e .txt ou .csv podem ir em texto puro. Venda repetida não duplica.
2. `get_card_reconciliation` com `date`: `only_acquirer` são vendas sem recebimento em cartão no caixa (entram no acerto), `only_cash_register` são recebimentos em cartão sem venda na adquirente, `settlements` são os acertos do dia.
3. Acerto: confirme dia, vendas, total e operador. `settle_card_sales` com `date`, `sale_ids` de `only_acquirer` (vazio acerta todas do dia) e `operator_user_id` do operador cujo caixa recebeu as vendas como dinheiro (`usuarioCaixaId` de `list_cash_closings`; vazio usa o usuário da conexão). As vendas passam a cartão e viram contas a receber da adquirente.
4. Acerto errado: `undo_card_settlement` com o `id` de `settlements`.
5. Uma venda e a previsão de cada parcela: `list_card_sales` e `get_card_sale`.

## Caixa, fluxo de caixa e DRE

- Saldo, entradas e saídas por caixa e banco: `cash_flow_dashboard` com `date_from` e `date_to` (sem datas, mês corrente). Transferência entre caixas aparece em cada caixa, mas não no total. Sem a permissão de ver o movimento de todos os usuários, os números são só do próprio usuário: avise.
- `list_cash_movements` traz os lançamentos mais recentes e busca por descrição ou pessoa, sem período: não use para somar.
- Fechamentos: `list_cash_closings` com a situação (PENDENTE, ACEITO, REJEITADO) e a diferença (negativa é falta, positiva é sobra). Falta de verdade é a do dinheiro; cartão e Pix são recebíveis.
- Lucro ou prejuízo: `dre_report` com o período, `regime` COMPETENCIA (padrão, resultado da operação) ou CAIXA (quando o dinheiro entrou ou saiu), `cost_center_id` de `list_cost_centers` e, para consolidar o grupo de empresas, `org_ids` de `get_company_group`. Mostre receita bruta, deduções, receita líquida, custo, lucro bruto, despesas e resultado, e sempre as `warnings`: o que está sem plano de contas fica fora dos totais (classifique como em Plano de contas), e produto sem custo aumenta o lucro bruto.
- Apoio: `list_receipts` (recibos), `list_credit_letters` (cartas de crédito com saldo e validade) e `list_card_transactions` (transações de cartão e Pix).

## Cobrança automática

- `billing_dashboard` com `from` e `to`: envios, falhas, lidas e clientes que alegaram pagamento.
- `simulate_billing_plan` mostra o que um plano cobraria num dia, sem enviar. O `plan_id` é a `key` de `planos` em `billing_dashboard` (vazia é envio manual); `next` true acha o próximo dia com cobrança.

## Como responder

- Valores como R$ 1.234,56 e datas como 05/10/2026. Identifique a conta pelo número do lançamento e pela pessoa, nunca pelo id.
- Comece pelo total e pelo que pede ação (vencido, falha, rejeitado, diferença de caixa), depois os maiores itens. Separe receber de pagar.
- Não mostre JSON, HTML nem base64. Diga quando a lista veio incompleta.
