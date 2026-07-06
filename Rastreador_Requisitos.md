5. Tracker de Rastreabilidade
Produza o arquivo docs/TRACKER.md, uma tabela markdown que mapeia cada item registrado nos seus documentos à origem na transcrição ou no código. O tracker funciona como uma referência cruzada: permite que qualquer leitor entenda de onde veio cada decisão, requisito ou restrição, e garante que a documentação está alinhada com o que foi efetivamente discutido e com o que existe no código.

O tracker não é um conceito padrão do mercado nem é um documento abordado diretamente no curso. É uma exigência específica deste desafio que ajuda a manter a integridade da documentação contra alucinações da IA.

Formato obrigatório da tabela:

ID	Documento	Tipo	Conteúdo (resumo)	Fonte	Localização
Onde:

ID: identificador único do item (ex: PRD-FR-01, RFC-ALT-02, FDD-CONTRATO-03, ADR-002)
Documento: arquivo onde o item aparece (docs/PRD.md, docs/RFC.md, docs/FDD.md, docs/adrs/ADR-002-...md)
Tipo: Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, entre outros
Conteúdo (resumo): descrição de uma linha do item
Fonte: TRANSCRICAO ou CODIGO
Localização: para TRANSCRICAO, timestamp + nome do falante (ex: [09:17] Diego). Para CODIGO, caminho do arquivo (ex: src/modules/orders/order.service.ts).
Cobertura mínima: pelo menos 80% dos itens identificáveis nos seus documentos devem ter linha correspondente no tracker.
