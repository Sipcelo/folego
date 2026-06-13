# Fôlego — Arquitetura do Backend

> Documento de arquitetura. Nenhuma implementação avança sem a aprovação da checklist final.
> Stack fixa: Java 21, Spring Boot 3.x, PostgreSQL, Spring Data JPA, Flyway, Spring Security + JWT, Bean Validation, JUnit 5 + Testcontainers, Maven, deploy Railway.

---

## 1. Visão geral e princípios

O backend é a **fonte única de verdade** do Fôlego. O frontend (Next.js/PWA existente) deixa de calcular qualquer valor financeiro e passa a ser um cliente magro: envia comandos, recebe valores prontos.

Princípios:

1. **Backend dono dos cálculos.** Saldo, totais, status de orçamento, previsões e insights são computados exclusivamente no servidor. O frontend nunca deriva valor financeiro — apenas formata e exibe.
2. **API REST versionada.** Todo endpoint vive sob `/api/v1`. Mudanças incompatíveis exigem nova versão; `/api/v1` não quebra contrato.
3. **Multiusuário com isolamento absoluto.** Todo dado pertence a um usuário; nenhuma query atravessa essa fronteira (ver §5).
4. **GRC como cidadão de primeira classe.** Auditoria de mutações, validação nas bordas e erros padronizados não são opcionais — fazem parte da definição de pronto de cada feature.
5. **TDD.** Nenhuma feature avança sem teste passando (ver §7).

---

## 2. Arquitetura em camadas

Fluxo único de dependência: `controller → service → repository`. Nunca o inverso.

| Camada | Responsabilidade | O que NÃO faz |
|---|---|---|
| **Controller** | Recebe HTTP, valida DTO de entrada (Bean Validation), delega ao service, devolve DTO de saída | Regra de negócio, acesso a repositório, montar entidade |
| **Service** | Regras de negócio, cálculos financeiros, orquestração transacional (`@Transactional`), gravação de auditoria | Conhecer HTTP (sem `HttpServletRequest`, sem status codes) |
| **Repository** | Acesso a dados via Spring Data JPA, queries derivadas/JPQL | Lógica de negócio, cálculo financeiro em SQL complexo não testável |

Contratos:

- **DTOs como `record`s** em todos os pontos de entrada e saída da API (`CreateTransactionRequest`, `TransactionResponse`, `BudgetStatusResponse`...).
- **Entidades JPA nunca saem da camada de service.** Mapeamento explícito entidade↔DTO em mappers dedicados (classes simples, sem MapStruct na primeira fase — mapeamento manual é auditável e suficiente).
- **Regras de negócio vivem nos services.** Entidades carregam apenas invariantes estruturais simples (não-nulo, relação obrigatória); cálculos ficam em services dedicados (§4).

---

## 3. Domínio e responsabilidades

| Entidade | Papel | Invariantes principais |
|---|---|---|
| **User** | Titular da conta; raiz de todo isolamento de dados. | E-mail único; senha apenas como hash BCrypt; nunca exposto em DTO de outra entidade. |
| **Account** | Conta financeira (carteira, banco) com saldo inicial. | Pertence a exatamente um user; saldo atual é **derivado** (inicial + transações), nunca armazenado redundante. |
| **Category** | Classificação de gastos/receitas (nome, cor, ícone, tipo). Existem **categorias globais** (`user_id` NULL, criadas por seed, somente leitura para usuários) e **categorias do usuário**. | Nome único por user (e único entre as globais); usuário não edita/arquiva categoria global; não pode ser excluída se referenciada (ou exige migração das transações). |
| **Transaction** | Movimentação financeira (valor, tipo entrada/saída, data, descrição). | Sempre pertence a um user, a uma account e a uma category do MESMO user; valor > 0 com tipo definindo o sinal; data obrigatória. |
| **Budget** | Limite mensal por categoria, com flag `rollover`. | Único por (user, category, mês de vigência da configuração); limite > 0; rollover é opt-in e default `false`. |
| **RecurringTransaction** | Lançamento recorrente (frequência semanal/mensal/anual). | Pertence a user + category; frequência e valor obrigatórios; materialização em Transaction é explícita, nunca implícita. |
| **Goal** | Meta de poupança (nome, valor alvo, prazo opcional). O valor atual é **derivado** de `goal_contributions` (sem coluna de estado). | Pertence a um user; valor alvo > 0; "valor atual ≥ 0" é regra de service: retirada (aporte negativo) não pode deixar a meta negativa; aportes só via operação dedicada (auditável). |
| **AuditLog** | Registro imutável de toda mutação (quem/o quê/quando/antes/depois). | Append-only: sem UPDATE/DELETE; gravado na mesma transação da mutação que o originou. |

---

## 4. Lógica financeira no servidor

Todos os cálculos abaixo são responsabilidade exclusiva do backend e expostos via API como valores prontos. Implementados como **funções puras sempre que possível**: services de cálculo recebem coleções/valores já carregados e retornam resultados, sem tocar em repositório — o que os torna unit-testáveis sem banco.

| Cálculo | Definição | Service responsável |
|---|---|---|
| Saldo (por conta e total) | saldo inicial + entradas − saídas | `BalanceService` |
| Totais do mês | soma de entradas e saídas do mês corrente | `MonthlySummaryService` |
| Gastos por categoria | agregação das saídas do mês por categoria | `MonthlySummaryService` |
| Status de orçamento com rollover | `saldo_mes_anterior = limite − gasto_mes_anterior` (pode ser negativo); `disponível = limite + saldo_mes_anterior`; sem rollover, `disponível = limite`; status (ok/alerta/estourado) calculado contra o disponível. Quando o mês anterior não tem budget registrado, usa-se o **limite vigente** no cálculo do saldo anterior (comportamento atual do app). | `BudgetService` |
| Total mensal de recorrentes | normalização: semanal × 52 ÷ 12; anual ÷ 12; mensal × 1 | `RecurringService` |
| Sobra livre do mês | `sobra_livre = saldo_total_atual − saídas recorrentes ATIVAS ainda pendentes no mês` (não é entradas−saídas do mês; semântica do app atual, fonte: `lib/insights.ts`) | `MonthlySummaryService` |
| Previsão de fim de mês | `previsao = saldo_atual + entradas_recorrentes_pendentes_do_mes − saidas_recorrentes_pendentes_do_mes − gasto_variavel_projetado`, onde `gasto_variavel_projetado = (saídas não-recorrentes do mês ÷ dias decorridos) × dias restantes`. Entrada variável não é projetada (conservador). | `ForecastService` |
| Progresso de metas | valor atual ÷ valor alvo (proteção contra divisão por zero) | `GoalService` |
| Insights rule-based | regras determinísticas sobre os agregados acima (orçamento estourado, gasto atípico, etc.), consumindo o status JÁ ajustado por rollover | `InsightService` |

Organização: a camada de service separa **services de aplicação** (orquestram repositórios, transações e auditoria) de **services de cálculo** (puros). Ex.: `BudgetService` (aplicação) carrega dados e delega a `BudgetCalculator` (puro) — espelhando o padrão já validado no frontend (`lib/rollover.ts`).

---

## 5. Segurança

- **Autenticação:** Spring Security + JWT.
  - *Access token* curto (15 min) + *refresh token* longo (7 dias) com rotação: cada refresh invalida o anterior (tabela de refresh tokens com revogação).
  - Senha com **BCrypt** (cost padrão do Spring, ≥ 10). Nenhuma senha ou hash em log.
- **Isolamento absoluto por usuário:**
  - Toda tabela de domínio tem `user_id NOT NULL` com FK.
  - **Toda query de repositório filtra por `user_id`** — métodos derivam de `findByIdAndUserId(...)`, nunca `findById(...)` em dados de domínio.
  - O `user_id` vem **exclusivamente do token autenticado** (SecurityContext), nunca de parâmetro do cliente.
  - Camada de defesa extra: services validam posse antes de mutar (carregar por id+user); recurso de outro usuário resulta em **404** (não 403, para não vazar existência).
  - Teste de regressão obrigatório por endpoint: usuário A não lê nem muta dado do usuário B.
- **Rate limiting básico:** limitador em memória (Bucket4j) nos endpoints de autenticação (login/refresh/registro) — ex.: 10 req/min por IP — e limite global mais frouxo no restante. Suficiente para single-instance no Railway.
- **CORS:** allowlist restrita ao(s) domínio(s) do frontend (env var), métodos e headers explícitos, sem `*` em produção.

---

## 6. GRC / Compliance

- **Auditoria:** toda mutação (create/update/delete) grava em `audit_log`. A autenticação é auditada pelas mutações de `refresh_tokens`: login = INSERT do token; logout/revogação (incluindo rotação no refresh) = DELETE do token — sem ação "LOGIN" à parte. Cada registro contém: `user_id` (quem), `entity_type` + `entity_id` + `action` (o quê), `timestamp` UTC (quando), `before` e `after` como JSONB (estado anterior/posterior, com campos sensíveis omitidos). Gravação **na mesma transação** da mutação: ou ambos persistem, ou nenhum. Tabela append-only (sem endpoints de edição; permissão de UPDATE/DELETE negada ao role da aplicação).
- **Validação rigorosa nas bordas:** Bean Validation em todos os DTOs de entrada (`@NotNull`, `@Positive`, `@Email`, limites de tamanho); validação semântica complementar nos services (ex.: categoria pertence ao user). Nada entra no domínio sem passar pelas duas.
- **Erros padronizados:** **Problem Details (RFC 7807)** via suporte nativo do Spring Boot 3 (`ProblemDetail`). Handler global (`@RestControllerAdvice`) mapeia validação (400), autenticação (401), não-encontrado (404), conflito (409) e erro interno (500, sem stack trace no corpo).
- **LGPD:**
  - *Minimização:* coleta-se apenas e-mail, nome e dados financeiros funcionais. Sem CPF, telefone ou dado que o produto não usa.
  - *Exclusão do titular:* endpoint autenticado `DELETE /api/v1/users/me` apaga dados financeiros (hard delete) e **anonimiza** o registro do user e as entradas de `audit_log` (substitui identificadores por pseudônimo irreversível), preservando a integridade do trilho de auditoria sem reter dado pessoal.
  - *Logs de aplicação* não registram corpo de requisição com dados pessoais ou financeiros.

---

## 7. Estratégia de testes (TDD)

Regra do projeto: **nenhuma feature avança sem teste passando.** Ordem: teste primeiro, implementação depois.

| Nível | Escopo | Ferramentas |
|---|---|---|
| **Unit** | Services de cálculo puros (rollover, normalização de recorrentes, previsão, sobra livre, insights) — casos de borda: mês sem dados, divisão por zero, rollover negativo | JUnit 5, AssertJ |
| **Integração (repositório)** | Queries com filtro por `user_id`, constraints, migrações Flyway aplicadas | Testcontainers (PostgreSQL real) |
| **Integração (API)** | Fluxos completos: registro → login → CRUD → cálculo; isolamento entre usuários; contratos de erro RFC 7807 | Spring Boot Test + Testcontainers + MockMvc/RestClient |

Convenções: nomenclatura `deve_<resultado>_quando_<condicao>`; dado de teste por builders; sem H2 (paridade com produção via Testcontainers); cobertura não é meta numérica — o critério é todo cálculo e toda fronteira de segurança terem teste.

---

## 8. Estrutura de pacotes e convenções

```
br.com.folego
├── config/              # Security, CORS, Jackson, OpenAPI
├── common/
│   ├── error/           # GlobalExceptionHandler, exceções de domínio
│   └── audit/           # AuditService, AuditLog (entidade + repo)
├── auth/                # AuthController, AuthService, JWT, RefreshToken
├── user/                # User, UserController (me/exclusão), UserService
├── account/             # Account + controller/service/repository/dto
├── category/            # Category + controller/service/repository/dto
├── transaction/         # Transaction + controller/service/repository/dto
├── budget/
│   ├── BudgetService    # aplicação (orquestra)
│   └── BudgetCalculator # cálculo puro (rollover, status)
├── recurring/           # RecurringTransaction + normalização mensal
├── goal/                # Goal + progresso
└── insight/             # InsightService (rule-based) + ForecastService
                         # + MonthlySummaryService, BalanceService
```

Organização **por feature** (não por camada técnica): cada pacote contém seu controller, service, repository, entidade e DTOs.

Convenções de nomes:
- Entidades no singular (`Transaction`); tabelas no plural snake_case (`transactions`).
- DTOs: `CreateXRequest` / `UpdateXRequest` / `XResponse` (records).
- Services de cálculo puro com sufixo `Calculator`; services de aplicação com sufixo `Service`.
- Migrações Flyway: `V<seq>__descricao_curta.sql` (ex.: `V3__create_budgets.sql`).
- Endpoints: substantivos no plural, kebab-case (`/api/v1/recurring-transactions`).

---

## 9. Mini-ADRs

**ADR-001 — Dinheiro como tipo exato**
Contexto: `double`/`float` acumulam erro binário em soma de valores monetários.
Decisão: `BigDecimal` no Java e `NUMERIC(14,2)` no PostgreSQL; arredondamento `HALF_EVEN` centralizado.
Motivo: exatidão é requisito de domínio financeiro e de auditoria; erro de centavos invalida o trilho.

**ADR-002 — Datas em UTC, date-only onde couber**
Contexto: transações e budgets são conceitos de "dia"/"mês", não de instante; fuso do usuário varia.
Decisão: timestamps de sistema (`created_at`, auditoria) em `TIMESTAMPTZ` UTC; datas de domínio (data da transação, mês do budget) como `DATE`/`LocalDate` sem fuso. Mês de referência sempre `YYYY-MM` derivado no servidor.
Motivo: evita o bug clássico de transação "mudar de dia" por timezone; UTC nos instantes garante ordenação consistente da auditoria.

**ADR-003 — Soft delete vs hard delete**
Contexto: auditoria pede rastreabilidade; LGPD pede capacidade de apagar.
Decisão: **hard delete** para dados de domínio (transações, metas, budgets), com o estado anterior preservado no `audit_log` (`before` JSONB). **Soft delete apenas para Category e Account** (flag `archived`), pois são referenciadas historicamente. Exclusão de conta do titular: hard delete + anonimização (§6).
Motivo: soft delete generalizado complica todas as queries (filtro extra = risco de vazamento) e conflita com LGPD; o audit_log já cumpre o papel de histórico.

**ADR-004 — Paginação padrão**
Contexto: listas de transações crescem sem limite.
Decisão: paginação obrigatória em toda listagem: `page` (0-based) + `size` (default 20, máx 100), ordenação default `date DESC, id DESC`; resposta com envelope (`content`, `page`, `totalElements`, `totalPages`).
Motivo: previne resposta sem limite (desempenho e abuso); ordenação estável com desempate por `id` evita itens duplicados/pulados entre páginas.

**ADR-005 — user_id do token, nunca do cliente**
Contexto: isolamento multiusuário é o maior risco do sistema.
Decisão: nenhum endpoint aceita `user_id` como parâmetro; identidade vem só do JWT validado; repositórios não expõem buscas sem filtro de user para entidades de domínio.
Motivo: elimina por construção a classe de vulnerabilidade IDOR (acesso a dado alheio por manipulação de id).

**ADR-006 — Recorrentes materializam, não simulam**
Contexto: lançamento recorrente pode virar transação real ou ser apenas projetado.
Decisão: recorrentes são templates; a materialização em `Transaction` é uma operação explícita e auditada. Projeções (total mensal normalizado) são cálculo de leitura, sem efeito colateral.
Motivo: separa fato (transação) de plano (recorrente); evita jobs implícitos criando dados sem trilha clara.

**ADR-007 — Rollover quando o mês anterior não tem budget**
Contexto: o budget é versionado por mês (`reference_month`); pode não existir registro para o mês anterior.
Decisão: na ausência de budget do mês anterior, o cálculo do `saldo_mes_anterior` usa o **limite vigente** (`saldo = limite_vigente − gasto_mes_anterior`); sem nenhum gasto anterior, saldo = 0.
Motivo: paridade com o comportamento atual do app (`lib/rollover.ts`), que deriva o saldo do limite atual; evita penalizar/beneficiar o usuário por não ter configurado o mês anterior.

---

## Checklist de revisão

Aprovar ou vetar cada decisão antes da próxima etapa (modelagem de banco e contrato da API):

- [ ] Backend como fonte única de verdade; frontend não calcula nada financeiro (§1)
- [ ] API versionada em `/api/v1` (§1)
- [ ] Camadas controller → service → repository com DTOs records e mapeamento manual explícito, sem expor entidades JPA (§2)
- [ ] Domínio com 8 entidades: User, Account, Category, Transaction, Budget (flag rollover), RecurringTransaction, Goal, AuditLog (§3)
- [ ] Saldo de conta sempre derivado, nunca armazenado redundante (§3)
- [ ] Cálculos financeiros em services dedicados, com núcleo em funções puras (`*Calculator`) testáveis sem banco (§4)
- [ ] Fórmula de rollover: disponível = limite + (limite − gasto do mês anterior), podendo carregar déficit (§4)
- [ ] Normalização de recorrentes: semanal ×52÷12, anual ÷12 (§4)
- [ ] JWT com access 15 min + refresh 7 dias com rotação e revogação (§5)
- [ ] BCrypt para senhas (§5)
- [ ] Isolamento por user_id em toda query; recurso alheio responde 404; user_id nunca vem do cliente (§5, ADR-005)
- [ ] Rate limiting em memória (Bucket4j) focado nos endpoints de auth (§5)
- [ ] CORS allowlist por env var, sem wildcard em produção (§5)
- [ ] Auditoria append-only com before/after JSONB, na mesma transação da mutação (§6)
- [ ] Erros no padrão Problem Details RFC 7807 (§6)
- [ ] LGPD: minimização + endpoint de exclusão com hard delete e anonimização do audit_log (§6)
- [ ] TDD com unit (cálculos), integração com Testcontainers (repos e API), sem H2 (§7)
- [ ] Estrutura de pacotes por feature, convenções de nomes propostas (§8)
- [ ] ADR-001: BigDecimal + NUMERIC(14,2), arredondamento HALF_EVEN (§9)
- [ ] ADR-002: UTC para instantes, LocalDate para datas de domínio (§9)
- [ ] ADR-003: hard delete no domínio + soft delete (archive) só em Category/Account (§9)
- [ ] ADR-004: paginação obrigatória, default 20, máx 100, ordenação estável (§9)
- [ ] ADR-006: recorrentes como templates com materialização explícita (§9)
- [ ] ADR-007: rollover sem budget no mês anterior usa o limite vigente (§9)
