# Tarefa 12 — LocationWorker (WorkManager 4x/dia)

## Visao geral

Voce **agenda** uma rotina pra rodar 4x ao dia (a cada 6h) que: busca a localizacao via `LocationProvider`, calcula sunrise/sunset com `SunCalculator`, e persiste no `SunCache`. Quando o Renderer for desenhar o gradiente (tarefa 14), ele le do cache que esse worker mantem fresco.

Aqui voce conhece o **WorkManager**, a forma moderna do Android pra rodar tasks em background com garantias (sobrevive reboot, respeita Doze mode, evita batalhar com bateria).

## Tecnologias e conceitos novos

**`WorkManager`.** Componente Android Jetpack pra executar trabalho **deferrable** e **garantido** em background. "Deferrable" = pode atrasar uns minutos se o sistema quiser economizar bateria. "Garantido" = vai rodar eventualmente, mesmo apos reboot do dispositivo.

WorkManager substitui o caos historico de APIs Android pra background:
- `JobScheduler` (API 21+) — quase certo, mas requer manifest setup
- `AlarmManager` — mais antigo, abusado, pode ser ignorado em Doze
- `Service` proprio (com foreground notification) — pra coisas em tempo real
- `BroadcastReceiver` com BOOT_COMPLETED — pra disparar coisas no boot

WorkManager unifica tudo isso. Voce so escreve um `Worker`, agenda, e o sistema cuida.

**`CoroutineWorker`.** Subclasse de `Worker` que suporta `suspend fun doWork()`. Voce pode chamar suspend functions diretamente (como `provider.fetchOnce()` que e suspend). Sem o `CoroutineWorker`, voce teria que usar `runBlocking` ou async manual — feio.

**`Result.success / retry / failure`.**
- `Result.success()` — task completou. Nao reagenda automaticamente.
- `Result.retry()` — falhou de forma transitoria (sem rede, sem GPS). WorkManager reagenda com backoff exponencial.
- `Result.failure()` — falhou permanentemente. WorkManager nao tenta de novo.

Voce retorna `retry` quando `provider.fetchOnce()` da null — se falhou agora, talvez funcione daqui a pouco.

**`PeriodicWorkRequestBuilder<T>(interval, unit)`.** Cria um request periodico. Minimum interval e **15 minutos** — voce nao pode pedir intervalo menor (limite imposto pelo sistema pra economizar bateria). Voce pediu 6 horas → totalmente ok.

```kotlin
val request = PeriodicWorkRequestBuilder<LocationWorker>(6, TimeUnit.HOURS).build()
```

**`enqueueUniquePeriodicWork(name, policy, request)`.** Agenda com nome unico (ex.: `"location-sync"`). Se ja houver um request com esse nome, a `policy` decide o que fazer:
- `KEEP` — mantem o antigo, ignora o novo. Bom pra agendamento idempotente.
- `REPLACE` — descarta o antigo, agenda o novo.

Voce escolheu `KEEP` porque o `WatchFaceService.onCreate` pode ser chamado multiplas vezes (sistema reciclando o servico) e voce nao quer reiniciar o agendamento toda vez.

**OneTimeWorkRequest pra primeira execucao.** PeriodicWorkRequest **nao** roda imediatamente — espera o primeiro intervalo. Pra popular o cache na primeira instalacao, voce dispara um one-shot adicional:

```kotlin
val oneShot = OneTimeWorkRequestBuilder<LocationWorker>().build()
WorkManager.getInstance(applicationContext).enqueueUniqueWork(
    "location-sync-now",
    ExistingWorkPolicy.KEEP,
    oneShot
)
```

Mesmo `Worker`, mesma logica, mas roda logo. Tambem `KEEP` pra nao duplicar.

**`Context` no Worker.** O construtor do Worker recebe `Context` e `WorkerParameters` automaticamente. Use `applicationContext` (acessivel via `applicationContext` ou `context`) pra evitar segurar Activity context.

**`adb logcat | grep WM-*`.** WorkManager loga todas as transicoes em tags com prefixo `WM-`. Util pra debugar agendamentos. Voce ve coisas como `WM-WorkSpec` (estado salvo no DB), `WM-WorkerWrapper` (execucao), etc.

## Passo a passo do que voce escreveu

**`LocationWorker.kt`:**

```kotlin
class LocationWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        val provider = LocationProvider(applicationContext)
        val cache = SunCache(applicationContext)

        val coords = provider.fetchOnce()
        if (coords != null) {
            val (lat, lon) = coords
            val zone = ZoneId.systemDefault()
            val today = LocalDate.now(zone)
            val window = SunCalculator.compute(lat, lon, today, zone)
            cache.store(window, lat, lon, today)
            return Result.success()
        }
        return Result.retry()
    }
}
```

Pipeline simples: pega coords, calcula sun, persiste, retorna success. Se nao conseguir coords, retry.

**No `WatchFaceService.onCreate`:**

```kotlin
override fun onCreate() {
    super.onCreate()
    val request = PeriodicWorkRequestBuilder<LocationWorker>(6, TimeUnit.HOURS).build()
    WorkManager.getInstance(applicationContext).enqueueUniquePeriodicWork(
        "location-sync",
        ExistingPeriodicWorkPolicy.KEEP,
        request
    )

    val oneShot = OneTimeWorkRequestBuilder<LocationWorker>().build()
    WorkManager.getInstance(applicationContext).enqueueUniqueWork(
        "location-sync-now",
        ExistingWorkPolicy.KEEP,
        oneShot
    )
}
```

`super.onCreate()` primeiro (importante!). Depois agenda o periodic e dispara um one-shot pra populacao inicial.

## Pegadinhas

- **PeriodicWork minimo 15min.** Se voce tentar `PeriodicWorkRequestBuilder<T>(5, MINUTES)`, o sistema arredonda pra cima ou ignora.
- **Atraso no agendamento.** O sistema pode atrasar o worker pra economizar bateria, especialmente se o relogio estiver em Doze mode (tela apagada por horas). 4x/dia pode virar 3-4x/dia na pratica. Pra essa watchface, irrelevante.
- **`Result.failure()` vs `Result.retry()`.** Failure significa "nunca mais tente". Use so pra erros de programacao (ex.: input invalido). Retry e o seu amigo pra falhas temporarias.
- **Worker rodando enquanto user usa app.** WorkManager pode rodar o worker concorrente com a UI. No nosso caso isso e ok porque o worker so escreve em SharedPreferences (que e thread-safe internamente).
- **`enqueueUniquePeriodicWork` em vez de `enqueue`.** Sem o "unique", cada vez que `onCreate` dispara um novo agendamento, voce empilha. Apos varios dias o WorkManager teria varias copias rodando. **Sempre** use unique.

## Conexao com a proxima tarefa

A tarefa 13 cria o `GradientInterpolator` que faz a matematica de transicao suave. A tarefa 14 finalmente liga tudo: o renderer le do `SunCache` (alimentado por esse worker) e passa pro interpolator.

## Pra sua proxima watchface

**WorkManager pra qualquer task periodica de background.** Sincronizar dados, fazer download, atualizar widget, processar fila — tudo via WorkManager. Esqueca AlarmManager pra essas coisas.

**`KEEP` policy = idempotente.** Sempre que voce agendar com `enqueueUniquePeriodicWork`, voce pode re-agendar varias vezes sem problema porque `KEEP` mantem o existente. Isso significa que voce pode coloca-lo em `onCreate` sem se preocupar com duplicacao.

**Combo periodic + one-shot pra primeira execucao.** Como periodic nao roda imediatamente, o truque do one-shot adicional e padrao em apps de producao. Lembre-se.

**Result.retry e backoff.** Voce nao precisa fazer logica de retry voce mesmo — o WorkManager ja faz com exponential backoff. Mais simples e mais correto que fazer `delay(60000); retry()` na mao.
