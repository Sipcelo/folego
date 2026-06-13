# Fôlego — API

API REST de finanças pessoais com foco em **exatidão financeira, auditoria e conformidade (LGPD)**. Backend que concentra toda a regra de negócio e os cálculos; o app cliente é apenas a interface.

> Projeto de portfólio. O objetivo não é volume de features, e sim demonstrar engenharia de backend cuidadosa em um domínio que exige rigor: dinheiro, trilha de auditoria e isolamento de dados entre usuários.

## Stack

Java 21 · Spring Boot 3 · PostgreSQL · Spring Data JPA · Flyway · Spring Security (JWT) · Bean Validation · JUnit 5 + Testcontainers · Maven · Railway

## Destaques de engenharia

- **Dinheiro exato:** `BigDecimal` + `NUMERIC(14,2)`, arredondamento `HALF_EVEN`. Sem ponto flutuante em valor monetário.
- **Auditoria append-only:** toda mutação grava antes/depois no `audit_log`, na mesma transação — append-only garantido por permissão de banco.
- **Isolamento de dados:** toda consulta filtra pelo usuário do token; recurso de outro usuário responde 404 (não vaza existência).
- **LGPD:** exclusão do titular com hard delete do grafo financeiro + anonimização da trilha de auditoria.
- **Cálculos como funções puras testáveis:** rollover de orçamento estilo envelope, previsão de fim de mês e sobra livre isolados em `*Calculator`, cobertos por testes de borda.
- **TDD com paridade de produção:** Testcontainers (PostgreSQL real), sem H2.
- **Erros padronizados:** RFC 7807 (Problem Details).

## Como rodar localmente

**Pré-requisitos:** Java 21, Maven 3.9+, Docker (para os testes com Testcontainers).

```bash
# variáveis de ambiente (copie e ajuste)
cp .env.example .env

# testes (sobe um PostgreSQL efêmero via Docker)
mvn clean verify

# subir a API (perfil dev)
mvn spring-boot:run
```

A documentação interativa da API (OpenAPI/Swagger) fica em `/swagger-ui.html` com a aplicação no ar.

## Documentação

O design é documentado antes do código (`docs/`):

- [`architecture.md`](docs/architecture.md) — camadas, decisões e ADRs.
- [`modelagem_inicial_banco.md`](docs/modelagem_inicial_banco.md) — modelo de dados e migrations.
- [`api_v1.md`](docs/api_v1.md) — contrato REST completo.
- [`roadmap.md`](docs/roadmap.md) — plano de implementação em fases.

## Status

Em desenvolvimento — ver [`docs/progress.md`](docs/progress.md).

## Licença

A definir.
