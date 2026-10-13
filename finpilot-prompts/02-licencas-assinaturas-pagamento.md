# FinPilot · Acesso, licenças, assinaturas e pagamento

> Cole este prompt na conversa do FinPilot no app do Claude **no notebook**. Rode **depois** do prompt de auditoria. Esta etapa começa com **proposta para eu revisar**. Só implemente depois que eu aprovar.

---

Quero transformar o **FinPilot** num produto que eu possa **vender por licença**. Outras pessoas entram com a conta Google delas, escolhem um plano, pagam, e eu faço a gestão de tudo num painel de administrador. Hoje isso é embrionário: existe login com Google, um item **"Meu plano"** (o antigo "Assinatura" do app) e uma seção **Admin**.

## Regras que valem para tudo

- **Git:**
  - Não mexa em `git config` (global ou local), `git remote`, `git push`, credenciais ou hooks.
  - Pode fazer commit local na pasta do projeto, como antes. Se o commit exigir mudar configuração, me avise e deixe os arquivos salvos.
- **Publicação e serviços externos:**
  - Nada de `firebase deploy`.
  - Não crie conta, produto ou preço em provedor de pagamento.
  - Não ative plano pago do Firebase.
  - Liste o que eu preciso fazer e me dê os comandos.
- **Meus dados:**
  - Eu sou o **dono** do app e continuo com acesso total e sem cobrança.
  - Nada dos meus dados pode ser apagado ou ficar inacessível durante a mudança.
- **Visual:** a identidade aprovada vale também para as telas de planos, checkout, paywall e admin. A tela de planos é a vitrine do produto e tem que transmitir a sensação de "feito com cuidado, por quem conhece".
- **Segurança em primeiro lugar:** quem decide se alguém tem acesso é o **servidor** (regra do Firestore ou Cloud Function). A interface nunca decide sozinha. Status de assinatura nunca pode ser gravado pelo cliente.

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
