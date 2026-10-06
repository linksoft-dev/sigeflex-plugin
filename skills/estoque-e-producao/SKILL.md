---
name: estoque-e-producao
description: Use quando o usuário perguntar o saldo de um produto, cadastrar, alterar ou importar produtos, consultar lotes, séries, movimentações ou compras, abrir, apontar, finalizar ou cancelar ordem de produção, produzir na hora, ver custo de produção ou o que falta produzir para os pedidos.
---

# Estoque e produção

Se o usuário falar de outra empresa, use `list_companies` e informe `company_id` nas chamadas.

## De onde vêm os ids

- Produto: `search_products` (nome, código, EAN ou referência).
- Categoria, fabricante e tributação: `list_categories`, `list_manufacturers` e `list_tributations`.
- Depósito: `list_stocks`. Número de série filtra pelo `id` (`stock_id`); ordem de produção e localização do produto usam o `nome` (o padrão é `PADRAO`).
- Ordem de produção: `list_production_orders`, pelo número, produto, situação ou lote.
- Item de pedido de venda a produzir: `list_pending_demands` (`sale_order_id` e `sale_order_item_id`).

## Cadastrar e alterar produto

1. A busca padrão só traz produtos ativos. Sem resultado em `search_products`, repita com `list_products` e `situation: INATIVO`; se achar, reative com `update_product` (`situacao: ATIVO`) em vez de cadastrar outro.
2. O sistema recusa código ou EAN repetido em produto ativo, mas aceita nome repetido. Produto repetido divide o saldo e as vendas entre dois cadastros.
3. `create_product` só exige `nome`: unidade padrão UN, código gerado sozinho, nasce ativo e com estoque zero. Para sair em nota o produto usa NCM e tributação: informe `ncm` e `tributacao_id` quando o usuário souber. `suggest_product_fiscal` sugere NCM, CEST e tributação de até 20 produtos, sem gravar: mostre e peça que o usuário confirme com a contabilidade.
4. Serviço tem catálogo próprio (`list_services`): não cadastre serviço como produto.
5. Não há ferramenta para criar categoria ou fabricante: use um existente, deixe em branco ou crie pela importação.
6. `update_product` muda só os campos informados, inclusive `tributacao_id`, `ncm` e `cest`. Preço e custo mexem na venda e na margem: confirme o valor antes. Não há exclusão por aqui: para tirar de uso, mude `situacao` para INATIVO.
7. Vários produtos de uma vez: `update_products_batch` com `ids` muda categoria, fabricante, unidade, situação, fracionamento, NCM, CEST, tributação e margens. Com `recalcular_precos`, refaz os preços de venda a partir do custo. Confirme antes: vale para todos os produtos da lista.

## Importar produtos

1. Aceita CSV ou JSON, pelo final do `file_name` (`.csv` ou `.json`); arquivo em UTF-8 ou Latin-1 é lido certo. CSV do usuário (separado por `;`, com o cabeçalho do modelo do sistema) vai como veio. Para montar a lista você mesmo, use JSON (lista de objetos) com as chaves `nome` (obrigatório), `codigo`, `codigoean`, `un`, `ncm`, `precovenda`, `precoaprazo`, `precoatacado`, `precocusto`, `categoria`, `fabricante` e `quantidade`. Planilha Excel não é aceita: converta antes.
2. Rode `import_products` com `validate_only: true` e mostre os erros por linha. "Duplicado" é produto que já existe pelo nome, código ou EAN. Preço a prazo não pode ficar abaixo do à vista, nem atacado acima.
3. Com o ok do usuário, rode de novo sem `validate_only`. `import_categoria` e `import_fabricante` criam as categorias e os fabricantes que faltarem. `update_if_exists` só completa campo vazio e só aumenta preço; para baixar preço ou corrigir dado, use `update_product`.
4. A coluna `quantidade` vira entrada de estoque no depósito padrão só para produto novo. Produto que já existia não recebe saldo pela importação.

## Saldo e consultas de estoque

- `check_stock` (`product_ids` ou `product_name`) dá o saldo total. Por nome traz até 5 parecidos: confirme qual é antes de responder.
- `get_product` traz o saldo por depósito (`stock_by_location`), preços, custo, NCM, categoria e fabricante.
- Produto com `batch_control` ou `serial_control`: responda pelo disponível, não pelo total. `list_batches` (`query` com o número do lote ou o nome do produto) traz validade e saldo de cada lote; separe vencido e bloqueado. `list_serials` com `product_id` e `status: AVAILABLE` lista as unidades disponíveis; a conta vale só para o que voltou (padrão 50, `limit` até 300).
- `get_product_group_stock` mostra o saldo nas outras empresas do grupo. Só consulta: cada empresa tem o próprio estoque.
- Recortes com `list_products`: `stock_movement` (`in_stock`, `out_of_stock`, `has_movement`, `no_movement`), `rentabilidade` (`abaixo_do_custo`, `sem_margem`, `sem_custo`, `sem_preco`), `markup_gte`, `markup_lte`, `category_id`, `manufacturer_id`. O `total_stock_cost` soma só os itens devolvidos (até 500 por chamada): com mais produtos, avise que o valor é parcial.
- `list_stock_movements` traz só os lançamentos manuais da tela Movimentação de Estoque, com situação. Venda, nota, ajuste e produção mexem no saldo sem aparecer ali.
- `list_stock_adjustments` e `list_inventories` consultam ajustes e inventários já lançados.

## Entrada, ajuste e inventário

O estoque entra por movimentação, e não há ferramenta para lançar entrada, saída, ajuste, inventário, lote ou número de série (não é falta de liberação). Oriente o usuário a fazer no sistema e depois confira com `check_stock`:

- contagem física ou saldo errado: Estoque > Ajuste de Estoque (informa o saldo real e o sistema lança a diferença);
- mercadoria sem nota: Estoque > Entrada Avulsa; transferência ou baixa manual: Estoque > Movimentação de Estoque;
- movimento lançado no produto errado: Estoque > Correção de Movimento.

A produção também movimenta o estoque (abaixo).

## Compras

- `list_purchase_orders` mostra fornecedor, situação e valor. Pedido parcial ou recebido já teve mercadoria recebida; rascunho, emitido ou confirmado ainda não.
- A entrada acontece quando o recebimento é registrado no pedido de compra, no sistema. Notas recebidas de fornecedor: `list_nfe` com `document_type: nfe_entrada`; o lançamento da nota pela foto e o vínculo dos itens seguem o fluxo de notas fiscais.

## Antes de produzir

1. `estimate_production` (`product_id`, `quantity`) diz se dá para produzir, o máximo possível e o insumo que limita. Não movimenta nada.
2. Erro de ficha técnica na estimativa quer dizer que o produto não tem composição. Sem ela, ordem e produção rápida só dão entrada no acabado, sem baixar insumo, valorizado pelo custo do cadastro: avise o usuário. Ficha técnica e rendimento se montam no cadastro do produto (Produto Composto, Composição), no sistema.
3. `preview_production_costs` mostra os custos extras (energia, embalagem) que entram sozinhos na ordem desse produto.

## Ordem de produção

1. `create_production_order` com `product_id`, `planned_quantity`, `expected_date` e, se não for o padrão, `input_stock_name` e `output_stock_name`. Não mande insumos: vêm da ficha. A ordem nasce pendente e numerada.
2. Pendente: `update_production_order` muda quantidade (nunca abaixo do total dos pedidos vinculados), data, depósitos e observação. `calculate_production_cost` recalcula os insumos pelo custo atual e grava `labor_cost` e `other_cost`; o que não for informado mantém o valor da ordem. Se a ordem tem custos extras detalhados, `other_cost` é recusado: ele é a soma desses custos, que se alteram na tela. Faça isso antes de iniciar.
3. `start_production` só em ordem pendente: congela o custo previsto e trava a ordem. Depois disso `update_production_order` só grava observação e ignora o resto sem dar erro.
4. Cada leva: `add_production_entry` com `good_quantity` e `lost_quantity` (refugo consome insumo e não entra no estoque). Só em ordem em produção, e a quantidade boa não passa do que falta. Baixa os insumos na proporção, dá entrada da parte boa e reparte entre os pedidos vinculados pela entrega mais próxima (`prioritize_stock` manda tudo para o estoque). O primeiro apontamento define o lote da ordem.
5. `finish_production` só em ordem em produção. Sem apontamento, processa tudo de uma vez: sem `produced_quantity` vale a quantidade planejada inteira, `batch_code` vazio o sistema gera, e `manufacture_date` e `expiration_date` vão para o lote. Com apontamento, só encerra: a quantidade é a soma das levas (para produzir mais, aponte antes).
6. Faltou insumo: apontamento e finalização param com a mensagem do que falta, e nada se move (salvo produto que permite estoque negativo). Mostre o insumo e a quantidade.
7. Desistir: `cancel_production` (com `reason`) ou `delete_production_order`. Nenhum vale para ordem finalizada ou com apontamento; nesse caso, finalize com o que já foi produzido.

## Produção rápida

`quick_production` cria, inicia e finaliza numa chamada (`product_id`, `quantity`, `lost_quantity`, lote, validade, `labor_cost`). É o caminho de quem produz e vende no mesmo dia. Se falhar por falta de insumo, a ordem criada é cancelada sozinha: resolva o estoque antes de repetir.

## Produzir para pedidos de venda

1. `list_pending_demands` lista, por produto, os itens de pedido ainda sem ordem, com cliente e data de entrega.
2. `generate_production_from_sale_orders` cria uma ordem por produto (`orders`, cada um com `product_id`, `demands` e, se quiser, `extra_quantity` para o estoque). Exige ficha técnica; item com composição própria no pedido vira ordem separada.
3. Ordem existente: `add_production_demands` vincula itens com a ordem pendente ou em produção (`increase_planned_quantity` aumenta a quantidade se precisar). `remove_production_demand` com o `demand_id` (o id da demanda, que vem em `get_production_order`) desvincula só enquanto nada foi produzido para aquele item.
4. O pedido acompanha sozinho: aguardando produção, em produção e pronto para expedição.

## Custos de produção

- Veja `list_production_costs` (detalhe em `get_production_cost`) antes de `create_production_cost`, para não duplicar. `type` é `FIXED` (por ordem) ou `PER_UNIT` (por unidade); `apply_by_default` faz o custo entrar sozinho em toda ordem nova.
- Cadastrado por aqui, o custo vale para qualquer produto. Restringir a categorias só no sistema (Produção > Custos de Produção). Ordem já criada não recebe o custo novo.

## Acompanhar e rastrear

- `production_dashboard`: demanda sem ordem, em andamento, atrasadas, pronto para expedir e números do mês.
- `list_production_orders` com `status` (`PENDING`, `IN_PRODUCTION`, `COMPLETED`, `CANCELED`), `only_late`, `product_id`, `number`, `batch_code` ou faixa de data prevista.
- `get_production_order` traz a ordem completa: insumos com quantidade e custo, apontamentos, demandas dos pedidos vinculados, lotes consumidos e custo previsto x realizado. `print_production_order` monta a impressão de uma ou várias ordens (`ids`).
- Lotes produzidos: `list_production_batches` (por produto, fabricação ou validade) e `get_production_by_batch`, que leva do código do lote às ordens que o geraram, com os lotes de matéria-prima consumidos e os fornecedores. O caminho inverso, do lote de matéria-prima para os produtos feitos com ele (recall), só pela tela Produção > Rastreabilidade.
- `run_production_report` com `report_type` `producaoListaOrdens`, `producaoCustoPrevistoRealizado`, `producaoConsumoInsumos` ou `producaoDemandaPedidos`, filtrando por `status` e `product_id`. Para desvio de custo, use só as finalizadas.

## Confirme antes de executar

Não há estorno por aqui; corrigir depois exige ajuste de estoque no sistema.

- `add_production_entry`, `finish_production` e `quick_production` baixam insumos e dão entrada no estoque: confirme produto, quantidade boa, refugo e lote.
- `cancel_production` e `delete_production_order` são definitivos; `start_production` trava a ordem.
- `import_products` sem `validate_only` grava tudo de uma vez, e `update_products_batch` muda todos os produtos da lista.

## Como apresentar

- Fale em nome, código e quantidade com unidade (12 KG). Ordem pelo número, nunca pelo id.
- Saldo: o total e, com mais de um depósito, a divisão por depósito. Aponte saldo negativo.
- Ordem finalizada: quantidade boa, refugo, perda %, custo unitário e lote.
- Estimativa: o máximo que dá e o insumo que limita.
- Importação: quantos entraram, quantos falharam e os erros agrupados por motivo.
- Valores em R$ no formato brasileiro e datas em dd/mm/aaaa.
