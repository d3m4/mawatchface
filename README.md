# Watchface — Galaxy Watch 8 Classic

Watchface minimalista pro Galaxy Watch 8 Classic (Wear OS 6).

## Visual

- Hora central em fonte **Caesar Dressing** (24h, somente hora — sem minutos)
- Anel de **60 tracos** ao redor da tela representando a bateria (cheia = 360°)
- Fundo em **gradiente preto-e-branco** que inverte conforme o ciclo dia/noite (sunrise/sunset reais via GPS)

## Stack

- Kotlin
- AndroidX Watch Face Library (`androidx.wear.watchface`)
- Render via `CanvasRenderer.Shared`
- `commons-suncalc` para calculo de nascer/por do sol
- `FusedLocationProviderClient` para localizacao (4 leituras/dia)

## Plataforma

- Galaxy Watch 8 Classic (46mm, tela redonda 480 × 480)
- Wear OS 6 / API 34
- Min SDK 30 (Wear OS 3+)

## Distribuicao

Sideload pessoal via ADB. Sem plano de publicacao na Play Store.

## Estrutura

```
docs/superpowers/specs/   # design aprovado e planos
app/                      # codigo Kotlin do watchface (a ser criado)
```

Veja [`docs/superpowers/specs/2026-05-08-galaxy-watchface-design.md`](docs/superpowers/specs/2026-05-08-galaxy-watchface-design.md) pro design completo.

## Status

Spec aprovada. Plano de implementacao em desenvolvimento.
