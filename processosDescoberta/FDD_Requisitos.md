3. FDD da feature
Produza o arquivo docs/FDD.md detalhando o "como implementar" da feature. O FDD é o documento mais técnico e precisa estar acionável o suficiente para um desenvolvedor pegar e começar a codar. Deve seguir o formato apresentado no curso e incluir, no mínimo:

Contexto e motivação técnica
Objetivos técnicos
Escopo e exclusões
Fluxos detalhados (criação do evento na outbox, processamento pelo worker, retry, DLQ)
Contratos públicos (endpoints HTTP com payloads de exemplo, headers, status codes, semântica)
Matriz de erros previstos com códigos no padrão WEBHOOK_*
Estratégias de resiliência (timeouts, retries, backoff, fallback)
Observabilidade (métricas, logs, tracing)
Dependências e compatibilidade
Critérios de aceite técnicos
Riscos e mitigação
Seção obrigatória adicional, específica deste desafio: "Integração com o sistema existente". Esta seção deve nomear pelo menos 4 caminhos de arquivo reais do código base e descrever como o módulo de webhooks vai se integrar com cada um (por exemplo, como o método changeStatus será estendido, como as classes de erro existentes serão reutilizadas).
