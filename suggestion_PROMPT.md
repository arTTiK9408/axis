# Prompt for Agent
## O1
```text
Leia integralmente o arquivo SPEC.md na raiz deste projeto e a documentação SOAP do WebISS em /docs/webiss-juazeiro antes de escrever qualquer código.

Configure o projeto do zero seguindo exatamente a stack definida na SPEC (Next.js + TS, PostgreSQL, Prisma, Tailwind/shadcn, NextAuth, Vitest, Playwright, pnpm).

TDD é obrigatório e não-negociável: para cada funcionalidade, escreva o teste primeiro, confirme que falha, só então implemente. Não escreva código de produção sem um teste correspondente.

Comece pela Fase 1 (RF01 - Autenticação) seguindo a ordem: setup do projeto → schema Prisma → testes de auth → implementação de auth. Antes de avançar para o próximo requisito funcional, me mostre o que foi feito e aguarde minha aprovação.
```

## O2
```text
Leia o SPEC.md na raiz do projeto. Não implemente nada ainda.

Primeiro, me apresente um plano de tarefas (task breakdown) para a Fase 1 inteira, na ordem que você executaria, respeitando TDD obrigatório (teste antes de implementação) em cada item. Quero revisar e aprovar esse plano antes de você começar a codar.
```

## O3
```text
Leia o SPEC.md e a documentação SOAP do WebISS em /docs/webiss-juazeiro.

Antes de montar o projeto inteiro, quero validar a integração mais crítica primeiro: crie um protótipo isolado (fora da estrutura final do Next.js) que, seguindo TDD, escreva testes com mocks baseados na documentação SOAP e implemente um cliente que consiga se autenticar e buscar movimentações no WebISS de Juazeiro. Só depois de validarmos esse fluxo eu quero que você monte o restante da aplicação (auth, dashboard, etc).
```
