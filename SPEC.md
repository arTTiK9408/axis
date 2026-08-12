# SPEC — NFS-e Monitor Hub

**Versão:** 1.0
**Status:** Aprovado para início de desenvolvimento (Fase 1)
**Metodologia obrigatória:** TDD (Test-Driven Development) — sem exceções

---

## 1. Visão Geral

### 1.1 Objetivo

Aplicação web (dashboard/hub) para monitorar movimentações fiscais de NFS-e (Nota Fiscal de Serviço Eletrônica) — cancelamentos, transferências, envios e outras movimentações — capturadas periodicamente dos portais das prefeituras onde a concessionária possui obrigações fiscais. O sistema serve como camada de conferência para alinhamento manual com o sistema interno da empresa.

### 1.2 Contexto de negócio

Empresa do setor de TI de uma concessionária de motos, pertencente a um grupo com atuação em múltiplas cidades. Escopo fiscal do time responsável:

| Cidade | UF | Portal / Gestor | Protocolo | Documentação disponível |
|---|---|---|---|---|
| Juazeiro | BA | WebISS (`juazeiroba.webiss.com.br`) | SOAP | ✅ Sim — na raiz do projeto |
| Petrolina | PE | Portal NFS-e Petrolina (`petrolina.pe.gov.br/nfs`) | A definir (provável scraping/API) | ❌ Não |
| Casa Nova | BA | Fisco.net (`fisco.net.br`) | A definir | ❌ Não |
| — | — | Emissor Nacional (`nfse.gov.br`) | A definir | ❌ Não |

### 1.3 Fases do projeto

- **Fase 1 (MVP — este documento cobre a implementação completa desta fase):** Integração exclusiva com **Juazeiro/WebISS** via SOAP, incluindo autenticação, dashboard, notificações por e-mail e toda a base de arquitetura reutilizável para as próximas integrações.
- **Fase 2:** Petrolina (integração a definir após levantamento técnico do portal)
- **Fase 3:** Casa Nova / Fisco.net
- **Fase 4:** Emissor Nacional

> A arquitetura da Fase 1 **deve** ser desenhada para permitir plugar novos "integradores" de portal (padrão Strategy/Adapter) sem reescrever o núcleo da aplicação.

### 1.4 Fora de escopo (Fase 1)

- Comparação/reconciliação automática com o sistema interno da empresa (MVP é só exibição para conferência manual)
- Integração com Petrolina, Casa Nova e Emissor Nacional (fases futuras)
- Permissões de acesso restritas por cidade/empresa (todo estagiário vê todas as cidades)

---

## 2. Stack Tecnológica (decidida)

| Camada | Escolha | Justificativa resumida |
|---|---|---|
| Framework fullstack | **Next.js 15 (App Router) + TypeScript** | Frontend + backend no mesmo projeto, um único deploy, curva de aprendizado adequada para manutenção solo |
| Banco de dados | **PostgreSQL** | Robusto, bom suporte a JSON (payloads brutos de NFS-e), gratuito |
| ORM | **Prisma** | Melhor DX para iniciantes, migrations automáticas, Prisma Studio |
| Estilização | **Tailwind CSS + shadcn/ui** | Produtividade alta, componentes prontos para telas de dashboard |
| Autenticação | **NextAuth.js (Auth.js)** — Credentials Provider | Gerencia sessão, proteção de rotas e papéis com pouco código |
| Testes unitários/integração | **Vitest** | Rápido, sintaxe moderna, integração nativa com TS/Next.js |
| Testes E2E | **Playwright** | Testa fluxos reais (login → dashboard → filtros) |
| Gerenciador de pacotes | **pnpm** | Mais rápido, mais leve, padrão atual para projetos Next.js |
| Cliente SOAP | `soap` ou `strong-soap` (Node.js) | Bibliotecas maduras para consumo de WSDL/SOAP |
| Agendamento de sincronização | **Cron do sistema operacional** (VPS) chamando script `pnpm run sync` | Mais resiliente que agendamento in-process, sem dependência de Redis |
| E-mail (notificações) | **SMTP próprio** (domínio/Gmail Workspace da empresa) via `nodemailer` | Sem custo adicional, aproveita e-mail corporativo já existente |
| Processo em produção | **PM2** | Mantém o processo Node vivo, restart automático, logs |
| Proxy reverso | **Nginx** | TLS (HTTPS via Let's Encrypt/Certbot), proxy para o processo Node |
| CI/CD | **GitHub Actions** | Bloqueia merge/deploy se os testes (Vitest + Playwright) falharem |
| Hospedagem | **KingHost — Cloud/VPS** | Único plano que suporta processo Node.js persistente + cron do SO |

---

## 3. Arquitetura

### 3.1 Visão de alto nível

```
┌─────────────────────────────────────────────────────────┐
│                    VPS KingHost (Linux)                  │
│                                                            │
│  ┌────────────┐     ┌─────────────────────────────────┐ │
│  │   Nginx    │────▶│   Next.js (Node.js via PM2)      │ │
│  │ (HTTPS/443)│     │   - Frontend (SSR/CSR)            │ │
│  └────────────┘     │   - API Routes / Server Actions   │ │
│                      │   - NextAuth (sessão/login)       │ │
│                      └───────────────┬───────────────────┘ │
│                                       │                     │
│                      ┌────────────────▼──────────────────┐ │
│                      │        PostgreSQL (local)         │ │
│                      └────────────────────────────────────┘ │
│                                                            │
│  ┌───────────────────────────────────────────────────┐   │
│  │ Cron do SO (crontab)                               │   │
│  │  */15 * * * *  pnpm run sync:juazeiro               │   │
│  └───────────────┬───────────────────────────────────┘   │
└──────────────────┼─────────────────────────────────────────┘
                    │ SOAP
                    ▼
        WebISS Juazeiro (juazeiroba.webiss.com.br)
```

### 3.2 Padrão de integração (Adapter/Strategy)

Cada portal municipal é implementado como um **Integrador** que respeita uma interface comum, permitindo adicionar novas cidades sem alterar o núcleo:

```typescript
// src/lib/integrations/types.ts
interface NfseIntegrator {
  cityCode: string; // ex: "juazeiro-ba"
  fetchMovements(params: FetchParams): Promise<RawMovement[]>;
  normalize(raw: RawMovement[]): NormalizedMovement[];
}
```

- `src/lib/integrations/juazeiro-webiss/` → implementação SOAP (Fase 1)
- `src/lib/integrations/petrolina/` → placeholder (Fase 2)
- `src/lib/integrations/casa-nova-fisconet/` → placeholder (Fase 3)
- `src/lib/integrations/emissor-nacional/` → placeholder (Fase 4)

> ⚠️ **Antes de implementar `juazeiro-webiss`, o agente deve ler integralmente a documentação SOAP disponível na raiz do projeto.** Nenhuma suposição sobre métodos, campos ou estrutura do WSDL deve ser feita sem consultar essa documentação primeiro.

### 3.3 Estrutura de pastas

```
/
├── docs/
│   └── webiss-juazeiro/          # documentação SOAP fornecida pelo usuário
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── src/
│   ├── app/
│   │   ├── (auth)/login/
│   │   ├── (dashboard)/
│   │   │   ├── page.tsx           # hub principal
│   │   │   ├── movimentacoes/
│   │   │   ├── usuarios/          # admin only
│   │   │   └── credenciais/       # admin only
│   │   └── api/
│   │       ├── auth/[...nextauth]/
│   │       └── movements/
│   ├── lib/
│   │   ├── integrations/
│   │   │   ├── types.ts
│   │   │   └── juazeiro-webiss/
│   │   ├── crypto.ts              # AES-256-GCM helpers
│   │   ├── db.ts                  # Prisma client singleton
│   │   ├── mailer.ts              # nodemailer wrapper
│   │   └── auth.ts                # config NextAuth
│   ├── components/
│   └── scripts/
│       └── sync-juazeiro.ts       # entry point chamado pelo cron do SO
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── .github/workflows/ci.yml
├── .env.example
└── ecosystem.config.js            # config do PM2
```

### 3.4 Segurança

- **Senhas de usuários do sistema** (login/estagiários): hash com `bcrypt` (via NextAuth Credentials Provider)
- **Credenciais dos portais municipais** (ex: login do WebISS): armazenadas **criptografadas no banco** com AES-256-GCM; chave mestra em variável de ambiente (`CREDENTIALS_ENCRYPTION_KEY`), nunca commitada
- Toda comunicação externa via **HTTPS** (Nginx + Certbot/Let's Encrypt)
- Proteção de rotas por sessão (middleware do Next.js) + verificação de papel (role) em cada rota/admin action
- Rate limiting no endpoint de login (proteção contra brute-force)
- Variáveis sensíveis nunca expostas ao client (uso de Server Actions/Route Handlers)
- Logs de sincronização não devem registrar credenciais em texto claro

---

## 4. Modelo de Dados (Prisma — visão inicial)

```prisma
enum Role {
  ADMIN
  ESTAGIARIO
}

enum MovementType {
  CANCELAMENTO
  TRANSFERENCIA
  ENVIO
  OUTRO
}

model User {
  id            String   @id @default(cuid())
  name          String
  email         String   @unique
  passwordHash  String
  role          Role     @default(ESTAGIARIO)
  createdAt     DateTime @default(now())
}

model City {
  id            String   @id @default(cuid())
  name          String
  state         String   // BA, PE
  slug          String   @unique // "juazeiro-ba"
  integratorKey String   // referencia ao adapter (ex: "juazeiro-webiss")
  active        Boolean  @default(true)
  companies     Company[]
  credentials   PortalCredential?
  syncLogs      SyncLog[]
}

model Company {
  id        String   @id @default(cuid())
  cityId    String
  city      City     @relation(fields: [cityId], references: [id])
  cnpj      String
  name      String
  movements Movement[]
}

model PortalCredential {
  id                  String   @id @default(cuid())
  cityId              String   @unique
  city                City     @relation(fields: [cityId], references: [id])
  encryptedUsername   String
  encryptedPassword   String
  extraConfig         Json?    // campos específicos por portal (ex: certificado)
  updatedAt           DateTime @updatedAt
}

model Movement {
  id             String       @id @default(cuid())
  companyId      String
  company        Company      @relation(fields: [companyId], references: [id])
  type           MovementType
  nfseNumber     String
  protocolNumber String?
  status         String
  occurredAt     DateTime     // data da movimentação no portal
  capturedAt     DateTime     @default(now()) // data que o sistema capturou
  rawPayload     Json         // payload bruto retornado pelo portal (auditoria)
  createdAt      DateTime     @default(now())

  @@index([companyId, occurredAt])
  @@index([type])
}

model SyncLog {
  id          String   @id @default(cuid())
  cityId      String
  city        City     @relation(fields: [cityId], references: [id])
  startedAt   DateTime @default(now())
  finishedAt  DateTime?
  status      String   // SUCCESS | ERROR
  errorMessage String?
  movementsFound Int   @default(0)
}

model NotificationLog {
  id          String   @id @default(cuid())
  movementId  String
  sentTo      String
  sentAt      DateTime @default(now())
  channel     String   // "email"
  success     Boolean
}
```

> Campos exatos de `Movement` (ex: `nfseNumber`, `protocolNumber`, `status`) devem ser **validados e ajustados durante a implementação** conforme a estrutura real de resposta do WSDL do WebISS, documentada em `/docs/webiss-juazeiro`.

---

## 5. Requisitos Funcionais

### RF01 — Autenticação
- Login via e-mail/senha (NextAuth Credentials Provider)
- Dois papéis: `ADMIN` e `ESTAGIARIO`
- Admin pode criar/editar/desativar usuários (estagiários)
- Estagiário tem acesso somente-leitura ao dashboard/movimentações (todas as cidades)

### RF02 — Dashboard / Hub
- Listagem das últimas movimentações capturadas, com: cidade, empresa (CNPJ), tipo de movimentação, número da NFS-e, data da movimentação, data de captura, status
- Filtros por: cidade, tipo de movimentação, período (data), empresa
- Busca por número de NFS-e/protocolo
- Indicador visual do status da última sincronização por cidade (sucesso/erro/data-hora)

### RF03 — Gestão de credenciais de portais (Admin only)
- Cadastrar/editar credenciais de acesso ao WebISS (e futuros portais) via interface — armazenadas criptografadas
- Testar conexão/credencial antes de salvar

### RF04 — Sincronização
- Sincronização automática periódica (via cron do SO) com o WebISS de Juazeiro
- Botão de "sincronizar agora" manual disponível para Admin
- Registro de log de cada execução de sincronização (sucesso, erro, quantidade de movimentações encontradas)

### RF05 — Notificações
- Envio de e-mail (SMTP corporativo) quando uma nova movimentação for capturada (ex: cancelamento)
- Configuração de destinatários das notificações (Admin only)

---

## 6. Requisitos Não Funcionais

### RNF01 — Responsividade
- Interface totalmente funcional e legível em dispositivos móveis (breakpoints Tailwind: `sm`, `md`, `lg`)
- Tabelas de movimentações devem ter um layout alternativo (cards) em telas pequenas

### RNF02 — Testes (TDD obrigatório — ver seção 7)

### RNF03 — Segurança
- Ver seção 3.4

### RNF04 — Performance
- Consultas ao dashboard paginadas (nunca carregar todas as movimentações de uma vez)
- Índices no banco nos campos usados em filtros (`type`, `occurredAt`, `companyId`)

### RNF05 — Observabilidade
- Logs estruturados de sincronização (`SyncLog`) consultáveis via interface admin

---

## 7. TDD — Regras Obrigatórias para o Agente

Estas regras são **inegociáveis** e devem ser seguidas em toda tarefa de implementação, sem exceção:

1. **Nenhuma linha de código de produção deve ser escrita sem um teste que falhe primeiro** (ciclo Red → Green → Refactor).
   - Red: escrever o teste que descreve o comportamento esperado e confirmar que ele falha.
   - Green: escrever o mínimo de código necessário para o teste passar.
   - Refactor: melhorar o código mantendo os testes verdes.
2. Toda nova função, endpoint, componente ou integração **começa pela escrita do(s) teste(s)** correspondente(s) em `tests/unit` ou `tests/integration`.
3. Fluxos críticos de usuário (login, visualizar dashboard, filtrar movimentações, sincronizar manualmente) devem ter cobertura em `tests/e2e` (Playwright).
4. Integrações com portais externos (SOAP/WebISS) devem ser testadas com **mocks/fixtures** do WSDL/respostas reais — nunca testes de integração que dependam do portal de produção estar no ar.
5. Nenhum Pull Request pode ser mergeado com testes falhando ou com cobertura reduzida em relação ao `main` (validado pelo pipeline de CI).
6. Antes de implementar o integrador do WebISS (`juazeiro-webiss`), o agente deve:
   - Ler a documentação SOAP disponível na raiz do projeto (`/docs/webiss-juazeiro`)
   - Escrever os testes com base nos contratos documentados (request/response esperados)
   - Só então implementar o cliente SOAP real
7. Meta de cobertura mínima: **80% em `src/lib` e `src/app/api`** (validado no CI via `vitest --coverage`).

---

## 8. CI/CD (GitHub Actions)

Pipeline mínimo (`.github/workflows/ci.yml`):

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm prisma migrate deploy
      - run: pnpm test:unit -- --coverage
      - run: pnpm test:e2e
      - name: Fail if coverage below threshold
        run: pnpm test:coverage-check
```

- Deploy manual (ou via job separado `deploy.yml` disparado só após `test` passar) para o VPS KingHost via SSH + `pm2 reload`.

---

## 9. Deploy — VPS KingHost

1. Provisionar VPS Linux (Ubuntu recomendado) no KingHost
2. Instalar Node.js (via `nvm`), PostgreSQL, Nginx, PM2, pnpm
3. Configurar Nginx como proxy reverso (porta 443 → processo Node em `localhost:3000`)
4. Certificado TLS via Certbot (Let's Encrypt)
5. Configurar `ecosystem.config.js` do PM2 para manter o processo Next.js vivo com restart automático
6. Configurar `crontab` do sistema:
   ```
   */15 * * * * cd /var/www/nfse-monitor && pnpm run sync:juazeiro >> /var/log/nfse-sync.log 2>&1
   ```
7. Variáveis de ambiente (`.env`) configuradas diretamente no servidor (nunca commitadas), incluindo:
   - `DATABASE_URL`
   - `NEXTAUTH_SECRET`
   - `CREDENTIALS_ENCRYPTION_KEY`
   - `SMTP_HOST`, `SMTP_USER`, `SMTP_PASS`

---

## 10. Convenções de Código

- ESLint + Prettier configurados desde o início
- Commits seguindo Conventional Commits (`feat:`, `fix:`, `test:`, `chore:`, `refactor:`)
- Todo módulo de integração de portal isolado em sua própria pasta, sem vazamento de detalhes de protocolo (SOAP, REST, scraping) para o resto da aplicação
- Tipagem estrita do TypeScript (`strict: true` no `tsconfig.json`)

---

## 11. Definition of Done — Fase 1 (MVP Juazeiro)

- [ ] Login funcional com dois papéis (Admin/Estagiário), com testes E2E
- [ ] CRUD de usuários (Admin), com testes
- [ ] Cadastro/edição de credenciais do WebISS, criptografadas, com testes
- [ ] Integrador `juazeiro-webiss` implementado e testado (mocks de resposta SOAP)
- [ ] Sincronização via cron do SO funcionando em ambiente de produção
- [ ] Dashboard responsivo listando movimentações com filtros (cidade, tipo, período, empresa)
- [ ] Notificação por e-mail disparada a cada nova movimentação capturada
- [ ] Cobertura de testes ≥ 80% em `src/lib` e `src/app/api`
- [ ] Pipeline de CI ativo e bloqueando merges com testes quebrados
- [ ] Aplicação implantada e acessível via HTTPS no VPS KingHost

---

## 12. Roadmap Pós-MVP

| Fase | Escopo | Pré-requisito |
|---|---|---|
| 2 | Integração Petrolina | Levantamento técnico do portal (SOAP? REST? Scraping?) |
| 3 | Integração Casa Nova (Fisco.net) | Levantamento técnico do portal |
| 4 | Integração Emissor Nacional | Levantamento técnico + possível certificado digital |
| 5 (futura) | Comparação automática com sistema interno | Definir formato de exportação/API do sistema interno |
| 6 (futura) | Permissão por cidade/empresa | Caso a equipe cresça e precise de segmentação |

---

## 13. Glossário

- **NFS-e**: Nota Fiscal de Serviço Eletrônica
- **WebISS**: Sistema de gestão de ISS/NFS-e usado por Juazeiro-BA, com API SOAP
- **Integrador/Adapter**: Módulo responsável por abstrair a comunicação com um portal municipal específico
- **TDD**: Test-Driven Development — desenvolvimento guiado por testes
