# Política de Privacidade e Conformidade LGPD — MotoQualify

> Versão: v1.0 | Data: junho de 2026 | Status: Vigente

---

## 1. Identificação do Controlador

| Campo | Informação |
|---|---|
| **Plataforma** | MotoQualify — Qualificação de Leads via WhatsApp |
| **Responsável pelo Tratamento (DPO)** | Diego Pereira |
| **Contato do DPO** | stones2006@gmail.com |
| **Canal de atendimento ao titular** | stones2006@gmail.com |
| **Versão da política** | v1.0 — junho de 2026 |

---

## 2. Base Legal

O tratamento de dados pessoais nesta plataforma é fundamentado nas seguintes hipóteses legais previstas no **Art. 7º da Lei 13.709/2018 (LGPD)**:

| Inciso | Base Legal | Aplicação |
|---|---|---|
| **Art. 7º, I** | Consentimento do titular | Coleta de nome e CPF mediante aceite explícito no fluxo do bot |
| **Art. 7º, V** | Execução de contrato ou procedimentos preliminares | Verificação de elegibilidade para financiamento de moto |
| **Art. 7º, IX** | Legítimo interesse do controlador | Prevenção de fraudes e anti-duplicidade de leads |

> O consentimento é coletado **antes** de qualquer dado pessoal ser solicitado. O registro de aceite (timestamp + número de WhatsApp) é persistido no PostgreSQL.

---

## 3. Dados Coletados e Finalidade

| Dado Pessoal | Categoria | Finalidade | Base Legal |
|---|---|---|---|
| **Nome completo** | Dado comum | Identificação do lead para o vendedor | Art. 7º, I |
| **CPF** | Dado comum (sensível em contexto financeiro) | Consulta de elegibilidade para crédito na BigDataCorp | Art. 7º, I e V |
| **Número de WhatsApp** | Dado comum | Canal de comunicação; anti-duplicidade | Art. 7º, I e IX |
| **Aceite de consentimento + timestamp** | Dado de controle | Prova de consentimento; conformidade legal | Art. 7º, I |
| **Origem do lead (UTM/anúncio)** | Dado de contexto | Análise de performance de campanhas | Art. 7º, IX |
| **Score de crédito (BigDataCorp)** | Dado de terceiro | Classificação do lead (aprovado/pendente) | Art. 7º, I e V |
| **Logs de interação** | Dado operacional | Auditoria, debugging e segurança | Art. 7º, IX |

> **Princípio da minimização (Art. 6º, III):** são coletados apenas os dados estritamente necessários para a finalidade declarada.

---

## 4. Tempo de Retenção por Tipo de Dado

| Dado | Sistema | Retenção | Justificativa |
|---|---|---|---|
| Sessão da conversa | Redis | 10 minutos (TTL ativo) | Dado operacional temporário |
| Hash CPF (dedup — aprovado) | Redis | 30 dias | Evitar reconsulta dentro do período |
| Hash CPF (dedup — pendente) | Redis | 7 dias | Permite nova tentativa após período |
| Hash WhatsApp (dedup) | Redis | 7 dias | Anti-duplicidade por canal |
| Lead completo (nome, CPF hash, resultado) | PostgreSQL | 24 meses | Histórico operacional e auditoria |
| Registro de consentimento | PostgreSQL | 5 anos | Obrigação legal de prova de consentimento |
| Logs de sistema | Arquivos / S3 | 90 dias | Segurança e debugging |
| Dados no Google Sheets | Google Sheets | Até exclusão manual pelo gestor | Operacional da loja |

> Após o período de retenção, os dados são excluídos automaticamente ou anonimizados, conforme o tipo.

---

## 5. Compartilhamento de Dados

| Destinatário | Dado Compartilhado | Finalidade | Base |
|---|---|---|---|
| **Vendedor da loja** (via WhatsApp) | Nome, resultado da consulta | Acompanhamento do lead | Art. 7º, I |
| **Gestor da loja** (via Google Sheets) | Nome, CPF mascarado, resultado | Gestão operacional | Art. 7º, I |
| **BigDataCorp** | CPF | Consulta de elegibilidade de crédito | Art. 7º, I e V |
| **Meta (WhatsApp Business API)** | Conteúdo das mensagens | Entrega das mensagens via canal | Art. 7º, I |

> **Não há compartilhamento de dados com terceiros para fins comerciais ou publicitários sem consentimento adicional explícito.**

---

## 6. Direitos do Titular

Conforme o **Art. 18 da LGPD**, o titular dos dados tem os seguintes direitos:

| Direito | Como Exercer | Prazo de Resposta |
|---|---|---|
| **Confirmação de tratamento** | E-mail para stones2006@gmail.com | 15 dias úteis |
| **Acesso aos dados** | E-mail para stones2006@gmail.com | 15 dias úteis |
| **Correção de dados incompletos ou inexatos** | E-mail para stones2006@gmail.com | 15 dias úteis |
| **Anonimização, bloqueio ou eliminação** | E-mail para stones2006@gmail.com | 15 dias úteis |
| **Portabilidade** | E-mail para stones2006@gmail.com | 15 dias úteis |
| **Eliminação dos dados tratados com consentimento** | E-mail para stones2006@gmail.com | 15 dias úteis |
| **Revogação do consentimento** | E-mail para stones2006@gmail.com | Imediato para novos tratamentos |
| **Oposição ao tratamento** | E-mail para stones2006@gmail.com | 15 dias úteis |

> Para exercer qualquer direito, o titular deve informar: nome completo, número de WhatsApp utilizado e a solicitação desejada. A identidade será verificada antes do atendimento.

---

## 7. Fluxo de Exclusão de Dados

```
[Titular envia solicitação de exclusão]
              │
              ▼
   [DPO recebe e-mail/solicitação]
              │
              ▼
   [Verificação de identidade do titular]
              │
       ┌──────┴──────┐
       │             │
       ▼             ▼
  [Verificado]  [Não verificado]
       │             │
       │             ▼
       │   [Solicita documentação adicional]
       │
       ▼
[Localiza registros no PostgreSQL]
              │
              ▼
[Remove / anonimiza dados pessoais]
  - Nome → substituído por "REMOVIDO"
  - CPF hash → excluído
  - WhatsApp → excluído
              │
              ▼
[Invalida chaves Redis (se ainda ativas)]
              │
              ▼
[Remove linha do Google Sheets (se aplicável)]
              │
              ▼
[Registra a exclusão com timestamp e motivo]
(sem dados pessoais no log de exclusão)
              │
              ▼
[Notifica o titular por e-mail — confirmação]
              │
              ▼
        [Prazo: até 15 dias úteis]
```

> **Exceção:** O registro de consentimento (sem dados pessoais identificáveis) pode ser mantido por até 5 anos para fins de prova legal, conforme orientação da ANPD.

---

## 8. Medidas de Segurança Implementadas

| Medida | Descrição | Escopo |
|---|---|---|
| **Hash SHA-256 do CPF** | O CPF nunca é armazenado em texto plano no Redis ou em logs | Redis, Logs |
| **Mascaramento em logs** | CPF exibido como `***.456.789-**` em todos os registros de log | Logs de sistema |
| **CPF mascarado em notificações** | O CPF não aparece integralmente em mensagens de WhatsApp ao vendedor | WhatsApp |
| **Redis TTL automático** | Dados de sessão e deduplicação expiram automaticamente | Redis |
| **Criptografia em repouso** | Dados sensíveis criptografados no PostgreSQL (AES-256) | PostgreSQL |
| **Criptografia em trânsito** | Toda comunicação via HTTPS/TLS 1.2+ | APIs, Webhooks |
| **Controle de acesso** | Acesso ao banco restrito por IP e credenciais rotacionadas | PostgreSQL |
| **Rate limiting** | Máximo de 3 tentativas de fluxo por WhatsApp em 24 horas | Bot |
| **Logs de auditoria** | Registro de todas as operações de leitura/escrita de dados pessoais | Sistema |
| **Backup criptografado** | Backups do PostgreSQL criptografados e armazenados em local seguro | Infraestrutura |

---

## 9. Transferência Internacional de Dados

| Serviço | País | Base para Transferência |
|---|---|---|
| **WhatsApp Business API (Meta)** | EUA | Cláusulas Contratuais Padrão (SCCs) |
| **Google Sheets / Google Drive** | EUA | Cláusulas Contratuais Padrão (SCCs) |
| **BigDataCorp** | Brasil | Nacional — sem transferência internacional |

---

## 10. Incidentes de Segurança

Em caso de incidente de segurança envolvendo dados pessoais:

1. O DPO é notificado imediatamente pela equipe técnica.
2. O incidente é avaliado quanto ao risco e à extensão.
3. A ANPD é notificada em até **72 horas** se houver risco relevante aos titulares (Art. 48, LGPD).
4. Os titulares afetados são notificados se o incidente puder causar dano significativo.
5. O incidente é registrado no log de incidentes (data, tipo, extensão, ações tomadas).

**Contato para reporte de incidentes:** stones2006@gmail.com

---

## 11. Histórico de Versões

| Versão | Data | Alterações |
|---|---|---|
| v1.0 | junho de 2026 | Versão inicial da política |

---

## 12. Contato

Para quaisquer dúvidas, solicitações ou exercício de direitos relacionados ao tratamento de dados pessoais:

**DPO (Encarregado de Proteção de Dados)**
- E-mail: stones2006@gmail.com
- Prazo de resposta: até 15 dias úteis

---

*Esta política foi elaborada em conformidade com a Lei nº 13.709/2018 (LGPD) e as diretrizes da Autoridade Nacional de Proteção de Dados (ANPD).*
