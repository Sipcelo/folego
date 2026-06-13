# CLAUDE.md — Manual de operação (Fôlego backend)

> **Leia este arquivo inteiro antes de qualquer ação.** Ele é o contrato de como trabalhamos.
> Depois dele, leia **na ordem**: `docs/architecture.md` → `docs/roadmap.md` → `docs/progress.md` → `docs/learn.md` → `docs/modelagem_inicial_banco.md` → `docs/api_v1.md`.
> Nenhuma linha de código antes de ter lido todos.

---

## 1. O que é o projeto

API REST de finanças pessoais (app **Fôlego**). Backend **grosso**: é dono de toda regra de negócio e de todo cálculo financeiro — o frontend (Next.js/PWA) vira cliente magro. Multiusuário autenticado.

**Por que existe:** peça de portfólio para vaga júnior + trilha GRC (Governança, Risco, Compliance). Os diferenciais que o projeto precisa demonstrar são **domínio financeiro exato**, **auditoria/trilha**, **isolamento de dados** e **LGPD** — não quantidade de features.

## 2. Stack (fixa — não trocar sem ADR)

Java 21 · Spring Boot 3.x · PostgreSQL · Spring Data JPA · Flyway · Spring Security + JWT · Bean Validation · JUnit 5 + AssertJ + Testcontainers · Maven · deploy Railway.

Sem H2 (paridade com produção via Testcontainers). Sem MapStruct na primeira fase (mapper manual, auditável). DTOs são `record`s.

## 3. Como trabalhamos (Anti-Vibe-Coding)

- **Dev = Navigator** (pensa, arquiteta, decide). **IA (você) = Piloto** (executa). O esqueleto — arquitetura, schema, regras — já está nos docs. Seu trabalho é preencher os órgãos sem reinterpretar o esqueleto.
- **TDD é obrigatório.** Teste primeiro, implementação depois. **Nenhuma feature avança com teste vermelho.** "Sem teste, toda mudança é aposta."
- **Não conserte código na mão.** Quando algo der errado, o dev explica o erro e **você corrige**. O erro vai documentado em `docs/learn.md`.
- **Uma fase por vez** (ver `docs/roadmap.md`). Não pule fases. Não implemente o que a fase atual não pede.
- **AI-Jail:** você roda dentro do WSL/containers. Migrations, testes e deps você pode executar. Ações destrutivas ou irreversíveis (apagar dados, force-push, mexer em config de produção/segredos) — **pare e peça confirmação ao dev**.

## 4. Comandos

```bash
mvn test                      # roda toda a suíte (sobe Postgres via Testcontainers; Docker precisa estar ligado)
mvn -Dtest=BudgetCalculatorTest test   # roda uma classe de teste
mvn spring-boot:run           # sobe a API local (perfil dev)
mvn clean verify              # build + testes (gate de CI)
mvn flyway:info               # estado das migrations (ou via app no boot)
docker ps                     # confirmar que o Docker está ativo antes de testar
```

Pré-requisitos do ambiente (já instalados): Java 21 (Temurin, via SDKMAN), Maven 3.9, Docker Desktop com integração WSL.

## 5. Estrutura de pacotes (por feature)

```
br.com.folego
├── config/        # Security, CORS, Jackson, OpenAPI
├── common/
│   ├── error/     # GlobalExceptionHandler, exceções de domínio, ProblemDetail
│   └── audit/     # AuditService, AuditLog (entidade + repo)
├── auth/          # AuthController, AuthService, JWT, RefreshToken
├── user/          # User, UserController (me/exclusão LGPD), UserService
├── account/       # CRUD + archive; currentBalance derivado
├── category/      # globais (read-only) + do usuário
├── transaction/   # CRUD + filtros + paginação
├── budget/        # BudgetService (orquestra) + BudgetCalculator (rollover/status, puro)
├── recurring/     # RecurringTransaction + normalização mensal + materialização
├── goal/          # Goal + progresso derivado de aportes
└── insight/       # InsightService, ForecastService, MonthlySummaryService, BalanceService
```

## 6. Convenções

- Entidades singular (`Transaction`); tabelas plural snake_case (`transactions`).
- DTOs: `CreateXRequest` / `UpdateXRequest` / `XResponse` (records).
- Cálculo puro → sufixo `Calculator`; aplicação/orquestração → sufixo `Service`.
- Migrations: `V<seq>__descricao_curta.sql` (ex.: `V3__create_budgets.sql`).
- Endpoints: substantivo plural, kebab-case (`/api/v1/recurring-transactions`).
- Testes: `deve_<resultado>_quando_<condicao>`; dado de teste por builders.

## 7. Regras de negócio invioláveis

Estas são o coração do domínio. Quebrar qualquer uma é bug crítico:

- **Dinheiro:** `BigDecimal` + `NUMERIC(14,2)`, arredondamento `HALF_EVEN` centralizado. Nunca `double`/`float` (ADR-001).
- **Datas:** instantes em UTC (`timestamptz`); datas de domínio em `LocalDate` (ADR-002).
- **Isolamento:** toda query filtra por `user_id` do **token** (nunca do cliente). Recurso de outro usuário → **404**, jamais 403 (ADR-005).
- **Valor > 0:** transações e aportes têm `amount > 0`; o `type` define o sinal.
- **Saldo derivado:** saldo de conta = inicial + entradas − saídas. Nunca armazenado redundante.
- **Rollover (ADR-007):** `disponivel = limite + (limite − gasto_mes_anterior)`; sem budget no mês anterior, usa o limite vigente. Fórmula idêntica a `lib/rollover.ts`.
- **Meta ≥ 0:** retirada (aporte negativo) não pode deixar a meta negativa (regra de service → 409).
- **Recorrentes são templates:** materialização em transação é explícita e auditada (ADR-006). Nunca jobs implícitos.
- **Auditoria:** toda mutação grava em `audit_log` na **mesma transação**. Append-only (sem UPDATE/DELETE). Login/logout = INSERT/DELETE de `refresh_tokens`.
- **Exclusão (ADR-003):** hard delete no domínio; soft delete (archive) só em Account/Category. `DELETE /users/me` = hard delete + anonimização do audit (LGPD).
- **JWT:** access 15 min, refresh 7 dias com rotação e revogação. BCrypt para senhas.
- **Erros:** RFC 7807 (`ProblemDetail`), sem stack trace no corpo.

## 8. Fonte de verdade dos cálculos

As fórmulas de **sobra livre**, **previsão de fim de mês**, **rollover**, **normalização de recorrentes** e **insights** já existem e estão especificadas em `docs/api_v1.md` (§10) e `docs/architecture.md` (§4), espelhando o frontend original. **Não invente fórmula nova.** Se um cálculo parecer ambíguo, pare e pergunte — não improvise.

## 9. Fora de escopo da v1

Não implemente (decisão registrada): gamificação (streak/desafio — fica no cliente), coach de IA, endpoint público de migração do localStorage, endpoints de leitura de auditoria.

## 10. Regras para você, agente

1. Leu todos os docs? Se não, pare e leia.
2. Trabalhe a fase atual do `roadmap.md`. Não pule.
3. Teste primeiro. Não marque feature como pronta com teste vermelho.
4. Ao fim de cada fase: atualize `docs/progress.md` (o que ficou pronto, quais testes passam).
5. Falhou algo / o dev corrigiu seu raciocínio? Registre em `docs/learn.md`.
6. Ação irreversível ou destrutiva? Pare e peça confirmação.
