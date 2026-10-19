# FinPilot · Rodada 2: ajustes, gastos por período e planos (custo zero)

> Cole este prompt inteiro na conversa do FinPilot (no app do Claude no notebook, ou numa sessão do Claude Code com o repositório). São **duas etapas**. Faça a **Etapa A** completa, entregue e **pare**. A **Etapa B** começa só quando eu disser "pode seguir para a Etapa B".

---

## Regras que valem para tudo

- **Git:**
  - Não mexa em `git config` (global ou local), credenciais ou hooks. O Git global do notebook é do meu trabalho.
  - Na pasta do notebook, faça só commit local, como antes, sem `git push` e sem `git remote`.
  - Numa sessão do Claude Code com o repositório no GitHub, trabalhe na branch indicada pela sessão e envie para ela.
- **Não publique.** Nada de `firebase deploy`. No fim, me passe os comandos.
- **Custo zero:**
  - Nada que gere cobrança: não ative o plano Blaze do Firebase e não use Cloud Functions.
  - Não crie conta, produto ou preço em provedor de pagamento.
  - Não use API paga nova.
  - Tudo tem que funcionar no plano gratuito (Spark) do Firebase: Auth, Firestore e Hosting.
  - Se alguma coisa só for possível pagando, não implemente: deixe preparado e me explique.
- **Meus dados reais:**
  - Não apague nada, não zere nada e não rode migração destrutiva.
  - Campos novos são opcionais.
  - O que for ambíguo vai para "Revisar".
  - Eu sou o **dono**: acesso total sempre, sem plano e sem cobrança. Nenhuma mudança pode me trancar para fora; isso tem que ter teste.
- **Visual:** a identidade aprovada vale para tudo (base verde quase preta, vidro fosco, Geist, verde só em valores positivos, champanhe só em detalhes nobres). Use só os tokens que já existem.
- **Qualidade:**
  - Regra de negócio ganha teste no Vitest.
  - `npm test`, lint e `npm run build` passando antes do commit.
  - Confira as telas em 390 px e no computador.
- **Atalhos do iPhone:**
  - Qualquer mudança vai para o passo a passo dentro do app e para a sua entrega.
  - Sem Toque Traseiro e sem Botão de Ação.
- **Contexto vivo:**
  - Crie ou atualize um `CONTEXTO.md` na raiz do projeto com o que uma sessão nova precisa saber: stack, regras de negócio, automação, decisões tomadas (incluindo as desta rodada), pendências e como publicar.
  - Atualize esse arquivo no fim de cada etapa.

## Onde o projeto está

- **Stack:** React 18 + Vite (PWA), Firebase Auth (Google) + Firestore, com um documento por usuário em `users/{uid}`, e Vitest com 250 testes.
- **Já feito e publicado:** a auditoria (`AUDITORIA.md`), puxar para atualizar, triagem de avisos do BB (compra negada, fatura, limite), crédito como padrão, cartão padrão, diagnóstico da automação, "Baixar meus dados" e o atalho "FinPilot · Lançar compra".
- **Orçamento na virada do cartão** (commit `4d56332`): confira se está publicado. Se não estiver, me avise na entrega.
- **Planos e acesso hoje:**
  - `config/app` guarda `plans` (`free`, `premium`, `family`), `featureFlags` globais e `licenses` por e-mail.
  - O root é decidido pelo e-mail na lista `rootEmails` (`wjroliveira88@gmail.com`) e recebe o plano `family`.
  - **Problemas conhecidos:**
    - as características dos planos são só texto e não bloqueiam nada;
    - o próprio usuário troca o próprio plano (`handleChangePlan`) sem pagar;
    - `config/app` é legível por qualquer usuário logado (item M4 da auditoria).

## Minhas decisões da auditoria (não reabrir)

| Item | Decisão |
|---|---|
| M5 · chave nova da automação | Não gerar agora |
| M10 · fatura sem detalhar de outubro | Deixar como está |
| A3 · migração para subcoleções | Fazer no início da Etapa B |
| Link do iCloud para atalhos | Sim, para clientes (Etapa B) |
| M7 · chips menores que 44 px | Aumentar (Etapa A) |
| B4 · leitor do Apple Pay | Manter |
| B3 · `toISOString` em `mensagens.js` | Corrigir (Etapa A) |
| Compra feita no dia 14 | Continua como está até eu conferir no BB |

---

# ETAPA A · Ajustes e gastos por período

### A1. Ajustes pequenos da auditoria
- **M7:** chips de período (Investimento, Financiamento, gráfico 3m/6m), "Abrir" em Ajuda financeira e "Paguei" e "Cobrado" em Assinaturas. Altura visível de 36 px e área de toque de 44 px, sem mudar o desenho.
- **B3:** trocar o cálculo de "ontem" com `toISOString` em `src/integrations/mensagens.js` por `somarDias` (`recorrencia.js`), com teste.

### A2. Boleto pago pelo pop-up
- **Problema:** pop-up do BB de **pagamento de boleto** ou de conta (luz, internet) sai da **conta**, mas pela regra "crédito por padrão" pode entrar no cartão.
- **O que fazer:**
  - Reconhecer "boleto", "pagamento de título", "pagamento realizado", "débito automático" e convênios. Esses avisos vão para a **conta padrão**, como pagamento.
  - Ao entrar um pagamento ou Pix enviado que case com algo em **Contas a pagar** (valor parecido, perto do vencimento), perguntar "É a sua conta de X?" e, ao confirmar, marcar a conta como paga. É a conciliação que já existe para receitas.
  - **Caso real para testar:** paguei a minha internet por Pix. Não pode aparecer duas vezes (Pix lançado e conta ainda em aberto).
- **Testes:** um por tipo de aviso novo, mais o caso de conciliação com Contas a pagar.

### A3. Gastos por dia, semana e período livre
**O que eu quero:** saber quanto gastei hoje, nesta semana, no último fim de semana ou entre duas datas que eu escolher, e não só no mês.

**Onde:**
- No painel de **Despesas**, o que abre ao tocar no KPI de Despesas, e em **Lançamentos**.
- Um seletor de período no topo, no lugar do seletor de mês atual, que continua como uma das opções.

**Opções do seletor:**
- **Atalhos:** Hoje, Ontem, Esta semana, Semana passada, **Último fim de semana**, Últimos 7 dias, Este mês, Mês passado, Últimos 30 dias, Ciclo do cartão (de 15 a 14, usando a regra que já existe no Orçamento).
- **Agrupar por:** Dia, Semana, Mês.
- **Personalizado:** escolher data inicial e final num calendário. Opção de **"só estes dias da semana"** (marcar sáb e dom, por exemplo) para responder "quanto gasto nos fins de semana nos últimos 3 meses".

**Regras:**
- **Semana:** de segunda a domingo.
- **"Último fim de semana":** sábado e domingo mais recentes já terminados. Se hoje for sábado ou domingo, mostrar "este fim de semana (até agora)" e, ao lado, o anterior.
- **Data que vale:** a **data da compra** (competência), com a mesma regra central de `consumo.js`. Parcelas contam na data de cada parcela, como no resto do app. Pagamento de fatura e transferência ficam fora, como já é.
- **Coerência:** com o período = mês, o total tem que ser **igual** ao KPI do mês. Teste isso.

**O que mostrar:**
- total do período;
- comparação com o período anterior equivalente ("+18% que o fim de semana anterior");
- gráfico de barras por dia ou semana;
- top categorias e top lugares;
- lista das compras.

**Também:**
- Card no Início com "Gastos da semana" e um toque para "Último fim de semana".
- Lembrar o último período escolhido no aparelho.
- Tudo funcionando no celular (seletor em folha que sobe de baixo) e no computador.
- **Testes:** cálculo de cada atalho, inclusive virada de mês e de ano, semana que atravessa dois meses e "último fim de semana" num sábado, domingo e segunda.

### A4. Android sem custo
- **Compartilhar para o FinPilot:**
  - No Android (Chrome), o app instalado pode aparecer em **Compartilhar**. Use o `share_target` do manifesto do PWA.
  - O print do pop-up do banco compartilhado vai direto para o leitor de print que já existe.
  - Se o Chrome não permitir ler a imagem dentro do app, explique a alternativa.
- **Guia MacroDroid (plano grátis)**, dentro de Configurações → Lançamento automático, aba Android:
  - gatilho: notificação do app do BB contendo "R$";
  - ação: requisição HTTP POST para o mesmo endereço da caixa de entrada, com a chave e o texto, `origem: "android"`;
  - avisos sobre dar acesso às notificações e tirar o MacroDroid da economia de bateria;
  - se der para gerar um arquivo de macro para importar, ofereça o download.
- A caixa de entrada e as regras do Firestore passam a aceitar `origem: "android"`. Me diga se as regras mudaram.

### A5. Despesas da mais recente para a mais antiga
- **Onde:** em **toda** lista de despesas e lançamentos, a ordem padrão é a **data mais recente primeiro**:
  - Lançamentos e seus grupos por mês;
  - painel de Despesas e de Receitas;
  - detalhe da fatia ou categoria;
  - compras dentro da fatura em Cartões;
  - Últimos lançamentos no Início;
  - Para completar e Revisar;
  - a lista do novo seletor de período (A3).
- **Desempate:** no mesmo dia, pela **hora** (lançamentos automáticos têm hora do aviso) e, sem hora, pela ordem em que foram criados (o mais novo primeiro).
- **Telas que já têm opção de ordenar** (como o detalhe da categoria): continuam com a escolha, mas começam em "mais recente".
- **Teste:** uma função única de ordenação usada por todas as listas, com os casos de mesmo dia, com e sem hora.

### A6. Envios que se perdem no 4G e hora no Diagnóstico

**O que aconteceu (relato real):**
- Saí para fazer compras no **4G**.
- O app mostrou o aviso de que estava **sem acesso à nuvem** e salvando só no aparelho.
- As compras feitas pela **Siri** e pelo **pop-up do BB** não apareceram no app. A da Siri entrou **depois de um tempo**. A do pop-up não entrou.
- No Diagnóstico, o envio da Siri mostra **só a data, sem a hora**, então não consigo saber se o atraso foi no iPhone, no envio ou no app.

**1. Três horários em cada envio, no Diagnóstico e no texto bruto de "Para completar":**
- **Feito no iPhone:** o `recebidoEm` que o atalho manda.
  - Confira os modelos de **todos** os atalhos (Siri, notificação, "Lançar compra", Android). Todos têm que mandar `yyyy-MM-dd HH:mm`.
  - Investigue por que o da Siri chega sem hora: modelo ou guia da Siri com outro formato, ou o leitor descartando a hora. Corrija a causa e o guia da Siri dentro do app.
- **Chegou na nuvem:** a hora em que o Firestore gravou o item. Use o `createTime` do documento (vem de graça na API) ou um campo de hora do servidor validado pela regra. Esse é o horário confiável.
- **Lançado pelo app:** a hora em que o app processou o item.
- Mostre a diferença entre eles ("chegou 1 h 12 min depois de feito", "lançado 40 min depois de chegar"), para eu enxergar onde está o atraso.
- **Data do lançamento:** se o `recebidoEm` vier sem hora ou inválido, use a hora de chegada na nuvem, não a do processamento.

**2. Diagnóstico de rede quando o app está sem nuvem:**
- Quando o app mostrar "sem acesso à nuvem", diga **o motivo provável** e o que conferir no iPhone:
  - Ajustes → Celular: **Safari** e **Atalhos** ligados (o app instalado na tela de início usa a permissão do Safari);
  - **Modo Dados Reduzidos** desligado;
  - **Retransmissão Privada do iCloud**, VPN ou DNS bloqueador interferindo em `firestore.googleapis.com`;
  - sinal fraco.
- Ao voltar a conexão, o app sincroniza sozinho o que ficou só no aparelho e avisa ("3 alterações enviadas para a nuvem"). Confirme isso com teste.

**3. Atalho que não perde a compra quando falha o envio:**
- O Atalhos não tem "tentar de novo". Monte isto em **todos** os atalhos de envio, e documente no passo a passo do app:
  - Depois de **Obter Conteúdo de URL**, um **Se**: se a resposta não contém `"name"` (o envio falhou), então:
    - **Adicionar ao Arquivo de Texto**: grava o envio numa linha de `FinPilot/pendentes.txt`, no iCloud Drive ou em "No Meu iPhone";
    - **Mostrar Notificação**: "Não consegui enviar ao FinPilot — guardei para reenviar".
  - Atalho novo **"FinPilot · Reenviar pendentes"**: lê o arquivo, envia linha por linha, apaga as que deram certo e avisa quantas foram.
  - Automação **"Ao conectar ao Wi-Fi"** (ou em horários fixos) que roda o "Reenviar pendentes" sozinha.
- **Sem duplicar:** cada envio leva um `idEnvio` gerado no iPhone (data e hora com segundos mais um número aleatório). O app ignora um `idEnvio` repetido. Teste isso.
- Avise que essa mudança **exige mexer nos atalhos atuais** e diga exatamente quais ações acrescentar, no mesmo formato de uma ação por tela.

**4. Testes:**
- horários e diferenças no Diagnóstico;
- envio sem hora usando a hora da nuvem;
- `idEnvio` repetido ignorado;
- reenvio de pendentes fora de ordem.

**5. Investigar o lado do código (não só o iPhone):**
- Por que o app perde a nuvem no 4G se o resto da internet funciona? Verifique a conexão do Firestore em rede móvel:
  - o canal em tempo real (WebChannel) costuma falhar em algumas operadoras;
  - avalie `experimentalAutoDetectLongPolling` (ou `experimentalForceLongPolling`) na inicialização do Firestore;
  - confira os tempos limite, a persistência offline e a reconexão ao voltar do segundo plano.
- Reproduza com rede lenta ou instável (limitação de rede do navegador) e mostre o antes e o depois.
- Se a causa provável for configuração do iPhone, inclua no Diagnóstico um botão **"Testar conexão agora"**. Ele faz uma gravação e uma leitura de teste e mostra o resultado e o tempo, para eu rodar na rua no 4G.

### A7. Cartão padrão já preenchido nos formulários

**Problema:** ao lançar ou editar uma despesa e escolher **Cartão de crédito**, eu ainda preciso escolher o cartão, mesmo tendo um só (Visa final 5720).

**Regra:**
- Ao escolher a forma de pagamento **Cartão de crédito**, em lançamento novo **ou** edição, o campo do cartão já vem preenchido com o **cartão padrão** (`config.cartaoPadraoId`). Sem padrão definido e com um único cartão ativo, vem com esse cartão.
- **Com um único cartão**, o campo aparece compacto, como "Visa final 5720 · padrão", com um "trocar" discreto. Não é um seletor que obriga a escolher.
- **Na edição de um lançamento que já tem cartão**, mantém o cartão dele. O padrão só preenche quando o campo está vazio ou quando eu mudo a forma de pagamento para crédito.
- **Quando eu mudo o cartão padrão** (Configurações ou Cartões), todo formulário novo passa a vir com o novo padrão. Os lançamentos antigos não mudam.
- **Mesma regra em todos os lugares** onde se escolhe cartão: botão +, edição em Lançamentos, Assinaturas no cartão, "Completar" de um item da caixa de entrada e o "Mudar para crédito" do Revisar.
- **Teste:** uma função só, `cartaoInicial(forma, lancamento, config, cartoes)`, cobrindo novo, edição com cartão, edição sem cartão, um cartão, vários cartões com e sem padrão, e padrão excluído.

### A8. Campo de parcelas preso no "1" (bug)

**Sintoma:** ao editar uma compra no cartão, o campo **Parcelas** fica sempre com "1". Apagar não funciona, e digitar 3 vira "13".

**Causa encontrada:** em `forms.jsx`, no formulário de lançamento, o campo está assim:
```jsx
<Input label="Parcelas" value={f.installments || '1'} onChange={v => set('installments', v.replace(/\D/g, ''))} ... />
```
- Quando o campo é apagado, `installments` fica `''`.
- `'' || '1'` mostra "1" de novo, então o campo nunca fica vazio e cada dígito é acrescentado depois do 1.
- Confira se ainda está assim na versão atual.

**Correção:**
- O campo mostra exatamente o que foi digitado (`value={f.installments ?? ''}`), com `placeholder="1"`.
- O **1** só é assumido ao salvar, quando o campo estiver vazio. A validação já usa `parseInt(...) || 1`.
- **Limite de 1 a 48.** Fora disso, mensagem clara.
- **No celular:** botões **− e +** ao lado do número (área de toque de 44 px), mais atalhos rápidos **1x, 2x, 3x, 6x, 10x, 12x**.
- Mostrar embaixo o valor de cada parcela ("3x de R$ 38,59"), com a mesma divisão de centavos da fatura.
- Mudar as parcelas de uma compra já lançada recalcula as faturas dos meses seguintes, sem duplicar.
- **Procure o mesmo padrão `valor || 'padrão'` em outros campos numéricos** (valor, dia de vencimento, dia de fechamento, limite, metas) e corrija onde ele impedir de apagar o campo.
- **Testes:**
  - apagar e digitar 3 resulta em 3;
  - vazio ao salvar vira 1;
  - 0 e 49 são recusados;
  - editar uma compra de 1x para 3x muda as faturas dos 3 meses e a soma continua igual ao valor da compra.

### A9. Avisos da Carteira (Apple Pay) sempre caem em "Para completar"

**Sintoma:** compras pagas com Apple Pay chegam, mas **sempre** vão para "Para completar". As que vêm pelo pop-up do app do BB entram direto.

**Causa, pelos prints reais:** são **dois avisos diferentes**, de dois apps diferentes.

**Aviso 1 · app Carteira (Apple Pay):**
```
App: Carteira
Título: Banco do Brasil
Corpo:  Pão de Açúcar. Brasília, DF
        R$ 27,19
```
- O título é o **nome do cartão/banco** na Carteira, não do app.
- O corpo é **"Loja. Cidade, UF"** e o valor numa linha à parte.
- **Não tem** "compra", "aprovada", "final" nem "cartão". Por isso a triagem trata como "não reconhecido" e manda para "Para completar". A regra "na dúvida, não lança" está funcionando; falta reconhecer o formato.
- Depois da limpeza do atalho (quebra de linha → espaço), o texto chega como `Pão de Açúcar. Brasília, DF R$ 27,19`.

**Aviso 2 · app do BB (este já funciona):**
```
Compra no valor de R$ 385,58, realizada em PG *MAXLOG IMPORT às 14:10 do dia 19/10, com cartão final 5720. Caso não reconheça, clique em Bloquear Cartão. Conheça os benefícios do seu...
```

**O que fazer:**
1. **Como os avisos da Carteira chegam (confirmado por mim):**
   - Tenho **duas automações** de notificação, as duas rodando o **mesmo atalho** "Pegar Valor BB": uma com o app **BB**, outra com o app **Carteira**. As duas mandam `origem: "notificacao"`.
   - O app tem que distinguir os dois **pelo formato do texto**, não pela origem.
   - **Não há duplicidade na prática:** pagamento por aproximação com o celular gera só o aviso da **Carteira**; compra com o cartão físico ou na internet gera só o do **BB**. Mantenha a regra de duplicidade apenas como proteção.
   - **Suspeita principal: a loja não está chegando.** No aviso da Carteira, "Pão de Açúcar. Brasília, DF" provavelmente é o **subtítulo** da notificação, e o atalho só envia **Título** e **Corpo**. O app receberia só "Banco do Brasil" + "R$ 27,19", sem a loja.
     - **Confirme** pelo texto bruto no Diagnóstico.
     - Se for isso, **ajuste o atalho**: no modelo, acrescente o campo `subtitulo` (variável Entrada do Atalho → Notificação → **Subtítulo**, se o iOS oferecer) com mais um marcador `XSUBX` e mais uma ação Substituir. Documente no passo a passo do app, uma ação por tela.
     - Se o iOS não oferecer o Subtítulo, diga isso e use o caminho abaixo.
   - **Mesmo sem loja, não parar em "Para completar":** aviso com valor e título igual ao banco ou nome de um cartão cadastrado **lança** no crédito desse cartão, com a descrição "Apple Pay · Banco do Brasil" e uma marca discreta "sem loja — toque para nomear". Não bloqueia.
2. **Novo formato reconhecido, "aviso da Carteira":**
   - Quando o **título** for o nome de um cartão ou banco cadastrado ("Banco do Brasil", ou o nome/apelido do cartão) e o corpo tiver `Loja. Cidade, UF` + `R$ valor`, é **compra aprovada no crédito**.
   - **Cartão:** o que casa com o título (banco ou nome). Senão, o único cartão ativo. Senão, o cartão padrão.
   - **Descrição:** só a loja ("Pão de Açúcar"). Cidade e UF vão para um campo opcional `local`, não para a descrição.
   - Também vale para um título "Apple Pay" ou com o nome de outro banco cadastrado.
   - Avisos da Carteira que **não** são compra (Apple Cash, passes, cartões de transporte, "transação recusada") **não lançam**. Recusada segue a regra de compra negada.
3. **Aviso do BB, melhorias com o texto real:**
   - **Usar a data e a hora do texto** ("às 14:10 do dia 19/10") como data e hora da compra. Isso ajuda no A6: a hora passa a vir do próprio banco. O ano é o atual, ou o anterior se a data ficaria no futuro.
   - **Limpar prefixos de intermediador** na loja: `PG *` (PagSeguro), `MP *` (Mercado Pago), `PAG*`, `IFD*`, `EC *`, `SUMUP *`, `PAYPAL *`. "PG *MAXLOG IMPORT" vira "MAXLOG IMPORT". O texto bruto continua guardado.
   - Ignorar o rodapé ("Caso não reconheça…", "Conheça os benefícios…").
4. **Final do cartão diferente (Apple Pay):** no Apple Pay o cartão tem um número próprio do aparelho, com outro final. Para não travar quando isso aparecer:
   - cada cartão aceita uma lista de finais (`finais: ['5720', ...]`), editável em Cartões;
   - final desconhecido **nunca** bloqueia quando há um único cartão ou um cartão padrão: entra nele, com a nota "final novo".
5. **Sem duplicar:** com Apple Pay podem chegar **os dois avisos da mesma compra**, o da Carteira e o do BB. Mesmo valor, mesmo dia e horário próximo é uma compra só. Fica a versão mais completa (a do BB, com hora e final), e a outra é registrada no Diagnóstico como "duplicada".
6. **Itens já parados:** em "Para completar", botão **"Lançar todos os avisos da Carteira"**, que reprocessa com o leitor novo.
7. **Testes, com os textos reais acima como base:**
   - aviso da Carteira lança R$ 27,19 em "Pão de Açúcar", crédito, cartão BB, local "Brasília, DF";
   - aviso do BB lança R$ 385,58 em "MAXLOG IMPORT", 19/10 às 14:10, cartão final 5720;
   - Carteira + BB da mesma compra lançam uma vez só;
   - título da Carteira que não casa com nenhum cartão, com um único cartão, entra nele;
   - aviso de transporte ou Apple Cash não lança;
   - prefixos `PG *` e `MP *` limpos;
   - data "19/10" em janeiro vira o ano anterior.

### Entrega da Etapa A (responda assim e pare)
1. Resumo em até 10 linhas.
2. O que mudou para mim, tela por tela.
3. Testes: quantos passam e quais foram adicionados.
4. Campos novos (opcionais) e a confirmação de que nada foi apagado.
5. O que eu faço no iPhone, se algo mudou. Se nada mudou, diga isso.
6. Comandos para publicar, dizendo se as regras do Firestore mudaram.

---

# ETAPA B · Acesso, planos e licenças sem custo

> Só comece quando eu disser "pode seguir para a Etapa B".

**Objetivo:** deixar o FinPilot pronto para ter outros usuários com planos, **sem cobrança automática por enquanto**. O pagamento entra depois, e esta etapa deixa o encaixe pronto.

### B0. Migração para subcoleções (primeiro passo, decisão A3)
- Siga o plano do `AUDITORIA.md`:
  - lançamentos em `users/{uid}/transacoes/{id}`;
  - o resto continua no documento principal;
  - leitura das duas fontes, gravação só na subcoleção;
  - cópia em lotes, com `migradoEm` marcado e a lista antiga intacta para voltar atrás.
- Confira que o volume cabe folgado na cota gratuita do Firestore no meu uso e com 50 usuários. Me mostre a conta.
- **Testes:** totais iguais antes e depois, e a volta atrás.

### B1. Papéis
- **Papéis:** `dono` (eu), `admin`, `suporte` e `usuario`.
- **Onde guardar:** numa coleção protegida (`papeis/{uid}`) que só o dono grava, lida pelas regras do Firestore com `get()`. Custom claims exigiriam servidor.
- O dono continua garantido também pelo e-mail verificado, para eu nunca perder o acesso.

### B2. Os três planos

| Plano | O que libera |
|---|---|
| **Básico** | Só **visão geral**: Início (painel, KPIs, gráficos), Lançamentos (ver, lançar e editar à mão), Patrimônio (contas e saldos) e Ajuda e Configurações básicas |
| **Avançado** | Tudo do Básico, mais **Contas a pagar**, **Assinaturas** e **Orçamento** (incluindo o ciclo do cartão) |
| **Premium** | **Acesso total**: tudo do Avançado, mais Cartões e faturas, importação de extrato e fatura, lançamento automático (atalhos, Android, diagnóstico), Metas, Investimento, Financiamento, Projeção, Ajuda financeira com mentor, gastos por período (A3) e os recursos futuros (WhatsApp, Open Finance, IA) |

- **Telas não listadas:** se existir tela que não está na tabela, siga a lógica **Básico = acompanhar, Avançado = organizar contas, Premium = automatizar e planejar**. Liste o que você decidiu na entrega. Me diga também se acha que algum item está no plano errado, por exemplo se cartões e faturas deveriam ficar no Avançado.
- **Teste grátis:** 14 dias de **Premium** para quem entra pela primeira vez, sem cartão.
- **Outros status:** `cortesia` (com ou sem prazo), `vitalicio` (fundadores), `expirado` e `cancelado`.
- **Quando o plano não cobre uma tela:**
  - A tela aparece no menu com um cadeado discreto.
  - Ao tocar, abre um **paywall elegante** mostrando o que o plano libera, com "Falar com o FinPilot" (abre o meu WhatsApp ou e-mail; configurável no Admin).
  - Nunca uma tela de erro.
- **Plano menor ou vencido:**
  - Nada é apagado. Os dados das telas bloqueadas ficam guardados e voltam quando o plano sobe.
  - Os dados do plano menor continuam visíveis, mas só para leitura.
  - "Baixar meus dados" funciona em **qualquer** plano.
- **Onde fica a regra:** uma tabela só, `src/acesso/planos.js`, com plano → telas e recursos. Nada de `if` de plano espalhado pelas telas. Testes para cada plano.

### B3. Licenças guardadas com segurança (sem servidor)
- **Coleção nova `licencas/{uid}`** com: plano, status, início, fim do teste, fim do período, origem (`teste`, `cortesia`, `pagamento_manual`, `vitalicio`), observação e histórico.
  - Leitura: o próprio usuário e admin/dono.
  - Escrita: **só admin/dono, pelas regras do Firestore**. O cliente nunca grava a própria licença.
  - Tire `licenses` de `config/app` (corrige o M4), migrando o que existir.
- **Criação do teste grátis:** a regra permite ao próprio usuário criar a sua licença **uma única vez**, e só com plano `premium`, status `trial` e `fimTeste` igual a `request.time` + 14 dias (com tolerância). Teste isso nas regras.
- **Vencimento sem tarefa agendada:**
  - As regras do Firestore comparam com `request.time`: depois do `fimTeste` ou `fimPeriodo`, a escrita nas áreas do plano é recusada pelo servidor.
  - O app mostra o mesmo estado lendo a licença.
  - Não é preciso Cloud Function.
- **Bloqueio de verdade, não só na tela:**
  - Depois da migração B0, as regras recusam gravação nas coleções e campos de recursos que o plano não cobre (por exemplo, `contasAPagar` no Básico).
  - Onde for complicado fazer pela regra com o documento atual, faça pela interface e liste o que ficou só na interface. Sem clientes pagantes isso é aceitável por enquanto, mas quero a lista.
- **Remova `handleChangePlan`**, ou faça a troca de plano virar um **pedido** que o admin aprova.

### B4. Pagamento manual (custo zero) e encaixe para o futuro
- **Por enquanto o fluxo é manual:**
  1. A pessoa escolhe um plano.
  2. Vê as instruções para pagar por **Pix**: minha chave Pix, configurável no Admin, e o valor do plano.
  3. Toca em "Já paguei" e anexa o comprovante (opcional) ou digita o ID.
  4. O pedido aparece no Admin.
  5. Eu confirmo e a licença é ativada por 1 mês ou 1 ano.
- **Preços:** configuráveis no Admin, mensal e anual.
- **Para o futuro:**
  - Crie uma interface `provedorPagamento` (por exemplo `iniciarCheckout`, `tratarWebhook`) com a implementação "manual" funcionando e um esboço comentado para Mercado Pago ou Stripe.
  - Escreva `PAGAMENTO_FUTURO.md` explicando o que vou precisar fazer quando quiser cobrança automática: plano Blaze, Cloud Function de webhook, conta no provedor, custos estimados e comparação entre Mercado Pago, Stripe e Asaas para pessoa física ou MEI.

### B5. Admin
- **Indicadores:** usuários totais, ativos nos últimos 30 dias, em teste, por plano, pedidos de pagamento pendentes, e receita do mês e do ano pelos pagamentos manuais confirmados.
- **Lista de usuários:** busca e filtro por plano e status. Para cada um: e-mail, entrada, último acesso, plano, status e vencimento.
- **Ações:**
  - confirmar ou recusar pagamento manual;
  - conceder cortesia (com prazo) ou vitalício;
  - estender teste;
  - trocar plano;
  - revogar;
  - mudar papel (só o dono).
- **Log de auditoria** em `auditoriaAdmin/{id}`, que ninguém apaga: quem, o quê, quando, antes e depois.
- **Cupons ou códigos de convite:** dias grátis ou plano de cortesia, com validade e limite de usos.
- **Privacidade:** o admin **não** vê as finanças dos usuários, só os dados da conta e da licença.
- **Feature flags:** continuam como chaves gerais de lançamento, separadas dos planos. A tela do Admin deixa isso claro.

### B6. Para outros usuários entrarem
- **Tela inicial para quem não está logado:** o que é o FinPilot, os três planos e "Entrar com Google". Esta é a vitrine e tem que ter o acabamento premium.
- **Antes de usar:**
  - aceite dos Termos e da Privacidade (rascunhos marcados "revisar com advogado"), com a versão e a data gravadas;
  - "Excluir minha conta", que apaga os dados de fato, depois de confirmar duas vezes e oferecer "Baixar meus dados" antes.
- **Caixa de entrada da automação:** já é por usuário, com chave própria. Confirme isso com teste nas regras.
- **Atalhos por link do iCloud:** um guia para eu gerar os links uma vez ("Perguntas de Importação" pedindo chave e endereço). No app, a pessoa só toca no link e cola a chave. A automação do BB continua sendo criada à mão, com uma ação "Executar Atalho".

### B7. Testes desta etapa
- **Vitest:** acesso por plano, transições de status, teste grátis, vencimento, pedido de pagamento manual e cupom.
- **Regras do Firestore no emulador** (`@firebase/rules-unit-testing`), que roda de graça:
  - usuário não lê nem grava dados de outro;
  - não grava a própria licença (exceto o teste grátis uma vez);
  - não grava área fora do plano;
  - o dono grava tudo e nunca é bloqueado.
- Se o emulador não rodar aí, me passe o comando para eu rodar.

### Entrega da Etapa B (responda assim)
1. Resumo.
2. A tabela final de plano → telas, incluindo o que você decidiu para as telas não listadas.
3. O que fica protegido pelas regras e o que ficou só na interface.
4. O que eu preciso fazer: publicar, configurar a chave Pix e os preços no Admin, e gerar os links do iCloud.
5. Comandos para publicar (as regras vão mudar).
6. Confirmação de que nada gera cobrança e de que eu, como dono, não perdi acesso a nada.
