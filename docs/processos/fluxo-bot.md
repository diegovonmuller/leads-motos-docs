# Fluxo do Bot WhatsApp — MotoQualify

> Versão: 1.0 | Data: 2026-06-16 | Status: Ativo

---

## Visão Geral dos Estados

O bot opera como uma máquina de estados. Cada lead possui um estado persistido no Redis que determina qual etapa da conversa está ativa.

```
IDLE → AWAITING_CONSENT → AWAITING_NAME → AWAITING_CPF
     → CONFIRMING_DATA → PROCESSING → COMPLETED / REJECTED / DUPLICATE
```

---

## Diagrama Completo do Fluxo

```
[Lead envia qualquer mensagem]
            │
            ▼
   [Estado: IDLE ou novo lead]
            │
            ▼
┌───────────────────────────────────┐
│  BOT: Apresenta termo de          │
│  consentimento LGPD               │
│  Estado → AWAITING_CONSENT        │
└───────────────────────────────────┘
            │
    ┌───────┴────────┐
    │                │
    ▼                ▼
[Aceita]         [Recusa / inválido]
    │                │
    │                ▼
    │     ┌──────────────────────────┐
    │     │ BOT: Mensagem de recusa  │
    │     │ Estado → REJECTED        │
    │     │ Registra recusa no DB    │
    │     └──────────────────────────┘
    │
    ▼
┌───────────────────────────────────┐
│  BOT: Solicita nome completo      │
│  Estado → AWAITING_NAME           │
└───────────────────────────────────┘
            │
    ┌───────┴────────┐
    │                │
    ▼                ▼
[Nome válido]    [Timeout 10min]
    │                │
    │                ▼
    │     ┌──────────────────────────┐
    │     │ Estado → IDLE            │
    │     │ Redis TTL expirado        │
    │     └──────────────────────────┘
    │
    ▼
┌───────────────────────────────────┐
│  BOT: Solicita CPF                │
│  Estado → AWAITING_CPF            │
└───────────────────────────────────┘
            │
    ┌───────┴──────────────────────┐
    │              │               │
    ▼              ▼               ▼
[CPF válido] [CPF inválido]  [Timeout 10min]
    │              │               │
    │              ▼               ▼
    │    ┌──────────────────┐  [Estado → IDLE]
    │    │ BOT: Informa erro │
    │    │ Solicita novamente│
    │    │ (máx. 3 tentativas)│
    │    └──────────────────┘
    │              │
    │         [3ª falha]
    │              ▼
    │    ┌──────────────────┐
    │    │ BOT: Encerra     │
    │    │ Estado → FAILED  │
    │    └──────────────────┘
    │
    ▼
┌───────────────────────────────────┐
│  BOT: Confirma dados              │
│  Estado → CONFIRMING_DATA         │
└───────────────────────────────────┘
            │
    ┌───────┴────────┐
    │                │
    ▼                ▼
[Confirma]      [Corrigir]
    │                │
    │                ▼
    │     [Volta para AWAITING_NAME]
    │
    ▼
┌───────────────────────────────────┐
│  SISTEMA: Anti-duplicidade        │
│  Verifica Redis → PostgreSQL      │
│  Estado → PROCESSING              │
└───────────────────────────────────┘
            │
    ┌───────┴────────┐
    │                │
    ▼                ▼
[Novo lead]     [Duplicado]
    │                │
    │                ▼
    │     ┌──────────────────────────┐
    │     │ BOT: Informa já cadastrado│
    │     │ Estado → DUPLICATE        │
    │     └──────────────────────────┘
    │
    ▼
┌───────────────────────────────────┐
│  SISTEMA: Consulta BigDataCorp    │
└───────────────────────────────────┘
            │
    ┌───────┴──────────────────┐
    │              │            │
    ▼              ▼            ▼
[CPF limpo]  [CPF sujo]   [API falhou]
    │              │            │
    ▼              ▼            ▼
[Aprovado]   [Pendente]   [Pendente +
    │              │       flag de erro]
    └──────┬────────┘
           │
           ▼
┌───────────────────────────────────┐
│  SISTEMA: Notifica grupo da loja  │
│  Registra Google Sheets           │
│  Salva PostgreSQL                 │
│  Atualiza Redis (TTL)             │
│  Estado → COMPLETED               │
└───────────────────────────────────┘
```

---

## Mensagens Exatas do Bot

### Etapa 1 — Boas-vindas e Consentimento LGPD

```
Olá! Sou o assistente da [Nome da Loja] 🏍️

Para te ajudar com sua consulta sobre financiamento de moto, preciso
coletar algumas informações.

Antes de continuar, preciso do seu consentimento para tratar seus dados
pessoais conforme a Lei Geral de Proteção de Dados (LGPD - Lei 13.709/18).

📋 *Dados coletados:* nome completo e CPF
🎯 *Finalidade:* verificar a elegibilidade para financiamento
🔒 *Armazenamento:* dados protegidos e não compartilhados sem seu consentimento
⏱️ *Retenção:* 30 dias após a consulta

Você concorda com o tratamento dos seus dados?

Digite *SIM* para continuar ou *NÃO* para encerrar.
```

### Etapa 2a — Consentimento Aceito

```
Ótimo! Seu consentimento foi registrado. ✅

Para começar, qual é o seu *nome completo*?
```

### Etapa 2b — Consentimento Recusado

```
Tudo bem! Respeitamos sua decisão. 🙏

Seus dados não serão coletados. Se mudar de ideia, é só nos chamar novamente!

Qualquer dúvida, nossa equipe está disponível. Até mais! 👋
```

### Etapa 3 — Coleta de CPF

```
Obrigado, *{nome}*! 😊

Agora preciso do seu *CPF* (somente números):
```

### Etapa 3a — CPF Inválido (tentativa 1 ou 2)

```
Hmm, esse CPF não parece válido. 🤔

Por favor, digite novamente apenas os *11 números* do CPF, sem pontos ou traços.

Exemplo: 12345678901
```

### Etapa 3b — CPF Inválido (tentativa 3 — encerra)

```
Não conseguimos validar o CPF informado após 3 tentativas.

Por favor, entre em contato diretamente com nossa equipe pelo número:
📞 [número da loja]

Até mais! 👋
```

### Etapa 4 — Confirmação dos Dados

```
Perfeito! Vou confirmar os dados antes de continuar:

👤 *Nome:* {nome_completo}
🪪 *CPF:* {cpf_mascarado}  _(ex: ***.456.789-**)_

As informações estão corretas?

Digite *SIM* para confirmar ou *NÃO* para corrigir.
```

### Etapa 5a — Lead Duplicado

```
Identificamos que você já realizou uma consulta recentemente. 😊

Nossa equipe já tem seus dados e entrará em contato em breve!

Qualquer dúvida, fale conosco: 📞 [número da loja]
```

### Etapa 6a — Lead Aprovado (CPF limpo)

```
Boa notícia! ✅

Sua consulta foi concluída com sucesso. Nossa equipe de vendas receberá
seu contato e entrará em breve para apresentar as melhores condições!

Até logo, *{nome}*! 🏍️
```

### Etapa 6b — Lead Pendente (CPF sujo)

```
Obrigado, *{nome}*! Recebemos sua solicitação. 😊

Nossa equipe analisará seu perfil e entrará em contato para apresentar
as opções disponíveis para você.

Até logo! 🏍️
```

### Etapa 7 — Falha na API BigDataCorp

```
Obrigado, *{nome}*! Recebemos sua solicitação. 😊

Nossa equipe entrará em contato em breve para dar continuidade ao atendimento.

Até logo! 🏍️
```

*(Vendedor recebe flag interno: "consulta indisponível — verificar manualmente")*

---

## Regras de Validação do CPF

```
1. Comprimento exato de 11 dígitos (após remoção de pontuação)
2. Não pode conter todos os dígitos iguais (ex: 111.111.111-11)
3. Dígito verificador 1:
   - Multiplica os 9 primeiros dígitos por [10, 9, 8, 7, 6, 5, 4, 3, 2]
   - Soma os produtos
   - Resto da divisão por 11
   - Se resto < 2: dígito = 0; senão: dígito = 11 - resto
   - Compara com o 10º dígito do CPF
4. Dígito verificador 2:
   - Multiplica os 10 primeiros dígitos por [11, 10, 9, 8, 7, 6, 5, 4, 3, 2]
   - Mesma lógica do passo 3
   - Compara com o 11º dígito do CPF
```

---

## Estados do Redis

| Chave | Valor | TTL | Descrição |
|---|---|---|---|
| `session:{whatsapp}` | `{state, name, cpf, attempts, consent_at}` | 10 min | Estado ativo da conversa |
| `dedup:cpf:{cpf_hash}` | `{lead_id, result, timestamp}` | 30 dias (CPF limpo) | Anti-duplicidade CPF aprovado |
| `dedup:cpf:{cpf_hash}` | `{lead_id, result, timestamp}` | 7 dias (CPF sujo) | Anti-duplicidade CPF pendente |
| `dedup:wa:{whatsapp}` | `{lead_id, timestamp}` | 7 dias | Anti-duplicidade por WhatsApp |
| `ratelimit:{whatsapp}` | `{count}` | 24h | Rate limit por número |

> O CPF nunca é armazenado em texto plano no Redis. A chave usa `SHA-256(cpf)`.

---

## Regras de Anti-Duplicidade

```
1. Ao receber CPF confirmado:
   a. Calcula SHA-256 do CPF
   b. Verifica chave Redis `dedup:cpf:{hash}`
      → Existe? → Informa lead duplicado. Encerra.
   c. Verifica PostgreSQL por CPF hash (fallback)
      → Existe dentro do período de retenção? → Informa lead duplicado. Encerra.
   d. Verifica Redis `dedup:wa:{whatsapp}`
      → Existe? → Informa lead duplicado. Encerra.
   e. Prossegue para consulta BigDataCorp

2. Após consulta:
   a. Grava `dedup:cpf:{hash}` com TTL conforme resultado
   b. Grava `dedup:wa:{whatsapp}` com TTL de 7 dias
   c. Persiste no PostgreSQL
```

---

## Fluxo de Retry — Falhas de API

### BigDataCorp indisponível

```
Tentativa 1 (imediata)
      │
      ▼ falha
Aguarda 2s
      │
      ▼
Tentativa 2
      │
      ▼ falha
Aguarda 5s
      │
      ▼
Tentativa 3
      │
      ▼ falha
Classifica como PENDING + flag "api_error"
Notifica vendedor com aviso de verificação manual
Registra erro no log (sem dados sensíveis)
```

### WhatsApp API indisponível (notificação ao vendedor)

```
Tentativa 1 (imediata)
      │
      ▼ falha
Aguarda 5s
      │
      ▼
Tentativa 2
      │
      ▼ falha
Aguarda 15s
      │
      ▼
Tentativa 3
      │
      ▼ falha
Salva mensagem na fila (Bull/Redis)
Reprocessa em até 5 minutos
Alerta admin se fila > 10 mensagens pendentes
```

### Google Sheets indisponível

```
Falha → Salva em fila de retry (Bull)
      → Reprocessa a cada 1 minuto por até 30 minutos
      → Após 30 min sem sucesso: notifica admin por e-mail
      → Dado já está no PostgreSQL (fonte de verdade)
```

---

## Tratamento de Timeout

| Situação | Timeout | Comportamento |
|---|---|---|
| Lead para de responder (qualquer etapa) | 10 minutos | Redis TTL expira; sessão reinicia na próxima mensagem |
| Sem resposta à confirmação de dados | 10 minutos | Idem |
| API BigDataCorp sem resposta | 8 segundos | Aciona retry (3 tentativas) |
| API WhatsApp sem resposta | 10 segundos | Aciona retry (3 tentativas) |

---

## Observações de Segurança

- O CPF é mascarado nos logs: apenas os 3 primeiros e 2 últimos dígitos são visíveis.
- O CPF nunca aparece integralmente em notificações de WhatsApp.
- Todas as chaves Redis com CPF usam hash SHA-256.
- Rate limit: máximo de 3 tentativas de fluxo por número de WhatsApp em 24 horas.
