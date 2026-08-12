Analisei a spec como um todo — arquitetura, modelo de dados, RFs/RNFs e as regras de TDD. Aqui vai o que eu cortaria, melhoraria e as lacunas reais que travariam a implementação se não forem resolvidas antes de começar a codar.

Lacunas de negócio (as mais críticas — decidem o modelo de dados)

1. PortalCredential 1:1 com City está provavelmente errado.
No WebISS (e na maioria dos portais de NFS-e municipais), o acesso costuma ser por prestador de serviço (CNPJ), não por prefeitura. O schema tem Company (com CNPJ) dentro de City, mas as credenciais estão amarradas só à cidade (cityId @unique). Se Juazeiro tiver mais de uma empresa do grupo emitindo notas, isso quebra — cada empresa provavelmente precisa de login/senha (ou certificado) próprios no WebISS. Isso precisa ser confirmado na documentação SOAP antes de fechar o schema, porque muda a modelagem inteira de credenciais.

2. Não existe RF para cadastro de Company (CNPJ).
RF03 cobre só credenciais de portal. Mas quem cadastra as empresas/CNPJs que serão monitoradas em cada cidade? Sem isso, não há como popular Movement (que depende de companyId). Falta um RF04-bis (ou expandir RF03) para CRUD de empresas, e um plano de seed inicial de City + Company para Juazeiro.

3. Critério de disparo de notificação (RF05) está ambíguo.
"quando uma nova movimentação for capturada (ex: cancelamento)" — isso dispara e-mail para todo tipo de movimentação ou só para tipos "críticos" (cancelamento/transferência)? Se for tudo, ENVIO (que deve ser o volume normal do dia a dia) vai gerar spam de e-mail e destruir a utilidade do alerta. Isso é uma decisão de produto que precisa ser fechada antes de implementar RF05, não durante.

4. Falta modelo de dados para "destinatários configuráveis" (RF03/RF05).
O RF05 diz que o Admin configura destinatários das notificações, mas não existe tabela nenhuma pra isso no schema (só NotificationLog, que é histórico de envio, não configuração). Falta uma NotificationRecipient (ou campo em City/global).

Lacunas de arquitetura / dados

5. Sem estratégia de idempotência/dedup em Movement.
O cron roda a cada 15 min. Se a janela de busca no WebISS tiver qualquer sobreposição (o que é comum fazer de propósito, como margem de segurança contra atraso de processamento do portal), o mesmo movimento vai ser inserido duas vezes. Falta uma constraint única, algo como @@unique([companyId, nfseNumber, type, protocolNumber]), e lógica de upsert no script de sync — não um create simples.

6. Interface NfseIntegrator não cobre RF03 ("testar conexão").
A interface só tem fetchMovements e normalize. RF03 exige testar credencial antes de salvar — isso precisa de um método explícito tipo testConnection(credentials) no contrato, senão cada integrador vai implementar isso de um jeito diferente e a abstração perde sentido.

7. NotificationLog.movementId não tem @relation declarada (diferente de Company/City/Movement, que têm FK explícita). Inconsistência no schema — sem integridade referencial garantida pelo Prisma/Postgres nesse ponto.

8. Sem plano para falha repetida de sincronização.
SyncLog registra erro, mas nada monitora "WebISS fora do ar há 3 execuções seguidas" e avisa alguém. Numa ferramenta de conferência fiscal, silêncio prolongado do sync é tão grave quanto um cancelamento não notificado — vale um RF06 simples de alerta operacional (e-mail se X syncs falharem seguidos).

9. Timezone dos dados do WebISS não é mencionado.
occurredAt vem do portal — se o SOAP devolver string sem timezone (comum em sistemas legados de prefeitura), é fácil introduzir bug de "-3h" silencioso. Vale registrar explicitamente no doc que isso precisa ser validado ao ler o WSDL.

Lacunas operacionais

10. Sem estratégia de backup do Postgres.
Isso guarda rawPayload (dado de auditoria fiscal) — perder o banco é perder histórico que talvez não seja recuperável do portal depois. A seção de deploy (9) não menciona backup/snapshot do banco. Isso deveria estar no DoD da Fase 1, não ser esquecido.

11. Deploy (seção 9) não roda migration em produção.
O CI roda prisma migrate deploy, mas o passo de deploy na VPS (seção 9) não menciona isso — só sobe o processo via PM2. Falta esse passo explícito no fluxo de deploy real.

12. Rotação da CREDENTIALS_ENCRYPTION_KEY não é endereçada. Se precisar trocar a chave um dia, não tem re-encriptação prevista. Não precisa resolver agora, mas vale uma nota de "risco aceito" no doc.

O que eu cortaria/simplificaria

13. Regra de TDD "nenhuma linha sem teste primeiro" é rígida demais para ser levada ao pé da letra.
Faz total sentido para src/lib/integrations, src/lib/crypto.ts, API routes — é onde bugs custam caro. Mas aplicada literalmente a scaffolding do Next.js, configuração do shadcn/ui, componentes puramente visuais, isso vira teatro (teste bobo só pra satisfazer a regra, sem valor real). Eu deixaria explícito: TDD estrito obrigatório em src/lib e src/app/api; nos componentes de UI, testes E2E cobrem o comportamento, não é necessário TDD unitário por componente. Isso evita que o agente (eu, executando) gaste ciclos escrevendo testes triviais de JSX estático.

14. Meta de cobertura 80% pode ser adiada para o final da Fase 1, não gate desde o primeiro PR.
Se o CI já bloqueia merge abaixo de 80% desde o commit zero, os primeiros PRs (scaffolding, schema, auth básica) vão travar antes de existir lógica de negócio suficiente pra cobrir. Sugiro habilitar o gate de cobertura só a partir de um marco intermediário (ex: depois do integrador Juazeiro implementado).

15. Playwright E2E completo para todos os "fluxos críticos" desde o dia 1 é ambicioso pra MVP solo.
Eu priorizaria E2E só para login + dashboard + filtro (que já cobre autenticação e o core da aplicação) e deixaria "sincronizar manualmente" como E2E com mock — sync real de portal externo em E2E é frágil e caro de manter.

Se quiser, posso já propor os ajustes concretos no schema.prisma (unique constraint em Movement, relação em NotificationLog, tabela de destinatários) e na interface NfseIntegrator (testConnection) antes de começar a Fase 1
