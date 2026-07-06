4. ADRs
Produza entre 5 e 8 ADRs em arquivos separados dentro de docs/adrs/, nomeados no formato ADR-NNN-titulo-em-kebab-case.md (ex: ADR-001-outbox-no-mysql.md).

Cada ADR deve seguir o formato MADR (ou variante padrão) com no mínimo as seções: Status, Contexto, Decisão, Alternativas Consideradas (pelo menos 1 alternativa real discutida ou plausível), Consequências (positivas e negativas, com trade-off explícito).

Pelo menos 1 ADR deve referenciar explicitamente arquivos, módulos ou padrões do código existente.

O conjunto de ADRs deve cobrir, no mínimo, 5 das 6 decisões principais discutidas na reunião:

Padrão Outbox no MySQL
Política de retry com backoff e DLQ
Autenticação HMAC-SHA256 com secret por endpoint
Garantia at-least-once com X-Event-Id
Worker em processo separado em polling
Reuso dos padrões existentes do projeto
Decisões técnicas secundárias (formato de payload, timeouts, headers, entre outras) podem virar ADRs adicionais ou ficar apenas no FDD, conforme você considerar mais adequado.
