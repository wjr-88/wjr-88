# FinPilot · Prompt único: auditoria, melhorias e licenças

> Cole este prompt inteiro na conversa do FinPilot no app do Claude **no notebook**, com a pasta do projeto acessível. Ele tem **duas etapas**. Faça a **Etapa 1** completa e me entregue. A **Etapa 2** começa só quando eu disser "pode seguir para a Etapa 2", e ela também para na proposta para eu aprovar antes de implementar.

---

Você vai trabalhar no **FinPilot**, meu PWA de finanças pessoais que já está em produção. O código está na pasta `Control Finance App\finpilot` do meu notebook.

- **Etapa 1:** uma **auditoria geral** do app e **cinco melhorias** (puxar para atualizar, disparo manual quando a automação falha, crédito como padrão, triagem de compra negada e diagnóstico da automação).
- **Etapa 2:** transformar o app num produto vendável por licença: acesso, planos, pagamento e gestão.

## Regras que valem para as duas etapas

- **Git: não mexa em configuração nenhuma.** Nada de `git config` (nem global, nem local), `git remote`, `git push`, credenciais ou hooks. Meu Git está configurado para o trabalho e não pode ser alterado. Pode fazer **commit local** na pasta do projeto, como nos commits anteriores, sem mudar autor nem configuração. Se o commit exigir mudar alguma configuração, não faça: me avise e deixe os arquivos salvos.
- **Não publique e não contrate nada.**
  - Nada de `firebase deploy`. No fim, me passe os comandos para eu rodar.
  - Não crie conta, produto ou preço em provedor de pagamento e não ative plano pago do Firebase. Liste o que eu preciso fazer.
- **Meus dados reais:**
  - Não apague nada, não zere nada e não rode migração destrutiva.
  - Todo campo novo é opcional e compatível com o que já existe.
  - O que for ambíguo vai para a lista "Revisar", como já fazemos.
  - Eu sou o **dono** do app: continuo com acesso total e sem cobrança, e nada meu pode ficar inacessível durante as mudanças.
- **Visual:** mantenha a identidade aprovada (base verde quase preta, vidro fosco, Geist, verde só em valores positivos, champanhe só em detalhes nobres). Nada de cores ou componentes fora do sistema de tokens que já existe. Isso vale também para planos, checkout, paywall e admin.
- **Qualidade:**
  - Tudo que for regra de negócio ganha teste no Vitest.
  - `npm test`, lint e `npm run build` têm que passar antes do commit.
  - Confira as telas que mudarem em largura de celular (390 px) e de computador.
- **Escopo:** não reescreva o que funciona.
- **Atalhos do iPhone:** toda mudança em atalho tem que estar documentada no passo a passo dentro do app e listada na sua entrega. Sem Toque Traseiro e sem Botão de Ação.

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

# ETAPA 1 · Auditoria geral e melhorias

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
  - virada de mês e de ano;
  - pop-up de **compra negada**, aviso de fatura e aviso de limite chegando pela automação (ver 2.4): nenhum pode virar gasto.
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

### 2.2 Disparo manual quando a automação falha

**Como funciona hoje:** chega o pop-up (notificação) do BB e a automação "Pegar Valor BB" dispara sozinha. Às vezes ela não dispara (hoje de manhã três compras não entraram). Quando eu rodo o atalho à mão pelo ▶, ele não tem notificação de entrada e manda um item vazio, sem valor nem nada.

**O que eu quero:**
- Quando falhar, deixo o pop-up do BB na Central de Notificações, rodo um atalho **à mão** e ele lança aquela compra.
- **Sem Toque Traseiro e sem Botão de Ação.** Eu rodo pelo app Atalhos ou por um widget.
- Em paralelo, **corrigir a causa das falhas** (ver 2.5), para o manual ser exceção.

**Limitação do iOS que você precisa respeitar (confirme antes de desenhar):** um atalho **não consegue ler uma notificação que já chegou**. Não existe ação "ler notificações da Central". Ele só recebe o texto da notificação no momento em que ela dispara a automação. Além disso, quando eu abro o app Atalhos para rodar à mão, a tela passa a ser o Atalhos, então uma captura de tela automática pegaria o Atalhos, não o pop-up. Por isso, "rodar à mão e ele lê sozinho o pop-up" não é possível do jeito literal. O desenho abaixo chega o mais perto disso.

**Solução esperada: um atalho manual, "FinPilot · Lançar compra", com dois caminhos**

1. **Pelo print do pop-up (lê o que está lá):**
   - Eu abro a Central de Notificações com o pop-up do BB à vista e tiro um print (botão lateral + volume).
   - Na miniatura do print, toco em **Compartilhar → FinPilot · Lançar compra**.
   - O atalho aceita **imagens pela Folha de Compartilhamento**, roda **Extrair Texto da Imagem** (leitura no próprio aparelho, sem servidor) e envia com `origem: "print"`.
   - Funciona também com um print da tela de detalhe da compra no app do BB.
   - Na limpeza do texto, troque quebra de linha por ` | ` em vez de espaço: em texto lido de imagem, a linha separa informação.
2. **Sem imagem (rodado direto pelo app Atalhos ou por widget):**
   - Pede **Valor** (teclado numérico) e **Onde** (texto curto), com a data e hora de agora.
   - Envia com `origem: "manual"`. Leva menos de 10 segundos.
   - Monte com a ação **Se**: "Se Entrada do Atalho tem algum valor" → caminho 1; senão → caminho 2.
3. **No app, um leitor de texto de print**, separado do leitor de notificação, porque o texto vem bagunçado, com várias linhas e vários valores. Ele tem que:
   - escolher o valor **da compra** e ignorar saldo, limite disponível, limite total, valor da fatura e cashback;
   - achar a loja e a **hora que aparece no pop-up** ("há 2 h", "09:41", "ontem"), não a hora do envio;
   - quando houver vários pop-ups do BB no mesmo print, lançar cada um, sem duplicar o que já entrou pela automação (regra de duplicidade que já existe);
   - ter testes com textos realistas de leitura de imagem, incluindo ruído (`R$1.234,56`, `RS 45,90` lido errado, vários valores na mesma tela).
   - Peça-me prints reais para calibrar, mas entregue já funcionando com os formatos comuns.
4. **O "Pegar Valor BB" rodado à mão** (sem notificação de entrada) não pode mais mandar item vazio: ele deve chamar o "FinPilot · Lançar compra" (ação **Executar Atalho**).
5. **No app, nunca mais criar item vazio.**
   - Envio sem texto aproveitável vira um aviso "Recebi um envio sem conteúdo — teste?", descartável com um toque.
   - Nunca vira lançamento nem "Para completar" vazio.
   - Guarde sempre o **texto bruto** recebido, visível em "Para completar" e no Diagnóstico.
6. **Guia dentro do app (Configurações → Lançamento automático):**
   - acrescente o "FinPilot · Lançar compra" no mesmo formato do passo a passo atual: uma ação por tela, botão Copiar e o desenho de como a ação fica;
   - inclua como ligar "Mostrar na Folha de Compartilhamento → Imagens" e como pôr o atalho num widget na tela de início;
   - inclua "como lançar quando a automação falhar": print do pop-up → Compartilhar → FinPilot.
   - Lembrete do que aprendemos: colar o modelo inteiro com marcadores (`XTITULOX`, `XMSGX`, `XDATAX`) e trocar com ações Substituir **funcionou**. Editar o meio do texto no iPhone **não funcionou**.
   - Toda mudança em atalho do iPhone tem que estar documentada nesse passo a passo **e** listada na sua entrega final.
7. **Avalie e me diga** se dá para distribuir os atalhos por **link do iCloud com "Perguntas de importação"**, que pediriam a chave e o endereço na instalação em vez de eu montar ação por ação. Se der, o guia passa a ser "1. Toque no link 2. Cole a chave". Explique o que eu preciso fazer uma vez para gerar os links (você não consegue gerar daqui).

### 2.3 Crédito como padrão nos lançamentos do pop-up

**Minha situação:** tenho **um único cartão de crédito** (Banco do Brasil, Visa, final 5720). Algumas compras que chegaram pelo pop-up entraram como **Pix**, e estavam erradas.

**Regra que eu quero:**
- Lançamento que vem do pop-up do BB (notificação, print ou manual) é **compra no cartão de crédito** por padrão, nesse cartão, como **a pagar** (entra na fatura, não mexe no saldo da conta).
- Só é **Pix** se o texto disser Pix de forma explícita ("Pix enviado", "Pix recebido", "Pix realizado"). Só é **débito** se disser débito de forma explícita.
- **Antes de mexer, investigue:**
  - Por que as compras viraram Pix? Pegue os textos brutos dessas compras e descubra a causa: regra antiga (já houve a correção "sem meio informado → crédito", ver se está publicada), palavra que confundiu o leitor ou outra coisa.
  - Corrija a causa com teste usando esses textos.
- **Lançamentos já gravados errado:** liste em "Revisar" os que vieram do pop-up como Pix sem a palavra Pix no texto bruto, com o botão **"Mudar para crédito"** (um a um ou todos). Nada muda sem eu confirmar.

**Cartão padrão (para quando houver mais de um cartão no futuro e para quem comprar o app):**
- Campo **"Cartão padrão para compras automáticas"** em Configurações → Lançamento automático. Com um único cartão, ele já vem preenchido com esse cartão.
- Ordem de decisão:
  1. final do cartão no texto, se casar com um cartão cadastrado;
  2. único cartão de crédito ativo;
  3. cartão padrão;
  4. "Para completar", com o motivo.
- Se o cartão padrão for excluído, o app volta para "sem padrão" e avisa. Nunca grava num cartão inexistente.
- O lançamento mostra de onde veio o cartão ("cartão padrão" ou "pelo final 5720").
- Campo novo opcional, por exemplo `config.cartaoPadraoId`. Testes para cada passo da ordem e para a regra "crédito por padrão, Pix só explícito".

### 2.4 Compra negada e outros avisos do BB que não são gasto

**Risco:** o filtro do iPhone é só "contém R$". Um pop-up de **compra negada** ("compra não aprovada", "transação recusada", "não autorizada", "saldo/limite insuficiente") tem "R$" e hoje pode virar gasto. O mesmo vale para outros avisos com valor que não são compra.

**Decisão de desenho:** a triagem fica **no app**, não no atalho. O atalho continua mandando tudo que tiver "R$", e eu não preciso refazer nada no iPhone.
- No app a regra tem teste, aparece no Diagnóstico e pode ser corrigida sem eu mexer no celular.
- Se você concluir que **algo precisa mudar no atalho**, a mudança tem que entrar no passo a passo do app (2.2, item 6) e na entrega final, com o passo exato.

**O leitor tem que classificar cada aviso, antes de qualquer lançamento, em:**

| Tipo de aviso | Exemplos de texto | O que o app faz |
|---|---|---|
| Compra aprovada | "compra aprovada", "compra realizada" | Lança no crédito (2.3) |
| **Compra negada** | "não aprovada", "negada", "recusada", "não autorizada", "bloqueada", "insuficiente", "tentativa de compra" | **Não lança.** Registra no Diagnóstico como "Ignorado: compra negada" e mostra um aviso discreto: "Compra negada de R$ X em LOJA — não lancei" |
| Compra cancelada ou estornada | "cancelada", "estorno", "devolução" | Estorno ligado à compra original, se achar; senão vai para "Revisar" |
| Fatura | "fatura fechou", "fatura disponível", "vence em", "pagamento da fatura recebido" | Não lança gasto. Pode atualizar ou confirmar a fatura, nunca somar como compra |
| Limite ou saldo | "limite disponível", "saldo" | Ignora |
| Pix enviado ou recebido, débito | palavras explícitas | Lança como Pix ou débito na conta padrão |
| Não reconhecido | qualquer outro | "Para completar", com o texto bruto. **Nunca** vira gasto por suposição |

- Em caso de dúvida entre aprovada e negada, **não lança**: manda para "Para completar".
- Testes obrigatórios: um por linha da tabela. Inclua o caso em que a mesma compra gera "negada" e, minutos depois, "aprovada" (lança uma vez só) e o caso "aprovada" seguida de "cancelada".
- **Na auditoria (Parte 1)**, inclua um achado específico: o que o leitor atual faz hoje com cada tipo da tabela, com o resultado de cada teste antes da correção.
- Peça-me prints reais de compra negada e de aviso de fatura do BB, se eu tiver, para calibrar.

### 2.5 Diagnóstico da automação

Para eu descobrir sozinho por que uma compra não entrou, crie em Configurações → Lançamento automático uma área **"Diagnóstico"** com:
- o último item recebido de cada origem (notificação, print, manual, Siri, WhatsApp), com data e hora;
- os últimos 20 envios recebidos com o texto bruto e o que o app fez com cada um (lançado, para completar, duplicado, vazio, ignorado como compra negada, fatura ou limite);
- uma lista de verificação do iPhone, com links das telas de Ajustes:
  - **dados celulares do Atalhos** (foi a causa de uma falha);
  - **Foco** silenciando o BB;
  - **Resumo Agendado** ou resumo por IA alterando o texto;
  - permissão "Sempre permitir" para `firestore.googleapis.com`;
  - "Confirmar antes de executar" desligado;
- um botão "Enviar teste" que mostra o endereço e um JSON de exemplo para eu testar no atalho.

---

## Entrega da Etapa 1 (responda assim e pare)

1. **Resumo** em até 10 linhas do que foi feito.
2. **Link ou caminho do `AUDITORIA.md`** e os 5 achados mais graves.
3. **O que mudou para mim no app**, tela por tela, em linguagem simples.
4. **Campos novos no banco** (todos opcionais) e a confirmação de que nenhum dado foi apagado.
5. **Testes:** quantos passam e quais foram adicionados.
6. **O que eu preciso fazer no iPhone**, passo a passo: atalho novo e qualquer mudança no "Pegar Valor BB". Se nada mudar no atalho atual, diga isso explicitamente.
7. **Comandos para publicar:**
   ```
   npm install
   npm test
   npm run build
   npx firebase deploy --only hosting
   ```
   Adicione `,firestore:rules` se as regras mudaram, e diga explicitamente se mudaram.
8. **Decisões pendentes**, com a sua recomendação.

---

# ETAPA 2 · Acesso, licenças, assinaturas e pagamento

> Só comece quando eu disser "pode seguir para a Etapa 2". Use o que a auditoria da Etapa 1 encontrou.

Quero transformar o **FinPilot** num produto que eu possa **vender por licença**. Outras pessoas entram com a conta Google delas, escolhem um plano, pagam, e eu faço a gestão de tudo num painel de administrador. Hoje isso é embrionário: existe login com Google, um item **"Meu plano"** (o antigo "Assinatura" do app) e uma seção **Admin**.

**Regra extra desta etapa, segurança em primeiro lugar:** quem decide se alguém tem acesso é o **servidor** (regra do Firestore ou Cloud Function). A interface nunca decide sozinha. Status de assinatura nunca pode ser gravado pelo cliente. A tela de planos é a vitrine do produto e tem que transmitir a sensação de "feito com cuidado, por quem conhece".

## Fase 0 · Diagnóstico do que existe (antes de propor)

Leia e me descreva em poucas linhas:
- como funciona hoje o login;
- o que é o "Meu plano" e o que é o "Admin";
- como o Admin é protegido: só na interface ou também na regra do Firestore;
- se existe algum campo de plano ou status no usuário;
- o que as regras do Firestore permitem;
- como a **caixa de entrada da automação** e o **robô do WhatsApp** identificam o usuário. Hoje o robô aceita só os números de `WHATSAPP_PERMITIDOS` e grava na minha conta, ou seja, não é multiusuário;
- o que já depende de serviço pago por uso: Claude API no robô, Groq e, no futuro, Pluggy/Open Finance, que tem custo alto por usuário.

## Fase 1 · Proposta (me entregue isto e PARE para eu aprovar)

Gere **`PROPOSTA_LICENCAS.md`** com:

### 1. Modelo de planos
Proponha planos, preços de referência em R$ (justificados pelo custo e pelo mercado de apps de finanças no Brasil) e o que cada um libera. Ponto de partida para discutir:

| Plano | Para quem | Inclui |
|---|---|---|
| Teste grátis (ex.: 14 dias) | Novo usuário | Tudo do Pessoal |
| Pessoal (mensal e anual com desconto) | 1 pessoa | App completo, atalhos do iPhone, assinaturas, metas, ajuda financeira |
| Premium | Quem quer automação total | + robô do WhatsApp (áudio e print), limites maiores de IA |
| Família | Casal ou família | Contas compartilhadas, vários usuários num mesmo espaço |
| Fundador ou vitalício (vagas limitadas) | Primeiros clientes | Pessoal ou Premium para sempre |
| Cortesia | Quem eu escolher | Concedido no Admin, com prazo ou sem |

- Diga quais recursos têm **custo variável** por usuário (IA, WhatsApp, Open Finance) e como limitar cada um por plano (cota mensal, por exemplo).
- Diga o que acontece quando a assinatura **vence**:
  - período de carência (ex.: 7 dias);
  - depois, **modo somente leitura**: o usuário vê e exporta os dados, mas não lança;
  - **nunca apagar dados** por falta de pagamento.

### 2. Provedor de pagamento
Compare **Stripe**, **Mercado Pago**, **Asaas** e **Pagar.me** para o meu caso:
- pessoa física ou MEI no Brasil;
- assinatura recorrente no cartão;
- Pix (inclusive Pix recorrente ou Pix Automático, se o provedor já tiver);
- boleto;
- taxas;
- webhook confiável;
- portal do cliente para trocar cartão e cancelar;
- emissão de nota fiscal ou integração com NFS-e;
- facilidade com Firebase.

Recomende **um** e justifique. Explique o que muda se eu ainda não tiver CNPJ.

### 3. Arquitetura

**Estrutura de dados:**
- `licencas/{uid}`: plano, status (`trial`, `ativa`, `em_atraso`, `cancelada`, `expirada`, `cortesia`, `vitalicia`), início, fim do período, fim do teste, renovação automática, id do cliente e da assinatura no provedor, origem (pagamento, cortesia, cupom) e histórico.
- **Somente o servidor escreve** em `licencas/{uid}`. As regras do Firestore negam qualquer escrita do cliente.

**Papéis:**
- `dono`, `admin`, `suporte` e `usuario`, guardados em custom claims ou numa coleção protegida.
- As regras do Firestore checam o papel. Esconder o botão não basta.

**Cloud Functions:**
- `criarCheckout`;
- `webhookPagamento`, com verificação de assinatura do webhook e idempotência pelo id do evento: o mesmo evento duas vezes não pode dar duas ativações;
- `portalCliente`;
- tarefa agendada que expira testes e assinaturas vencidas;
- funções do Admin (conceder cortesia, estender teste, revogar, reembolsar).
- Explique que isso exige o **plano Blaze** do Firebase, e quanto deve custar no meu volume.

**No app:**
- leitura da licença com cache offline;
- bloqueio com **paywall elegante**, nunca tela de erro;
- marcação de recursos por plano num lugar só (por exemplo `recursos.js`), sem `if` espalhado pelas telas.

**Isolamento entre usuários:**
- Revise as regras para garantir que ninguém lê ou grava dados de outro.
- A **caixa de entrada da automação** de cada usuário tem chave própria.
- O **robô do WhatsApp** passa a mapear `número → uid`, com verificação do número por código enviado no próprio WhatsApp.

**Limite de 1 MiB por documento:** com muitos usuários isso vira problema de verdade. Use o que a auditoria apontou e diga se a migração para subcoleções tem que vir **antes** da venda.

### 4. Jornada do cliente (desenhe cada tela)
1. Página de apresentação com os planos. Pode ser a tela inicial para quem não está logado.
2. Entrar com Google.
3. Início do teste grátis, sem cartão.
4. Onboarding, que já existe.
5. Avisos de fim de teste dentro do app (e por e-mail, se for viável), em D-3 e D-0.
6. Paywall e checkout.
7. Confirmação.
8. "Meu plano": plano atual, próxima cobrança, trocar plano, forma de pagamento, notas, cancelar (com pergunta de motivo e oferta de pausa).
9. Pagamento recusado: o que o usuário vê e como regulariza.

Ao mesmo tempo, para cada passo, diga o que **eu, como dono,** vejo no Admin.

### 5. Painel do administrador
- **Indicadores:** usuários totais e ativos, em teste, pagantes por plano, MRR, ARR, cancelamentos no mês (churn), conversão de teste para pago, receita do mês, inadimplentes, e custo estimado de IA e WhatsApp por usuário.
- **Lista de usuários:** busca e filtros por plano e status. Para cada um: e-mail, data de entrada, último acesso, plano, status, valor pago e uso dos recursos com custo.
- **Ações:** conceder cortesia (com prazo), estender teste, trocar plano, revogar, marcar como vitalício e reembolsar via provedor.
- **Toda ação do admin vai para um log de auditoria** (quem, o quê, quando, antes e depois), que não pode ser apagado.
- **Cupons e códigos de convite:** percentual ou valor, validade, limite de usos e planos válidos.
- **Suporte:** o admin **não** vê as finanças dos usuários. Se eu precisar acessar a conta de alguém, que seja só com consentimento registrado e com prazo.

### 6. Jurídico e LGPD (o mínimo para vender)
- **Termos de Uso e Política de Privacidade:** quais dados coleto, para quê, com quem compartilho (Firebase/Google, provedor de pagamento, Anthropic, Groq, Evolution).
- **Aceite registrado:** versão e data.
- **Direitos do usuário:** exportar todos os dados (JSON e CSV) e excluir a conta, apagando de fato os dados, com prazo.
- **Aviso claro:** o app não é instituição financeira e não faz recomendação de investimento. A Ajuda financeira já não indica produto específico.
- **Nota fiscal:** o que eu preciso como pessoa física versus MEI.
- Marque o que precisa de revisão de advogado. Não invente texto jurídico definitivo.

### 7. Plano de implementação
Divida em etapas pequenas e publicáveis. Para cada uma: o que entra, o que eu preciso fazer fora do código e como testar. Sugestão:
1. papéis e regras;
2. licença e teste grátis sem cobrança;
3. paywall;
4. pagamento em modo de teste do provedor;
5. Admin;
6. LGPD e exportação;
7. produção.

Inclua como testar tudo **sem dinheiro real**: emulador do Firebase, modo de teste do provedor e simulação de webhook (pago, recusado, cancelado, reembolsado, evento repetido e fora de ordem).

### 8. Riscos e perguntas para mim
Liste o que você precisa que eu decida: preços, nome dos planos, CNPJ, domínio próprio, e-mail de suporte e se a Família entra já ou depois.

## Fase 2 · Implementação (só depois do meu "aprovado")
- Siga o plano aprovado, uma etapa por vez.
- Cada etapa termina com testes passando, build ok, commit local e a lista do que eu preciso fazer.
- Regras de negócio da licença com testes no Vitest: transições de status, carência, expiração, idempotência do webhook e liberação de recursos por plano.
- Regras do Firestore com testes no emulador (`@firebase/rules-unit-testing`). Se o emulador não rodar aí, me diga e me passe o comando para eu rodar.
- Minha conta precisa estar marcada como `dono` antes de qualquer bloqueio entrar em vigor, para eu nunca ficar trancado para fora. Inclua um teste disso.
