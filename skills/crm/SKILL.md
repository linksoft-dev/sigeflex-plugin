---
name: crm
description: Use quando o usuário falar de oportunidades ou negociações, funil e etapas, follow-up, tarefas e ligações com clientes, fechar venda como ganha ou perdida, gerar pedido ou orçamento de uma negociação, previsão de vendas, automações do funil, ou quiser prospectar empresas e salvar como leads.
---

# CRM e prospecção

Se o usuário falar de outra empresa, use `list_companies` e informe `company_id` nas chamadas.

Funil é o pipeline, oportunidade é a negociação (deal) e lead é o cadastro de uma pessoa ou empresa que ainda não é cliente.

## Regras que valem para tudo

- Escolha funil e etapa pelo nome que o usuário disse, entre os devolvidos por `list_pipelines`. Se o nome da etapa existir em mais de um funil, pergunte qual.
- Ache a oportunidade com `list_deals`: `query` busca por título, cliente ou número (`#12`). Sem `outcome`, a lista traz só as abertas; para as fechadas, use `WON` ou `LOST`.
- `search_people` sem `type` não mostra quem é só lead: para incluir leads, use `type` `customer,lead`. Só cadastre com `create_person` depois disso; para cadastrar como lead, `type` lead (sem ele, vira cliente).
- Datas aceitam data e hora com fuso, como `2026-10-09T14:00:00-03:00`, ou só a data, que vale no fuso do sistema: início do dia em `due_at`, `close_forecast` e no começo dos períodos, e o dia inteiro no fim do período (`due_to`, `created_to`).
- Nenhuma ferramenta escolhe ou troca o responsável (nem `update_deal`): a oportunidade nasce tendo como responsável o usuário que autorizou a conexão, e a troca é feita na tela.
- Mover para uma etapa chamada "Fechado" ou "Ganho" não fecha a oportunidade. Fechar é só com `close_deal`.
- Oportunidade ganha ou perdida não aceita `update_deal` nem `move_deal_stage`. Para mexer nela, é preciso reabrir com `reopen_deal`: pergunte antes.

## Abrir oportunidade

1. `list_pipelines` para achar o funil e a etapa inicial.
2. Ache o cliente ou lead (regra acima). O cliente é opcional, mas sem ele o pedido ou orçamento gerado depois sai sem cliente.
3. `create_deal` com `title`, `pipeline_id`, `stage_id`, `person_id`, `person_name` (o nome não é preenchido a partir do id), `amount` e, se fizer sentido, `tags` e `notes`.
4. A previsão de fechamento não entra na criação: em seguida, chame `update_deal` com `close_forecast`.
5. Responda com o número (`sequence`), funil, etapa e valor.

Lead qualificado vira oportunidade assim, vinculando o `person_id` dele. Quando a oportunidade é ganha, o lead passa a ser cliente sozinho.

## Alterar e mover

- `update_deal` muda título, valor, cliente (`person_id` com `person_name`), `close_forecast` e `tags`. `tags` substitui a lista inteira: leia as atuais com `get_deal` e envie todas. Não muda responsável: essa alteração fica para a tela.
- `move_deal_stage` com `id` e `stage_id`. Para levar a outro funil, informe também `pipeline_id`.
- Criar e mover disparam as automações ativas do funil que agem quando a oportunidade entra na etapa: elas podem criar tarefas e mandar e-mail ao contato. Antes de mover várias de uma vez, confira com `list_crm_automations`.

## Atividades e histórico

1. `create_activity` com `deal_id`, `title` (3 a 160 caracteres), `type` (`TASK`, `CALL`, `EMAIL`, `WHATSAPP` ou `MEETING`; sem tipo, vira `TASK`) e `due_at`. Toda atividade pertence a uma oportunidade.
2. `list_activities` filtra por `deal_id`, `person_id`, `type`, `due_from` e `due_to` e ordena do vencimento mais tarde para o mais cedo. `pending_only: true` descarta as concluídas depois de aplicar o `limit` (padrão 50, máximo 200): restrinja o período para não perder pendências.
3. `complete_activity` com o id da atividade. Desfazer só pela tela, então confirme qual é antes.
4. `add_deal_interaction` registra ligação, visita ou resposta do cliente no histórico da oportunidade e não envia nada ao cliente. Use `is_internal: true` para anotação interna.
5. `list_deal_interactions` vem do registro mais antigo para o mais novo e corta no `limit` (padrão 30, máximo 100). Para achar o último contato, peça o máximo.

## O que precisa de atenção (follow-up)

1. `crm_alerts`: oportunidades abertas com fechamento previsto para os próximos 7 dias, sem atualização há mais de 10 dias ou com a previsão vencida. Agrupe por tipo.
2. `list_activities` com `pending_only: true` e `due_to` no momento atual: tarefas atrasadas.
3. `list_deals` com `idle_days`: abertas paradas na mesma etapa há pelo menos N dias (cada item traz `dias_parado`).
4. Proponha o próximo passo de cada uma. Crie atividades ou mova etapas só depois do ok do usuário.

## Fechar e reabrir

- Ganha: `close_deal` com `outcome: WON` e, se houver, `notes`, que vão para o histórico. O contato passa a ser cliente automaticamente. Fechar não gera pedido (veja a seção seguinte).
- Perdida: `close_deal` com `outcome: LOST` exige `loss_reason_id`, um motivo de perda do funil da oportunidade. Os motivos de cada funil vêm em `loss_reasons` de `list_pipelines`: escolha pelo nome que o usuário disse.
- Reabrir: `reopen_deal` devolve a oportunidade aberta na mesma etapa e limpa o motivo de perda. Não cancela pedido ou orçamento já gerado nem tira a marca de cliente do contato.

## Gerar pedido ou orçamento

1. `get_deal` para conferir cliente e situação. Sem cliente, vincule com `update_deal` enquanto ela estiver aberta.
2. Confirme com o usuário a oportunidade, o cliente e o tipo de documento.
3. `convert_deal_to_order` com `id` e `tipo`: `ped` para pedido, `orc` para orçamento. Sem `tipo`, gera orçamento.
4. Cada chamada gera um documento novo. Chame uma vez só; se der erro, leia a mensagem antes de tentar de novo.
5. O documento leva o cliente e os itens do escopo comercial da oportunidade, cadastrados na tela. Sem itens no escopo, ele nasce vazio: inclua com `add_product_to_order` e `add_service_to_order` (`document_type` quote no orçamento).
6. Responda com o número (`order_number`). Para detalhes, `get_order` com o `order_id`.
7. Se a venda foi fechada, pergunte se fecha a oportunidade como ganha com `close_deal`. No orçamento, ela costuma seguir aberta.

## Previsão e resultados

- `crm_forecast`: oportunidades abertas de todos os funis, com valor bruto e ponderado pela probabilidade da etapa, por mês da previsão de fechamento, e as de maior valor ponderado. Oportunidade sem valor fica de fora; sem previsão, cai em "Sem previsão". Sugira completar com `update_deal`.
- `crm_dashboard`: últimos 12 meses, agrupados pelo mês em que a oportunidade foi criada (não pelo fechamento), com valores em aberto, ganhos e perdidos, principais responsáveis, etapas e motivos de perda.
- `list_deals`: `count` e `total_amount` somam só o que voltou (padrão 50, máximo 300). Para o total do funil, use as duas acima.
- Por responsável não há filtro pelo nome: liste com `limit` alto e separe por `responsible_name`.

## Automações do funil

1. `list_crm_automations`, com filtro opcional por `pipeline_id` e `status` (`draft`, `active` ou `paused`).
2. `set_crm_automation_status` com `status` `active` ou `paused`. Ativada, a automação age sozinha nas oportunidades do funil: cria tarefas, manda e-mail ao contato, move etapa e fecha como ganha ou perdida. Pausar interrompe todos os disparos dela.
3. Criar ou editar as regras de uma automação é só na tela.

## Prospecção: buscar empresas e salvar leads

1. `search_companies` busca na base externa, não nos cadastros. Use `state` (UF), `size_range` (`MEI`, `ME`, `EPP`, `MEDIO` ou `GRANDE`) e, para a atividade, `cnae_code` junto com `sector` em texto, porque cada fonte usa um. Nem toda fonte filtra por `city`: confira cidade e UF nos resultados.
2. `search_companies` recusa `cpf_cnpj`: nenhuma fonte busca por CNPJ. Para saber se a empresa já está no cadastro, use `search_people` com o CNPJ e `type` `customer,lead`.
3. Mantenha o padrão, que oculta quem já é cliente ou lead, e diga quantos foram ocultados (`ocultas_clientes` e `ocultas_leads`).
4. Lista vazia e `erros_do_provedor` preenchido: explique a mensagem. Costuma ser fonte de dados não conectada, que o usuário conecta na tela Prospectar, aba Fontes de dados.
5. Cada busca traz até `limit` empresas (padrão 20, máximo 100) e não tem página seguinte. Ela pode consumir créditos: não repita a mesma busca à toa.
6. Mostre nome, cidade e UF, atividade e se tem telefone ou e-mail, e pergunte quais salvar.
7. `save_companies_as_leads` com as empresas escolhidas exatamente como vieram, mantendo `cpf_cnpj` e `provider`. O CNPJ é o que evita duplicar: sem ele, a empresa é cadastrada de novo a cada chamada.
8. Para já abrir oportunidades, envie `create_deal: true` com `pipeline_id` e `stage_id` de `list_pipelines` e, se houver, `expected_value`. Só os leads novos ganham oportunidade; quem já existia entra em `ja_existiam`.
9. Responda com quantos foram salvos, quantos já existiam, quantas oportunidades foram criadas e os erros.
10. Revelar o contato de um sócio ou decisor não é feito pelas ferramentas, só na tela Prospectar.

## Confirme antes de chamar

- `convert_deal_to_order`: cria pedido ou orçamento, e cada chamada cria outro.
- `set_crm_automation_status` com `active`: e-mail que a automação mandar ao contato não volta.
- `save_companies_as_leads`: cadastra em lote e, com `create_deal: true`, abre oportunidades que disparam as automações ativas do funil escolhido.
- `close_deal`: ganha marca o contato como cliente, e isso fica mesmo se a oportunidade for reaberta.
- Criar oportunidades, mover etapas ou criar atividades em quantidade.

## Fora das ferramentas

Criar ou editar funil, etapa e motivo de perda, cadastrar o escopo comercial, trocar responsável, excluir oportunidade e editar ou reagendar atividade são feitos na tela. Diga isso em vez de procurar outro caminho.

## Como apresentar

- Identifique a oportunidade por número e título (`#12 Implantação Empresa X`), com cliente, etapa, valor em R$ e previsão de fechamento. Não mostre ids.
- Em listas, comece pelo resumo (quantas e valor total) e mostre as mais relevantes; ofereça o restante.
- Depois de uma alteração, diga em uma linha o que mudou.
