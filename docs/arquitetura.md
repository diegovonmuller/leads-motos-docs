# Arquitetura — Qualificação de Leads via WhatsApp

## Visão geral

Plataforma B2B SaaS de qualificação automática de leads para redes de concessionárias de motos. O bot recebe o lead via WhatsApp, consulta o CPF em tempo real, classifica o perfil e encaminha automaticamente para o vendedor (lead limpo) ou nurturing (lead sujo).

---

## Stack tecnológica

| Camada | Tecnologia | Função |
|--------|-----------|--------|
| Runtime | Node.js + TypeScript | Backend da aplicação |
| WhatsApp | Evolution API (Meta Business) | Canal de entrada e saída |
| Qualificação | BigDataCorp API | Consulta de CPF / score de crédito |
| Cache | Redis | Evitar re-consultas, sessões de conversa |
| Banco de dados | PostgreSQL | Persistência de leads, histórico, lojas |
| Orquestrador | N8N | Fluxos de automação e integrações |
| IA | Claude API (Haiku) | Interpretação de mensagens, geração de respostas |
| Infraestrutura | AWS (EC2 + RDS + ElastiCache) | Hospedagem escalável |

---

## Fluxo principal

```
Tráfego pago (Meta Ads)
        │
        ▼
   WhatsApp (lead entra)
        │
        ▼
  Evolution API recebe mensagem
        │
        ▼
   N8N orquestra o fluxo
        │
        ▼
  Bot coleta dados via Claude Haiku
  (nome, CPF, interesse, valor pretendido)
        │
        ▼
   Redis — verifica cache do CPF
        │
   ┌────┴────┐
   │ Hit     │ Miss
   │         ▼
   │   BigDataCorp API
   │   consulta CPF em tempo real
   │         │
   └────┬────┘
        │
   Classificação do lead
        │
   ┌────┴────────────────┐
   │                     │
   ▼                     ▼
LEAD LIMPO          LEAD SUJO
(score OK,          (restrições,
 sem restrições)     score baixo)
   │                     │
   ▼                     ▼
Encaminha para      Entra em fluxo
vendedor humano     de nurturing
via WhatsApp        (N8N + mensagens
                     automáticas)
        │
        ▼
   PostgreSQL
   salva tudo
   (lead, score, classificação,
    loja, timestamp, resultado)
```

---

## Arquitetura de serviços

```
┌─────────────────────────────────────────────────────────┐
│                        AWS                              │
│                                                         │
│  ┌──────────────┐    ┌──────────────┐                  │
│  │   EC2 (App)  │    │  EC2 (N8N)   │                  │
│  │  Node.js +   │◄──►│  Orquestrador│                  │
│  │  TypeScript  │    │  de fluxos   │                  │
│  └──────┬───────┘    └──────────────┘                  │
│         │                                               │
│  ┌──────▼───────┐    ┌──────────────┐                  │
│  │  ElastiCache │    │  RDS         │                  │
│  │  (Redis)     │    │  (PostgreSQL)│                  │
│  └──────────────┘    └──────────────┘                  │
└─────────────────────────────────────────────────────────┘
         │                        │
         ▼                        ▼
  Evolution API            BigDataCorp API
  (WhatsApp Meta)          (consulta CPF)
         │
         ▼
  Claude API (Haiku)
  (interpretação IA)
```

---

## Estrutura do projeto

```
leads-motos/
├── src/
│   ├── bot/           # Lógica de conversa e coleta de dados
│   ├── services/      # Integrações (BigDataCorp, Evolution, Claude)
│   ├── queue/         # Filas de processamento
│   ├── db/            # Models PostgreSQL e Redis
│   └── api/           # Endpoints internos (webhooks, painel)
├── config/            # Variáveis de ambiente e configurações
├── scripts/           # Deploy, migrations, seed
├── docs/              # Documentação técnica
└── n8n/               # Exports dos fluxos N8N
```

---

## Modelo de negócio

### Precificação

| Item | Valor |
|------|-------|
| Licença por loja/mês | R$ 950,00 |
| Meta API (WhatsApp) | No cartão do cliente |
| BigDataCorp por consulta | Custo operacional (~R$0,30–R$1,00/CPF) |

### Projeção — 300 lojas

| Métrica | Valor |
|---------|-------|
| Receita bruta | R$ 285.000/mês |
| Custo estimado (infra + APIs) | ~R$ 28.500/mês (10%) |
| **Margem estimada** | **~R$ 256.500/mês** |

### Responsabilidades

- **Cliente (loja):** paga a Meta API diretamente — sem risco de inadimplência na infraestrutura do WhatsApp
- **Plataforma:** cobra somente a licença de uso + suporte

---

## Decisões arquiteturais

**Por que Evolution API?**  
Suporte oficial à Meta Business API com webhook nativo, multi-instância por loja e SDK bem documentado.

**Por que Redis para cache de CPF?**  
Evita re-consultas ao BigDataCorp para o mesmo CPF dentro de uma janela de tempo (ex: 24h), reduzindo custo operacional.

**Por que Claude Haiku?**  
Menor custo por token no portfólio Anthropic, latência baixa (~200ms), ideal para interpretação de mensagens em fluxo de chat em tempo real.

**Por que N8N como orquestrador?**  
Permite que configurações de fluxo por loja (horários, mensagens personalizadas, escaladas) sejam modificadas sem deploys de código.

---

## Variáveis de ambiente necessárias

```env
# WhatsApp
EVOLUTION_API_URL=
EVOLUTION_API_KEY=

# BigDataCorp
BIGDATACORP_TOKEN=

# Claude
ANTHROPIC_API_KEY=

# Banco de dados
DATABASE_URL=
REDIS_URL=

# Aplicação
PORT=3000
NODE_ENV=production
```

---

## Roadmap inicial

- [ ] Setup infraestrutura AWS (EC2, RDS, ElastiCache)
- [ ] Integração Evolution API + webhook
- [ ] Fluxo de coleta de dados via bot (Claude Haiku)
- [ ] Integração BigDataCorp (consulta CPF + cache Redis)
- [ ] Roteamento lead limpo → vendedor / lead sujo → nurturing
- [ ] Persistência PostgreSQL (leads, lojas, resultados)
- [ ] Painel básico por loja (histórico de leads)
- [ ] N8N: fluxos de nurturing e follow-up
- [ ] Onboarding multi-loja (instância por loja na Evolution API)
- [ ] Dashboard de métricas (conversão, volume, score médio)
