você vai atuar como um engenheiro de software senior, especializado em orquestração de agentes e spec driven development

quero que você crie a SPEC de uma aplicação

> terminantemente necessário e obrigatório testes antes de qualquer implementação, estamos falando de TDD obrigatório para o agente

> a hospedagem vai ser feita no KingHost

---

1. Visão geral
- quero um webapp/website com um hub/dashboard que mostre algumas movimentações fiscais específicas, como cancelamente, transferências, envio e outras movimentos, referentes a nfs-e
- trabalho no setor de TI de uma concessionária de motos de um grupo que atua em diversas cidades, mas tenho obrigações com Remanso, Casa Nova e Juazeiro na BA e Petrolina em PE. E preciso monitorar as movimentações para alinhas com o sistema que usamos internamente
- a princípio apenas a implementação referente ao monitoramento das notas de juazeiro os seguintes websites são os os gestores
  - https://juazeiroba.webiss.com.br/
  - https://petrolina.pe.gov.br/nfs/
  - https://www.nfse.gov.br/EmissorNacional/Login
  - https://www.fisco.net.br/notafiscal/NFA/Empresa/AcessoEmpresaNovo.aspx?prefeitura=PREFEITURA%20MUNICIPAL%20DE%20CASA%20NOVA
> o webiss de juazeiro é o mais importante dos gestores, e o primeiro que deve ser implementado, e tambem, ele é por ora, o único que ja possuo a documentação disponível, ela utiliza SOAP. Essa documentação vai estar na raiz do projeto para quando o agente for iniciar as tarefas

2. Tech Stack
- quanto a decisões sobre framework, tecnologias, estilização, e o que mais for de necessário decidir... quero decidir antes de você criar a spec, para que isso ja esteja inserido desde o inicio... 
- faça uma entrevista guiada para preencher todas as lacunas....

3. Arquitetura e estrutura
- quanto a decisões sobre segurança, testes, decisões de arquitetura de software e design, banco de dados, e o que mais for de necessário decidir... quero decidir antes de você criar a spec, para que isso ja esteja inserido desde o inicio...
- faça uma entrevista guiada para preencher todas as lacunas....

4. Requisitos funcionais
- devo conseguir acessar através de um login e senha, que posso compartilhar com meus estagiários, ou criar usuários para eles
- ao acessar, posso visualizar um dashboard ou hub com informações como as ultimas movimentações feitas, com datas e outras informações úteis que as prefeituras possam ceder através das formas de requisições que vamos usar (cada uma usa um padrão diferente)

5. Requisitos não funcionais
- autenticação é outro assunto que vou precisar de sua ajuda para decidir, entrevista guiada
- quero responsividade com celulares

6. Modelos de dados
- entrevista guiada para responder a esse tópico tambem

7. Instruções de execução e entrega
- o gerenciador de pacotes, acredito se relacionar com a stack que usaremos
- quero testes unitários

tudo relacionado a entrevista guiada você vai precisar apresentar alternativas com pros e contras

ao fim da cosntrução da spec, apresente tambem alternatias de prompt, para eu enviar quando acionar o agente e a construção do projeto
