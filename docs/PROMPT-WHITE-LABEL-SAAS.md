# Prompt mestre — Plataforma SaaS White Label multi-empresa

Use este documento como **briefing único** para arquitetura, desenvolvimento e integrações. Pode ser colado em ferramentas de IA, enviado a fornecedores ou usado como PRD interno.

---

## Contexto e visão

Construir uma **plataforma SaaS white label** que permita vender o mesmo núcleo de software para **diversos tipos de negócio** (ex.: carro por assinatura, barbearia, clínica de estética, lava-jato, outros estabelecimentos). Cada cliente da plataforma é uma **empresa white label** com marca própria, domínio/subdomínio, dados isolados e **módulos contratados sob pagamento**.

Existem **dois mundos financeiros** distintos e explícitos:

1. **Receita da plataforma (vocês):** mensalidades e módulos pagos por cada white label (cartão de crédito e/ou PIX).
2. **Receita do white label:** pagamentos que **os clientes finais** daquela empresa fazem **para o white label**; o sistema deve permitir que cada empresa cadastre **meios de recebimento próprios** (PIX, cartão via adquirente/gateway de escolha).

---

## Objetivos principais

- **Multi-empresa (multi-tenant):** isolamento lógico e de dados por empresa white label.
- **Intranet master (super admin):** cadastro e gestão de white labels, domínios, planos, módulos, cobrança da plataforma e visão financeira consolidada.
- **Intranet por empresa:** administradores de cada white label gerenciam só o que pertence à sua empresa (conforme módulos contratados).
- **Site/app público por tenant:** landing, cadastros, tracking e fluxos específicos do vertical, com branding por empresa.
- **Controle financeiro duplo:**  
  - **Plataforma:** faturas, assinaturas, renovações, inadimplência, histórico de pagamentos dos white labels.  
  - **White label:** conciliação e registros dos recebimentos dos **clientes finais** (quando o produto exigir).
- **Cobrança da plataforma:** assinatura recorrente + **módulos com valor agregado**; pagamento via **cartão de crédito** e **PIX** (com QR Code / copia e cola onde aplicável).
- **Módulos licenciáveis:** cada funcionalidade relevante é um **módulo** com preço; a empresa contrata só o que precisa; a UI e as APIs respeitam o que está ativo.
- **APIs de pagamento “prontas”:** camada de integração abstrata (adapter pattern) com implementações concretas; o cliente white label **só configura credenciais e chaves** no painel, sem reescrever código por banco.

---

## Personas e permissões

| Persona | Descrição |
|--------|-----------|
| **Super Admin (plataforma)** | Acessa intranet master; cria/edita white labels; define preços de planos e módulos; vê financeiro global; suporte e auditoria. |
| **Admin do white label** | Gestão da própria empresa: branding, domínios, usuários internos, módulos contratados (visualização), configuração de gateways para **recebimento dos clientes finais**, CRM, relatórios permitidos. |
| **Usuários operacionais** | Papéis granulares por empresa (ex.: financeiro, marketing, atendimento), limitados por módulo. |
| **Visitante / cliente final** | Interage com o site/checkout do white label; pagamentos vão para as contas/credenciais configuradas pelo white label (quando houver checkout). |

---

## Requisitos funcionais — Intranet master

- CRUD de **empresas (tenants / white labels)** com: nome fantasia, razão social, documento, slug, status (trial, ativo, suspenso, cancelado).
- Associação de **domínios**: subdomínio na infraestrutura da plataforma (ex.: `barbearia.dominiodaplataforma.com`) e **domínio customizado** (ex.: `cliente.com.br` → verificação DNS/CNAME).
- **Catálogo de módulos** com nome, descrição, código técnico estável, **preço mensal** (e opcionalmente preço de setup).
- **Planos base** (opcional) + composição por módulos; ou apenas “mensalidade base + add-ons de módulos”.
- **Assinaturas da plataforma:** vínculo empresa ↔ plano/módulos ativos; data de renovação; trial; histórico de alterações de módulos.
- **Cobrança para a plataforma:** geração de faturas/cobranças; integração com gateway para **cartão** (recorrência quando possível) e **PIX**; webhooks de confirmação; conciliação.
- **Dashboard financeiro master:** MRR, churn, inadimplentes, receita por módulo, por empresa.
- **Auditoria:** log de ações críticas (quem alterou módulo, suspendeu empresa, etc.).

---

## Requisitos funcionais — Empresa white label

- Painel próprio com **isolamento total** de dados de outras empresas.
- Configuração de **marca**: logo, favicon, cores, textos legais, redes, WhatsApp, etc.
- **Gestão de módulos contratados** (somente leitura ou upgrade solicitado, conforme política comercial).
- **Recebimento dos clientes finais:**  
  - Cadastro de **chave PIX** (tipo e chave) para exibição/instruções ou integração com PSP.  
  - Cadastro de **credenciais de gateway/adquirente** para cartão (por ambiente sandbox/produção).  
  - Regra clara: **pagamentos de clientes finais não passam obrigatoriamente pela conta da plataforma** (split/marketplace só se produto exigir); padrão = **dinheiro vai para o white label**.
- Relatórios permitidos pelos módulos (ex.: CRM, tracking, financeiro local).

---

## Requisitos — Módulos (exemplos)

Definir catálogo versionado. Exemplos:

- **Core / Site:** landing, formulários básicos, SEO básico.  
- **CRM / Clientes**  
- **Tracking / Analytics**  
- **Automação / Bot (N8N ou similar)**  
- **Planilhas / catálogo dinâmico** (ex.: preços, serviços)  
- **Depoimentos**  
- **Agendamento** (vertical barbearia/clínica)  
- **Financeiro avançado** (conciliação, DRE simplificada) — se aplicável  

Cada módulo: `code`, `name`, `description`, `price_monthly`, `dependencies[]`, `vertical_tags[]` (opcional).

**Enforcement:** backend e RLS devem negar acesso a dados e rotas de módulos não contratados.

---

## Arquitetura técnica (diretrizes)

- **Single codebase**, deploy único (ex.: Vercel + Supabase ou stack equivalente), com **resolução de tenant** por: `Host` (domínio customizado), subdomínio, ou header em API.
- **Modelo de dados:** tabela `tenants` (empresas); `tenant_id` em **todas** as tabelas de negócio; `tenant_domains`; `subscriptions` (plataforma); `subscription_modules`; `module_catalog`; `platform_invoices` / `platform_payments`.
- **Pagamentos clientes finais:** tabelas `tenant_payment_profiles` ou `tenant_gateway_credentials` (criptografadas em repouso), `payment_providers` enum.
- **Segurança:** RLS no Postgres por `tenant_id`; JWT ou session com `tenant_id` e roles; segredos nunca em front-end em claro.
- **APIs:** REST ou Edge Functions com versionamento (`/v1/...`); documentação OpenAPI.

---

## Camada de pagamentos — abstração e PSPs

Implementar um **Payment Provider Interface** comum, por exemplo:

- `createCharge`, `createSubscription`, `refund`, `webhookHandler`, `getPixQrCode`, `tokenizeCard` (conforme suporte de cada PSP).

**Provedores a prever implementação ou adapter stub + documentação de credenciais** (nomes normalizados):

| Provedor | Observação |
|-----------|------------|
| **Mercado Pago** (Mercado Livre / MP) | PIX, cartão, assinaturas conforme API atual. |
| **PagBank** | Ex-PagSeguro v2; unificar com PagSeguro se API convergir. |
| **Itaú** | API banking / cobrança conforme produto contratado pelo white label. |
| **Santander** | Cobrança / API disponível ao PJ. |
| **Caixa** | Boletos/PIX conforme produtos da instituição. |
| **Bradesco** | APIs de cobrança. |
| **PagSeguro** | Gateway e-commerce. |
| **Stone** | Adquirência. |
| **Rede** | Itaú Rede / e.Rede. |
| **Cielo** | Braspag / Cielo e-commerce. |
| **Inter** | Banking / cobrança / PIX. |
| **Infini Pay** | Conforme documentação oficial do provedor. |
| **Nubank** | NuPay / APIs empresariais disponíveis para parceiros. |

**Importante:** muitos desses nomes agregam **vários produtos** (PIX direto, gateway e-commerce, adquirência). O prompt exige **mapear cada um para o produto real** (documentação oficial) e implementar **adapters**; não misturar “conta PJ” com “gateway de loja” sem especificação.

**Para a plataforma (cobrança do white label):** escolher 1–2 PSPs **estratégicos** para MVP (ex.: Mercado Pago + PIX) e expandir.  
**Para o white label (recebimento dos clientes):** o painel permite **selecionar provedor** e preencher credenciais; o sistema chama o adapter correspondente.

---

## Entregáveis esperados (para quem for desenvolver)

1. Diagrama ER com `tenants`, domínios, assinaturas, módulos, pagamentos plataforma vs pagamentos tenant.  
2. Matriz de permissões (RBAC) master vs tenant.  
3. Fluxo de onboarding: criação de empresa → DNS → primeiro pagamento → liberação de módulos.  
4. Especificação OpenAPI dos endpoints públicos e admin.  
5. Plano de migração do sistema atual (mono-tenant) para primeiro `tenant_id` default.  
6. Política de LGPD: dados por empresa, exportação, exclusão.  
7. Testes: isolamento entre tenants (testes automatizados de RLS).

---

## Critérios de sucesso

- Nenhum dado de empresa A acessível por empresa B (testes e auditoria).  
- Super admin cobra e renova assinatura com cartão e PIX.  
- White label configura recebimento próprio sem deploy novo.  
- Módulo desligado = UI invisível + API 403 + queries impossibilitadas por RLS.  
- Novo PSP adicionado sem alterar regras de negócio centrais (só novo adapter + config).

---

## Como usar este prompt

Cole o bloco abaixo em uma conversa com IA ou anexo junto a este arquivo:

```
Quero implementar uma plataforma SaaS white label multi-tenant conforme o documento PROMPT-WHITE-LABEL-SAAS.md anexo. 
Priorize: (1) modelo de dados e RLS, (2) intranet master + billing da plataforma com PIX e cartão, (3) módulos com preço, (4) camada abstrata de pagamentos com adapters para os PSPs listados, (5) configuração pelo white label para receber pagamentos dos clientes finais. 
Entregue primeiro o desenho arquitetural e o schema SQL; depois o plano de fases de implementação.
```

---

*Documento gerado como especificação mestre — ajuste nomes de produtos bancários conforme contratos e APIs vigentes no Brasil.*
