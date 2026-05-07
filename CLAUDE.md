# CLAUDE.md — Watchface Galaxy Watch 8 Classic

Projeto pessoal de watchface custom pro Galaxy Watch 8 Classic.

## Diretrizes deste projeto

### Idioma
- Tudo em **portugues brasileiro**: commits, comentarios, documentacao, mensagens de erro voltadas ao usuario
- Acentuacao opcional em codigo/commits curtos, mas obrigatoria em texto longo (README, docs, comentarios explicativos)

### Commits
- Mensagens **curtas e diretas**, em pt-BR
- Sem `Co-Authored-By: Claude` ou similar — assinaturas de IA NAO devem aparecer nos commits
- Sem `🤖 Generated with Claude Code` ou linhas equivalentes
- Sem `--no-verify` em hooks de pre-commit
- Nunca commitar `.env`, credenciais, keystores ou `.claude/`

### Branch
- Trabalhar em `main` direto neste projeto pessoal (sem necessidade de feature branches)
- Para mudancas grandes/experimentais, branch dedicada antes de merge

## Stack

- Kotlin + AndroidX Watch Face Library
- Render via `CanvasRenderer.Shared`
- `commons-suncalc` pra calculo de nascer/por do sol
- `FusedLocationProviderClient` pra localizacao
- Distribuicao via ADB sideload (sem Play Store)

## Plataforma alvo

Galaxy Watch 8 Classic (46mm, 480×480, Wear OS 6, API 34).

## Spec

Design aprovado em [`docs/superpowers/specs/2026-05-08-galaxy-watchface-design.md`](docs/superpowers/specs/2026-05-08-galaxy-watchface-design.md).

## Workflow

1. Spec → plano de implementacao → implementacao incremental
2. Cada etapa testada via `./gradlew installDebug` no relogio real
3. Validacao final pelos criterios de aceite listados na spec
