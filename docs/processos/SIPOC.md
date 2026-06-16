# SIPOC — MotoQualify

**Plataforma de Qualificação de Leads via WhatsApp para Rede de Motos**

> Versão: 1.0 | Data: 2026-06-16 | Status: Ativo

---

## Visão Geral

O SIPOC descreve o processo de ponta a ponta da qualificação de leads gerados por anúncios pagos, passando pela conversa automatizada no WhatsApp, consulta de CPF e entrega do lead qualificado ao vendedor da loja.

---

## Tabela SIPOC

| **S — Suppliers** | **I — Inputs** | **P — Process** | **O — Outputs** | **C — Customers** |
|---|---|---|---|---|
| Meta Ads | Lead: nome, CPF, WhatsApp | 1. Lead clica no anúncio e inicia conversa no WhatsApp | Lead qualificado (CPF limpo) → notificação ao vendedor | Vendedor da loja |
| Google Ads | Dados do anúncio (UTM, origem) | 2. Bot apresenta termo de consentimento LGPD | Lead pendente (CPF sujo) → mensagem suave + notificação ao vendedor | Gestor da loja |
| WhatsApp Business API (Meta) | Score de crédito (BigDataCorp) | 3. Lead aceita ou recusa o consentimento | Registro completo no Google Sheets | Dono da rede |
| BigDataCorp | Consentimento LGPD (aceite ou recusa) | 4. Bot coleta nome completo | Histórico persistente no PostgreSQL | Empresa de dados (uso futuro, com consentimento) |
| Google Sheets | Histórico de consultas (Redis / PostgreSQL) | 5. Bot coleta CPF e valida formato | Cache de deduplicação no Redis | — |
| — | — | 6. Bot confirma os dados com o lead | — | — |
| — | — | 7. Sistema verifica anti-duplicidade (Redis / PostgreSQL) | — | — |
| — | — | 8. Sistema consulta CPF na BigDataCorp | — | — |
| — | — | 9. Sistema classifica lead: aprovado ou pendente | — | — |
| — | — | 10. Sistema notifica grupo da loja no WhatsApp | — | — |
| — | — | 11. Sistema registra lead no Google Sheets | — | — |
| — | — | 12. Sistema salva dados no PostgreSQL | — | — |

---

## Detalhamento do Processo

### Fluxo Principal

```
[Anúncio] ──► [Início da conversa no WhatsApp]
                        │
                        ▼
              [Apresentação do termo LGPD]
                        │
              ┌─────────┴─────────┐
              │ Aceita            │ Recusa
              ▼                   ▼
     [Coleta nome]        [Encerra conversa]
              │                   │
              ▼                   ▼
     [Coleta CPF]        [Registra recusa no DB]
              │
              ▼
     [Validação de formato do CPF]
              │
     ┌────────┴────────┐
     │ Válido          │ Inválido
     ▼                 ▼
[Confirmação]   [Solicita CPF novamente]
     │
     ▼
[Anti-duplicidade — Redis/PostgreSQL]
     │
     ┌──────────────────┐
     │ Já existe        │ Novo
     ▼                  ▼
[Informa lead]   [Consulta BigDataCorp]
[já cadastrado]         │
                ┌───────┴───────┐
                │ CPF limpo     │ CPF sujo
                ▼               ▼
         [Aprovado]        [Pendente]
                │               │
                └───────┬───────┘
                        ▼
            [Notifica grupo da loja]
                        │
                        ▼
            [Registra no Google Sheets]
                        │
                        ▼
            [Salva no PostgreSQL]
                        │
                        ▼
              [Cache no Redis]
```

---

## Descrição Detalhada das Etapas

| # | Etapa | Responsável | Sistema | SLA |
|---|---|---|---|---|
| 1 | Lead clica no anúncio e inicia conversa no WhatsApp | Automático | Meta / Google Ads | Imediato |
| 2 | Bot apresenta consentimento LGPD | Bot | WhatsApp Business API | < 1s |
| 3 | Lead aceita ou recusa consentimento | Lead | WhatsApp | — |
| 4 | Bot coleta nome completo | Bot | WhatsApp Business API | < 1s |
| 5 | Bot coleta CPF e valida formato | Bot | WhatsApp Business API | < 1s |
| 6 | Bot confirma os dados com o lead | Bot | WhatsApp Business API | < 1s |
| 7 | Verificação de anti-duplicidade | Sistema | Redis / PostgreSQL | < 100ms |
| 8 | Consulta de CPF na BigDataCorp | Sistema | BigDataCorp API | < 3s |
| 9 | Classificação do lead | Sistema | Lógica interna | < 50ms |
| 10 | Notificação ao grupo da loja | Sistema | WhatsApp Business API | < 2s |
| 11 | Registro no Google Sheets | Sistema | Google Sheets API | < 2s |
| 12 | Persistência no PostgreSQL | Sistema | PostgreSQL | < 100ms |

---

## Suppliers — Fornecedores

| Fornecedor | Tipo | Responsabilidade |
|---|---|---|
| **Meta Ads** | Tráfego pago | Gera leads via campanhas no Instagram e Facebook |
| **Google Ads** | Tráfego pago | Gera leads via campanhas de pesquisa e display |
| **WhatsApp Business API (Meta)** | Canal de comunicação | Canal oficial para o fluxo conversacional |
| **BigDataCorp** | Consulta de crédito | Retorna score e situação do CPF |
| **Google Sheets** | Registro e gestão | Planilha de leads para acompanhamento da equipe |

---

## Inputs — Entradas

| Input | Origem | Obrigatório |
|---|---|---|
| Nome completo do lead | Fornecido pelo lead no chat | Sim |
| CPF do lead | Fornecido pelo lead no chat | Sim |
| Número de WhatsApp | Capturado automaticamente | Sim |
| Dados do anúncio (UTM, origem) | Meta / Google Ads | Não (enriquecimento) |
| Score de crédito | BigDataCorp | Sim |
| Aceite de consentimento LGPD | Fornecido pelo lead | Sim |
| Histórico de consultas | Redis / PostgreSQL | Sim (anti-duplicidade) |

---

## Outputs — Saídas

| Output | Destino | Condição |
|---|---|---|
| Lead **Aprovado** (CPF limpo) | Vendedor da loja via WhatsApp | Score positivo na BigDataCorp |
| Lead **Pendente** (CPF sujo) | Vendedor da loja via WhatsApp | Score negativo na BigDataCorp |
| Registro no **Google Sheets** | Gestor da loja | Sempre (aprovado ou pendente) |
| Histórico no **PostgreSQL** | Sistema interno | Sempre |
| Cache no **Redis** (30 dias — CPF limpo / 7 dias — CPF sujo) | Sistema interno | Sempre |

---

## Customers — Clientes

| Cliente | Papel | Necessidade |
|---|---|---|
| **Vendedor da loja** | Receptor do lead | Receber notificação imediata com dados do lead |
| **Gestor da loja** | Acompanhamento operacional | Visualizar todos os leads no Google Sheets em tempo real |
| **Dono da rede** | Visão estratégica | Relatórios consolidados de performance e conversão |
| **Empresa de dados** | Uso futuro | Acesso aos dados apenas com consentimento LGPD explícito |

---

## Conformidade LGPD

> **Base legal:** Consentimento (Art. 7º, I, LGPD)

| Requisito | Implementação |
|---|---|
| Consentimento explícito | Etapa obrigatória antes de qualquer coleta de dado (passo 2 e 3) |
| Finalidade declarada | Informada no termo apresentado pelo bot |
| Minimização de dados | Coleta apenas nome, CPF e WhatsApp |
| Direito de recusa | Lead pode recusar a qualquer momento; conversa é encerrada sem registro |
| Retenção de dados | Definida por política interna (Redis: 7–30 dias; PostgreSQL: conforme política) |
| Portabilidade e exclusão | Previsto no roadmap (endpoint de exclusão de dados) |
| Registro de consentimento | Timestamp de aceite salvo no PostgreSQL junto ao registro do lead |

---

## Métricas de Sucesso do Processo

| Métrica | Descrição | Meta |
|---|---|---|
| **Taxa de conversão do anúncio** | Leads que iniciam conversa / cliques no anúncio | > 20% |
| **Taxa de aceite LGPD** | Leads que aceitam o consentimento / leads que iniciam | > 85% |
| **Taxa de conclusão do fluxo** | Leads que completam CPF e confirmação / aceitam LGPD | > 90% |
| **Taxa de duplicidade bloqueada** | Leads duplicados detectados / total de tentativas | Monitoramento contínuo |
| **Taxa de aprovação** | Leads com CPF limpo / leads consultados | Referência de mercado |
| **Tempo médio de qualificação** | Tempo entre início da conversa e notificação ao vendedor | < 60 segundos |
| **Disponibilidade do bot** | Uptime do serviço | > 99,5% |
| **Latência da consulta BigDataCorp** | Tempo de resposta da API de crédito | < 3 segundos |

---

## Observações Técnicas

- **Anti-duplicidade:** O Redis é a primeira camada de verificação (< 100ms). O PostgreSQL é a fonte de verdade persistente. Um lead com o mesmo CPF dentro do TTL definido não gera nova consulta na BigDataCorp.
- **Retry e fallback:** Em caso de falha na consulta à BigDataCorp, o lead é classificado como **pendente** e o vendedor é notificado com flag de "consulta indisponível".
- **Idempotência:** Todas as etapas de persistência são idempotentes para evitar registros duplicados em caso de reprocessamento.
- **Dados sensíveis:** O CPF é armazenado criptografado no PostgreSQL e nunca é exibido integralmente nas notificações ao WhatsApp.
