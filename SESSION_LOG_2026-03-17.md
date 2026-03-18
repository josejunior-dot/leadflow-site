# Session Log — 17-18 Março 2026

## Resumo Executivo

Sessão focada em 3 frentes: (1) revisão de código e segurança do leadflow-site, (2) atualização de pricing para modelo de créditos, (3) integração do widget de chat com o ComercialFlow CRM via WhatsApp Business API.

---

## 1. Repositório: leadflow-site (Landing Page)

**Branch:** main
**Commits:** 8
**Arquivos alterados:** index.html, proposta.html (+900 linhas, -244 linhas)

### 1.1 Revisão de Segurança e Qualidade
- Lucide fixado em v0.460.0 com SRI integrity hash (previne supply chain attack)
- 62 inline event handlers removidos → addEventListener
- innerHTML substituído por DOM methods seguros (createElement/textContent)
- Magic numbers documentados com constantes nomeadas (TAXA_PERDA, CONV_BASE, etc.)
- URL params validados com bounds check e proteção contra NaN
- setInterval limpo via visibilitychange (sem memory leak)
- Seletores DOM cacheados (particles no mousemove)

### 1.2 SEO e Acessibilidade
- Schema.org JSON-LD (SoftwareApplication + ofertas)
- ARIA attributes (aria-expanded no FAQ, aria-label no menu)
- Botão mobile close com aria-label="Fechar menu"

### 1.3 Pricing v3 — Modelo de Créditos
- 5 planos: Unitário (R$179), Starter (R$349), Growth (R$697), Performance (R$1.490), Enterprise
- Modelo mudou de "leads inclusos/mês" para "créditos de canal + leads ilimitados"
- Mockup do painel de consumo atualizado
- ROI calculator custoLF atualizado
- Schema.org atualizado com novos preços
- Pricing gate: "14 dias grátis" em vez de "Garantia 30 dias"

### 1.4 Widget de Chat
- **Versão final:** Chat em tempo real com bot do ComercialFlow
- Fluxo simplificado de 3 etapas (INTERESSE → NOME → CONTATO → CONFIRMADO)
- Botões clicáveis para opções numeradas do bot
- Estado mantido no client (stateless no servidor)
- Botão flutuante verde do WhatsApp com pulse animation
- Número WhatsApp atualizado para +55 34 9904-4870

---

## 2. Repositório: leadflow (CRM Imobiliário)

**Branch:** main
**Commits:** 3
**Arquivos alterados:** 5

### 2.1 CORS para Endpoints Públicos
- `/api/public/interest` — CORS com whitelist (leadflow-ia.vercel.app, leadflow-crm.vercel.app)
- `/api/public/properties` — CORS com whitelist + cache headers preservados
- OPTIONS handler para preflight em ambos endpoints
- Middleware atualizado: `/api/public/*` adicionado como rota pública

### 2.2 Fix Billing (bug pré-existente)
- `buy-credits/route.ts` — referenciava `credits_1000` e `credits_5000` que não existem nos ADD_ONS
- `ConsumptionPanel.tsx` — mesma referência inválida
- Corrigido para pacotes reais: credits_500 (R$249), credits_1500 (R$690), credits_3000 (R$1.390)

### 2.3 Aplicabilidade ao LeadFlow
Os seguintes itens podem ser portados para o LeadFlow:
- **CORS pattern** — mesmo padrão para /api/public/* (já aplicado)
- **Middleware /api/public/*** — mesma mudança (já aplicado)
- **Billing fix** — NÃO aplicável, billing do LeadFlow é diferente
- **.trim() nas env vars** — aplicável a qualquer projeto com WhatsApp API

---

## 3. Repositório: comercialflow (CRM B2B)

**Branch:** develop
**Commits desta sessão:** 12
**Arquivos alterados:** 33 (3028 inserções, 1553 remoções — inclui mudanças de outras sessões no período)

### 3.1 Endpoint /api/public/chat (NOVO)
- Funil simplificado de 3 etapas para widget do site
- Stateless — widget envia step/interesse/nome a cada request
- Cria lead automaticamente no CRM com interesse registrado
- Inicia bot WhatsApp após captura do contato
- CORS configurado para leadflow-ia.vercel.app

### 3.2 Endpoint /api/public/leads (melhorado)
- Registra interação no timeline do lead (antes não registrava)
- Formata telefone com formatPhoneDisplay
- Inicia bot WhatsApp automaticamente (antes era fire-and-forget que não executava)
- Mudado de fire-and-forget para await direto com try/catch

### 3.3 Middleware
- `/api/public/*` adicionado como rota pública (antes não estava)

### 3.4 WhatsApp — Bugs Críticos Corrigidos
| Bug | Causa | Fix |
|-----|-------|-----|
| Mensagens não entregues em produção | `WHATSAPP_PHONE_NUMBER_ID` com `\n` no final | `.trim()` em todas env vars do whatsapp.ts |
| Chat interno não entregava ao lead | Número salvo como `034984260970` (com zero, sem 55) | Normalização no chat/send: remove zero, adiciona 55 |
| Bot não iniciava pelo widget | fire-and-forget encerrava antes do bot executar | Mudado para await direto |
| Template não entregava | Template na WABA errada (conta sem telefone) | Recriado na WABA correta (ID: 1456149002553180) |
| Frontend sem feedback de erro | catch block só fazia console.log | Alert com mensagem de erro |
| fetch sem credentials | Cookie de sessão podia não ser enviado | Adicionado credentials: 'include' |

### 3.5 Bot Orchestrator
- Fallback para texto simples quando template WhatsApp falha
- Tenta sendTemplateMessage → se falhar → sendTextMessage

---

## 4. Configurações Meta Business (WhatsApp)

### Template leadbot_greeting
- **Conta WABA:** ComercialFlow - ID: 1456149002553180
- **Número:** +55 34 9904-4870
- **Status:** Active
- **Categoria:** Marketing
- **Idioma:** pt_BR
- **Botões:** "Sim, quero conhecer" / "Agora não, obrigado"

### Webhook
- **URL:** https://comercialflow.vercel.app/api/webhooks/whatsapp
- **Status:** Verificado e ativo

### Observações
- Template Marketing não entrega sem verificação de negócio (is_official_business_account: false)
- Fallback para texto simples funciona dentro da janela de 24h
- Número do lead no banco pode ter formato inconsistente (com/sem zero, com/sem 55)

---

## 5. O que Portar para o LeadFlow (Merge Candidates)

### Alta prioridade
1. **`.trim()` nas env vars do WhatsApp** — mesmo bug pode existir no LeadFlow
2. **Normalização de número BR no chat/send** — remover zero do DDD, adicionar 55
3. **CORS nos endpoints /api/public/*** — já foi feito nesta sessão
4. **Middleware /api/public/*** — já foi feito nesta sessão

### Média prioridade
5. **Fallback texto quando template falha** — bot orchestrator
6. **credentials: 'include' no WhatsAppChat.tsx** — garantir envio de cookie
7. **Alert no catch do handleSend** — feedback visual de erro

### Verificar
8. **Billing packages** — verificar se LeadFlow tem referências a pacotes inexistentes (já corrigido nesta sessão)
9. **Formato de número no BotConversation** — verificar se leads salvam com formato consistente

---

## 6. Status Final

| Sistema | Status |
|---------|--------|
| Widget no site (leadflow-ia.vercel.app) | ✅ Funcionando — chat em tempo real |
| Lead criado no ComercialFlow | ✅ Funcionando |
| Bot WhatsApp (lead manda primeiro) | ✅ Funcionando |
| Chat interno (atendente → lead) | ✅ Funcionando (após normalização de número) |
| Template leadbot_greeting | ⚠️ Aprovado mas não entrega (conta não verificada) |
| Pricing no site | ✅ Atualizado — 5 planos, modelo de créditos |
| Segurança do site | ✅ Revisado — SRI, ARIA, sem innerHTML |
