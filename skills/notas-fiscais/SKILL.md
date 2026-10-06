---
name: notas-fiscais
description: Use quando o usuário pedir para emitir, gerar, conferir ou cancelar nota fiscal (NF-e, NFC-e ou NFS-e), faturar um pedido ou orçamento, mandar o DANFE ou o XML, entender uma rejeição da SEFAZ, consultar notas, lançar nota de fornecedor pela foto, ver a tributação dos produtos ou os números fiscais do período.
---

# Notas fiscais

Se o usuário falar de outra empresa, use `list_companies` e informe `company_id` nas chamadas.

## Antes de começar

- Produto sai em NF-e ou NFC-e; serviço sai em NFS-e. Pedido com produtos e serviços gera duas notas.
- NFC-e é o cupom do consumidor final, dentro do estado. NF-e é para empresa, para revenda, para outro estado ou quando o cliente pede NF-e. Na dúvida, pergunte.
- O id de uma nota só vale no módulo dela: repita o mesmo `document_type` (`nfe`, `nfce` ou `nfe_entrada`) em `list_nfe`, `get_nfe`, `get_danfe`, `send_nfe` e `cancel_nfe`. `nfe_entrada` é a nota emitida pelo fornecedor: só aparece nas consultas, não se emite nem se cancela (veja Nota de fornecedor).
- Situações da NF-e e da NFC-e: DIGITADA (gerada, ainda sem valor fiscal), PENDENTE (aguardando autorização, em contingência), AUTORIZADA, REJEITADA e CANCELADA.
- Não têm ferramenta aqui e são feitos na tela do sistema: transmitir, imprimir e cancelar NFS-e; carta de correção; nota de devolução avulsa; inutilização; alterar uma nota já gerada; enviar a nota por e-mail ou WhatsApp. Diga isso ao usuário em vez de procurar outro caminho.

## Gerar e emitir a nota de um pedido

1. Ache o pedido com `list_orders` (`query` com número ou cliente). Orçamento: o id vem de `create_quote` ou do link do orçamento em `get_public_document`. Orçamento aprovado costuma virar pedido; se `create_nfe` recusar o orçamento por não estar fechado, gere pelo pedido.
2. `get_order`: confira situação, cliente, itens e pagamentos. Só documento FECHADO vira nota (a não ser que a empresa permita faturar venda em aberto); cancelado, nunca. NF-e e NFC-e levam os produtos; NFS-e, os serviços.
3. Mostre o resumo e peça um "sim" explícito para gerar e emitir: tipo da nota, cliente com CPF/CNPJ, itens e quantidades, total em R$ e forma de pagamento. Se o usuário pediu só para gerar (vai conferir na tela), gere e pare.
4. `create_nfe` com `order_id` e `document_type` (`nfe`, `nfce` ou `nfse`). Vários pedidos do mesmo cliente numa nota só: `order_ids` no lugar de `order_id`. `customer_id` só se o usuário pedir outro destinatário. A nota nasce DIGITADA, sem ir à SEFAZ.
5. Confira `note_id` na resposta. Sem ele, nada foi gerado (por exemplo, a empresa não tem o módulo fiscal daquele tipo): avise e pare. NFS-e continua na seção NFS-e.
6. `get_nfe` com o `note_id` e o mesmo `document_type` traz os itens com CFOP, NCM e tributos (ICMS, IPI, PIS e COFINS com CST e alíquota): destinatário, itens, tributos e total precisam bater com o resumo. Se não baterem, mostre a diferença e pergunte antes de seguir.
7. `send_nfe` com `id` e `document_type`.

Recusas de `create_nfe` e o que fazer:

- "não está fechado": a venda precisa ser fechada antes (no fluxo de vendas, `add_order_payment` fecha o pedido quando o total fica pago).
- "já possui nota fiscal ... AUTORIZADA": a nota já existe. Ache com `list_nfe` (`query` com nome ou CPF/CNPJ do cliente).
- "gerada e ainda não enviada": não gere outra. Ache a DIGITADA com `list_nfe` (`status` DIGITADA) e emita essa.
- Pedidos de clientes diferentes não saem na mesma nota.

## Resultado da emissão

- AUTORIZADA (`emitted` verdadeiro): informe número, série, chave de acesso e os links `danfe_url` e `xml_url`. Na NF-e, se o cliente tem e-mail, o sistema já enviou a nota para ele.
- PENDENTE: a SEFAZ estava fora do ar e a nota saiu em contingência (`reason` diz por quê). O sistema autoriza sozinho quando ela voltar. Não reenvie nem gere outra; confira depois com `get_nfe`.
- Erro: veja Rejeição e erros.
- Sem resposta clara (tempo esgotado, falha de conexão): consulte `get_nfe` antes de qualquer nova tentativa.

## Emissão direta, sem pedido

Use só quando o usuário quer a nota sem registrar a venda: a emissão direta não cria pedido. Se a venda existe, gere pelo pedido.

1. `search_people` para o destinatário (opcional na NFC-e) e `search_products` para cada item. O produto precisa estar cadastrado.
2. Resumo e confirmação, como no fluxo do pedido.
3. `send_nfe` com `document_type`, `idempotency_key` e `note`, por exemplo:
   `{"pessoa":{"id":"..."},"produtos":[{"produtoId":"...","quantidade":2}],"pagamentos":[{"formaPagamentoNome":"PIX","valor":150}]}`
   Preço, unidade e tributação vêm do cadastro do produto; impostos e totais, o sistema calcula. Em `pagamentos`, `formaPagamentoNome` é DINHEIRO, PIX, CREDITO etc., e `valor` é o total pago pelo cliente.
4. Gere uma `idempotency_key` nova para cada nota. Se a chamada cair sem resposta, repita com a mesma chave: ela devolve a nota da primeira chamada em vez de emitir outra. A mesma chave também devolve a mesma rejeição: depois de corrigir os dados, use chave nova.

## Rejeição e erros

- Erro com "Rejeitado pela SEFAZ" é recusa da SEFAZ e já vem explicado, com o código (cStat). Sem esse texto, o erro é do sistema antes do envio (dado obrigatório, certificado, SEFAZ fora do ar): leia a mensagem, ela diz o que falta.
- Nota já rejeitada: `get_nfe` traz em `rejection` o código (`cstat`) e o motivo (`message`) da última rejeição. Para uma explicação com o passo a passo, `explain_dfe_rejection` com `documento` (nfe, nfce ou nfse) e `id`; ela consome créditos de IA quando a empresa não tem integração de IA própria.
- Explique em poucas palavras o que está errado e onde corrigir. Não reenvie sem mudar nada: a rejeição se repete, e insistir faz a SEFAZ bloquear os envios da empresa por cerca de uma hora (consumo indevido, cStat 656).
- Onde corrigir nos casos comuns:
  - IE, endereço ou UF do cliente (IE inválida ou inativa, CFOP de dentro x fora do estado): confira com `get_person` e corrija com `update_person` (`ie` por UF, `ident_free` para quem é isento, `address` completo).
  - Cliente sem Inscrição Estadual em venda para revenda (cStat 696): ele só recebe nota de consumidor final.
  - NCM ou dados de combustível do produto: cadastro do produto (`get_product`; `update_product` troca `ncm` e `cest`).
  - Certificado vencido, CSC e token da NFC-e, empresa não credenciada na SEFAZ: configuração fiscal da empresa, com a contabilidade.
  - SEFAZ paralisada ou com erro interno (cStat 108, 109 ou 999): a nota está certa; tente mais tarde.
  - Duplicidade (cStat 204 ou 539): consulte com `list_nfe` antes de emitir outra.
- Depois que o usuário corrigir (no cadastro ou na própria nota, pela tela), reenvie com `send_nfe` e o mesmo id.

## Cancelar

1. Ache a nota com `list_nfe` e confira com `get_nfe`: precisa estar AUTORIZADA.
2. NF-e e NFC-e só cancelam até 24 horas depois da emissão (`issued_at`). Passou disso, não tente: o caminho é a nota de devolução, gerada na tela da nota original (Opções > Gerar NF-e de devolução), e vale alinhar com a contabilidade.
3. Erro que não muda valor, imposto, mercadoria nem destinatário (texto, transportador, natureza da operação): sugira carta de correção pela tela da nota, sem cancelar.
4. Confirme número, cliente, valor e motivo. `cancel_nfe` com `id`, `reason` (15 a 255 caracteres, com o motivo dito pelo usuário) e `document_type` (`nfe` ou `nfce`, padrão `nfe`).
5. Informe a situação CANCELADA e o protocolo de cancelamento. Cancelar a nota não cancela a venda: o pedido continua fechado e pode receber outra nota.

Para desfazer a venda inteira, `cancel_order` no pedido (fluxo de vendas) cancela também a nota ou, fora do prazo, gera a devolução. NFS-e se cancela na tela de NFS-e, com o prazo da prefeitura.

## NFS-e

1. Gere pelo pedido com serviços: `create_nfe` com `document_type` `nfse`.
2. Avise que a transmissão à prefeitura (botão Enviar), a impressão e o cancelamento são feitos na tela de NFS-e do sistema.
3. Acompanhe com `list_nfse` (`query` com número ou tomador; `status` DIGITADA, AUTORIZADA, REJEITADA ou CANCELADA).
4. Rejeição costuma ser código de tributação municipal do serviço, dados do cliente incompletos ou credenciais da prefeitura. O motivo aparece na nota, e `explain_dfe_rejection` com `documento` nfse explica o passo a passo.

## DANFE e XML

- `get_danfe` com `id` e `document_type` devolve `url` (link público do PDF para NF-e e NFC-e). Só existe DANFE depois da autorização.
- NF-e de entrada não tem link público: o DANFE dela sai pela tela de NF-e de entrada.
- O link do XML (`xml_url`) vem só na resposta de `send_nfe` autorizada. Depois disso, nenhuma ferramenta devolve o XML: ele sai pela tela da nota.

## Nota de fornecedor (entrada)

- Consulta: `list_nfe` e `get_nfe` com `document_type` `nfe_entrada` (situações também ACEITA e MANIFESTADA).
- Foto do DANFE ou do cupom: `import_nfe_entrada_from_image` com `image_base64` e `mime_type` (JPEG, PNG ou WebP, até 5 MB). NF-e segue pela chave lida (XML da SEFAZ); NFC-e vira entrada pendente com os itens da foto. Confirme antes e mostre os avisos do que conferir.
- Nota pendente com itens sem produto: `link_nfe_entrada_items` vincula ao produto do estoque ou sugere a categoria do cadastro novo; os vinculados ficam marcados para conferir. `suggest_nfe_entrada_prices` sugere o preço de venda à vista de cada item, sem gravar.
- Essas três usam IA e consomem créditos quando a empresa não tem integração de IA própria. O aceite (entrada no estoque) é feito na tela de NF-e de entrada.

## Tributação

- `create_nfe` e `send_nfe` não recebem tributação. O imposto de cada item vem da tributação do produto; sem ela, da categoria; sem ela, da tributação padrão da empresa. Na venda para outro estado, o sistema troca o CFOP sozinho.
- Venda para quem revende: a NF-e sai sem consumidor final e usa a tributação de revenda, quando cadastrada. No pedido, isso é o `resale` de `create_order`.
- `list_tributations` (`query` com nome ou CFOP) devolve id, nome, CFOP de saída e alíquota de ICMS. O id vai em `create_product` ou `update_product` (`tributacao_id` e `tributacao_revenda_id`); `update_product` também troca `ncm`, `cest` e os ajustes de ICMS, IPI e ST. Vários produtos de uma vez: `update_products_batch`.
- `suggest_product_fiscal` (até 20 produtos) sugere NCM, CEST e tributação sem gravar; consome créditos de IA quando a empresa não tem integração própria.
- Não escolha tributação, CST, CSOSN ou alíquota por conta própria: mostre as opções ou a sugestão e peça que o usuário confirme com a contabilidade. Mudar o produto vale para as próximas notas, não para as já autorizadas.

## Relatórios e indicadores

- Totais do período: `fiscal_dashboard` com `date_from` e `date_to` (AAAA-MM-DD; sem datas, mês corrente). Traz valor emitido, canceladas, impostos, principais CFOPs, totais por tipo e os últimos 12 meses, pela data de emissão.
- Lista de notas: `list_nfe` (`document_type`, `query`, `status`, `date_from`, `date_to`, `limit`). As datas aceitam a data simples (2026-09-01; no fim do período vale o dia inteiro) ou data e hora com fuso; data inválida é recusada. O `total` de `list_nfe` soma só as notas devolvidas: para a soma do período, use `fiscal_dashboard`.
- A tela de NFS-e filtra pela competência e o painel pela emissão: o mesmo mês pode dar números diferentes.
- Relatório pronto: `list_reports` com `module` `fiscal` para achar o nome, depois `run_fiscal_report` com `report_name` (aceita `query` e `status`, não período).

## Como apresentar

- Situação em palavras ("autorizada", "rejeitada pela SEFAZ", "em contingência"); cStat só entre parênteses.
- Número e série, cliente, valor em R$ (1.234,56) e data dd/mm/aaaa. Sem ids internos nem JSON.
- Emissão autorizada: chave de acesso e link do DANFE.
- Listas: quantidade e total primeiro; detalhe só se pedirem.
