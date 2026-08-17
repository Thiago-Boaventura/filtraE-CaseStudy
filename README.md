<div align="center">

# FiltraÊ — Triagem inteligente de clientes via WhatsApp com IA

**Case study de um SaaS B2B em produção, construído solo do zero: arquitetura, IA, tempo real e pagamentos.**

![NestJS](https://img.shields.io/badge/NestJS-E0234E?style=for-the-badge&logo=nestjs&logoColor=white)
![React](https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white)
![Prisma](https://img.shields.io/badge/Prisma-2D3748?style=for-the-badge&logo=prisma&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Socket.io](https://img.shields.io/badge/Socket.io-010101?style=for-the-badge&logo=socket.io&logoColor=white)
![Google Gemini](https://img.shields.io/badge/Google%20Gemini-4285F4?style=for-the-badge&logo=google&logoColor=white)

</div>

---

## O problema

Pequenos e médios negócios que atendem pelo WhatsApp perdem clientes por dois motivos opostos: demoram para responder fora do horário comercial, ou respondem rápido demais com um atendente que gasta os primeiros minutos da conversa fazendo perguntas repetitivas ("qual seu nome?", "o que você precisa?") antes de conseguir ajudar de fato.

O **FiltraÊ** resolve isso automatizando só a parte que vale a pena automatizar: o primeiro contato. Uma IA assume a conversa 24/7, conduz a triagem com base em um prompt configurável pelo próprio usuário (sem código), e transfere para o responsável humano só quando já tem contexto suficiente — nome, interesse, dados relevantes — entregue como um resumo, não como um histórico cru para o atendente garimpar.

```
Cliente manda mensagem no WhatsApp
        ↓
IA assume o atendimento (24/7)
        ↓
Conduz a conversa com base no prompt configurado pelo usuário
        ↓
Transfere para o humano + resumo da conversa
```

## Visão geral do produto

- **Multi-tenant**: cada usuário conecta o próprio número de WhatsApp (via QR code) e configura seu próprio prompt de IA, plano e equipe — sessões e dados isolados por conta.
- **Painel em tempo real**: atendentes acompanham conversas, métricas e status da conexão do WhatsApp ao vivo, via WebSocket.
- **Cobrança recorrente**: planos com checkout via PIX e cartão, integrado a um gateway de pagamentos brasileiro.
- **Landing page própria** para aquisição de clientes, com formulário de contato e conteúdo institucional.

## Arquitetura

Monorepo com **npm workspaces**, três aplicações e pacotes compartilhados:

```
├── apps/
│   ├── api/     → Backend: NestJS + Prisma + PostgreSQL
│   ├── admin/   → Painel do cliente: React + TanStack + Zustand
│   └── web/     → Landing page: React + Vite
├── packages/    → Tipos, utils e UI compartilhados entre apps
└── docker-compose.yml
```

A API é organizada em módulos de domínio (auth, whatsapp, conversations, ai, billing, settings, dashboard, audit...), cada um com responsabilidade única — o que manteve o backend navegável mesmo crescendo para mais de uma dúzia de módulos.

## Stack

| Camada | Tecnologias |
|---|---|
| Backend | NestJS, Prisma ORM, PostgreSQL, Socket.io, JWT + Argon2 |
| Sessão WhatsApp | whatsapp-web.js sobre Puppeteer (Chromium headless) |
| IA | Google Gemini — geração de respostas e resumos de conversa |
| Painel (admin) | React 18, TanStack Router/Query, Zustand, Tailwind CSS v4, Framer Motion |
| Landing (web) | React 18, Vite, Tailwind CSS v3 |
| Pagamentos | Gateway brasileiro (PIX + cartão), com tokenização de cartão |
| Infra | Docker, Nginx, deploy em VPS |

## Desafios técnicos

Um sistema que fala com clientes reais em tempo real via WhatsApp tem uma característica que não aparece em CRUD comum: **concorrência e estado de sessão são o problema central**, não um detalhe de implementação. Alguns dos desafios mais interessantes:

**1. Sessão do WhatsApp como recurso frágil e stateful**
whatsapp-web.js mantém uma sessão de navegador (Puppeteer) por usuário conectado — não é uma API stateless. Quedas de conexão são esperadas, não exceção. A solução foi reconexão automática com **backoff exponencial** (30s → 60s → 120s → 240s, com teto e contagem de tentativas por usuário), evitando tanto reconexões agressivas que sobrecarregam o processo quanto sessões mortas que ninguém percebe.

**2. Evitar a IA respondendo em duplicidade**
Quando um cliente manda várias mensagens em sequência rápida ("Oi", "tudo bem?", "queria saber sobre X"), processar cada uma isoladamente gera respostas fragmentadas e, pior, pipelines de IA rodando em paralelo para o mesmo contato — cada um gerando uma resposta diferente. A solução combina **debounce** (agrupa mensagens que chegam em uma janela curta em um único texto) com um **lock de concorrência por conversa** (se já existe um pipeline de IA em andamento para aquele contato, novas mensagens são reenfileiradas em vez de disparar um segundo pipeline). O lock é sempre liberado no `finally`, mesmo em caso de erro na chamada à IA.

**3. Handoff IA → humano sem perda de contexto**
A transferência de uma conversa da IA para um atendente humano precisa acontecer sem o cliente perceber uma "troca de sistema" e sem o atendente reabrir a conversa do zero. Isso significa: distinguir com precisão uma resposta automática de um sistema terceiro (bot de outro serviço, notificação automática) de uma mensagem real do cliente, e garantir que o estado da conversa (quem está respondendo — IA ou humano) fique sempre consistente mesmo quando o atendente inicia o contato antes do cliente responder.

**4. Compliance em pagamentos sem virar o foco do produto**
Para cobrança recorrente por cartão, dados sensíveis nunca tocam a lógica de negócio: o número do cartão é **tokenizado no gateway de pagamento como primeira operação**, e só o token segue para a criação da assinatura — nada de dado bruto de cartão persistido ou logado.

## Meu papel

Projeto construído solo, do zero, incluindo:
- Arquitetura do monorepo e dos módulos da API
- Design e implementação do pipeline de IA (prompt configurável, geração de resposta, resumo de transferência)
- Integração com WhatsApp via Puppeteer e toda a resiliência de sessão/reconexão
- Painel administrativo em tempo real (React + Socket.io)
- Modelagem de dados multi-tenant no PostgreSQL/Prisma
- Integração de pagamentos (PIX/cartão) e faturamento recorrente
- Deploy, Docker e operação em VPS de produção

## Aprendizados

- Em sistemas conversacionais com IA, o desafio raramente é "fazer a IA responder bem" — é orquestrar **quando** ela deve responder, quando deve calar e passar a bola, e como não duplicar trabalho quando eventos chegam fora de ordem.
- Concorrência real (mensagens simultâneas, múltiplas sessões de WhatsApp, múltiplos tenants) expõe bugs que testes unitários isolados não pegam — boa parte dos ajustes mais delicados do projeto vieram de comportamento observado em produção, não de especificação prévia.
- Compliance (LGPD, dados de pagamento) é mais barato de tratar como restrição de arquitetura desde o início do que como retrofit depois que dado sensível já vazou para logs ou banco.

---

> Screenshots do painel (dashboard, conversas, conexão do WhatsApp, configurações — modo claro e escuro) podem ser adicionados a este repositório antes da publicação, usando os prints já existentes do produto que não contenham dados de clientes reais.
