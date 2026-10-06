# Sigeflex

Opere o ERP Sigeflex conversando com o seu assistente de IA: consulte vendas, pedidos e contas, emita notas fiscais, gere boletos, acompanhe o funil do CRM, monte campanhas e posts, confira estoque, produção e a pista do posto.

O plugin junta o conector do sistema (`https://app.sigeflex.com/mcp`) com instruções de uso por área (vendas, notas fiscais, financeiro, CRM, marketing, estoque e produção, posto), para o assistente seguir o fluxo certo de cada operação. Ele não traz código executável: são só o endereço do conector e os textos das instruções.

A conexão com assistentes externos está disponível para empresas no plano Empresarial.

## Instalar

- **Claude Code**: `/plugin marketplace add linksoft-dev/sigeflex-plugin` e depois `/plugin install sigeflex@sigeflex`
- **Só o conector, sem as instruções**: na tela IA > IA externa do sistema há botões e comandos prontos para Claude, ChatGPT, Cursor, VS Code, Claude Code, Codex e Gemini CLI

## Como usar

1. Instale o plugin e conecte o conector **Sigeflex**.
2. Entre com o seu usuário do sistema, escolha a empresa padrão e marque o que o assistente pode fazer em cada área.
3. Peça em português, por exemplo: "quanto tenho a receber esta semana?", "quais oportunidades estão paradas há mais de 15 dias?", "emite a nota do pedido 1520".

Quem tem acesso a mais de uma empresa pode pedir para trabalhar em outra do plano Empresarial a qualquer momento ("agora na Loja Centro"), sem reconectar.

## Segurança e dados

O assistente age em nome do usuário que autorizou a conexão, limitado às áreas liberadas na autorização e às ferramentas ligadas pela empresa. Ações que alteram ou apagam dados pedem a sua confirmação no assistente antes de rodar. A conexão pode ser revogada a qualquer momento na tela IA > IA externa do sistema.

Os dados da conversa necessários para cada operação (cliente, produto, valores, datas) são enviados apenas ao servidor do sistema em app.sigeflex.com, com a autorização do usuário. O plugin não guarda nada por conta própria e não fala com nenhum outro serviço.

- Política de privacidade: https://sigeflex.com/politica-de-privacidade
- Termos de uso: https://sigeflex.com/termos-de-uso
- Suporte: https://sigeflex.com/suporte

## Estrutura

- `.claude-plugin/plugin.json`: manifesto do plugin
- `.claude-plugin/marketplace.json`: catálogo para instalar pelo Claude Code
- `.mcp.json`: endereço do conector
- `skills/`: uma instrução por área do sistema
