# ADR-004: Autenticação HMAC-SHA256 com Secret por Endpoint

## Status

Aceito — [09:22] Sofia

## Contexto

Webhooks expõem dados de pedidos para endpoints fora da infraestrutura da plataforma. Clientes precisam validar que a requisição é autêntica e que o payload não foi adulterado em trânsito.

Histórico interno: um cliente já vazou secret em log de aplicação — rotação segura é requisito, não luxo.

## Decisão

1. **Assinatura HMAC-SHA256** sobre o corpo JSON do request.
2. Header **`X-Signature`** com a assinatura.
3. **Secret única por endpoint** de webhook (armazenada na tabela de configuração junto com `url`, `customer_id`, `active` e filtro de status). Não usar secret global da plataforma.
4. **Secret gerada pelo sistema** na criação do webhook e devolvida ao cliente uma vez.
5. **Rotação via API:** endpoint para solicitar nova secret; a secret anterior permanece válida por **24 horas** (grace period) para migração.
6. **TLS obrigatório:** URLs devem ser `https`; `http` rejeitado na validação Zod (`WEBHOOK_INVALID_URL`).

## Alternativas Consideradas

### Secret global compartilhada entre todos os clientes

**Descartada.** Sofia: se uma vazar, compromete todos os endpoints.

### Assinatura assimétrica (RSA/ECDSA)

**Não discutida na reunião; descartada por simplicidade.** HMAC-SHA256 é padrão de mercado (Stripe, GitHub) e toda biblioteca HTTP do cliente suporta.

### Sem assinatura (confiar apenas em TLS)

**Descartada.** Sofia exigiu validação explícita de autenticidade e integridade do payload.

## Consequências

### Positivas

- Padrão de mercado, fácil de documentar no portal do desenvolvedor.
- Secret por endpoint limita blast radius de vazamento.
- Grace period de 24 h reduz downtime em rotações.

### Negativas

- Responsabilidade do cliente armazenar e verificar secrets corretamente.
- Implementação de rotação com duas secrets ativas aumenta complexidade do worker (tentar secret atual e anterior durante grace period).
- **Trade-off explícito:** segurança robusta com operação manual de rotação, em troca de não usar infraestrutura de chaves assimétricas gerenciadas.
