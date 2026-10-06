---
name: posto
description: Use quando o usuário falar do posto de combustível, como bomba, bico, tanque, frentista, automação da pista, troca de preço na bomba, abastecimento pendente, aferição, encerrante, fechamento de turno, medição de tanque, LMC, convênio de frota, veículo da frota ou fatura do convênio.
---

# Posto de combustível

Se o usuário falar de outro posto ou outra empresa, use `list_companies` e informe `company_id` nas chamadas.

## Regras da área

- Bico, bomba e tanque se ligam pelo número (o mesmo que a automação usa), não pelo id. O `id` para consultar, alterar ou excluir vem de `list_pump_nozzles`, `list_fuel_pumps` e `list_fuel_tanks`.
- Combustível é produto: o `product_id` vem de `search_products`. Se ele não existir, peça para cadastrá-lo no sistema com o código ANP antes do tanque e do bico; `create_product` não tem esse campo.
- Frentista, cliente do convênio e dono do veículo são pessoas: `person_id` e `person_name` vêm de `search_people`.
- O `query` das listagens busca pelo número (bicos e bombas), pelo nome (tanques e frentistas), pelo combustível (comandos), pelo cliente (convênios) e pela placa completa ou pelo cliente (veículos).
- Data sem hora vale no dia certo: nos períodos das listagens, `2026-09-21` vale o dia inteiro. Em `date` do LMC, em `applied_at` dos lacres e em `invoice_date` das entradas do LMC, mande só a data (`2026-09-21`).
- `list_fuel_pumps` e `list_fleet_vehicles` trazem só a quantidade de lacres, intervenções, combustíveis e motoristas; `get_fuel_pump` e `get_fleet_vehicle` trazem as listas completas. A lista enviada numa alteração substitui a gravada: para mudar um lacre, uma intervenção, um combustível ou um motorista, leia a lista inteira com o get correspondente e reenvie tudo, com o que já existia. Nunca mande lista parcial. As linhas do LMC seguem a mesma regra em `update_lmc_sheet`.
- A venda do abastecimento é feita no caixa (PDV da pista). Não venda combustível da pista com `create_order`: o abastecimento continuaria pendente e a nota sairia sem bico e encerrante.
- Convênio, aqui, é o de frota (`list_fleet_agreements`), não o de `list_partnership_members`.

## Confirme antes de executar

Diga o que vai acontecer (combustível, preço, bomba, bico, cliente, valor) e só chame depois do sim:

- `change_fuel_price` e `send_pump_command`: chegam à bomba de verdade.
- `mark_fueling_as_calibration`: não se desfaz.
- `invoice_fleet_agreement`: só volta atrás cancelando a fatura em Financeiro > Contas a Receber.
- `start_forecourt_pairing` e `unpair_forecourt_agent`: mexem na ligação da pista com o sistema.
- `delete_lmc_sheet`: a folha não volta.
- `delete_pump_nozzle`, `delete_fuel_pump`, `delete_fuel_tank`, `delete_pump_attendant`, `delete_tank_measurement`, `delete_fleet_agreement` e `delete_fleet_vehicle`: o registro vai para a lixeira. Com o módulo Lixeira, `list_deleted_records` (`module` `posto.bico`, `posto.bomba`, `posto.tanque`, `posto.frentista`, `posto.medicao`, `posto.convenio` ou `posto.veiculo`) e `restore_deleted_record` desfazem.

## Pista e automação

1. `get_forecourt_status` diz se a pista está funcionando: agente pareado e conectado, estado e encerrante de cada bico e abastecimentos pendentes. Consulte antes de troca de preço e comando.
2. Sem pareamento: `start_forecourt_pairing` devolve o código (`code`), de uso único e válido por 10 minutos (gerar outro invalida o anterior), e o endereço do sistema (`address`). O usuário digita os dois no agente instalado no computador do posto.
3. Troca do computador do agente: `unpair_forecourt_agent` antes de parear o novo. Até lá, nenhum abastecimento ou medição chega.
4. Desconectado é o computador do agente desligado ou sem internet. Os abastecimentos ficam guardados no concentrador e chegam quando ele volta.

## Troca de preço

1. Ache o combustível com `search_products` e veja em `list_pump_nozzles` os bicos dele: a troca vale para todos e é recusada se nenhum bico vende esse combustível.
2. Confirme o preço à vista, o a prazo (vazio fica igual ao à vista) e quando (vazio troca agora).
3. `change_fuel_price` com `product_id`, `cash_price`, `credit_price` e `apply_at`. Troca agora exige o agente conectado; a agendada é aceita com ele fora do ar e acontece quando ele voltar.
4. Volta um comando por bico; o agendado fica aguardando o agente até a hora marcada. Acompanhe com `get_pump_command` ou `list_pump_commands` (`query` com o nome do combustível): EXECUTADO é a bomba confirmando, e FALHOU traz o motivo (confira o preço na bomba e mande a troca de novo).
5. O cadastro do combustível só recebe o preço novo quando a bomba confirma, e o preço que `list_pump_nozzles` mostra vem desse cadastro. Não troque o preço só no produto (`update_product`): a bomba continuaria cobrando o antigo.

## Comandos na bomba

1. Confira bomba e bico com `list_fuel_pumps` e `list_pump_nozzles`.
2. `send_pump_command` com `type` e `pump_number`:
   - `preset` limita o próximo abastecimento de um bico e exige `nozzle`, `limit_by` (`value` em reais ou `liters`), `limit` e `price_level` (1 à vista, 2 a prazo);
   - `authorize`, `block`, `unblock` e `stop` valem para a bomba inteira, com todos os bicos.
3. O comando vale 60 segundos. Confira com `get_pump_command`: EXECUTADO, FALHOU (o motivo diz, por exemplo, que a automação não aceita pré-determinação por litros) ou EXPIRADO (o agente não executou e o comando não vale mais). Só mande de novo se o usuário pedir.

## Abastecimentos e aferição

- `list_fuelings` filtra por `status` (PENDENTE sem venda, EM_VENDA no caixa, VENDIDO, AFERICAO), `nozzle_number` e período; `get_fueling` mostra a venda em que ele saiu.
- Abastecimento com `bloqueio` (bico sem cadastro ou sem combustível) não vai para a venda: cadastre o bico com o combustível.
- Aferição (combustível devolvido ao tanque): ache o abastecimento PENDENTE pelo bico, pelos litros e pelo horário, confirme e chame `mark_fueling_as_calibration`. Só pendente vira aferição, e ela entra no LMC do bico.
- `reserve_fueling` só se o usuário pedir: a reserva não lança o abastecimento em venda nenhuma, só o tira da lista dos outros caixas. O `attendant_id` é a pessoa do frentista (`pessoaId` em `list_pump_attendants`, não o id do frentista) e só vale quando a bomba não identificou ninguém.

## Fechamento por encerrante (turno ou dia)

1. `fueling_closing_by_nozzle` com `date_from` e `date_to`: o dia (`2026-09-21`) ou os horários do turno (`2026-09-21T06:00:00-03:00`).
2. Por bico vêm os litros pelo encerrante, o vendido (litros e valor), a aferição, o pendente, o que está no caixa e a diferença entre o encerrante e o resto.
3. Diferença fora de zero e avisos (litros que passaram pelo encerrante sem abastecimento registrado, encerrante que virou, pendentes, bico sem cadastro): aponte o bico. Os horários dos avisos já vêm no horário local. Pendente no fim do turno é abastecimento ainda sem venda.
4. Litros e valor por combustível: some os bicos dele. Venda por frentista: `sales_by_seller` com o `pessoaId` de cada frentista, porque o frentista do primeiro combustível da venda vira o vendedor.

## Tanques e medições

- `list_fuel_tanks` dá id, número, combustível e capacidade.
- Régua: `create_tank_measurement` com `tank_id` e `volume` em litros, mais `water`, `temperature` e `level` (mm) se houver. Sem `date_time` vale agora; para outro horário, mande data, hora e fuso. O volume não passa da capacidade.
- `list_tank_measurements` (`tank_id`, período) traz as medições manuais e as do medidor automático.

## LMC (uma folha por dia e combustível)

1. Faça as folhas em ordem de dia: a abertura dos tanques e dos bicos vem da folha anterior do mesmo combustível. `list_lmc_sheets` (`product_id`, período) mostra os dias feitos e os totais.
2. O fechamento de cada tanque é a última medição do dia: confira em `list_tank_measurements` e, se faltar, lance a régua antes.
3. `fill_lmc_sheet` com `date` (`2026-09-21`) e `product_id` monta a folha sem gravar: `sheet` traz as linhas (`tanques`, `vendas` e `entradas`) e os totais, e `warnings` os avisos (sem folha anterior, tanque sem medição, encerrante que virou ou que não bate com a folha anterior). Mostre os avisos e resolva com o usuário antes de gravar.
4. Grave com `create_lmc_sheet` (`date` e `product_id`) passando as linhas da folha montada: `tanques` em `tanks`, `vendas` em `sales` e `entradas` em `entries`. Abertura e fechamento viram `opening` e `closing` (no tanque, o fechamento é `conciliacao`); `numeroBico` e `numeroTanque` viram `nozzle_number` e `tank_number`; `afericao` vira `calibration`; `viradaEncerrante` vira `rollover`; `numeroNf`, `dataNf` e `volumeRecebido` viram `invoice_number`, `invoice_date` e `volume`.
5. Confira: os totais da folha gravada (o sistema calcula) têm de bater com os da folha montada. Corrija com `update_lmc_sheet`; só as listas enviadas mudam, e cada uma vai inteira.
6. Se o usuário preferir, a tela do LMC tem o botão Gerar folhas do dia, que monta e grava as folhas de todos os combustíveis.
7. `print_lmc` com os `ids` das folhas e `report_type` vazio (folha do livro), `LmcPerdasSobras` ou `LmcAfericoes` devolve os dados do relatório, não o arquivo.

## Cadastro da pista

Ordem: tanques, bombas e bicos. Antes de criar, confira na listagem se o número já existe.

1. `create_fuel_tank` com `number`, `product_id` e `capacity` em litros.
2. `create_fuel_pump` com `number`. Para o SPED Fiscal do posto, informe também `manufacturer`, `model`, `serial_number` e `measurement_type` (`digital` ou `analogica`).
3. `create_pump_nozzle` com `number` (o da automação), `product_id`, `tank_number` (tanque cadastrado), `pump_number` (bomba cadastrada), `model` (obrigatório, de 2 a 20 caracteres) e `price_type` (`avista` ou `aprazo`).
4. Lacres e intervenções vão em `update_fuel_pump` (`seals`, `interventions`), com a lista inteira lida em `get_fuel_pump`. A intervenção leva número, data e hora, técnico com CPF válido e o encerrante de cada bico antes e depois. Registre no dia do reparo, senão o salto do encerrante parece venda.
5. Antes de excluir bomba ou tanque, veja em `list_pump_nozzles` se algum bico aponta para ele.

## Frentistas

- `create_pump_attendant` com `person_id` e `person_name`; cada pessoa tem um cadastro de frentista só.
- `pin` (4 a 6 números), `card` e `automation_id` (identificação na bomba) não repetem na empresa. Senha e cartão nunca voltam na consulta: `update_pump_attendant` mantém os gravados, e `remove_pin` ou `remove_card` apagam.
- Para afastar um frentista, `active` false basta: inativo não se identifica no caixa.

## Convênio de frota

1. Ache o cliente com `search_people` e veja em `list_fleet_agreements` se ele já tem convênio: é um por cliente.
2. A forma de pagamento (`list_payment_methods`) precisa lançar em contas a receber, como o crediário. No crediário, o cliente precisa de limite de crédito no cadastro, senão o caixa recusa a venda.
3. `create_fleet_agreement` com `person_id`, `person_name`, `payment_method_id`, `closing` (obrigatório: `monthly`, `weekly` ou `biweekly`), `closing_day` (`monthly`: dia 1 a 31; `weekly`: 1, segunda, a 7, domingo; `biweekly` fecha no dia 15 e no último dia e ignora o campo), `due_days` e as exigências da venda (`require_km`, `require_driver`, `issue_nfe`, `block_overdue_days`).
4. Veículos: confira a placa em `list_fleet_vehicles` e use `create_fleet_vehicle` com `person_id`, `person_name` e `plate`, mais `model`, `tag`, `fuels`, os limites e `drivers` (nome e CPF). Limite zero não limita; sem combustíveis ou motoristas na lista, qualquer um vale. Para mudar, `update_fleet_vehicle`, com as listas completas lidas em `get_fleet_vehicle` (o último km vem das vendas e não se altera).
5. O veículo entra na venda pelo caixa (bloco Frota do PDV da pista). `authorize_fleet_sale` com o `order_id` (de `list_orders`) diz se a venda passa, com os motivos da recusa, e quanto o veículo já abasteceu no dia e no mês.
6. A fatura sai sozinha no dia seguinte ao fechamento. Para antecipar, mostre o que está em aberto (`list_receivables` com o nome do cliente e `status` PENDENTE) e avise que entram todos os títulos de venda dele sem pagamento e sem boleto, não só os de combustível. Depois do sim, `invoice_fleet_agreement` com o id do convênio: vence hoje mais `due_days`, e `invoiced` false quer dizer que não havia nada em aberto.
7. A fatura é uma conta a receber (`receivable_id`). Para cobrar, siga com `send_receivable_payment_link` ou `generate_receivable_boleto`.

## Como apresentar

- Litros e encerrantes com 3 casas (1.234,567 L), reais com 2 (R$ 1.234,56) e preço por litro com 3 (R$ 6,299/L).
- Horários dos registros voltam em UTC (terminam em Z): mostre no horário de Brasília, com datas como 21/09/2026. Os avisos de `fueling_closing_by_nozzle` já vêm no horário local.
- Fale de bico, bomba e tanque pelo número, de veículo pela placa e de pessoa pelo nome. Id só se pedirem, e nunca mostre JSON.
- Situações em português simples: pendente, no caixa, vendido, aferição; comando aguardando o agente, enviado, executado, falhou (com o motivo) ou expirado.
- Fechamento e LMC em tabela curta, por bico ou por dia, com a diferença e as perdas e sobras em destaque. Perdas e sobras é o medido menos o escritural: negativo é perda.

## Erros comuns

- "agente de automação da pista está fora do ar": computador do agente desligado ou sem internet; a troca de preço agendada continua possível.
- "concentrador desta pista não recebe comandos": a automação instalada não aceita comandos; preço e comando são feitos no painel do concentrador.
- "A licença cobre N bicos": para cadastrar mais, é preciso contratar bicos adicionais.
- "Já existe a folha": o dia já tem folha desse combustível; corrija a existente com `update_lmc_sheet`.
- "já tem convênio": altere o convênio do cliente com `update_fleet_agreement`.
- Sem permissão: diga qual falta (Comandar bombas e Parear agente, em Posto/Automação; Registrar aferição, em Posto/Abastecimento; Faturar, em Posto/Convênio). Quem libera é o administrador.
