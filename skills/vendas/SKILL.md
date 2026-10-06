---
name: vendas
description: Use quando o usuário pedir para criar pedido ou orçamento, incluir produtos e serviços, registrar o pagamento e fechar a venda, cancelar pedido, converter orçamento em pedido, consultar vendas e faturamento, acompanhar etapa de pedido, OS ou contrato, registrar resposta de orçamento, mexer em tabela de preço, reajustar contratos, ver metas e devoluções, mandar documento para assinar ou cadastrar cliente.
---

# Vendas

Se o usuário falar de outra empresa, use `list_companies` e informe `company_id` nas chamadas.

## O que fica na tela

Alterar ou tirar item, estornar pagamento, clonar pedido, enviar a proposta ao cliente e criar ordem de serviço, contrato, devolução ou meta são feitos na tela do sistema. Não há ferramenta para isso, e não é área bloqueada: diga isso ao usuário e faça a parte que cabe aqui.

## Peça o sim antes

Mostre o resumo do que vai acontecer e espere a confirmação antes de `create_order`, `add_order_payment`, `cancel_order`, `convert_quote_to_order`, `send_quote_reminder`, `send_for_signature`, `resend_signature_request`, `cancel_signature_envelope`, `generate_readjustment_batch`, `approve_readjustment_batch`, `renumber_documents`, qualquer exclusão e importação que grava.

## Cliente

1. Procure com `search_people` (nome, CPF/CNPJ, código ou e-mail). Sem `type`, a busca esconde quem é só lead; sem `include_inactive`, esconde os inativos. Antes de dizer que não existe, repita com `type` `customer,lead` e `include_inactive` true.
2. Achou inativo: reative com `update_person` (`status` ACTIVE).
3. Não achou: confirme os dados e use `create_person` com `type` customer, vendor ou lead (sem ele, cadastra cliente). Para venda com NF-e, peça CPF/CNPJ e endereço completo com CEP. Se `create_person` recusar por CPF/CNPJ já cadastrado, inclusive de um lead, a mensagem traz o nome do cadastro existente: use esse.
4. `update_person` só muda o que vier preenchido, inclusive Inscrição Estadual por UF (`ie`), isenção de IE (`ident_free`) e endereço (`address`). No endereço, mande todos os campos: logradouro, número, bairro, cidade, UF e CEP.
5. Antes de vender a prazo, `get_person` mostra limite de crédito, saldo devedor, crédito disponível e vendedor do cliente.
6. Lista em arquivo: `import_people` grava direto, sem passada de conferência. Quem já existe pelo CPF/CNPJ ou pelo nome só tem os campos vazios completados. Aceita CSV separado por `;`, com o cabeçalho do modelo do sistema, ou JSON; arquivo em UTF-8 ou Latin-1 é lido certo.
7. Convênio é uma pessoa do tipo `partnership`: `list_partnership_members` (beneficiários, ou os convênios de uma pessoa com `person_id`), `list_partnership_services` (valor combinado e repasse), `list_partnership_professionals` e `list_portal_partnerships`.

## Criar um pedido

1. Use `customer_id` vindo de `search_people`. Pedido só com `customer_name` fica sem vínculo com o cadastro e, ao fechar, vira um cadastro novo (duplicado, se o cliente já existia e o CPF/CNPJ não foi informado).
2. Pergunte se o cliente vai revender a mercadoria: `resale` true faz a NF-e sair sem consumidor final e com a tributação de revenda. Com o pedido fechado, isso não muda mais.
3. Sem `price_table`, vale a tabela do cliente (`get_customer_price_table` com `channel` order mostra qual). Para outra, use avista, aprazo, atacado ou o id de `list_price_tables`.
4. `purchase_order` é o número que o cliente deu à compra no sistema dele (até 15 caracteres) e sai em cada item da NF-e. Não confunda com `list_purchase_orders`, que lista as compras da empresa com fornecedores.
5. Campos personalizados: nomes e formatos em `list_custom_fields` (`entity_type` pedido).
6. `create_order` cria o pedido pendente, ainda sem itens. Guarde o id.
7. Produtos: `add_product_to_order` com `order_id`, `product_id` (de `search_products`) e `quantity`. Sem `unit_price`, vale o preço da tabela do pedido; `discount` é em reais no item. Serviços: `add_service_to_order` com `service_id` (de `list_services`), `quantity` e `unit_price`, que é obrigatório. Cada chamada devolve os itens e o total. Só documento em aberto recebe item.
8. Confira com `get_order`: cliente, itens e total. A nota sai na ordem dos itens; para mudar, `reorder_order_items` com o `item_id` de cada item de `get_order` em `product_ids` e `service_ids`, todos os atuais, sem faltar nem repetir.
9. Pagamento: `add_order_payment` com `payments` (`payment_method_id` de `list_payment_methods`, `amount` e, se a forma parcela, `installments`). Os pagamentos têm de quitar o saldo do pedido; em dinheiro, o excedente vira troco. Com o total pago, a venda fecha: baixa o estoque e lança contas a receber e caixa. Não se desfaz por aqui: estorno é na tela.
10. Responda com o número e a situação. A nota do pedido fechado segue o fluxo de notas fiscais (`create_nfe`).
11. Venda que nasceu de oportunidade do funil: `convert_deal_to_order` com `tipo` ped (sem ele, vira orçamento) leva o cliente e os itens do escopo comercial. Sem itens no escopo, o pedido nasce vazio e os itens entram como no passo 7.

## Cancelar um pedido

1. Confira com `get_order` e explique o efeito: o cancelamento desfaz o que o pedido lançou (estoque e financeiro) e, se há nota fiscal, ela é cancelada na SEFAZ ou, fora do prazo, sai a nota de devolução.
2. Com o sim, `cancel_order` com `order_id` e `reason`. Vale para pedido pendente ou fechado e não se desfaz.

## Orçamento até o pedido

1. `create_quote` recebe os mesmos dados do pedido, menos `purchase_order` e `resale` (`entity_type` orcamento nos campos personalizados). Os itens entram com `add_product_to_order` e `add_service_to_order` com `document_type` quote. Guarde o id: não há ferramenta que liste orçamentos.
2. `get_order` abre orçamento, pedido ou OS pelo id, com itens, valores e pagamentos.
3. Orçamento antigo: peça o link que o cliente recebeu e abra com `get_public_document` (id, situação, etapa, validade e última resposta).
4. Cliente respondeu por telefone ou WhatsApp: `respond_public_document` com o `link` e a `response`. approved exige `name`; adjustment exige `message`; meeting, declined e more_time exigem `option`. A resposta entra no histórico, move o orçamento para a etapa dela e avisa a equipe. Registre só o que o cliente disse de fato.
5. `send_quote_reminder` (`quote_id`) manda agora o lembrete da validade, ou o aviso de vencido, pelo canal do último envio ou, sem envio, pelo e-mail ou WhatsApp do cliente. Só vale para orçamento em aberto com validade. Se vier `failures`, diga qual canal não saiu e por quê.
6. Aprovado: `convert_quote_to_order` com `quote_id` gera o pedido com cliente, itens e preços e devolve `order_number`. Cada chamada gera outro pedido: chame uma vez só. Orçamento com total zero é recusado.

## Etapas e acompanhamento

1. `list_order_stages` com `type` order (pedido e contrato), service_order ou quote devolve id e código de cada etapa.
2. `set_order_stage` com `order_id` e `stage_id` (ou `stage_code`). Sem etapa, a chamada tira a etapa do documento.
3. Etapa é separada da situação (PENDENTE, FECHADO, CANCELADO) e muda até em documento fechado. Produção e romaneio trocam a etapa sozinhos.
4. Em orçamento, trocar a etapa tira a resposta do cliente da listagem e reabre as opções no link; ela continua no histórico.
5. `get_order_tracking` (`type` service_order para OS) traz situação, etapa, histórico e o `public_link` que o cliente abre sem login. Entregue esse link quando pedirem para mandar o andamento ao cliente.
6. Ordens de serviço: `list_service_orders` (`status` PENDENTE, FECHADO ou CANCELADO). Entregas: `list_shipments` (romaneios com rota, situação e pedidos).

## Consultar vendas

- Pedido pelo número ou cliente: `list_orders` com `query`, depois `get_order` com o id.
- Faturamento somado por categoria, produto, cliente, vendedor, forma de pagamento, dia ou mês: `sales_summary` (`date_from` e `date_to` como 2026-08-01, `group_by`, `limit` até 100, `document_type` orcamento para orçamentos). Ele soma pela data de criação, inclui pendentes e mostra cancelados à parte. O painel de vendas do sistema conta o faturado pela data do fechamento: se os números não baterem, explique a diferença. Se vier `aviso` do teto de 3000 pedidos, divida o período.
- `list_orders` só traz pedidos, com `status` PENDENTE, FECHADO ou CANCELADO. `date_from` e `date_to` filtram pela data de criação e aceitam a data simples (2026-08-01; no fim do período vale o dia inteiro) ou data e hora com fuso. Data inválida é recusada. O `total` soma só os pedidos devolvidos (até 500 com `limit`) e deixa os cancelados de fora: para faturamento, use `sales_summary`.
- Por vendedor: `sales_by_seller` com `seller_ids` (vendedor é pessoa: `search_people` com `type` salesman), `start_date` e `end_date` no formato completo (2026-08-01T00:00:00-03:00) e `base` documento (valor do documento fechado) ou pagamento (valor pago, na data do pagamento). É a mesma conta das metas.
- Grupo de empresas: `group_sales_dashboard` (`mes` AAAA-MM) ou `list_orders` com `group_view`.
- Relatórios prontos: `list_reports` (`module` sales) e `run_sales_report` com o `report_name` exato da lista; volta no máximo 200 linhas.
- Outros recortes (data de fechamento, pagamento, etapa): campos em `list_filter_fields` (`module` pedido) para o `filters` de `list_orders`.

## Tabela de preço

1. Consulte com `list_price_tables` e `get_price_table` (este traz as exceções).
2. `create_price_table`: `base` (avista, aprazo, atacado, custo, custo_medio, ou table com `base_table_id`), `adjust_percent` (negativo é desconto), `rounding`, `application` (automatic entra sozinha ao informar o cliente; manual só quando escolhida), `clients` (ids de `search_people`), `groups`, `channels` e `exceptions` por produto (`search_products`), marca (`list_manufacturers`) ou categoria (`list_categories`), cada uma com `price` fixo ou `adjust_percent` próprio.
3. Para alterar, leia antes com `get_price_table`: em `update_price_table`, `exceptions`, `clients`, `groups` e `channels` substituem a lista inteira. Mande a lista completa, com o que já existia.
4. Confira com `resolve_price_table_prices` (`client_id` obrigatório em tabela restrita) e mostre preço, preço-base e margem. Preço zero é produto sem preço-base.
5. Regras: exceção do produto vence a da marca, que vence a da categoria; entre tabelas automáticas vale a de maior `priority`; cliente listado vence grupo; sem clientes nem grupos vale para todos; venda com pagamento lançado não troca de tabela. Promoção por período fica nas regras de desconto (`list_discounts`, só consulta).
6. Não há ferramenta que liste grupos de pessoas: use `clients` ou um grupo que já apareça em outra tabela.
7. `delete_price_table` não altera vendas já feitas.

## Reajuste de contratos

Não há ferramenta que liste ou crie contratos: o reajuste parte do plano e identifica o contrato pelo cliente e pelo número. Contratos vindos de planilha: `import_contracts`.

1. Veja os planos com `list_readjustment_plans` e `get_readjustment_plan`.
2. `create_readjustment_plan` exige `name` e `type`: percent ou amount com `value`, ou index com `index` (ipca, igpm, inpc) e, se houver, `additional`. Defina `interval_months`, `count_from`, `notice_days`, `scope` (plan, criteria com `criteria`, ou manual com `contracts`), `approval` e `skip_overdue`. `default` true faz o contrato novo nascer com o plano (só um plano é padrão). Canal whatsapp exige `whatsapp_integration_id`, que sai de `list_integrations` com `channel` whatsapp.
3. `update_readjustment_plan` vale para todos os contratos ligados ao plano; `delete_readjustment_plan` ou `active` false fazem esses contratos deixarem de reajustar. Confirme antes.
4. Antes de gerar, rode `simulate_contract_readjustment` (`plan_id`, não grava) e mostre quantos contratos, o total por mês antes e depois, o reajuste médio e os maiores aumentos.
5. `generate_readjustment_batch` grava o lote: os contratos sobem a partir do próximo ciclo. Com `approval` no plano o lote fica pendente, e só contrato de lote aprovado é reajustado e avisado; sem `approval`, o lote já nasce aprovado.
6. Lote pendente: revise com `get_readjustment_batch` e tire contratos com `remove_contracts_from_readjustment_batch` (`person_id` e `number` de cada um).
7. `approve_readjustment_batch` reajusta e avisa os clientes: confirme mostrando os totais. `cancel_readjustment_batch` pula este reajuste; `delete_readjustment_batch` só apaga lote pendente ou cancelado. `list_readjustment_batches` mostra a situação de cada lote.
8. `create_readjustment_batch` serve só para lote montado à mão, com contratos e valores informados.

## Metas e devoluções

- `list_sales_goals` mostra período e progresso. `recalculate_sales_goal` refaz o progresso (meta criada com o período já começado ou depois de cancelar venda). `replicate_sales_goal` copia a meta para os meses seguintes (`months` vazio vai até dezembro).
- `list_returns` só consulta devoluções; a carta de crédito gerada nelas aparece em `list_credit_letters`.

## Assinatura eletrônica

1. `send_for_signature` com `items` (`module` contract, service_order, order, quote ou upload, e o `record_id`). Sem `signers`, assina o cliente do documento, que precisa ter e-mail ou telefone no cadastro.
2. Acompanhe com `list_signature_envelopes` (`origin_record_id` filtra por documento) e `get_signature_envelope`.
3. `resend_signature_request` invalida o link anterior; `cancel_signature_envelope` exige `reason`.
4. `download_signed_document` devolve o link do PDF (o assinado, quando concluído); `verify_signature_code` confere o código da folha de assinaturas; `finalize_signature_envelope` refaz a finalização que falhou.

## Importar e renumerar

- `import_orders` e `import_contracts`: rode primeiro com `validate_only` true, mostre total e erros e só grave depois do sim do usuário.
- `renumber_documents` renumera todos os documentos do tipo e a numeração anterior se perde. É recusado se algum tem pagamento, nota ou já foi fechado. Use só a pedido explícito.

## Como apresentar

- Mostre o número do documento, não o id. Valores em R$ 1.234,56 e datas em dd/mm/aaaa.
- Em todo total, diga o período e o critério (data de criação ou de fechamento, com ou sem pendentes).
- Em lista longa, mostre os mais relevantes e o total, e ofereça o resto.
- Depois de gravar, diga o que foi feito e o que ainda falta fazer na tela.
