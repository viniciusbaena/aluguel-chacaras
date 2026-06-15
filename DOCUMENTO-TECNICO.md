# Documento Técnico — AluguelDeChácaras.com.br
## Guia para implementação via Cowork

---

## 1. VISÃO GERAL DO PROJETO

Plataforma de marketplace para aluguel de chácaras para eventos em Maringá e região do Paraná. O modelo de negócio é baseado em **comissão de 8%** por reserva confirmada e paga, descontada automaticamente do repasse ao proprietário.

---

## 2. FLUXO PRINCIPAL DE RESERVA

```
1. Cliente visita o site e encontra uma chácara
2. Cliente preenche formulário de solicitação (check-in, check-out, evento, contato)
3. Sistema notifica o proprietário via:
   - WhatsApp (mensagem automática via Z-API)
   - E-mail (via SMTP ou serviço de e-mail)
4. Proprietário confirma ou recusa (pode responder por):
   - WhatsApp (IA interpreta a resposta via Claude Haiku)
   - E-mail (link de confirmação/recusa)
   - Painel web (botão confirmar/recusar)
5. Se confirmado → cliente recebe notificação para pagar
6. Cliente paga via Mercado Pago (cartão, Pix ou boleto)
7. Mercado Pago faz split automático:
   - 8% fica na conta da plataforma
   - 92% fica retido até o check-out
8. Após check-out → 92% transferido para conta MP do proprietário
9. Sistema libera endereço completo e telefone do proprietário para o cliente
10. Calendário da chácara é atualizado automaticamente com a data bloqueada
```

---

## 3. STACK TECNOLÓGICA DEFINIDA

| Componente | Tecnologia | Função |
|---|---|---|
| Frontend | HTML/CSS/JS (GitHub Pages) | Site público e painéis |
| Backend/Automação | N8N self-hosted (VPS) | Orquestração de todos os fluxos |
| WhatsApp API | Z-API | Envio e recebimento de mensagens |
| IA interpretadora | Claude Haiku 4.5 (Anthropic API) | Interpreta respostas do proprietário no WhatsApp |
| Pagamentos | Mercado Pago + split automático | Cobrança e repasse com comissão |
| Banco de dados | A definir (sugestão: Supabase ou Airtable) | Armazenar reservas, proprietários, clientes |
| E-mail | A definir (sugestão: Resend ou SendGrid) | Notificações por e-mail |

---

## 4. FLUXOS N8N A CONFIGURAR

### Fluxo 1 — Nova solicitação de reserva
```
Trigger: webhook recebe dados do formulário do site
→ Salva reserva no banco com status "pendente"
→ Envia WhatsApp ao proprietário via Z-API:
   "Olá João! Nova solicitação de reserva na AluguelDeChácaras.com.br
    Cliente: Maria Fernanda
    Check-in: 20/07/2026 | Check-out: 22/07/2026 (3 dias)
    Evento: Aniversário | ~80 pessoas
    Valor: R$ 3.600
    Mensagem: 'Será uma festa surpresa...'
    Responda: CONFIRMAR ou RECUSAR"
→ Envia e-mail ao proprietário com botões de ação
→ Agenda verificação de expiração em 24h
```

### Fluxo 2 — Resposta do proprietário via WhatsApp
```
Trigger: Z-API recebe mensagem no número da plataforma
→ Envia mensagem para Claude Haiku API:
   "O proprietário respondeu: '[mensagem]'. 
    Ele está confirmando ou recusando a reserva? 
    Responda apenas: CONFIRMAR ou RECUSAR"
→ Claude retorna CONFIRMAR ou RECUSAR
→ Se CONFIRMAR:
   - Atualiza status da reserva para "confirmada"
   - Notifica cliente via WhatsApp e e-mail para pagar
   - Gera link de pagamento Mercado Pago
→ Se RECUSAR:
   - Atualiza status para "recusada"
   - Notifica cliente com motivo
```

### Fluxo 3 — Pagamento confirmado pelo Mercado Pago
```
Trigger: webhook do Mercado Pago (payment.approved)
→ Atualiza status da reserva para "paga"
→ Bloqueia as datas no calendário da chácara
→ Envia ao cliente:
   - WhatsApp com endereço completo e telefone do proprietário
   - E-mail de confirmação com todos os detalhes
→ Envia ao proprietário:
   - WhatsApp confirmando pagamento recebido
   - Dados do cliente (nome, WhatsApp)
```

### Fluxo 4 — Expiração de solicitação (24h sem resposta)
```
Trigger: agendamento após 24h da solicitação
→ Verifica se reserva ainda está "pendente"
→ Se sim:
   - Atualiza status para "expirada"
   - Notifica cliente que o proprietário não respondeu
   - Sugere outras chácaras disponíveis
   - Notifica proprietário que perdeu a reserva
```

### Fluxo 5 — Repasse após check-out
```
Trigger: agendamento para o dia seguinte ao check-out
→ Verifica se reserva foi concluída
→ Libera repasse de 92% para conta MP do proprietário
→ Envia comprovante ao proprietário
→ Solicita avaliação ao cliente
```

---

## 5. CONFIGURAÇÃO Z-API

- Criar conta em z-api.io
- Conectar número WhatsApp da plataforma via QR Code
- Configurar webhook para receber mensagens no N8N
- Número sugerido: número exclusivo para a plataforma (não o pessoal)
- Configurar mensagens de template para: notificação de reserva, confirmação, pagamento, endereço liberado

---

## 6. CONFIGURAÇÃO MERCADO PAGO

- Criar conta Mercado Pago para a plataforma (CNPJ ou CPF)
- Habilitar Marketplace/Split de pagamento
- Cada proprietário precisa ter conta no Mercado Pago e vincular via OAuth
- Configurar webhook para eventos: payment.approved, payment.cancelled
- Comissão: 8% para a plataforma, 92% para o proprietário (retido até check-out)
- Taxa do MP (~5%) é descontada do valor total antes do split

---

## 7. BANCO DE DADOS — ESTRUTURA SUGERIDA

### Tabela: chácaras
```
id, proprietario_id, nome, cidade, bairro, descricao,
capacidade, valor_diaria, valor_fds, checkin_hora, checkout_hora,
fotos[], tags[], tipos_evento[], regras,
whatsapp, email, status (ativo/inativo),
mp_access_token (token OAuth do proprietário no MP),
criado_em
```

### Tabela: reservas
```
id, chacara_id, cliente_nome, cliente_whatsapp, cliente_email,
checkin, checkout, dias, valor_total, comissao (8%), valor_liquido,
tipo_evento, num_pessoas, mensagem_cliente,
status (pendente/confirmada/paga/concluida/recusada/expirada),
mp_payment_id, criado_em, confirmado_em, pago_em
```

### Tabela: proprietários
```
id, nome, cpf, email, whatsapp, mp_access_token, criado_em
```

### Tabela: clientes
```
id, nome, email, whatsapp, criado_em
```

---

## 8. CONFIGURAÇÃO VPS PARA N8N

- Provedor sugerido: DigitalOcean, Contabo ou Hetzner (~R$ 35/mês)
- Ubuntu 22.04 LTS
- Instalar Docker + Docker Compose
- Instalar N8N via Docker
- Configurar domínio/subdomínio: n8n.alugueldecharas.com.br
- Configurar SSL (Let's Encrypt)
- Configurar backup automático dos workflows

---

## 9. VARIÁVEIS DE AMBIENTE NECESSÁRIAS

```env
# N8N
N8N_HOST=n8n.alugueldecharas.com.br
N8N_PROTOCOL=https

# Z-API
ZAPI_INSTANCE_ID=xxx
ZAPI_TOKEN=xxx
ZAPI_PHONE=5544999999999

# Claude API
ANTHROPIC_API_KEY=xxx
CLAUDE_MODEL=claude-haiku-4-5

# Mercado Pago
MP_ACCESS_TOKEN=xxx
MP_CLIENT_ID=xxx
MP_CLIENT_SECRET=xxx
MP_WEBHOOK_SECRET=xxx

# Banco de dados
DATABASE_URL=xxx

# E-mail
SMTP_HOST=xxx
SMTP_USER=xxx
SMTP_PASS=xxx
FROM_EMAIL=noreply@alugueldecharas.com.br

# Site
SITE_URL=https://alugueldecharas.com.br
COMMISSION_RATE=0.08
```

---

## 10. PÁGINAS DO SITE (já desenvolvidas)

| Arquivo | URL | Descrição |
|---|---|---|
| index.html | / | Página inicial com busca e filtros |
| chacara.html | /chacara | Página individual da chácara + fluxo de reserva |
| fornecedor.html | /fornecedor | Página individual do fornecedor |
| painel-proprietario.html | /painel-proprietario | Painel completo do proprietário |
| painel-fornecedor.html | /painel-fornecedor | Painel completo do fornecedor |

---

## 11. PRÓXIMOS PASSOS (ordem sugerida)

1. **Provisionar VPS** e instalar N8N
2. **Configurar Z-API** e conectar número WhatsApp da plataforma
3. **Configurar Mercado Pago** com split e webhooks
4. **Criar banco de dados** (Supabase recomendado — gratuito para começar)
5. **Implementar Fluxo 1** (solicitação de reserva)
6. **Implementar Fluxo 2** (resposta do proprietário via WhatsApp com Claude)
7. **Implementar Fluxo 3** (pagamento confirmado)
8. **Implementar Fluxo 4** (expiração 24h)
9. **Implementar Fluxo 5** (repasse após check-out)
10. **Conectar frontend** com os webhooks do N8N
11. **Testes end-to-end** com reserva real
12. **Go live** 🚀

---

## 12. CUSTOS MENSAIS ESTIMADOS

| Serviço | Custo |
|---|---|
| VPS (N8N self-hosted) | ~R$ 35/mês |
| Z-API (WhatsApp) | ~R$ 70/mês |
| Claude Haiku API | ~R$ 20/mês |
| Mercado Pago | ~5% por transação |
| Supabase (banco) | Grátis para começar |
| Resend (e-mail) | Grátis até 3.000/mês |
| GitHub Pages (site) | Grátis |
| **Total fixo** | **~R$ 125/mês** |

