# Da Reunião ao Documento: Design Docs Gerados por IA

## Descrição

Neste desafio você vai transformar a transcrição de uma reunião técnica em um pacote completo de design docs, usando IA como ferramenta principal de produção.

**Cenário:** uma empresa que opera um Order Management System (OMS) em produção vai construir uma nova feature, um Sistema de Webhooks de Notificação de Pedidos. A decisão técnica já foi tomada em uma reunião entre tech lead, PM, engenheiros e segurança, mas nada foi registrado além da transcrição da call (`TRANSCRICAO.md`).

**Sua tarefa:** produzir, a partir da transcrição e do código existente, a documentação técnica da feature, em nível acionável o suficiente para o time de engenharia iniciar a implementação.

## Método de trabalho
1. Inicialmente solicitei a IA uma avaliação completa do sistema existente a fim de documentar estrutura, arquitetura, funcionalidades e demais abordagens: `C:\_aulasMBA\Documentos_Gerados_Por_IA\mba-ia-desafio-design-docs-com-ia` Após avaliação, pedi a geração do documento `achados.md`

2. Em separado do repositório inicial, pedi que a IA lesse a `TRANSCRICAO.md` e tendo por base o `achados.md`, criasse um `toDo.md`. Comparei esse arquivo mais compacto com a leitura inicial da trascrição e validei os pontos a serem incluídos. Não excluí os pontos anotados como "para o futuro" para deixar documentado o motivo de não entrar nas tarefas atuais.

3. Depois da exploração inicial, segui a orientação final da tarefa para encadeamento dos documentos a serem gerados:
    a) ADRs primeiro: identifique e produza as decisões principais antes dos demais documentos. As decisões formam o esqueleto do "como implementar";
    b) RFC: consolide a proposta técnica em cima das decisões. As alternativas descartadas e as questões em aberto da reunião têm lugar natural aqui. Referencie os ADRs já escritos;
    c) FDD: com as decisões formalizadas e a proposta consolidada, o desenho técnico se constrói em cima delas. Lembre da seção obrigatória "Integração com o sistema existente";
    d) PRD: produza o PRD por último entre os grandes documentos. Como ele é mais alto nível, com RFC, FDD e ADRs em mãos vira praticamente uma consolidação.
    e) TRACKER: monte em paralelo com os outros documentos ou no fim, varrendo os documentos prontos.

## Ferramenta de IA
Composer 2.5 Fast

## Prompts

1. Leia a transcrição da reunião de pedido de alterações do sistema, @TRANSCRICAO.md , e pontue o que está sendo pedido.
Pontue os pedidos de alteração marcados como decisão e deixe no final tudo que for anotações para o futuro.
As anotações sobre uma exploração inicial para entender estrutura e  padrões atual do sistema está em @achados.md.
Cie um arquivo chamado toDo.md na raiz do projeto com os pontos a serem tratados nesse pedido.

2. Olhe para @processosDescoberta/toDo.md e, seguindo os padrões elencados em @processosDescoberta/ADRs_Requisitos.md , crie os ADRs solicitados. Crie a pasta docs na raiz do projeto para abrigar os documentos.

3.Olhando para os ADRs e seguindo o padrão de @processosDescoberta/RFC_Requisitos.md, crie o documnto @docs/RFC.md lembrando referenciar os 7 ADRs já criados.
Faça apenas o RFC


## Iterações e ajustes
Inicialmente não havia separado a transcrição dos arquivos do projeto e a IA acabou lendo a transcrição e assumiu que a exploração inicial já deveria prever as mudanças solicitadas. Acabei reiniciando o processo, separando os arquivos para diminuir ao máximo a janela de contexto. A abordagem resultou vantajosa e mais fácil de validar.

Cada documento gerado era lido em sua totalidade e poucas vezes foi necessário intervenção, o que foi feito de forma manual.

## Como navegar a entrega:
Vale a pena ler primeiro os documentos `achados.md`, em seguida `toDo.md` e só depois ler os documentos gerados.
