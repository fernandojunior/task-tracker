# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Visão

**Task Tracker API** — REST API para gerenciamento de tarefas, construída
com metodologia AI-First (spec → plano → execução → validação).

**Idioma do projeto:** documentação e comentários em **português**.
Identificadores de código (variáveis, funções, classes) em inglês.

## Stack

- **Linguagem:** Python 3.12
- **Framework:** FastAPI + Pydantic v2
- **Server:** Uvicorn
- **Banco de dados:** PostgreSQL + SQLAlchemy 2 + Alembic _(planejado, ainda não implementado)_
- **Qualidade:** ruff (lint + format), mypy --strict (types), pytest (testes)
- **Gerenciador de pacotes:** uv

## Estrutura de pastas

O projeto segue Clean Architecture com 4 camadas. Dependências apontam
sempre de fora para dentro.

```
app/
  ├── domain/          # Entidades, value objects, regras puras (sem libs externas)
  ├── application/     # Use cases — futuramente em commands/ e queries/ (CQRS)
  ├── infrastructure/  # Adapters: DB, persistência, integrações externas
  ├── presentation/    # Routers FastAPI, schemas Pydantic, dependências
  └── main.py          # Inicialização do FastAPI

tests/
  ├── unit/            # Sem I/O
  ├── integration/     # Toca DB ou serviços
  └── e2e/             # Fluxo HTTP completo

docs/
  ├── constitution.md  # Princípios não-negociáveis (a criar)
  ├── specs/           # Specs de feature antes do código
  └── plans/           # Planos técnicos antes da implementação
```

Essa separação permite: lógica de negócio independente de framework/banco;
testes em cada camada; fluxo de dependência claro; troca de infra sem
mexer no domínio.

## Convenções obrigatórias

- Type hints em **toda** função pública (`mypy --strict` deve passar).
- `snake_case` para funções/variáveis; `PascalCase` para classes.
- Imports absolutos a partir de `app.`.
- Toda função pública tem teste unitário ou de integração.
- Erros de negócio estendem `DomainError`; nunca `raise Exception` solto.
- Pydantic somente em `presentation/`; entidades de `domain/` são puro Python (dataclasses).
- `domain/` **NUNCA** importa de `infrastructure/` ou `presentation/`.
- `application/` **NUNCA** importa de `infrastructure/` ou `presentation/`.

## Anti-padrões

- ❌ Importar de `infrastructure` dentro de `domain` ou `application`.
- ❌ Usar `Any` em código de `domain` ou `application`.
- ❌ Logar segredos (DB_URL com senha, JWT_SECRET, API keys).
- ❌ Usar `print()` em código de aplicação (use logger estruturado).
- ❌ `raise Exception` sem subclasse específica.
- ❌ Commits diretos em `main` — sempre branch + PR.
- ❌ Modificar migrations Alembic já aplicadas em produção.

## Comandos comuns

**Instalar dependências:**
```bash
uv sync --all-groups
```

**Rodar servidor de desenvolvimento:**
```bash
uv run uvicorn app.main:app --reload
```

**Testes:**
```bash
uv run pytest                    # todos
uv run pytest tests/unit/        # só unit
uv run pytest -v                 # verbose
uv run pytest -k <test_name>     # teste específico
uv run pytest --cov              # com coverage
```

**Lint e formatação:**
```bash
uv run ruff check .              # lint
uv run ruff check --fix .        # auto-fix
uv run ruff format .             # formatar
```

**Type checking:**
```bash
uv run mypy app/
```

**Pipeline local (rodar antes de commitar):**
```bash
uv run ruff check . && uv run mypy app && uv run pytest
```

## Workflow de execução (plan-first)

Para qualquer feature nova:

1. **Plan first.** Antes de qualquer código, produzir
   `docs/plans/<numero>-<feature>.md` com: objetivo, contratos
   (request/response), modelo de domínio impactado, passos numerados
   de implementação, testes a criar. **NÃO IMPLEMENTAR AINDA.**
2. Esperar revisão humana do plano.
3. Após aprovado, executar passo a passo, criando testes **ANTES** da
   implementação (TDD).
4. Rodar `ruff + mypy + pytest` ao final de cada passo.
5. Propor commits atômicos (um por mudança lógica), nunca um commit gigante.

## Git

- **Conventional Commits:** `feat|fix|refactor|docs|test|chore|style(escopo): descrição`.
- Subject ≤ 72 chars, imperativo no presente ("add", não "added").
- Branches: `feat/<desc>`, `fix/<desc>`, `chore/<desc>`, `refactor/<desc>`.
- Antes de cada commit: rodar `ruff + mypy + pytest` (o pre-commit fará isso quando configurado).
- Sem `git push --force` em branches compartilhadas (`main`).
- PRs devem incluir: Summary, Changes, Testing notes.

## Diretrizes para o agente

- **Não narre passos intermediários.** Execute direto e reporte apenas o que mudou e bloqueios.
- **Verifique antes de afirmar.** Se você acha que um arquivo existe, leia. Se uma função existe, busque. Não invente.
- **Quando não souber, pergunte.** Melhor uma pergunta que uma alucinação.
- **Limite o escopo.** Modifique apenas os arquivos solicitados; se precisar tocar em outros, peça permissão.
- **Após edição, rode os testes** se o projeto tiver `pytest` configurado.
- **Cite código sempre com `arquivo:linha`** em relatórios e respostas.
- **Se um comando retornou output truncado ou não retornou**, diga isso explicitamente. Não invente o resultado.

## Estado atual do projeto

- ✅ Scaffold inicial criado (estrutura de pastas, FastAPI, health check)
- ✅ Ferramentas de qualidade configuradas (ruff, mypy, pytest)
- 🔄 CLAUDE.md sendo refinado (este arquivo)
- ⏳ `docs/constitution.md` — a criar
- ⏳ Primeira spec e implementação (Project entity) — Cap. 4 do handbook
- ⏳ PostgreSQL + SQLAlchemy + Alembic — implementação planejada
- ⏳ Pre-commit hooks e GitHub Actions CI — Cap. 8 do handbook

## Variáveis de ambiente

Documentadas em `.env.example` (a criar). Nunca commitar `.env`.

- `DATABASE_URL` — string de conexão Postgres _(quando o banco for adicionado)_
- `APP_ENV` — `development` | `staging` | `production`