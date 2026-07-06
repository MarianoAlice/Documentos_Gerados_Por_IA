Estrutura do desafio
O desafio consiste em produzir um pacote de design docs: PRD, RFC, FDD, ADRs, Tracker e o README do processo a partir da transcrição e do código.

Objetivo
Entregar, em um repositório público no GitHub (fork do repositório base), o seguinte pacote de documentação:

PRD (Product Requirement Document) da feature
RFC (Request for Comments) com a proposta técnica da solução, submetida à equipe para revisão
FDD (Feature Design Document) da feature
Entre 5 e 8 ADRs (Architecture Decision Records) das decisões discutidas
Tracker de rastreabilidade ligando cada item à origem na transcrição ou no código
README atualizado documentando o processo de produção
Toda informação registrada nos documentos deve ser rastreável à transcrição ou ao código fonte da aplicação. Não é permitido inventar requisitos, decisões ou restrições sem origem identificável.

O pacote de documentos e o papel de cada um
Os documentos não se repetem: cada um opera em uma altura diferente. Antes de produzir, entenda a fronteira entre eles: conteúdo duplicado entre documentos é sinal de que algo está no lugar errado.

Documento	Papel	Altura	Pergunta que responde
PRD	Problema, público, escopo e métricas de sucesso	Produto / negócio	Por que e o quê?
RFC	Proposta técnica da solução para revisão: abordagem geral, alternativas e questões em aberto	Arquitetura	Como pretendemos resolver, e o que ainda está em aberto?
ADRs	Cada decisão arquitetural isolada, com contexto e consequências	Decisão pontual	Por que decidimos exatamente assim?
FDD	Especificação de implementação: fluxos, contratos, erros, integração com o código	Implementação	Como construir, em detalhe?
Tracker	Rastreabilidade de cada item ao código ou à transcrição	Transversal	De onde veio cada coisa?
Em uma frase: o RFC propõe e abre para revisão, os ADRs registram cada decisão fechada e o FDD detalha como construir. O RFC é conciso (2 a 4 páginas) e fala em decisão; o FDD é profundo e fala em implementação. Não repita no RFC o nível de detalhe do FDD.

Contexto
A aplicação existente
O repositório base contém uma aplicação Node.js + TypeScript funcional: um Order Management System com módulos de autenticação, usuários, clientes, produtos e pedidos. Banco MySQL via Prisma. O ciclo de vida do pedido tem máquina de estados controlada, controle transacional de estoque e auditoria de mudanças de status.

A aplicação não tem nenhum mecanismo de notificação externa, eventos, filas ou webhooks. Esse vácuo é proposital. É exatamente o que a feature discutida na reunião pretende preencher.

A transcrição
O arquivo TRANSCRICAO.md contém a gravação literal da reunião técnica. Cinco participantes discutem por aproximadamente 55 minutos no formato [hh:mm] Nome: fala.

A transcrição inclui decisões fechadas, requisitos funcionais explícitos, restrições, ganchos com o código existente, pontos descartados ou adiados para fases futuras e detalhes técnicos secundários. Nem tudo que foi mencionado vira requisito. Algumas coisas foram explicitamente descartadas, outras foram adiadas. Identificar o que NÃO entra é tão importante quanto identificar o que entra. Use a IA com prompts dirigidos para fazer essa filtragem, não pedidos genéricos.

Tecnologias e ferramentas
Liberdade total na escolha de ferramentas de IA. Você pode usar qualquer combinação de Claude, ChatGPT, Cursor, Copilot Chat, Gemini, agentes, prompts customizados, skills ou plugins. Aproveite os prompts e plugins disponibilizados pelo professor durante o curso como ponto de partida.

Os documentos devem ser entregues em formato Markdown.

A entrega é puramente documental: você não deve mexer no código da aplicação (src/, prisma/, tests/, configurações). O código serve de contexto e referência.

Ordem de execução sugerida
Fork e setup: faça o fork do repositório base: já feito e presente em: C:\_aulasMBA\mba-ia-desafio-design-docs-com-ia
Contextualização com IA: forneça à IA acesso ao código (via Claude Code, Cursor lendo o repo, ou colando trechos relevantes) e à transcrição. Peça uma exploração inicial para entender estrutura, padrões e o que a feature precisa endereçar.
ADRs primeiro: identifique e produza as decisões principais antes dos demais documentos. As decisões formam o esqueleto do "como implementar".
RFC: consolide a proposta técnica em cima das decisões. As alternativas descartadas e as questões em aberto da reunião têm lugar natural aqui. Referencie os ADRs já escritos.
FDD: com as decisões formalizadas e a proposta consolidada, o desenho técnico se constrói em cima delas. Lembre da seção obrigatória "Integração com o sistema existente".
PRD: produza o PRD por último entre os grandes documentos. Como ele é mais alto nível, com RFC, FDD e ADRs em mãos vira praticamente uma consolidação.
Tracker: monte em paralelo com os outros documentos ou no fim, varrendo os documentos prontos.
README do processo: deixe por último, quando o processo já está completo e você pode documentá-lo com clareza.
Revisão final: passe pela checklist de critérios de aceite item por item antes do push final.
Itere: é esperado que o processo demande 3 a 5 ciclos de geração, revisão crítica, ajuste de prompt e nova geração. Se você gerou tudo de primeira sem ajustes, os documentos provavelmente estão genéricos demais.
Dicas Finais
A qualidade do prompt determina a qualidade do documento. Prompts vagos do tipo "gere um PRD a partir dessa transcrição" produzem documentos vazios e genéricos. Aproveite os prompts disponibilizados pelo professor no curso como base e adapte-os ao contexto deste desafio.

O tracker é seu melhor aliado contra alucinações da IA. Se você não consegue preencher a coluna "Localização" para uma linha do PRD ou do FDD, é sinal de que aquela informação não tem origem identificável e provavelmente foi inventada pela IA. Ajuste ou remova.

Cuidado com o que NÃO entra na documentação. A reunião descarta explicitamente algumas ideias. Se essas coisas aparecerem como requisito nos seus documentos, é sinal de que a IA não está sendo cuidadosa com o que você pediu.

A restrição de não alterar o código da aplicação é absoluta: o código serve de contexto e referência, e o entregável é puramente documental.

Itere bastante. Os primeiros documentos que a IA gerar provavelmente serão superficiais ou redundantes. Volte com correções, peça refinamento de pontos específicos, peça para remover trechos vagos, peça exemplos concretos. O resultado final deve parecer escrito por alguém que pensou no problema com a IA ao lado, não por alguém que copiou e colou da transcrição.