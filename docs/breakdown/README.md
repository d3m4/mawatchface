# Breakdown — Aprenda Kotlin + Android + Wear OS construindo este watchface

Esta pasta contem **16 documentos didaticos**, um por tarefa do plano de implementacao. O alvo: voce ja sabe programar, mas nao conhece Kotlin, Java, Android, Wear OS, nem Gradle. Cada arquivo te leva por uma fatia pequena do projeto explicando os conceitos novos antes de mostrar o codigo.

## Como usar

1. Le os arquivos **em ordem** — eles foram desenhados pra construir conhecimento incrementalmente. A tarefa 5 assume que voce entendeu o que aconteceu na 4.
2. Pra cada tarefa, abra **junto** o arquivo correspondente do plano (`docs/superpowers/plans/2026-05-08-galaxy-watchface-implementation.md`). O plano tem o codigo cru e os passos exatos; o breakdown explica **por que** o codigo e assim.
3. Voce nao precisa decorar nada. Concentra em entender **o padrao**, nao a sintaxe.

## Indice

| # | Arquivo | O que aprende |
|---|---------|---------------|
| 1 | [01-bootstrap-gradle.md](01-bootstrap-gradle.md) | Gradle, Kotlin DSL, estrutura de projeto Android, AndroidX |
| 2 | [02-watchface-service-renderer-minimo.md](02-watchface-service-renderer-minimo.md) | AndroidManifest, Service, intent-filters, CanvasRenderer da Watch Face Library |
| 3 | [03-fonte-caesar-dressing.md](03-fonte-caesar-dressing.md) | assets vs res, embedar fontes TTF, Typeface |
| 4 | [04-renderizar-hora.md](04-renderizar-hora.md) | Paint, Canvas.drawText, FontMetrics, ZonedDateTime, centralizacao de texto |
| 5 | [05-gradiente-dia-fixo.md](05-gradiente-dia-fixo.md) | LinearGradient, Shader, color stops, performance de redraws |
| 6 | [06-logica-dia-noite-horario-fixo.md](06-logica-dia-noite-horario-fixo.md) | Refatoracao incremental, "stub que sera substituido" como tecnica |
| 7 | [07-battery-provider-anel.md](07-battery-provider-anel.md) | BroadcastReceiver, sticky broadcasts, geometria polar (sin/cos) |
| 8 | [08-sun-calculator.md](08-sun-calculator.md) | java.time, object singleton, builder pattern, integrar libs externas |
| 9 | [09-sun-cache.md](09-sun-cache.md) | SharedPreferences, persistencia leve, properties customizadas em Kotlin |
| 10 | [10-permission-activity.md](10-permission-activity.md) | Activity, ActivityResultContracts, runtime permissions modernas |
| 11 | [11-location-provider.md](11-location-provider.md) | FusedLocationProviderClient, suspendCancellableCoroutine, callback → suspend |
| 12 | [12-location-worker.md](12-location-worker.md) | WorkManager, CoroutineWorker, agendamento periodico no Android moderno |
| 13 | [13-gradient-interpolator.md](13-gradient-interpolator.md) | Lerp (linear interpolation) entre cores, Duration, design de logica pura |
| 14 | [14-wire-sunrise-sunset.md](14-wire-sunrise-sunset.md) | Composicao de unidades, fallback gracioso, padrao "wiring" |
| 15 | [15-aod-desligado.md](15-aod-desligado.md) | DrawMode.AMBIENT, lifecycle de watchface, economia de bateria |
| 16 | [16-calibracao-final.md](16-calibracao-final.md) | Validacao manual, criterios de aceite, evitar polish creep |

## Bibliografia rapida

Quando precisar aprofundar em algum conceito, esses sao os recursos canonicos:

- **Kotlin:** [kotlinlang.org/docs/home.html](https://kotlinlang.org/docs/home.html) — secoes "Basic syntax", "Coroutines"
- **Android Jetpack:** [developer.android.com/jetpack](https://developer.android.com/jetpack)
- **Wear OS Watch Face Library:** [developer.android.com/training/wearables/watch-faces](https://developer.android.com/training/wearables/watch-faces)
- **Canvas drawing:** [developer.android.com/reference/android/graphics/Canvas](https://developer.android.com/reference/android/graphics/Canvas)
- **WorkManager:** [developer.android.com/topic/libraries/architecture/workmanager](https://developer.android.com/topic/libraries/architecture/workmanager)

## Apos terminar

Voce vai ter visto, em ordem natural:

1. Como **configurar** um projeto Android do zero
2. Como criar um **servico** que o sistema chama (e o que e Service em Android)
3. Como **desenhar** com Canvas + Paint + Shader
4. Como ler estado do dispositivo (**bateria, localizacao**)
5. Como agendar **trabalho em background** sem destruir bateria
6. Como **persistir** dados leves
7. Como pedir e tratar **permissoes** runtime
8. Como integrar uma **biblioteca externa** Java/Kotlin
9. Como pensar **lifecycle** (ambient mode, onDestroy, etc)

Esse subconjunto cobre ~80% do que voce precisa pra construir qualquer watchface custom Wear OS daqui pra frente. Os outros 20% sao APIs especificas (complications, user style, hardware sensors) — voce ja vai ter o vocabulario pra ler a doc oficial sozinho.
