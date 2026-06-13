# roadmap.md — Plano de implementação (Fôlego backend)

> Uma fase por vez. **Critério de pronto = testes verdes** (ver `architecture.md` §7). Ao concluir uma fase, atualizar `progress.md`.
> Toda fase pressupõe os docs lidos (`CLAUDE.md` §10).

---

## Fase 0 — Fundação do projeto
**Objetivo:** projeto Maven compilando, vazio porém saudável.
**Entregáveis:**
- `pom.xml` (Java 21, Spring Boot 3.x, deps: web, data-jpa, validation, security, flyway, postgresql, testcontainers, springdoc-openapi).
- Estrutura de pacotes `br.com.folego` por feature (esqueleto vazio).
- `application.yml` com perfis `dev` / `test` / `prod`; segredos via env (`JWT_SECRET`, `DATABASE_URL`). `.env.example` versionado, `.env` no `.gitignore`.
- `.gitignore` (target/, .idea, *.iml, .env).
- Testcontainers configurado para o perfil de teste (Postgres real, sem H2).
**Pronto quando:** `mvn clean verify` passa com um teste de contexto (`contextLoads`) usando Postgres via Testcontainers.

## Fase 1 — Documentação congelada
**Objetivo:** docs como fonte única de verdade dentro do repo.
**Entregáveis:** `CLAUDE.md` (raiz) + `docs/` completo (architecture, roadmap, progress, learn, modelagem, api_v1). ADRs revisados.
**Pronto quando:** os 7 documentos existem e estão coerentes entre si (nomes de coluna, fórmulas, status de erro batem).

## Fase 2 — Migrations + schema
**Objetivo:** banco modelado e versionado.
**Entregáveis:** `V1__schema_inicial.sql` (10 tabelas, constraints, índices, triggers `set_updated_at`, REVOKE no audit_log) + `V2__seed_categorias.sql` (8 categorias globais). Fiel a `modelagem_inicial_banco.md`.
**Pronto quando:** teste de integração sobe o container, aplica as migrations e valida constraints-chave (unicidade de e-mail, partial unique de categorias, CHECK de reference_month, append-only do audit).

## Fase 3 — Domínio + repositórios
**Objetivo:** entidades JPA + repositórios com isolamento por `user_id`.
**Entregáveis:** entidades de todas as tabelas; repositórios Spring Data com queries derivadas filtrando por usuário.
**Pronto quando:** testes de repositório (Testcontainers) provam: persistência correta, isolamento entre usuários, ON DELETE (CASCADE/RESTRICT/SET NULL) conforme a modelagem §3.

## Fase 4 — Cálculos financeiros (núcleo puro)
**Objetivo:** `*Calculator` puros, testados sem banco. **Coração do projeto.**
**Entregáveis:** BalanceCalculator, BudgetCalculator (rollover/status), RecurringNormalizer (mensal/anual), ForecastCalculator (sobra livre + previsão), InsightCalculator (rule-based).
**Pronto quando:** testes unitários cobrem cada fórmula e suas bordas — mês sem dados, divisão por zero, rollover negativo, início de mês, recorrente cancelado. Resultados idênticos aos do frontend (`lib/insights.ts`, `lib/rollover.ts`).

## Fase 5 — Services de aplicação + auditoria
**Objetivo:** orquestração transacional e trilha de auditoria.
**Entregáveis:** `*Service` por feature; `AuditService` gravando before/after na mesma transação; validação semântica (posse cruzada de categoria/conta).
**Pronto quando:** testes provam que toda mutação gera registro de auditoria atômico (rollback da mutação = rollback do audit) e que a regra "meta ≥ 0" e a materialização de recorrentes funcionam.

## Fase 6 — Segurança (auth + JWT)
**Objetivo:** autenticação e isolamento de ponta a ponta.
**Entregáveis:** register/login/refresh/logout; JWT access 15min + refresh 7d com rotação; BCrypt; filtro que injeta `user_id` no contexto; rate limit básico; CORS allowlist.
**Pronto quando:** testes de integração cobrem o fluxo de auth completo, rotação/revogação de refresh, e que requisição sem token → 401 e a recurso alheio → 404.

## Fase 7 — Controllers + DTOs (contrato REST)
**Objetivo:** expor a `api_v1.md` fielmente.
**Entregáveis:** controllers e DTOs (records) de todos os recursos; mappers manuais; `GlobalExceptionHandler` com RFC 7807; paginação só em `/transactions`; springdoc-openapi publicando o contrato.
**Pronto quando:** testes de API (MockMvc/RestClient + Testcontainers) validam cada endpoint, códigos de status, envelope de erro e isolamento — batendo com o contrato.

## Fase 8 — Read models / endpoints de cálculo
**Objetivo:** entregar os valores prontos do backend grosso.
**Entregáveis:** `/dashboard`, `/insights`, `/budgets` (status), `/reports/spending-by-category`, `/reports/weekly-spending`.
**Pronto quando:** testes de API confirmam que os números batem com os `*Calculator` da Fase 4.

## Fase 9 — Deploy + CI/CD
**Objetivo:** projeto rodando e demonstrável (portfólio).
**Entregáveis:** deploy no Railway (Postgres gerenciado + env vars); GitHub Actions rodando `mvn clean verify` no push; README com link vivo; OpenAPI público.
**Pronto quando:** pipeline verde no CI e API acessível pela URL pública.

---

### Decisões pendentes de bater antes da Fase 7
(herdadas da checklist da `api_v1.md`)
- Dinheiro como string vs número no JSON.
- Categoria global imutável → 403 (confirmar).
- Aporte além do alvo: permitir ultrapassar vs. manter teto do app.
- Insights: texto pronto vs. componentes numéricos.
