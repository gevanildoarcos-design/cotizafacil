# 🧁 CotizaFácil

**Orçamentos bonitos e rápidos em 30 segundos.**
Aplicativo web responsivo (PWA) para pequenos negócios da área de alimentação — pensado para **padarias e confeitarias**.
Interface em **🇧🇷 Português** e **🇪🇸 Español**, com troca de idioma em 1 clique.

Funciona no navegador do celular, tablet e computador. **Não precisa de versão nativa**: basta abrir no navegador
e usar "Adicionar à tela inicial" para virar um atalho/app.

---

## ✨ O que ele faz

| Módulo | O que entrega |
|---|---|
| 🏠 **Apresentação** | Logo, frase de valor ("Orçamentos bonitos e rápidos em 30 segundos"), botões **Começar Grátis / Entrar** e os três destaques: *Sem complicação · Envio por WhatsApp · Sem papel perdido* |
| 🔐 **Cadastro / Login** | E-mail + senha, **entrar com Google**, escolha do idioma 🇧🇷/🇪🇸 e apresentação dos planos logo após o cadastro |
| ⭐ **Planos e assinatura** | Gratuito, Iniciante R$14,90, Completo R$29,90 e Crescimento R$49,90 — **anual = 10 mensalidades (2 meses grátis)** |
| 📝 **Novo orçamento** | Produto (Torta · Bolo · Doces · Pão especial · Outro), Tipo/Tamanho, Sabor, Recheio, Decoração, Mensagem, Data e hora de entrega, Cliente (nome + telefone), valor calculado com ajuste manual |
| 📄 **PDF + 📲 WhatsApp** | PDF profissional com o **logo do estabelecimento** e link do WhatsApp com mensagem pré-preenchida no idioma do usuário |
| 📅 **Histórico e calendário** | Visão em lista **e** calendário mensal, filtros *pendente / pago / entregue*, ações: reenviar, marcar como pago, editar, excluir |
| 👥 **Clientes** | Nome, telefone, histórico de pedidos, total vendido e **observações** (ex.: "sem chocolate") |
| 🧾 **Minha assinatura** | Plano atual, valor, próxima cobrança, status, alterar plano, cancelar e histórico de pagamentos |
| 🔔 **Lembrete por e-mail** | E-mail automático **3 dias antes do vencimento** (Vercel Cron + Resend) |
| 📱 **PWA** | `manifest.json` + service worker: instala como atalho e abre rápido mesmo com internet ruim |

---

## 🧱 Stack

- **Next.js 15 (App Router) + React 19 + TypeScript** — um único projeto que roda site + APIs.
- **Supabase** (Postgres + Auth) — login por e-mail/senha e Google; dados protegidos por **Row Level Security**.
- **Gateway de pagamento** intercambiável: **Mercado Pago** (`preapproval`) ou **Stripe Billing** (`subscription`).
- **Resend** — e-mails de lembrete e confirmação.
- **jsPDF** — o PDF é gerado **no próprio aparelho**, sem servidor.
- **Vercel** — hospedagem + Cron diário.

> **Não precisa de servidor caro no começo**: o plano gratuito da Vercel e do Supabase já sustentam os primeiros clientes.

---

## 🚀 Instalação em 6 passos

### 1. Pré-requisitos
```bash
node -v   # precisa ser 18.18 ou superior (recomendado 20+)
npm -v
```

### 2. Baixar e instalar
```bash
cd cotizafacil
npm install
```

### 3. Criar o arquivo de ambiente
```bash
cp .env.example .env.local
```
Abra `.env.local` e preencha (veja os passos 4 e 5). **Nunca** comite chaves reais.

### 4. Configurar o Supabase (banco + login)
1. Crie uma conta em [supabase.com](https://supabase.com) e um novo projeto (guardar a senha do banco).
2. Vá em **SQL Editor → New query**, cole **todo** o conteúdo de `supabase/schema.sql` e clique em **Run**.
   Isso cria as tabelas `profiles`, `clients`, `quotes`, `payments`, `subscriptions`, as políticas de segurança (RLS),
   o gatilho que cria o perfil no cadastro e o **gatilho que bloqueia o 6º orçamento do plano gratuito**.
3. Opcional: rode `supabase/seed.sql` para ter clientes e orçamentos de exemplo (troque antes o `<SEU_USER_ID>`).
4. Em **Project Settings → API**, copie:
   - `Project URL` → `NEXT_PUBLIC_SUPABASE_URL`
   - `anon public` → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `service_role` → `SUPABASE_SERVICE_ROLE_KEY` ⚠️ **somente no servidor, nunca no navegador**
5. Login com Google (opcional): **Authentication → Providers → Google → Enable**, cole o
   *Client ID* e *Client Secret* criados no [Google Cloud Console](https://console.cloud.google.com/apis/credentials).
   Em **Redirect URLs**, adicione `https://SEU-DOMINIO/auth/v1/callback` e `http://localhost:3000/auth/v1/callback`.
6. Em **Authentication → URL Configuration**, coloque sua URL do site em *Site URL* e adicione
   `http://localhost:3000/**` em *Redirect URLs*.

### 5. Configurar o gateway de pagamento
Escolha **um** provedor e ajuste `config/payment.config.ts`.

#### Opção A — Mercado Pago (recomendado para Brasil / Latam)
1. [Painel de desenvolvedor](https://www.mercadopago.com.br/developers/panel/app) → crie uma aplicação → copie o **Access Token**.
2. `.env.local`:
   ```env
   PAYMENT_PROVIDER="mercadopago"
   PAYMENT_CURRENCY="BRL"
   MP_ACCESS_TOKEN="APP_USR-..."
   MP_WEBHOOK_SECRET="sua_chave_secreta_do_webhook"
   MP_BACK_URL="https://SEU-DOMINIO/assinatura"
   ```
3. Em **Webhooks**, cadastre a URL `https://SEU-DOMINIO/api/webhooks/mercadopago` e marque os eventos
   **`subscription_preapproval`** e **`subscription_authorized_payment`**.
   A assinatura é criada via `POST https://api.mercadopago.com/preapproval` com `auto_recurring`
   (`frequency: 1 ou 12`, `frequency_type: "months"`, `transaction_amount`, `currency_id`).
4. Teste com as credenciais de **teste** (sandbox) antes de usar as de produção.

#### Opção B — Stripe
1. [Dashboard → Developers → API keys](https://dashboard.stripe.com/apikeys) → copie a **Secret key**.
2. Crie 6 preços recorrentes em **Products** (mensal e anual de cada plano) e copie os `price_id`.
3. `.env.local`:
   ```env
   PAYMENT_PROVIDER="stripe"
   PAYMENT_CURRENCY="BRL"
   STRIPE_SECRET_KEY="sk_live_..."
   STRIPE_WEBHOOK_SECRET="whsec_..."
   NEXT_PUBLIC_STRIPE_PRICE_STARTER_MONTHLY="price_..."
   # ... os outros 5 preços
   ```
4. Webhook: `stripe listen --forward-to localhost:3000/api/webhooks/stripe` para testar localmente, e
   `https://SEU-DOMINIO/api/webhooks/stripe` em produção (eventos `checkout.session.completed`,
   `invoice.paid`, `invoice.payment_failed`, `customer.subscription.updated`, `customer.subscription.deleted`).

#### Opção C — Sem gateway (só para testar o app)
Deixe `PAYMENT_PROVIDER` apontando para um provedor **sem chave**: a API responde `{ mode: "demo" }` e o app
ativa o plano localmente, mostrando o aviso "Gateway de pagamento não configurado: assinatura simulada".

### 6. Rodar
```bash
npm run dev
# abra http://localhost:3000
```
Para produzir a build de produção:
```bash
npm run build && npm start
```

---

## 💌 Lembrete 3 dias antes do vencimento

O agendamento já está em `vercel.json`:
```json
{ "crons": [{ "path": "/api/cron/lembretes", "schedule": "0 12 * * *" }] }
```
A rota `/api/cron/lembretes` lê a view `subscriptions_due_in_3_days` (definida no `schema.sql`),
envia o e-mail pelo **Resend** e grava `reminder_sent_at` para nunca repetir o mesmo aviso.

1. Crie uma conta em [resend.com](https://resend.com), verifique seu domínio e gere uma API key.
2. `.env.local`:
   ```env
   RESEND_API_KEY="re_..."
   MAIL_FROM="CotizaFácil <lembretes@SEU-DOMINIO>"
   CRON_SECRET="uma_string_aleatoria_longa"
   ```
3. Na Vercel, adicione `CRON_SECRET` nas variáveis do projeto (o Cron envia `Authorization: Bearer <CRON_SECRET>`).
4. Teste manualmente:
   ```bash
   curl -H "Authorization: Bearer $CRON_SECRET" https://SEU-DOMINIO/api/cron/lembretes
   ```
   Sem `RESEND_API_KEY` o envio vira *dry-run* e só aparece no log — ótimo para desenvolvimento.

> **Sem Vercel?** Também funciona com o `pg_cron` do Supabase chamando a mesma URL, ou qualquer agendador
> que faça um GET diário nessa rota com o header `Authorization`.

---

## ☁️ Publicar (Vercel + domínio próprio)

```bash
npm i -g vercel
vercel            # primeiro deploy (preview)
vercel --prod     # produção
```
Ou publique pelo site da Vercel conectando o repositório do GitHub.

1. **Environment Variables**: cole todas as chaves do `.env.local` em
   *Project → Settings → Environment Variables* (Production **e** Preview) e faça um novo deploy.
2. **Domínio**: *Project → Settings → Domains → Add* → `coti-facil.com` e `www.coti-facil.com`.
   No seu registrador, crie os registros que a Vercel mostrar:
   - `A` para `coti-facil.com` → `76.76.21.21` (a Vercel confirma o IP na tela)
   - `CNAME` para `www` → `cname.vercel-dns.com`
3. Ajuste `NEXT_PUBLIC_APP_URL="https://coti-facil.com"` e **reimplante**.
4. Atualize as URLs nos painéis: webhook do gateway, *Site URL* do Supabase e `MP_BACK_URL`.
5. Abra no celular → menu do navegador → **Adicionar à tela inicial**. Pronto, virou app. 📱

---

## ✅ Roteiro de teste (Passo 5 do briefing)

| Teste | Como verificar |
|---|---|
| Cadastro novo usuário | `/entrar?tab=signup` → cria conta → cai na tela de planos |
| Limite do plano gratuito (5/mês) | Crie 5 orçamentos e tente o 6º: aparece o aviso e o botão **Assinar o Iniciante** |
| Bloqueio real no servidor | `insert` direto pela API do Supabase com o 6º orçamento devolve `free_limit_reached` (gatilho no banco) |
| Gerar orçamento e enviar por WhatsApp | "📲 Enviar por WhatsApp" salva e abre `wa.me` com a mensagem pronta |
| PDF com logo | Configurações → envie o logo → "📄 Gerar PDF" baixa o arquivo |
| Assinar plano e liberar | Assine o Iniciante → webhook grava `plan=starter` e `subscription_status=active` |
| Trocar idioma PT/ES | Botão 🇧🇷/🇪🇸 no topo — toda a interface e a mensagem do WhatsApp mudam |
| Celular e computador | Abra o mesmo endereço em ambos; no PC aparece o menu superior, no celular a barra inferior |
| Lembrete de vencimento | `curl` na rota do cron com o `CRON_SECRET` e confira o e-mail |

---

## 🗂️ Estrutura de arquivos

```
cotizafacil/
├── config/
│   └── payment.config.ts          # ⚙️ único arquivo para trocar de gateway
├── lib/
│   ├── plans.ts                   # planos, preços, anual = 10× mensal
│   ├── pricing.ts                 # sugestão automática de valor
│   ├── whatsapp.ts                # wa.me + mensagem PT/ES
│   ├── pdf.ts                     # PDF com logo (jsPDF, no navegador)
│   ├── email.ts                   # Resend: lembrete + confirmação
│   ├── repo.ts                    # dados (Supabase ou localStorage)
│   ├── types.ts                   # tipos TypeScript
│   ├── i18n.ts                    # traduções PT/ES
│   └── supabase/{client,server}.ts
├── locales/
│   ├── pt.json                    # 🇧🇷 textos
│   └── es.json                    # 🇪🇸 textos (mesmas chaves)
├── app/
│   ├── layout.tsx  globals.css    # tema amarelo / branco / marrom
│   ├── page.tsx                   # 1. apresentação
│   ├── entrar/                    # 2. cadastro e login + idioma
│   ├── planos/                    # 3. planos e assinatura
│   ├── novo/                      # 4. orçamento + PDF + WhatsApp
│   ├── historico/                 # 5. lista e calendário
│   ├── clientes/                  # 6. clientes
│   ├── assinatura/                # 7. minha assinatura
│   ├── configuracoes/             # logo, loja, idioma, PWA
│   ├── painel/                    # início do app
│   └── api/
│       ├── checkout/              # cria a assinatura no gateway
│       ├── subscription/cancel/   # cancela a assinatura
│       ├── webhooks/stripe|mercadopago/
│       └── cron/lembretes/        # e-mail 3 dias antes
├── components/                    # AppShell, PlanCards, Calendar, QuoteCard…
├── supabase/{schema.sql,seed.sql}
├── public/{manifest.json,sw.js,icon-*.png}
├── vercel.json                    # agendamento do cron
├── .env.example
└── README.md
```

---

## 🔐 Segurança e boas práticas já embutidas

- **RLS em todas as tabelas**: cada usuário só lê e escreve os próprios dados.
- **Limite do plano gratuito no banco** (`enforce_quote_limit`), não apenas na interface.
- **Chaves secretas só no servidor** (`SUPABASE_SERVICE_ROLE_KEY`, tokens dos gateways).
- **Webhook do Mercado Pago validado por HMAC** (`x-signature` + `x-request-id`).
- **Webhook do Stripe validado** por `stripe.webhooks.constructEvent`.
- Telefone normalizado para **E.164 sem símbolos** antes de montar o link `wa.me`.

---

## 🌎 Notas sobre espanhol (ES)

Os textos em espanhol ficam em `locales/es.json`, com **exatamente as mesmas chaves** do `pt.json`
(assim nenhuma tela fica sem tradução). Diferenças culturais já aplicadas:

| PT | ES |
|---|---|
| Orçamento | Presupuesto |
| Bolo | Pastel |
| Doces | Dulces |
| Pão especial | Pan especial |
| Recheio | Relleno |
| Data de entrega | Fecha de entrega |
| Valor | Monto / Valor |
| Assinatura | Suscripción |

Para adicionar um terceiro idioma, copie `pt.json`, traduza e registre em `lib/i18n.ts`.

---

## 🧰 Comandos úteis

```bash
npm run dev         # desenvolvimento
npm run build       # build de produção
npm start           # roda a build
npm run typecheck   # checagem de tipos
```

---

Feito com carinho para padarias e confeitarias da América Latina. 🥖🎂
