# FinPilot · Auditoria geral + 4 melhorias

> Cole este prompt na conversa do FinPilot no app do Claude **no notebook**, com a pasta do projeto acessível.

---

Você vai trabalhar no **FinPilot**, meu PWA de finanças pessoais que já está em produção. O código está na pasta `Control Finance App\finpilot` do meu notebook. A tarefa tem duas partes, nesta ordem:

1. uma **auditoria geral** do app;
2. **quatro melhorias**: puxar para atualizar, disparo manual lendo a tela do BB, cartão padrão e diagnóstico da automação.

## Regras que valem para tudo

- **Git: não mexa em configuração nenhuma.** Nada de `git config` (nem global, nem local), `git remote`, `git push`, credenciais ou hooks. Meu Git está configurado para o trabalho e não pode ser alterado. Pode fazer **commit local** na pasta do projeto, como nos commits anteriores, sem mudar autor nem configuração. Se o commit exigir mudar alguma configuração, não faça: me avise e deixe os arquivos salvos.
- **Não publique.** Nada de `firebase deploy`. No fim, me passe os comandos para eu rodar.
- **Meus dados reais:**
  - Não apague nada, não zere nada e não rode migração destrutiva.
  - Todo campo novo é opcional e compatível com o que já existe.
  - O que for ambíguo vai para a lista "Revisar", como já fazemos.
- **Visual:** mantenha a identidade aprovada (base verde quase preta, vidro fosco, Geist, verde só em valores positivos, champanhe só em detalhes nobres). Nada de cores ou componentes fora do sistema de tokens que já existe.
- **Qualidade:**
  - Tudo que for regra de negócio ganha teste no Vitest.
  - `npm test`, lint e `npm run build` têm que passar antes do commit.
  - Confira as telas que mudarem em largura de celular (390 px) e de computador.
- **Escopo:** não reescreva o que funciona. Corrija o que a auditoria apontar como crítico ou alto e liste o resto.

## Contexto do que já existe

- **Stack:**
  - React 18 + Vite (PWA, `sw.js` com cache versionado), estilos inline com tokens do tema e recharts.
  - Firebase Auth (Google) + Firestore, com um documento por usuário em `users/{uid}` guardando listas (`transactions`, `bills`, `creditCards`, `invoicePayments`, `goals`, contas, etc.).
  - Vitest, com cerca de 136 testes.
  - Hosting no projeto `finpilot-bab89`.
- **Onde está cada coisa:**
  - telas principais em `App.jsx`, modais em `forms.jsx`;
  - regras em `src/finance/` (`consumo`, `natureza`, `recorrencia`, `contas`, `assinaturas`, `metas`, `dinheiro`, `faturas`, `fluxo`);
  - componentes em `src/components/`;
  - leitura de mensagens do banco em `integrations/mensagens.js`, conciliação em `integrations/conciliacao.js`;
  - onboarding em `onboarding.jsx`, importação em `import.jsx`.
- **Lançamento automático:**
  - Um atalho do iPhone ("Pegar Valor BB") dispara na **notificação do app BB** cujo Corpo contém `R$`.
  - Ele envia por POST, para a API REST do Firestore (caixa de entrada do usuário), o JSON abaixo. A regra do Firestore só aceita a criação quando `chave` confere.
    `{"fields":{"chave":…,"origem":"notificacao","titulo":…,"texto":…,"recebidoEm":"yyyy-MM-dd HH:mm"}}`
  - O app processa a caixa de entrada **quando está aberto**. Item completo vira lançamento. Item incompleto vai para "Para completar".
  - Há deduplicação: mesmo valor, data com até 3 dias de diferença, mesmo cartão.
  - Há conciliação com receitas e contas previstas ("é o mesmo?").
  - Notificação de compra sem meio de pagamento informado entra como **crédito a pagar**.
- **O que existe mas ainda não foi ligado:** o robô do WhatsApp (Evolution + n8n + Claude) e o Open Finance (Pluggy).

---

## PARTE 1 · Auditoria geral

Leia o projeto inteiro antes de mexer. Rode os testes e o build e use o app de verdade, em celular e computador, com dados de exemplo que cubram os casos abaixo. Gere o relatório em **`AUDITORIA.md`**, na raiz do projeto.

### O que auditar

**1. Cálculos financeiros**
- Confira em todas as telas que o mesmo número bate: KPIs do Início, painéis de Receitas e Despesas, gráfico de categorias e o detalhe da fatia, evolução, Mês a mês, Orçamento, Lançamentos (totais por mês), Cartões, Contas a pagar, Assinaturas, Patrimônio, projeção, "para usar até o fim do mês" e nota de saúde.
- Casos obrigatórios:
  - compra parcelada atravessando o fechamento;
  - compra feita **no dia do fechamento** e no dia seguinte (em qual fatura cai);
  - estorno;
  - pagamento parcial de fatura e fatura parcelada;
  - fatura "sem detalhar" mais as compras detalhadas depois;
  - Pix entre contas próprias (transferência);
  - receita prevista e depois realizada pela notificação;
  - assinatura anual (Smart TV R$ 50/ano);
  - conta a pagar quitada fora do app;
  - lançamento com data anterior ao saldo informado;
  - virada de mês e de ano.
- **Dinheiro:** procure soma de valores em ponto flutuante fora de `dinheiro.js` e arredondamentos inconsistentes.
- **Datas e fuso:** já houve um bug que gravava o dia seguinte depois das 21h. Procure qualquer outro uso de UTC, `toISOString()` ou `new Date('yyyy-mm-dd')` que desloque o dia.

**2. Integridade e modelo de dados**
- **Limite de 1 MiB por documento do Firestore.** Com tudo dentro de `users/{uid}`, estime em quantos meses de uso eu chego no limite. Proponha (sem executar) a migração para subcoleções, com plano de compatibilidade.
- **Escritas concorrentes:**
  - o que acontece se o celular e o computador gravarem ao mesmo tempo;
  - o que acontece se a caixa de entrada for processada em duas abas.
  - Procure gravação do documento inteiro que sobrescreve mudanças (perda de atualização). Avalie transação, `updateDoc` por campo ou `arrayUnion`.
- **Caixa de entrada:**
  - item processado é apagado ou marcado?
  - pode ser processado duas vezes?
  - o que acontece com item malformado?
- **Backup e exportação:** existe um jeito de eu baixar todos os meus dados? Se não existir, é recomendação alta.

**3. Segurança**
- **Regras do Firestore, linha a linha:**
  - um usuário consegue ler ou escrever dados de outro?
  - a caixa de entrada limita campos, tipos e tamanhos?
  - dá para criar sem chave?
  - dá para ler a chave?
- **Chave da automação:** tamanho, onde fica guardada, rotação, e o que dá para fazer com ela se vazar.
- **Admin:** quem consegue acessar, e a checagem é feita na regra do Firestore ou só na interface?
- **Texto vindo de notificação, print ou importação:** risco de injeção na interface (HTML ou markdown renderizado).
- **Segredos:** chaves no bundle ou no repositório; `.env` no `.gitignore`.

**4. PWA, desempenho e atualização**
- Ciclo de atualização do service worker:
  - o usuário recebe a versão nova sem precisar fechar o app duas vezes?
  - existe aviso de "nova versão disponível"?
- Bundle de 1,3 MB: proponha divisão de código com `import()` dinâmico por tela, `manualChunks` para recharts e firebase, e lazy load de telas pouco usadas. Implemente o que for seguro.
- Tempo até a primeira tela no iPhone e uso offline.

**5. Interface, usabilidade e acessibilidade**
- Navegue por **todos os menus** e liste:
  - textos que confundem;
  - botões que parecem estado (como o antigo "✓ Paga");
  - janelas cortadas;
  - estados vazios sem orientação;
  - fluxos com passos demais.
- Áreas de toque de pelo menos 44 px, contraste de pelo menos 4.5:1, navegação por teclado no computador e `aria-label` em botões só com ícone.

**6. Testes e qualidade do código**
- Regras sem teste, código morto (sobras do "E se?" e do Apple Pay) e duplicação de lógica entre telas.

### Formato do `AUDITORIA.md`

1. **Resumo executivo** em até 10 linhas: estado geral e os 5 riscos principais.
2. **Tabela de achados** com as colunas: ID, Área, Severidade (Crítico, Alto, Médio, Baixo), O que acontece, Como reproduzir, Arquivo:linha, Correção proposta, Status (Corrigido, Pendente, Precisa de decisão).
3. **Correções feitas** nesta rodada, com os testes adicionados.
4. **Decisões que dependem de mim**, cada uma com a sua recomendação.
5. **Roteiro sugerido** para os pendentes.

Corrija nesta rodada tudo que for **Crítico** ou **Alto**, desde que seja local e seguro. Mudança estrutural, como a migração para subcoleções, fica só proposta.

---

## PARTE 2 · Melhorias

### 2.1 Puxar para atualizar

Quero o gesto do Chrome no celular: na tela que está no topo, arrastar para baixo mostra um indicador e, ao soltar depois de um limite, o app atualiza.

- **Por que precisa ser feito à mão:** no iPhone, o PWA instalado na tela de início não tem esse gesto nativo. Ele tem que ser implementado no app, e precisa funcionar no PWA instalado, no Safari e no Chrome Android.
- **Comportamento:**
  - só começa se a área rolável estiver em `scrollTop === 0` e o arrasto for vertical para baixo;
  - o indicador acompanha o dedo com resistência;
  - o limite é de uns 70 px;
  - o ícone gira ao passar do limite;
  - feedback visual de "atualizando" e depois "atualizado", dentro do visual do app.
- **O que "atualizar" faz:**
  1. recarrega os dados do Firestore;
  2. processa a caixa de entrada da automação;
  3. procura versão nova do service worker. Se houver, ativa (`skipWaiting`) e recarrega a página.
  - Isso substitui o "feche e abra o app duas vezes" que eu faço hoje.
- **Não pode disparar:**
  - com modal ou gaveta aberta;
  - dentro de carrosséis ou listas com rolagem horizontal;
  - durante seleção de texto;
  - no computador com mouse. No computador, ofereça um botão de atualizar discreto.
- Use `overscroll-behavior-y: contain` onde for preciso para não brigar com o elástico nativo do iOS. Respeite `prefers-reduced-motion`.
- **Teste:** a lógica do gesto (limite, cancelamento, bloqueios) isolada num hook com teste unitário.

### 2.2 Disparo manual lendo a tela do BB

**Problema:** às vezes a automação da notificação não dispara (hoje de manhã três compras não entraram). Quando eu rodo o atalho manualmente pelo ▶, ele não tem notificação de entrada e manda um item vazio, sem valor nem nada.

**O que eu quero:** estou com a tela do app do BB aberta, mostrando a compra (detalhe da transação ou extrato do cartão). Disparo um atalho e o gasto entra no FinPilot com valor, loja, data e cartão.

**Solução esperada:**

1. **Novo atalho, "FinPilot · Ler tela"**, com as ações nesta ordem:
   - **Tirar Captura de Tela**;
   - **Extrair Texto da Imagem** (OCR no próprio aparelho, sem servidor);
   - limpeza com a mesma expressão `[\n\r"\\]` → espaço. Atenção: aqui a quebra de linha pode ser informação útil. Avalie trocar por um separador ` | ` em vez de espaço;
   - Formatar Data;
   - Texto com o modelo, com `origem: "tela"`;
   - Obter Conteúdo de URL (POST).

   Ele será disparado por **Toque Traseiro** (Ajustes → Acessibilidade → Toque → Toque Traseiro) ou pelo Botão de Ação.
2. **No app, um leitor de texto de tela (OCR)** separado do leitor de notificação, porque o texto vem bagunçado, com várias linhas e vários valores. Ele tem que:
   - escolher o valor **da transação** e ignorar saldo, limite disponível, limite total, valor da fatura e cashback;
   - achar a loja, a data e a hora da tela (e não a hora do envio);
   - achar o final do cartão, as parcelas ("em 3x", "parcela 1/3") e o tipo (compra, Pix enviado ou recebido, débito);
   - quando houver mais de uma transação na tela (extrato), pegar a **mais recente**, ou mandar todas para "Para completar" como lista para eu escolher. Proponha a melhor das duas;
   - ter testes com textos de OCR realistas, incluindo ruído (`R$1.234,56`, `R$ 45,90`, `RS 45,90` lido errado, valores negativos e o "limite disponível" na mesma tela).

   Peça-me prints reais para calibrar, mas entregue já funcionando com os formatos comuns.
3. **Atalho de notificação mais robusto:** quando o "Pegar Valor BB" for rodado à mão (sem notificação na Entrada do Atalho), em vez de mandar vazio ele deve:
   - cair no fluxo de captura de tela;
   - ou, se você achar melhor, perguntar o valor ("Pedir Entrada", número) e a descrição.
   - Monte isso com a ação **Se** do Atalhos: "Se Entrada do Atalho não tem nenhum valor".
4. **No app, nunca mais criar item vazio.** Payload sem texto aproveitável vira:
   - um aviso "Recebi um envio sem conteúdo — teste?", que eu descarto com um toque;
   - nunca um lançamento nem um "Para completar" sem nada.
   - Guarde sempre o **texto bruto** recebido, visível em "Para completar", para eu entender o que chegou.
5. **Guia dentro do app:**
   - acrescente o atalho "Ler tela" ao passo a passo de Configurações → Lançamento automático, no mesmo formato de uma ação por tela, com botão Copiar e o desenho de como a ação fica;
   - inclua a configuração do Toque Traseiro.
   - Lembrete do que aprendemos: colar o modelo inteiro com marcadores (`XTITULOX`, `XMSGX`, `XDATAX`) e trocar com ações Substituir **funcionou**. Editar o meio do texto no iPhone **não funcionou**.
6. **Avalie e me diga** se dá para eu distribuir os atalhos por **link do iCloud com "Perguntas de importação"**. Isso pediria a chave e o endereço na instalação, em vez de eu montar ação por ação. Se der, o guia passa a ter "1. Toque no link 2. Cole a chave". Explique o que eu preciso fazer uma vez para gerar os links (você não consegue gerar daqui).

### 2.3 Cartão de crédito padrão

**Problema:** quando a notificação não informa o cartão, o gasto fica sem cartão e eu preciso escolher toda vez.

**O que eu quero:**
- Em **Configurações → Lançamento automático**, um campo **"Cartão padrão para compras automáticas"**.
- Atalho para o mesmo campo na tela de **Cartões**, como "Definir como padrão", com uma marca visual discreta no cartão escolhido.
- Na mesma linha, **"Conta padrão para Pix e débito"**.

**Ordem para decidir o cartão de um lançamento automático:**
1. Final do cartão no texto, se casar com um cartão cadastrado.
2. Se só existir um cartão de crédito ativo, ele.
3. O **cartão padrão**.
4. Se nada disso resolver, "Para completar", com o motivo.

**Regras:**
- Pix enviado e débito usam a conta padrão, com a mesma lógica.
- Se o cartão padrão for excluído ou arquivado, o app volta para "sem padrão" e me avisa. Nunca grava num cartão inexistente.
- Os itens que já estão em "Para completar" por falta de cartão ganham o botão **"Usar cartão padrão em todos"**.
- O lançamento automático mostra de onde veio o cartão ("cartão padrão" ou "pelo final 5720"), para eu desconfiar quando precisar.
- Campos novos opcionais nas configurações do usuário (por exemplo `config.cartaoPadraoId`, `config.contaPadraoId`).
- Testes para cada passo da ordem de decisão.

### 2.4 Diagnóstico da automação

Para eu descobrir sozinho por que uma compra não entrou, crie em Configurações → Lançamento automático uma área **"Diagnóstico"** com:
- o último item recebido de cada origem (notificação, tela, Siri, WhatsApp), com data e hora;
- os últimos 20 envios recebidos com o texto bruto e o que o app fez com cada um (lançado, para completar, descartado como duplicado, vazio);
- uma lista de verificação do iPhone, com links das telas de Ajustes:
  - **dados celulares do Atalhos** (foi a causa de uma falha);
  - **Foco** silenciando o BB;
  - **Resumo Agendado** ou resumo por IA alterando o texto;
  - permissão "Sempre permitir" para `firestore.googleapis.com`;
  - "Confirmar antes de executar" desligado;
- um botão "Enviar teste" que mostra o endereço e um JSON de exemplo para eu testar no atalho.

---

## Entrega final (responda assim)

1. **Resumo** em até 10 linhas do que foi feito.
2. **Link ou caminho do `AUDITORIA.md`** e os 5 achados mais graves.
3. **O que mudou para mim no app**, tela por tela, em linguagem simples.
4. **Campos novos no banco** (todos opcionais) e a confirmação de que nenhum dado foi apagado.
5. **Testes:** quantos passam e quais foram adicionados.
6. **O que eu preciso fazer no iPhone**, passo a passo, para os atalhos novos.
7. **Comandos para publicar:**
   ```
   npm install
   npm test
   npm run build
   npx firebase deploy --only hosting
   ```
   Adicione `,firestore:rules` se as regras mudaram, e diga explicitamente se mudaram.
8. **Decisões pendentes**, com a sua recomendação.
