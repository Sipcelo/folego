# Fôlego — Contrato da API v1

> Derivado de `docs/architecture.md` e `docs/modelagem_inicial_banco.md` (aprovados).
> Nenhum código aqui — apenas o contrato REST. Nomes de campo em inglês, espelhando as colunas da modelagem.
> Toda fórmula financeira reproduz **exatamente** a do frontend (`lib/insights.ts`, `lib/rollover.ts`, `lib/store.tsx`) — o backend é a fonte única de verdade; o front vira cliente magro.

---

## 1. Convenções gerais

### 1.1 Base e formato
- Base path: **`/api/v1`**. Toda rota abaixo é relativa a ele.
- Corpo de requisição/resposta: **JSON UTF-8**. `Content-Type: application/json`.
- Idioma de mensagens de erro: pt-BR.

### 1.2 Autenticação
- Esquema **Bearer JWT**: `Authorization: Bearer <access_token>`.
- Toda rota exige autenticação, **exceto** `POST /auth/register`, `POST /auth/login` e `POST /auth/refresh`.
- O `user_id` é extraído **exclusivamente do token** (SecurityContext). Nenhuma rota aceita `userId` no path, query ou body (ADR-005).

### 1.3 Tipos de dado na borda
- **IDs**: `UUID` (string). Ex.: `"3f1c…"`.
- **Dinheiro**: **string decimal** com 2 casas, ex.: `"1450.00"`. *Decisão (ver checklist):* dinheiro trafega como string, não número — evita qualquer perda de precisão de ponto flutuante no cliente e espelha o `BigDecimal`/`NUMERIC(14,2)` (ADR-001). Na entrada, aceita string ou número; sempre validado como > 0 onde o domínio exige.
- **Datas de domínio** (`occurredOn`, `referenceMonth`, `nextChargeOn`, `deadline`, `contributedOn`): `LocalDate` no formato **`yyyy-MM-dd`** (ADR-002).
- **Instantes de sistema** (`createdAt`, `updatedAt`, `occurredAt`): ISO-8601 **UTC** com `Z`, ex.: `"2026-06-13T01:02:03Z"`.
- **Mês de referência**: sempre o **dia 1** (`"2026-06-01"`), canônico (CHECK no banco).

### 1.4 Paginação (ADR-004)
Aplica-se **apenas** a coleções que crescem sem limite — neste contrato, **somente `GET /transactions`**. Coleções limitadas (contas, categorias, orçamentos, recorrentes, metas) retornam **array simples**, sem envelope.

Query params da paginação: `page` (0-based, default `0`), `size` (default `20`, máx `100`).
Envelope de resposta:
```json
{
  "content": [ /* … */ ],
  "page": 0,
  "size": 20,
  "totalElements": 137,
  "totalPages": 7
}
```
Ordenação default das transações: `occurredOn DESC, id DESC` (estável, com desempate por id).

### 1.5 Erros — Problem Details (RFC 7807)
Todo erro responde `application/problem+json`:
```json
{
  "type": "https://folego.app/errors/validation",
  "title": "Requisição inválida",
  "status": 400,
  "detail": "Um ou mais campos são inválidos.",
  "instance": "/api/v1/transactions",
  "errors": [
    { "field": "amount", "message": "deve ser maior que zero" }
  ]
}
```
- `errors[]` só aparece em **400 de validação** (Bean Validation + validação semântica).
- Nunca expõe stack trace no corpo (nem em 500).

| Situação | Status | Notas |
|---|---|---|
| Sucesso com corpo | 200 | GET/PATCH/POST de leitura/atualização |
| Recurso criado | 201 | POST de criação; header `Location` com a URL do recurso |
| Sucesso sem corpo | 204 | DELETE, archive, logout |
| Validação (sintática/semântica) | 400 | inclui `errors[]` |
| Sem/!inválido token | 401 | — |
| Recurso **inexistente OU de outro usuário** | 404 | nunca 403 para dado de outro user (não vaza existência — §5/ADR-005) |
| Operação proibida em recurso visível mas não mutável | 403 | ex.: editar/arquivar **categoria global** |
| Conflito de estado/unicidade | 409 | ex.: e-mail já cadastrado; orçamento já existe para (categoria, mês) |
| Erro interno | 500 | sem detalhe sensível |

### 1.6 Isolamento e auditoria (transversais)
- Toda query de dado de domínio filtra por `user_id` do token. Recurso de outro usuário → **404**.
- Toda **mutação** (POST/PATCH/DELETE e materialização) grava em `audit_log` na **mesma transação** (§6). É transparente ao cliente — não há endpoints de auditoria nesta versão.

---

## 2. Autenticação — `/auth`

### `POST /auth/register`  *(público)*
Cria usuário. Não autentica automaticamente (cliente faz login em seguida).
- **Request**
```json
{ "name": "Marcelo", "email": "marcelo@exemplo.com", "password": "•••••••• (mín. 8)" }
```
- **201** → `{ "id": "<uuid>", "name": "...", "email": "..." }` (nunca retorna hash).
- **400** validação (`name` não vazio, `email` válido, `password` ≥ 8).
- **409** e-mail já cadastrado (unicidade case-insensitive, `citext`).

### `POST /auth/login`  *(público)*
- **Request** `{ "email": "...", "password": "..." }`
- **200**
```json
{
  "accessToken": "<jwt>",
  "refreshToken": "<opaco>",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```
`accessToken` expira em **15 min**; `refreshToken` em **7 dias**, com rotação (§5). Login = INSERT em `refresh_tokens` (auditado).
- **401** credenciais inválidas (mensagem genérica, sem distinguir e-mail/senha).

### `POST /auth/refresh`  *(público; usa o refresh token)*
- **Request** `{ "refreshToken": "<opaco>" }`
- **200** → mesmo shape do login, com **novo par** (rotação: o refresh anterior é revogado/DELETE auditado).
- **401** refresh inválido, expirado ou já revogado.

### `POST /auth/logout`  *(autenticado)*
- **Request** `{ "refreshToken": "<opaco>" }`
- **204**. Revoga o refresh informado (DELETE auditado). Tokens de acesso expiram naturalmente.

---

## 3. Usuário — `/users/me`

### `GET /users/me`
- **200** → `{ "id", "name", "email", "createdAt" }`.

### `DELETE /users/me`  *(LGPD)*
Exclusão do titular: **hard delete** de todo o grafo financeiro (CASCADE) + **anonimização** das entradas de `audit_log` (SET NULL no `user_id`, payloads pseudonimizados) — em uma transação (§6).
- **204**. Após isso, tokens do usuário tornam-se inválidos.

---

## 4. Contas — `/accounts`

Soft delete via `archived` (ADR-003); não há DELETE. Saldo é **derivado**, nunca armazenado.

### `GET /accounts`
- **200** → array (inclui arquivadas? **não** por default; `?includeArchived=true` para incluir).
```json
[{
  "id": "<uuid>",
  "name": "Conta Principal",
  "type": "conta",            // conta | dinheiro | cartao
  "initialBalance": "0.00",
  "currentBalance": "9096.70", // derivado: initial + entradas − saídas
  "archived": false,
  "createdAt": "…", "updatedAt": "…"
}]
```

### `GET /accounts/{id}` → **200** | **404**

### `POST /accounts`
- **Request** `{ "name", "type", "initialBalance": "0.00" }` (`type` ∈ conta|dinheiro|cartao; `initialBalance` pode ser negativo — cartão).
- **201** + `Location`. **409** se `(user, name)` já existe (UNIQUE).

### `PATCH /accounts/{id}`
- Campos opcionais: `name`, `type`, `initialBalance`. **200** | **400** | **404** | **409** (nome duplicado).

### `POST /accounts/{id}/archive` / `POST /accounts/{id}/unarchive`
- **204** | **404**. Arquivar não apaga; transações históricas permanecem (FK RESTRICT impede exclusão real).

---

## 5. Categorias — `/categories`

Globais (`userId` NULL, seed, **somente leitura**) + categorias do usuário.

### `GET /categories`
- **200** → array com globais **e** do usuário (`?type=gasto|receita` opcional; `?includeArchived=true`).
```json
[{
  "id": "<uuid>", "name": "Mercado", "icon": "shopping-cart",
  "color": "#22C55E", "type": "gasto",   // gasto | receita
  "global": true,            // true = global (read-only); false = do usuário
  "archived": false, "createdAt": "…", "updatedAt": "…"
}]
```

### `POST /categories`
- Cria categoria **do usuário**. Request `{ "name", "icon", "color" (#RRGGBB), "type" }`.
- **201**. **409** nome duplicado para o usuário (case-insensitive).

### `PATCH /categories/{id}` / `POST /categories/{id}/archive`
- **200/204** para categoria do usuário.
- **403** se for **categoria global** (visível, porém imutável — não é 404 porque não é dado de outro usuário; é recurso público read-only).
- **404** se inexistente.

---

## 6. Transações — `/transactions`

### `GET /transactions`  *(paginado)*
Query: `page`, `size`, `month` (`yyyy-MM`), `categoryId`, `type` (entrada|saida), `from`/`to` (`yyyy-MM-dd`), `q` (busca em `description`).
- **200** → envelope de paginação; cada item:
```json
{
  "id": "<uuid>", "type": "saida", "amount": "150.00",
  "categoryId": "<uuid>", "accountId": "<uuid>",
  "occurredOn": "2026-06-11", "description": "Gasolina",
  "createdAt": "…", "updatedAt": "…"
}
```

### `GET /transactions/{id}` → **200** | **404**

### `POST /transactions`
- **Request** `{ "type", "amount", "categoryId", "accountId", "occurredOn", "description"? }`.
- Regras (service): `amount > 0` (sinal vem de `type`); `categoryId` e `accountId` devem **pertencer ao mesmo usuário** (senão **404** — não revela posse alheia); `occurredOn` obrigatória.
- **201** + `Location`. **400** validação.

### `PATCH /transactions/{id}`
- Campos parciais (mesma validação de posse cruzada). **200** | **400** | **404**.

### `DELETE /transactions/{id}`
- **Hard delete** (ADR-003); estado anterior preservado no `audit_log`. **204** | **404**.

---

## 7. Orçamentos — `/budgets`

Versionado por **(categoria, mês)**. A leitura já devolve o **status calculado com rollover** (espelha `orcamentoMes` do store).

### `GET /budgets`
Query: `month` (`yyyy-MM`, default = mês corrente).
- **200**
```json
{
  "referenceMonth": "2026-06-01",
  "items": [{
    "categoryId": "<uuid>",
    "category": { "id": "...", "name": "Lazer", "icon": "...", "color": "...", "type": "gasto" },
    "spent": "210.00",          // saídas da categoria no mês
    "monthlyLimit": "200.00",
    "rollover": false,
    "carryover": "0.00",        // saldoMesAnterior: + sobra / − estouro; 0 sem rollover
    "available": "200.00",      // disponivel = limite + carryover
    "percentage": 105.0,        // gasto / disponivel * 100
    "status": "estourado"       // ok (<80) | alerta (80–100) | estourado (>100)
  }],
  "totalSpent": "...", "totalLimit": "...", "remaining": "..."
}
```
> Fórmula do rollover (idêntica a `lib/rollover.ts`): se o mês anterior tem saídas na categoria, `carryover = limiteVigente − gastoMesAnterior` e `available = limite + carryover`; senão `carryover = 0`, `available = limite` (ADR-007). `status`/`percentage` sempre contra `available`.

### `PUT /budgets`  *(upsert)*
- **Request** `{ "categoryId", "referenceMonth": "2026-06-01", "monthlyLimit": "800.00", "rollover": true }`.
- Cria ou atualiza o orçamento de `(categoria, mês)`. `monthlyLimit > 0`. **200** (atualizado) | **201** (criado) | **400** | **404** (categoria de outro user).

### `DELETE /budgets`
- Query `categoryId` + `month`. Remove o orçamento daquela categoria/mês (paridade com `setOrcamento` que apaga quando limite ≤ 0). **204** | **404**.

---

## 8. Recorrentes — `/recurring-transactions`

Templates; materialização é explícita e auditada (ADR-006).

### `GET /recurring-transactions`
Query: `status` (ativo|cancelado) opcional.
- **200**
```json
{
  "items": [{
    "id": "<uuid>", "name": "Amazon Prime", "type": "saida", "amount": "19.90",
    "categoryId": "<uuid>", "accountId": "<uuid>",
    "frequency": "mensal",        // semanal | mensal | anual
    "status": "ativo",            // ativo | cancelado
    "nextChargeOn": "2026-06-12",
    "monthlyEquivalent": "19.90", // normalizado: semanal×52÷12, anual÷12, mensal×1
    "createdAt": "…", "updatedAt": "…"
  }],
  "summary": {
    "monthlyActiveTotal": "165.03",  // só saídas ativas
    "annualActiveTotal": "1980.36",
    "upcomingCharges": [ /* ativos ordenados por nextChargeOn ASC */ ]
  }
}
```

### `GET /{id}` → **200** | **404**

### `POST /recurring-transactions`
- **Request** `{ "name", "type", "amount", "categoryId", "accountId", "frequency", "nextChargeOn" }`. Cria com `status: "ativo"`. **201** | **400** | **404** (posse cruzada).

### `PATCH /{id}` → atualização parcial. **200** | **400** | **404**

### `POST /{id}/cancel` / `POST /{id}/reactivate`
- Alterna `status` (espelha `toggleStatusRecorrente`). **204** | **404**.

### `POST /{id}/post`  *(materializar — "Lançar agora")*
Cria uma **transação real** a partir do template e avança `nextChargeOn` conforme a frequência (semanal +7d, mensal +1 mês, anual +1 ano). Operação única, auditada (ADR-006).
- **Request** (opcional) `{ "occurredOn": "2026-06-13" }` (default = hoje).
- **201**
```json
{
  "transaction": { /* TransactionResponse criada */ },
  "recurring": { /* recorrente com nextChargeOn avançado */ }
}
```
- **404** | **409** se o recorrente estiver `cancelado` (não materializa cancelado).

### `DELETE /{id}` → hard delete (audit guarda o estado). **204** | **404**.

---

## 9. Metas — `/goals`

`currentAmount` e progresso são **derivados** de `goal_contributions` (sem coluna de estado).

### `GET /goals`
- **200**
```json
[{
  "id": "<uuid>", "name": "Reserva", "targetAmount": "5000.00",
  "currentAmount": "1000.00",     // soma dos aportes (derivado)
  "progress": 20.0,               // currentAmount / targetAmount * 100 (0 se alvo 0)
  "deadline": "2026-12-01",       // ou null
  "createdAt": "…", "updatedAt": "…"
}]
```

### `GET /{id}` → **200** | **404**

### `POST /goals`
- **Request** `{ "name", "targetAmount", "deadline"?, "initialAmount"? }`.
- `targetAmount > 0`. Se `initialAmount > 0`, cria **1 aporte inicial** em `goal_contributions` (espelha `addMeta` com `valorAtual`). **201** | **400**.

### `PATCH /{id}` → `name`, `targetAmount`, `deadline`. **200** | **400** | **404**.

### `POST /{id}/contributions`  *(aporte / retirada)*
- **Request** `{ "amount": "200.00", "contributedOn"? }`. `amount ≠ 0`; **negativo = retirada**.
- **Regra de service (invariante "valor atual ≥ 0"):** retirada não pode deixar `currentAmount` negativo → **409** se exceder o saldo da meta.
- **201** → meta atualizada com `currentAmount`/`progress` recalculados.
> Observação de paridade: o app hoje **limita o aporte ao alvo** (`Math.min(... , valorAlvo)`). No backend isso é uma **decisão a confirmar** (ver checklist): permitir ultrapassar o alvo (mais honesto contábil) ou manter o teto do app.

### `GET /{id}/contributions` → histórico de aportes (array, mais recente primeiro). **200** | **404**.

### `DELETE /{id}` → hard delete (CASCADE nos aportes; audit guarda). **204** | **404**.

---

## 10. Read models / Cálculos (backend grosso)

Endpoints de **leitura** que entregam valores prontos (§4). Núcleo em `*Calculator` puro; estes endpoints só orquestram.

### `GET /dashboard`
Resumo da home, all-time + projeções do mês.
```json
{
  "balance": { "total": "9096.70", "totalIncome": "7000.00", "totalExpense": "2283.30" },
  "freeLeftover": {                 // calcularSobraLivre
    "value": "8930.00", "perDay": "470.00", "daysRemaining": 19, "status": "confortavel"
  },
  "forecast": {                     // calcularPrevisaoFimMes
    "predictedEndBalance": "5505.05",
    "projectedVariableSpend": "3424.95",
    "safeDailySpend": "496.11",
    "daysRemaining": 19, "currentDay": 12, "daysInMonth": 30,
    "startOfMonth": false, "status": "confortavel"
  }
}
```
> `balance.*` são **all-time** (espelham `saldoTotal`/`totalEntradas`/`totalSaidas` do store, sem filtro de mês — é o que o card "Saldo total" exibe). `daysRemaining` usa a fonte única `diasRestantesNoMes` (inclui hoje). Status: `confortavel|apertado|negativo`.

### `GET /insights`
Feed rule-based, já ordenado por prioridade (orçamento estourado > maior gasto > economia com cancelados), consumindo o status **já ajustado por rollover**.
```json
[{
  "type": "orcamentoEstourado",      // orcamentoEstourado | maiorGasto | economiaCancelados
  "tone": "alerta",                  // alerta | neutro | positivo
  "icon": "AlertTriangle",
  "title": "Orçamento estourado",
  "text": "Lazer passou R$ 10,00 do limite.",
  "target": "categorias"             // categorias | recorrentes
}]
```
> O texto formatado em pt-BR (`R$ …`) é responsabilidade de exibição; o backend pode entregar texto pronto **ou** os componentes numéricos. *Decisão a confirmar (checklist):* texto pronto vs. campos numéricos crus.

### `GET /reports/spending-by-category`
Query: `month` (`yyyy-MM`) opcional; **sem `month` = all-time** (espelha `gastosPorCategoria` do store, base do Relatório).
```json
[{ "categoryId": "<uuid>", "category": { /* … */ }, "total": "1450.00", "percentage": 64.0 }]
```

### `GET /reports/weekly-spending`
Query: `month` (`yyyy-MM`, default mês corrente). Saídas agregadas por **semana do mês** (gráfico de barras).
```json
{
  "referenceMonth": "2026-06-01",
  "weeks": [
    { "label": "Sem 1", "from": "2026-06-01", "to": "2026-06-07", "total": "0.00" },
    { "label": "Sem 2", "from": "2026-06-08", "to": "2026-06-14", "total": "2283.30" }
  ]
}
```

---

## 11. Fora de escopo da API v1

Registrado para não haver ambiguidade na implementação:
- **Gamificação** (streak de registro, desafio da semana, marcos celebrados): permanece **client-side** (decisão da modelagem §5; não há tabela nem endpoint).
- **Coach de IA**: rota separada, fora deste contrato de domínio.
- **Migração do `localStorage`**: processo único de importação (usa o de-para da modelagem §5), não um endpoint público versionado.
- **Endpoints de auditoria**: a auditoria é gravada, mas não há leitura via API nesta versão (append-only, acesso fora de banda).

---

## 12. Rastreabilidade — endpoint → fonte

| Endpoint | Cálculo / regra | Fonte |
|---|---|---|
| `GET /accounts` (`currentBalance`) | saldo derivado | store `saldoTotal`/derivação por conta |
| `GET /budgets` | rollover + status | `lib/rollover.ts`, `orcamentoMes`, ADR-007 |
| `GET /recurring-transactions` (`summary`) | normalização mensal | `valorMensal` (store/insights) |
| `POST /{id}/post` | materialização + avanço | ADR-006 |
| `GET /dashboard` (`freeLeftover`) | sobra livre | `calcularSobraLivre` |
| `GET /dashboard` (`forecast`) | previsão fim de mês | `calcularPrevisaoFimMes` |
| `GET /insights` | feed rule-based | `calcularInsights` |
| `GET /reports/spending-by-category` | gastos por categoria | store `gastosPorCategoria` |
| `GET /goals` (`currentAmount`/`progress`) | derivado de aportes | modelagem §2.7/2.8 |

---

## Checklist de revisão

Aprovar ou vetar antes da implementação no Claude Code:

- [ ] Base `/api/v1`; auth Bearer JWT; rotas públicas só register/login/refresh
- [ ] **Dinheiro como string decimal** na borda (não número) — decisão a confirmar
- [ ] Datas de domínio `yyyy-MM-dd`; instantes ISO-8601 UTC; mês de referência no dia 1
- [ ] Paginação (envelope ADR-004) **só** em `GET /transactions`; demais listas são arrays
- [ ] Erros RFC 7807 com mapa de status; `errors[]` só em 400; sem stack trace
- [ ] Isolamento: recurso de outro usuário → **404** (nunca 403); `userId` só do token
- [ ] Categoria global imutável → **403** (não 404, pois é recurso público read-only) — decisão a confirmar
- [ ] Auth: register/login/refresh/logout; refresh com rotação; login/logout auditados via `refresh_tokens`
- [ ] `DELETE /users/me` = hard delete + anonimização do audit (LGPD)
- [ ] Contas: archive em vez de delete; `currentBalance` derivado; `initialBalance` pode ser negativo
- [ ] Categorias: globais + do usuário no `GET`; criação/edição só de categoria do usuário
- [ ] Transações: filtros (month/categoryId/type/from/to/q); posse cruzada validada no service; hard delete
- [ ] Budgets: `GET` devolve status calculado com rollover (idêntico ao front); `PUT` upsert por (categoria, mês); `DELETE` por categoria+mês
- [ ] Recorrentes: summary (mensal/anual/próximas); cancel/reactivate; **`POST /{id}/post`** materializa + avança `nextChargeOn`; não materializa cancelado (409)
- [ ] Metas: `currentAmount`/`progress` derivados; aporte negativo = retirada; retirada além do saldo → 409
- [ ] **Aporte além do alvo**: permitir ultrapassar (backend) vs. manter teto do app — decisão a confirmar
- [ ] `GET /dashboard`: `balance.*` all-time; `freeLeftover` e `forecast` com as fórmulas exatas do front
- [ ] `GET /insights`: texto pronto vs. componentes numéricos — decisão a confirmar
- [ ] `GET /reports/spending-by-category`: sem `month` = all-time (paridade com o Relatório)
- [ ] `GET /reports/weekly-spending`: agregação por semana do mês
- [ ] Fora de escopo: gamificação, coach de IA, migração do localStorage, leitura de auditoria
