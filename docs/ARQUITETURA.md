# Back-end do Trimox: análise e proposta de arquitetura

Este documento avalia como o front-end ([trimox-front-end](https://github.com/lhrangelis/trimox-front-end)) está estruturado do ponto de vista de dados, compara stacks de back-end e propõe como começar com **Supabase**.

**Resumo:** Supabase é uma boa escolha para o Trimox. O domínio é relacional (agenda, clientes, caixa, comissões, assinaturas), multi-empresa e com papéis bem definidos, exatamente onde Postgres + Row Level Security (RLS) se encaixam. Vale começar por ele, com três cuidados:

1. As regras críticas (impedir horário duplo, fechar caixa, fechar comissão) ficam **no banco**, em constraints e funções, e não espalhadas pelo front.
2. Tudo que fala com terceiros (WhatsApp, pagamentos) fica em **Edge Functions** com chave de servidor. Nunca no navegador.
3. O schema vive **versionado neste repositório** (migrations + testes). Nada de criar tabela pelo painel do Supabase.

---

## 1. Como o front-end está hoje, do ponto de vista do back-end

### O que já ajuda

- **Os dados estão isolados em `src/lib/data/`**, com tipos TypeScript. É um bom rascunho do contrato de dados e mostra quais entidades existem.
- **Cada página é um Server Component fino** (`page.tsx`) que renderiza uma *view* cliente (`AgendaView`, `CashView`…). Esse formato permite buscar os dados no servidor e passá-los como props, sem reescrever as telas.
- **As regras de cálculo já aparecem no código** (comissão = faturado × %, MRR = preço × assinantes, gaveta = troco + dinheiro − sangrias). Elas servem de especificação para as funções do banco.

### O que precisa mudar antes de ligar o back-end

| Problema no front | Por que importa | O que fazer |
|---|---|---|
| **18 arquivos importam os mocks direto** (`@/lib/data/...`) | Trocar mock por API exigiria mexer em todas as telas | Criar uma camada `src/lib/api/` (ex.: `getAgenda(unitId, date)`). As telas passam a receber dados por props, e só essa camada muda |
| **Relacionamentos por nome, não por id** (`pro: "Rafael"`, `preferredPro: "Camila"`, alertas e faturas apontam para `account: "Studio Bella Pele"`) | Renomear alguém quebra os vínculos, e há homônimos | Toda referência vira `uuid` com chave estrangeira |
| **Datas como texto** (`"13/10"`, `"Há 18 dias"`, `"hoje"`) e um "hoje" fixo (`BUSINESS.today`, `NOW_HOUR`) | Impossível ordenar, filtrar por período ou calcular "sumido há 45 dias" | `timestamptz` no banco, fuso `America/Sao_Paulo`, formatação só no front |
| **Dinheiro em `number` de reais** | Erros de arredondamento em comissão e caixa (ex.: 45% de R$ 33,33) | Guardar em **centavos** (`integer`/`bigint`) |
| **Indicadores escritos à mão** (taxa de retorno 64%, faltas 3,1%, churn 2,1%, "+72 este mês", ocupação 71%…) | Viram mentira assim que os dados forem reais | Views SQL ou funções que calculam a partir dos lançamentos |
| **Sem sessão, papéis nem empresa ativa.** O login só faz `router.push`, e o seletor de unidade não faz nada | Sem isso não há como isolar os dados de cada barbearia | Supabase Auth + tabela de membros com papel + unidade ativa |
| **Ações só mudam o estado local** (salvar, pagar, suspender, enviar) | Somem ao recarregar a página | Server Actions chamando o Supabase, com estado de carregando e de erro |
| **Sem `loading.tsx` / `error.tsx`** | Com dados reais vai haver espera e falha de rede | Adicionar por segmento (`/app`, `/admin`) |
| **Usuários e e-mails fixos em Configurações** | — | Virão da tabela de membros |

Nenhum desses itens é grave. É o esperado para um front construído sem back-end, e a correção é mecânica.

---

## 2. Modelo de domínio

O Trimox tem **dois níveis de cliente**, e isso define toda a segurança:

- **Contas (tenants):** a barbearia ou salão que assina o Trimox. Uma conta tem uma ou mais **unidades** (o plano Rede permite várias).
- **Clientes finais:** as pessoas que cortam o cabelo. Pertencem a uma conta.

E há **três tipos de usuário**:

| Usuário | Onde entra | Vê o quê |
|---|---|---|
| Membro da conta (proprietário, recepção, profissional) | `/app` | Só os dados da própria conta, filtrados pelas permissões do papel |
| Admin da plataforma (CEO, suporte) | `/admin` | Todas as contas, mas por funções de servidor e com registro em log |
| Cliente final (anônimo) | `/agendar/[slug]` | Só horários livres e o próprio agendamento |

### Tabelas propostas (primeira versão)

```
plans                 id, name, price_cents, max_pros, features (jsonb)
accounts              id, name, type, plan_id, status, trial_ends_at, created_at
units                 id, account_id, name, slug (único), address, timezone, whatsapp
unit_hours            unit_id, weekday, opens_at, closes_at
members               id, account_id, user_id → auth.users, role, permissions (jsonb)
professionals         id, unit_id, member_id (nullable), name, commission_pct
professional_hours    professional_id, weekday, starts_at, ends_at, break_start, break_end
service_categories    id, account_id, name
services              id, account_id, category_id, name, duration_min, price_cents,
                      commission_pct, online, deposit_cents
professional_services professional_id, service_id
customers             id, account_id, name, phone (E.164), notes, consent_at, created_at
appointments          id, unit_id, professional_id, customer_id, service_id,
                      during (tstzrange), status, source (link|recepção), price_cents
payments              id, unit_id, appointment_id (nullable), customer_id, method,
                      amount_cents, products_cents, paid_at, cash_session_id
cash_sessions         id, unit_id, opened_by, opened_at, opening_cents,
                      closed_at, counted_cents
cash_movements        id, cash_session_id, kind (sangria|suprimento), amount_cents
commission_periods    id, unit_id, starts_on, ends_on, closed_at, paid_at
commission_entries    id, period_id, professional_id, payment_id, base_cents,
                      pct, amount_cents        ← "foto" do % no momento do pagamento
advances              id, period_id, professional_id, amount_cents, note
reminder_settings     unit_id, confirm_on_book, remind_24h, remind_2h, comeback_days, template
notifications         id, appointment_id, channel, kind, status, sent_at, provider_id

-- plataforma (só admin)
subscriptions         id, account_id, plan_id, provider, provider_id, status, current_period_end
invoices              id, account_id, amount_cents, due_on, status, method, provider_id
feature_overrides     account_id, feature, enabled
feature_flags         key, rollout_pct
announcements         id, title, body, audience, channels, sent_at
admin_audit_log       id, admin_user_id, action, account_id, payload, created_at
```

Duas decisões importantes estão nesse modelo:

- **`commission_entries` guarda o % usado.** Se o proprietário mudar a comissão do Bruno de 45% para 50% no dia 20, a quinzena anterior não pode mudar.
- **`appointments.during` é um intervalo (`tstzrange`).** Isso permite que o próprio Postgres impeça dois agendamentos sobrepostos para o mesmo profissional (seção 4).

---

## 3. Comparação de stacks

| Opção | Pontos fortes | Pontos fracos | Para o Trimox |
|---|---|---|---|
| **Supabase** (Postgres + Auth + RLS + Storage + Edge Functions + Realtime + Cron/Queues) | Muito pronto: login, banco, arquivos, tempo real e jobs no mesmo lugar. É Postgres de verdade, então dá para sair se precisar. Região **São Paulo** (bom para a LGPD e a latência). CLI com migrations locais | RLS mal escrita vira brecha ou lentidão. Edge Functions (Deno) têm limite de tempo e memória. Parte da lógica fica em SQL, então o time precisa saber SQL | **Recomendado para começar** |
| **Next.js como back-end** (Route Handlers / Server Actions) + Drizzle ou Prisma + Postgres gerenciado (Neon, RDS) + Auth.js ou Better Auth | Tudo em TypeScript num repo só, com controle total | Você monta e mantém auth, jobs, filas, arquivos e tempo real na mão. Multi-tenant fica todo no código da aplicação (um `WHERE` esquecido vaza dados) | Bom se o time não quiser SQL/RLS. Mais trabalho no início |
| **API separada** (NestJS, Fastify ou Hono) + Postgres | Arquitetura clássica, fácil de testar, serve app mobile e integrações | Mais um serviço para hospedar, versionar e monitorar | Faz sentido **depois**, se surgir app nativo ou API pública (o plano Rede promete "API e integrações") |
| **Firebase** | Rápido para protótipo | Banco de documentos é ruim para relatórios, somas e joins (caixa, comissões, MRR) | **Não recomendado** |
| **Laravel / Rails** | Muito produtivos para SaaS | Stack diferente do front, sem reaproveitar tipos | Só se o time já dominar |

**Por que Supabase e não "tudo no Next":** o Trimox precisa de login (incluindo OTP por WhatsApp no futuro), jobs agendados (lembrete 24 h antes), fila de mensagens, upload de logo, agenda atualizando em tempo real na recepção e isolamento rígido entre barbearias. O Supabase entrega tudo isso pronto. No Next puro, cada item seria uma biblioteca ou serviço a mais.

**Lock-in:** é moderado. O dado está num Postgres padrão (dá para usar `pg_dump` e migrar). O que prende são Auth, Storage e Edge Functions, que são substituíveis com algum trabalho.

---

## 4. Arquitetura proposta com Supabase

```
                 ┌────────────────────────── Supabase (região São Paulo) ─────────────────────────┐
 Navegador       │                                                                                 │
 ─────────       │  Auth ──► Postgres (RLS por conta) ◄── pg_cron ──► Queues ──► Edge Functions ───┼──► WhatsApp (Meta)
 Next.js         │              ▲   ▲                                              │               │
 (Vercel)  ──────┼── leitura ───┘   └── RPC (funções SQL: agendar, fechar caixa…)   └──────────────┼──► Pagamentos (Pix/cartão)
 Server Comp. /  │                                                                                 │        │
 Server Actions  │  Storage (logos)      Realtime (agenda da recepção)     Webhooks ◄──────────────┼────────┘
                 └─────────────────────────────────────────────────────────────────────────────────┘
```

### Onde fica cada tipo de lógica

| Tipo de lógica | Onde | Exemplo |
|---|---|---|
| Leitura e CRUD simples | Tabela + **RLS** | Listar clientes, editar serviço |
| Regra com várias tabelas ou que precisa ser atômica | **Função SQL** (`rpc`) | `create_appointment`, `close_cash_session`, `close_commission_period` |
| Fala com terceiros ou usa segredo | **Edge Function** | Enviar WhatsApp, criar cobrança, receber webhook |
| Recorrente | **pg_cron** → fila → Edge Function | Lembrete 24 h antes, fechar quinzena no dia 15, marcar teste vencido |
| Indicadores | **Views** (ou views materializadas, se ficar pesado) | Ocupação do dia, ticket médio, MRR, churn |

### No front-end

- `@supabase/ssr` para a sessão em cookies. O `proxy.ts` (nome novo do *middleware* no Next 16) só renova a sessão e redireciona quem não está logado. **A autorização de verdade é a RLS.**
- Server Components leem os dados, e Server Actions fazem as mutações e chamam as funções SQL.
- Tipos gerados com `supabase gen types typescript`, copiados para o front (`src/lib/database.types.ts`).
- A chave `service_role` **nunca** chega ao navegador nem ao front. Ela só existe nas Edge Functions.

---

## 5. Pontos críticos (onde costuma dar errado)

### 5.1 Horário duplo
Duas pessoas agendando o mesmo horário pelo link ao mesmo tempo é o bug mais provável. Checar no front não basta. A proteção é uma constraint no banco:

```sql
create extension if not exists btree_gist;

alter table appointments
  add constraint no_overlap
  exclude using gist (professional_id with =, during with &&)
  where (status in ('confirmed', 'pending', 'in_progress'));
```

Com isso, o segundo `insert` falha, e a função `create_appointment` devolve "esse horário acabou de ser ocupado".

### 5.2 Agendamento público (`/agendar/[slug]`)
O cliente final não tem login. **Não abra as tabelas para o papel `anon`.** Exponha só duas funções `security definer`:
- `get_available_slots(slug, service_id, professional_id, date)`, que devolve os horários livres sem nomes de outros clientes;
- `create_public_booking(...)`, que cria ou reaproveita o cliente pelo telefone e cria o agendamento.

Proteja com **Cloudflare Turnstile** (captcha invisível) e limite de tentativas por IP e telefone. Sem isso, um robô lota a agenda de uma barbearia.

### 5.3 Fuso horário
Guarde tudo em `timestamptz` e o fuso em `units.timezone`. "Amanhã às 16:30" é calculado no fuso da unidade, não do servidor. Isso evita erro no horário de verão (se ele voltar) e em unidades fora de São Paulo.

### 5.4 Dinheiro
Use só centavos inteiros. O % de comissão é guardado com o lançamento. Arredonde em um único lugar (na função SQL) e documente a regra (ex.: arredondar meio centavo para cima).

### 5.5 Admin e "Acessar como cliente"
- Os admins da plataforma ficam numa tabela própria (`platform_admins`). As telas `/admin` chamam funções que checam essa tabela.
- **"Acessar como cliente"** não deve entrar com a senha do cliente. Deve abrir uma sessão de leitura marcada e registrar quem, quando e o quê em `admin_audit_log`, com uma faixa visível "você está vendo a conta X".
- Suspender conta, reenviar cobrança e enviar comunicado também vão para o log.

### 5.6 LGPD
- Use a região **São Paulo (sa-east-1)**.
- Observações como "sensível a amônia" podem ser dado de saúde (dado sensível). Restrinja quem lê as observações por papel e registre o consentimento (`customers.consent_at`).
- Ofereça exportação e exclusão de dados do cliente final. O próprio design já cita "pediu ajuda para exportar clientes".
- Para lembretes por WhatsApp, guarde o opt-in. A API oficial exige isso.

### 5.7 Testes de RLS
Erro de RLS é silencioso: a tela só fica vazia, ou pior, mostra dado de outra conta. Escreva testes **pgTAP** (`supabase test db`) para cada tabela: "membro da conta A não lê nada da conta B", "profissional sem permissão não vê faturamento" e assim por diante. Rode no CI a cada PR.

---

## 6. Integrações (decisões pendentes)

| Necessidade | Opções | Observação |
|---|---|---|
| **WhatsApp** (confirmação, lembrete, convite de retorno) | **API oficial da Meta** (WhatsApp Cloud API), direto ou via BSP (Twilio, 360dialog, Gupshup…) · APIs não oficiais (Z-API, Evolution) | A oficial exige templates aprovados e é cobrada por conversa, mas não derruba o número. As não oficiais são baratas e arriscadas: o número do cliente pode ser **banido**. Para um SaaS, recomendo a oficial |
| **Assinatura do Trimox** (Solo, Equipe, Rede) | Asaas, Pagar.me, Mercado Pago, Stripe (Brasil) | Precisa de cartão recorrente, Pix e boleto, além de webhooks. O "Financeiro" do CEO (falhou, atrasada, reenviar cobrança) depende disso |
| **Sinal via Pix no agendamento** | O mesmo provedor, idealmente com subconta ou split por barbearia | O dinheiro é da barbearia, não do Trimox. Exige subconta, o que tem implicação fiscal e contratual |
| **E-mail** (recuperar senha, comunicados) | Resend, SES, Postmark | O Supabase Auth manda os e-mails de login, mas o SMTP padrão tem limite baixo. Configure um SMTP próprio |
| **Login com Google** | Supabase Auth | Já previsto nas telas |

### Decisões tomadas

**Pagamento da assinatura: Mercado Pago.** A cobrança recorrente usa as assinaturas do Mercado Pago, e os webhooks atualizam `subscriptions` e `invoices`. Antes de implementar, confirmar quais formas de pagamento a assinatura recorrente aceita. No cartão é automática, mas Pix e boleto podem exigir gerar uma cobrança avulsa por mês.

**O Trimox não intermedeia o pagamento do cliente final.** O corte é pago direto à barbearia, na maquininha ou no Pix dela. O Caixa só **registra** o que entrou. Isso evita custódia de dinheiro, subconta, split, regras de subadquirente e responsabilidade fiscal sobre o serviço do salão. A consequência é no **sinal via Pix** do design:
- **Agora:** a barbearia cadastra a própria chave Pix. A página de agendamento mostra o Pix copia-e-cola, e a recepção confirma o recebimento à mão. Não há integração nem dinheiro passando pelo Trimox.
- **Depois (opcional):** a barbearia conecta a **própria** conta Mercado Pago (OAuth). O Trimox gera a cobrança na conta dela e recebe só o aviso de pago. O dinheiro continua indo direto para a barbearia.

**WhatsApp: sem API paga no início.**
- **Fase inicial:** botões "Enviar no WhatsApp" (link `wa.me` com a mensagem pronta) na agenda e na ficha do cliente. A recepção envia do próprio WhatsApp com um toque. Custo zero, sem risco de banimento, mas é manual. Os lembretes automáticos vão por **e-mail**.
- **Quando houver receita:** API oficial. O custo é cobrado por mensagem, e o preço por mensagem de lembrete pode ser incluído no valor dos planos. Consulte a tabela atual da Meta.
- **Por que não a API não oficial:** além do risco de banir o número da barbearia, ela exige manter uma sessão do WhatsApp Web rodando 24 h em um servidor próprio (não roda na Vercel nem nas Edge Functions).
- **Arquitetura:** o envio fica atrás de uma interface única (`notifications` + uma função `send`). Trocar "link manual" por "API oficial" é configuração, não reescrita.

---

## 7. Organização deste repositório

```
trimox-back-end/
  supabase/
    config.toml
    migrations/            SQL versionado (supabase migration new ...)
    seed.sql               os dados de exemplo do front (Barbearia Central, equipe, serviços…)
    functions/             Edge Functions (Deno/TypeScript)
      send-whatsapp/
      payments-webhook/
      process-reminders/
    tests/                 testes pgTAP de schema e RLS
  docs/
    ARQUITETURA.md         este documento
    decisoes/              registro de decisões (ADRs curtos)
  .github/workflows/       CI: subir o banco local, rodar migrations e testes, gerar tipos
```

**Ambientes:** local (`supabase start`, que roda em Docker), **staging** e **produção** como dois projetos separados no Supabase. As migrations sobem pelo CI (`supabase db push`), nunca pelo painel.

**Contrato com o front:** a cada mudança de schema, o CI gera os tipos. O front copia o arquivo, ou ele é publicado como pacote mais tarde, se os repositórios crescerem.

---

## 8. Roteiro sugerido

| Fase | Entrega | Telas que ficam reais |
|---|---|---|
| **0. Fundação** | Projeto Supabase (São Paulo), CLI, CI, `plans`/`accounts`/`units`/`members`, Auth (e-mail + Google), RLS e testes | Entrar, Criar conta |
| **1. Agenda** | Serviços, profissionais, horários, clientes, agendamentos com proteção contra horário duplo, Realtime | Agenda, Clientes, Serviços, Profissionais, Configurações (negócio, horário) |
| **2. Agendamento público** | `get_available_slots`, `create_public_booking`, Turnstile, limite de tentativas | `/agendar/[slug]` |
| **3. Dinheiro** | Caixa, pagamentos, sangria, fechamento; comissões com período e vales | Caixa, Comissões |
| **4. Lembretes** | Fila + cron + Edge Function com lembrete por e-mail e botões `wa.me`. Depois, API oficial do WhatsApp com templates e opt-in | Lembretes, indicadores de lembrete |
| **5. Relatórios** | Views de indicadores | Relatórios, resumo da agenda |
| **6. Plataforma** | Assinaturas e webhooks do provedor, faturas, overrides de recursos, feature flags, comunicados, log de auditoria | Todo o `/admin` |

As fases 0 a 2 já permitem um **piloto com uma barbearia real**.

---

## 9. Decisões

| Tema | Decisão |
|---|---|
| Back-end | Supabase (região São Paulo) |
| Hospedagem do front | **Vercel** no início, mantendo o projeto portátil (ver abaixo) |
| Assinatura do Trimox | **Mercado Pago** |
| Pagamento do cliente final | **Direto para a barbearia.** O Trimox só registra |
| WhatsApp | Links `wa.me` manuais + lembretes por e-mail. API oficial quando houver receita |

### Portabilidade da hospedagem
O front não usa nada exclusivo da Vercel. O back-end (Supabase) é independente de onde o front roda. Para manter a migração fácil:
- Não usar pacotes `@vercel/*` (KV, Blob, Analytics) para coisas essenciais. Usar Supabase ou serviços neutros.
- Toda configuração em variáveis de ambiente.
- Para sair da Vercel, rodar em um **servidor Node** (VPS ou plano com suporte a Node.js): `next build` + `next start`, de preferência em Docker com `output: "standalone"`. Hospedagem compartilhada só de PHP/HTML **não** roda o projeto, porque ele precisa de um servidor Node (login, páginas dinâmicas).
- Fora da Vercel, perdem-se CDN global, *preview* por PR e escala automática. Com mais de uma instância, é preciso configurar um cache compartilhado do Next.

### Ainda em aberto
- Regras de negócio: preços e duração do teste, arredondamento de comissão, política de cancelamento e o que acontece com o link de agendamento quando a assinatura atrasa.
- Provedor de e-mail transacional (Resend, SES ou Postmark).
