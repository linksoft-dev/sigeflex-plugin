---
name: marketing
description: Use quando o usuário pedir campanha de e-mail, WhatsApp ou SMS, público ou segmento de contatos, criativo, post para Instagram ou outra rede, arte do Canva, aprovar, agendar ou publicar post, resultado de campanha, post ou anúncio, ou mudar o texto e a conta das mensagens que o sistema envia aos clientes.
---

Se o usuário falar de outra empresa, use `list_companies` e informe `company_id` nas chamadas.

## Público: lista ou segmento

Lista (`kind` LIST) guarda os contatos gravados. Segmento (SEGMENT) guarda critérios e é recalculado a cada uso, inclusive no disparo. Não há ferramenta para alterar nem excluir público: acerte antes de criar.

1. Procure com `list_segments` (`query`, `kind`) e reaproveite o que existir.
2. Para criar, `create_segment`. Com `criteria` vira segmento, e a pessoa só entra se atender a todos. Campos e valores:
   - `person.tag`: customer (cliente), vendor (fornecedor), salesman (vendedor) ou lead
   - `person.status`: PERSON_STATUS_ACTIVE ou PERSON_STATUS_INACTIVE
   - `contract.status`: CONTRACT_STATUS_ACTIVE, CONTRACT_STATUS_TRIAL, CONTRACT_STATUS_TRIAL_EXPIRED, CONTRACT_STATUS_SUSPENDED ou CONTRACT_STATUS_CANCELED
   - `pedido.situacao`: PENDENTE, FECHADO ou CANCELADO (quem tem pedido nessa situação)
   - `person.city`, `person.state` (UF, ex.: SP), `person.group` (nome do grupo)
   - `list.id`: id de uma lista; com `ne`, exclui quem está nela

   Operador: `eq` (padrão), `contains` ou `ne`. Valor escrito de outro jeito não dá erro, só traz público vazio.
3. Contato avulso só entra em lista, com `import_segment_contacts` (segmento recusa). Rode antes com `validate_only: true`. Cada contato é procurado no cadastro de pessoas pelo e-mail e, sem e-mail, pelo telefone; quem não é achado vira cadastro novo como cliente. Contato que já está na lista só muda com `update_if_exists`, que grava o nome, o e-mail, o telefone e os atributos que vierem preenchidos.
4. Confira o público com `resolve_segment_contacts` antes de usar.

### Quantos recebem

- `resolve_segment_contacts` traz nome, e-mail e telefone de quem entra hoje, já sem os desinscritos, até o `limit` (padrão 20, máximo 200). `has_more` diz se há mais gente além da amostra.
- `total_count` só vem quando é exato. Sem ele e com `has_more`, peça `limit: 200` e, se continuar assim, diga "200 ou mais". O `contact_count` de `list_segments` não mede segmento.
- Só recebe quem tem o dado do canal: e-mail na campanha de e-mail, telefone no WhatsApp e no SMS. Conte na amostra e diga ao usuário.
- Público acima da cota do plano é recusado inteiro no disparo, com o teto na mensagem: divida o público.

## Criativo (conteúdo da campanha)

1. Procure com `list_creatives` (`template_type`: email, whatsapp ou sms).
2. Para criar, `create_creative`: e-mail leva `html_content`; WhatsApp e SMS levam `text_content`. Personalize com `{{nome}}`, `{{email}}`, `{{telefone}}` e campos extras da lista, como `{{list.campo}}`.
3. Imagem no e-mail: traga com `import_canva_design` ou `upload_file` e use a URL devolvida no HTML.
4. Confira com `preview_creative` (`creative_id`, `segment_id`): mostra a mensagem com os dados de um contato real e as `missing_variables`. Variável sem valor chega vazia ou com as chaves no texto: ajuste com `update_creative`.

## Campanha: do rascunho ao disparo

1. Tenha o `segment_id` com o público conferido e o `creative_id` do mesmo canal da campanha (o sistema não confere o canal do criativo).
2. Escolha a conta que envia: `list_integrations` com o `channel` da campanha devolve as contas conectadas em Integrações. O envio nativo de e-mail do sistema não serve para campanha.
3. `create_campaign` com `name`, `segment_id`, `creative_id`, `channel`, `integration_id` e, no e-mail, `subject` (obrigatório para disparar, aceita `{{nome}}`), `sender_name` e `sender_email`. Sem `integration_id`, a campanha não dispara. Sem `scheduled_at`, ela fica em rascunho e nada sai.
4. Pergunte um e-mail ou telefone do próprio usuário e rode `simulate_campaign_send` (`id`, `destination`). O teste chega com "[SIMULAÇÃO]" no assunto ou no começo do texto. Ajuste o que ele pedir.
5. Confirme numa mensagem só: nome da campanha, canal, quantos recebem, assunto e quando sai. Envio em massa não se desfaz.
6. Com o sim, `dispatch_campaign` com o `id`. A campanha fica em enviando (SENDING) e o envio segue em segundo plano.

Armadilhas:

- `scheduled_at` já agenda: com data, a campanha nasce agendada e sai sozinha na hora marcada, sem `dispatch_campaign`. Confirme antes, como num disparo. Com data, `integration_id` é obrigatório. `dispatch_campaign` com `schedule: true` só confirma a data já gravada.
- Não há ferramenta para alterar nem excluir campanha. Rascunho errado: crie outro e não dispare o errado.
- Campanha concluída não dispara de novo: `clone_campaign` cria um rascunho com o mesmo público, criativo e conta.

## Resultado da campanha

- `campaign_stats` (`id`): enviados, entregues, abertos, cliques, rejeitados, falhas e `error_message`. Para achar ou comparar campanhas, `list_campaigns` com `query` ou `status`.
- Apresente os números e as taxas de abertura e clique sobre os entregues, as falhas com o motivo e a situação em português (rascunho, agendada, enviando, pausada, concluída, falhou).
- Logo depois do disparo os números ainda estão chegando: diga que está enviando em vez de concluir.

## Post em rede social

1. `list_social_accounts`: escolha conta com `active` verdadeiro e anote `integration_id` e `network`. Sem conta ativa da rede, peça para conectar em Integrações antes de agendar.
2. Texto: escreva com o usuário ou use `generate_social_post` (gasta crédito de IA, devolve texto e hashtags para revisar e não salva nada). Informe em `platform` a rede de menor limite entre as escolhidas. Ela não gera arte: imagem e vídeo vêm de `import_canva_design` ou `upload_file`.
3. Mídia que vem de fora (arquivo do usuário, link de terceiro) entra com `upload_file`, que devolve URL definitiva; link de fora expira e a publicação falha depois. Arte do Canva: `import_canva_design` com o id do design (o trecho depois de /design/ no link), com o Canva conectado nas Integrações do sistema. Arquivo já enviado: `list_files`. Arquivo grande: `get_storage_usage` mostra o espaço livre e o máximo por arquivo.
4. `create_social_post` com `content`, `title` (nome interno), `hashtags` sem o # e `media_url` (vídeo: `media_type` video). O post nasce rascunho. Carrossel: `update_social_post` com `media_urls` na ordem.
5. Post que pede aprovação só é publicado depois de aprovado. Com a permissão de aprovar posts, `review_social_post` com `id` e `approve: true` aprova; `approve: false` com `reason` devolve pedindo ajuste. Mudar texto, hashtags ou mídia depois da aprovação devolve o post para aprovação.
6. Confirme rede, conta, data, hora e texto final. Então `schedule_social_post` por rede (`post_id`, `network`, `integration_id`, `scheduled_at` e, se precisar, `custom_content` só para aquela rede). Agendado sai sozinho na hora. O `id` da publicação nova vem em `publications` na resposta: é o `publication_id` das próximas chamadas.
7. Publicar agora: agende primeiro (é o que cria a publicação) e, com o sim do usuário, chame `publish_social_post` (`post_id`, `publication_id`).
8. Acompanhe em `list_social_posts`: situação de cada publicação, `approval_status` com a situação da aprovação, `error_message` da que falhou e `external_url` da que saiu, para mandar ao usuário. Aprovação que chega depois do horário deixa a publicação em falha: remarque com `reschedule_social_post` ou publique com `publish_social_post`.

| Rede | Texto, contando hashtags | Mídia |
|---|---|---|
| Instagram | 2.200 | obrigatória, até 10 |
| Facebook | 63.206 | até 10 |
| LinkedIn | 3.000 | até 20 imagens ou 1 vídeo sozinho |
| X | 280 | não é enviada, sai só o texto |
| TikTok | 2.200 | exatamente 1 vídeo |
| Threads | 500 | até 20 |

Texto acima do limite faz a publicação falhar: encurte ou use `custom_content` naquela rede.

## Mudar ou desfazer post

- `reschedule_social_post` muda horário e texto de publicação ainda não publicada, mas não troca a conta. Para mudar a conta, `remove_social_post_publication` e agende de novo com `schedule_social_post`.
- Não deve mais sair: `cancel_social_post_publication` (só agendada; o registro fica no post) ou `remove_social_post_publication` (tira a rede do post). Nenhuma das duas mexe no que já foi publicado.
- Já publicado: `unpublish_social_post` apaga da rede e não se desfaz. O TikTok não permite: apague pelo aplicativo.
- `delete_social_post` apaga só o registro no sistema; o que saiu continua no ar. Para tirar do ar, despublique antes.
- `delete_file` faz a imagem sumir dos posts que apontam para ela: confira antes em `list_social_posts`.
- Confirme com o usuário antes de `unpublish_social_post`, `delete_social_post` e `remove_social_post_publication`.

## Métricas de post e anúncios

- `sync_social_post_metrics` (`post_id`) busca os números do momento nas redes e recalcula o score. Apresente o score (0 a 100) com o nível, o principal motivo, a primeira recomendação e os números por rede. Sem métrica sincronizada o nível é "Sem dados". TikTok e Threads não informam cliques nem salvamentos: zero ali não é resultado ruim.
- Anúncios do Meta Ads e do Google Ads são só leitura e dependem da conta de anúncios conectada em Integrações; criar ou mudar anúncio é na própria plataforma. `ads_performance` (`date_from` e `date_to` no formato 2026-08-01) traz investimento, impressões, cliques, CTR e CPC médios e a divisão por plataforma.
- `ads_roi` ainda não está disponível: a chamada é recusada. Para o desempenho dos anúncios, use `ads_performance`.

## Modelos de mensagem (envios do sistema ao cliente)

Textos de orçamento, pedido, cobrança, nota fiscal, avaliação e afins, por e-mail ou WhatsApp. Não confunda com criativo, que é de campanha.

1. `list_message_event_types`: pegue a `key` do tipo de envio, os canais e as `variables`.
2. `list_message_templates` com `event_type` e `channel`: mostra o modelo do sistema (`system_default`), os da empresa e qual vale no envio (`used_on_send`). Parta do conteúdo atual.
3. Escreva com as variáveis exatamente como vêm em `expression` (template Jet; o `{{nome}}` do criativo não vale aqui). Para testar campo vazio, use `{{ if .Campo }}...{{ end }}`, nunca `!= nil`. E-mail é HTML e exige `subject`; WhatsApp é texto.
4. `preview_message_template` com `event_type`, `channel`, `content` e `subject` ainda não salvos: mostre ao usuário como fica com os dados de exemplo.
5. Salve com `update_message_template` no modelo do sistema (grava uma cópia da empresa, com outro `id`, e o original continua) ou com `create_message_template` (nome único no tipo e canal). Conteúdo que não compila é recusado.
6. Depois de salvar, confira `used_on_send` em `list_message_templates`. Se o modelo novo não estiver valendo, use `set_default_message_template`.
7. Voltar ao modelo do sistema: `set_default_message_template` com o id dele, ou `delete_message_template` na cópia da empresa.

### Conta que envia

1. `list_integrations` com `channel` (email ou whatsapp): contas conectadas, a padrão do canal (`is_default`) e se está ativa.
2. `set_message_template_account` com `event_type`, `channel` e `integration_id` (ou `native_email`, só no e-mail). Sem `integration_id`, volta para a conta padrão do canal.
3. Vale para todo envio desse tipo, inclusive os automáticos. Se a conta escolhida for desconectada, esses envios param até escolher outra: escolha só conta ativa e avise o usuário.

## Feito na tela do sistema

Escolher conta ou data de campanha já criada, alterar ou excluir campanha e público, automações de marketing e sincronizar anúncios na hora não têm ferramenta, e não é falta de liberação. Diga ao usuário que isso se faz no menu Marketing do sistema e siga com o resto.

## Ao responder

- Use nomes, não ids. Situações em português, datas em dd/mm/aaaa hh:mm, valores em R$.
- Depois de criar, diga em que estado ficou (rascunho, agendado, aguardando aprovação) e o próximo passo.
- Depois de disparar ou publicar, diga o que saiu, para quantos ou em qual rede, e como acompanhar.
