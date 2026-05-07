# Tarefa 11 — LocationProvider

## Visao geral

Voce abstrai a leitura de GPS num componente unico que oferece **uma operacao**: `fetchOnce()` retorna `Pair<Double, Double>?` (lat, lon ou null se falhar). Internamente usa o `FusedLocationProviderClient` do Google Play Services e converte a API baseada em **callbacks** numa **suspend function** Kotlin — isso encaixa direto no `CoroutineWorker` da tarefa 12.

## Tecnologias e conceitos novos

**`FusedLocationProviderClient`.** API moderna do Google Play Services pra localizacao. "Fused" significa que ele combina varias fontes (GPS, Wi-Fi, celula, sensores) automaticamente, escolhendo a melhor pra cada momento. Mais inteligente e eficiente que o velho `LocationManager` do Android.

Voce obtem com:
```kotlin
val client = LocationServices.getFusedLocationProviderClient(context)
```

E chama `getCurrentLocation` ou `lastLocation`.

**`getCurrentLocation` vs `lastLocation`.**
- `lastLocation` — devolve a ultima localizacao **conhecida** (cacheada). Pode ser muito velha ou null se o dispositivo nao tem nenhuma. Rapido.
- `getCurrentLocation(priority, cancellationToken)` — pede uma **nova** medicao. Pode demorar segundos pra resolver. Mais preciso, mais caro de bateria.

Pra um worker que roda 4x/dia, vale `getCurrentLocation` — voce quer dado fresco, nao cache.

**`Priority`.** Enum com 4 niveis:
- `PRIORITY_HIGH_ACCURACY` — GPS ativo, mais preciso, mais bateria
- `PRIORITY_BALANCED_POWER_ACCURACY` — Wi-Fi/celula prioritarios, GPS se necessario
- `PRIORITY_LOW_POWER` — so Wi-Fi/celula, ~10km de precisao
- `PRIORITY_PASSIVE` — so reaproveita o que outros apps ja pediram

Pra calculo solar, `LOW_POWER` e mais que suficiente — voce so precisa saber em que cidade voce esta.

**`CancellationTokenSource`.** Um objeto que voce pode "cancelar" pra abortar uma operacao em andamento. Voce passa o token associado ao `getCurrentLocation`; se o coroutine for cancelado (ex.: WorkManager mata o worker), voce cancela o token e a chamada de localizacao para de gastar bateria.

**Callback API → suspend function.** O `getCurrentLocation` usa callbacks (`addOnSuccessListener`, `addOnFailureListener`). Suspend functions usam `await/resume`. Pra fazer ponte:

```kotlin
return suspendCancellableCoroutine { cont ->
    client.getCurrentLocation(priority, token)
        .addOnSuccessListener { loc -> cont.resume(loc?.let { it.latitude to it.longitude }) }
        .addOnFailureListener { cont.resume(null) }
    cont.invokeOnCancellation { token.cancel() }
}
```

`suspendCancellableCoroutine` da o "controle do coroutine" pra voce. `cont.resume(value)` e o jeito de dizer "ok, prossegue com esse valor". `cont.invokeOnCancellation { ... }` registra um cleanup pra quando o coroutine for cancelado.

Esse padrao `suspendCancellableCoroutine` e o canivete suico pra adaptar **qualquer** API baseada em callbacks pra coroutines.

**`@SuppressLint("MissingPermission")`.** O Lint do Android avisa "voce esta chamando uma API que precisa de permissao mas nao tem checagem". Voce **tem** a checagem (no inicio do `fetchOnce` voce faz `if (!hasPermission()) return null`), mas o Lint nao consegue analisar. `@SuppressLint` silencia o warning. Use com sobriedade — so onde voce realmente verificou.

**`ContextCompat.checkSelfPermission`.** Pergunta se o app tem uma permissao concedida. Retorna `PackageManager.PERMISSION_GRANTED` ou `PERMISSION_DENIED`. Voce sempre verifica antes de cada uso de API que precisa de permissao — porque o usuario pode revogar nas Settings a qualquer hora.

## Passo a passo do que voce escreveu

**`hasPermission()`** — utility que checa permissao COARSE.

**`fetchOnce()` (suspend)**:
1. Se nao tem permissao, retorna null instantaneamente.
2. Cria o cliente fused.
3. Cria token de cancelamento.
4. Em `suspendCancellableCoroutine`:
   - Chama `getCurrentLocation` com `LOW_POWER` e o token.
   - No success: pega `latitude/longitude` (se loc nao for null) e resume com `Pair`.
   - No failure: resume com null.
   - Em caso de cancelamento do coroutine: cancela o token (para de gastar bateria).

A funcao retorna `Pair<Double, Double>?`. Quem chama destrutura facil: `val (lat, lon) = provider.fetchOnce() ?: return`.

## Pegadinhas

- **`getCurrentLocation` pode retornar null.** Mesmo com permissao, GPS pode estar desligado, sem sinal, ou simplesmente falhar. Sempre trate como `Location?`. Voce ja faz com `loc?.let { ... }`.
- **Nao chame em main thread** sem coroutine. `getCurrentLocation` retorna `Task<Location>` (assincrono), entao tecnicamente nao bloqueia. Mas se voce for ler em `LocationManager.requestLocationUpdates` (API antiga), atencao com threading.
- **`PRIORITY_LOW_POWER` em ambientes fechados pode falhar.** Se o relogio estiver dentro de casa sem Wi-Fi conhecido, a estimativa pode vir nula. Acceptable pro nosso fallback.
- **Play Services tem que estar disponivel.** Em alguns dispositivos Wear OS antigos ou modificados, Play Services pode estar ausente. O `LocationServices.getFusedLocationProviderClient` jogaria excecao. Watch 8 Classic tem Play Services oficial — sem problema.

## Conexao com a proxima tarefa

A tarefa 12 envelopa essa funcao num `CoroutineWorker` que roda 4x/dia. O worker chama `provider.fetchOnce()`, alimenta o `SunCalculator`, persiste no `SunCache`. Pipeline completo.

## Pra sua proxima watchface

**Adapter callback → suspend.** O padrao `suspendCancellableCoroutine` aparece em todo lugar quando voce moderniza codebases Android. Sempre que ver uma API com `addOnSuccessListener/addOnFailureListener`, voce sabe como envelopar.

**`Priority.LOW_POWER` quase sempre basta.** Localizacao precisa de GPS so pra navegacao em tempo real ou tracking de atividade. Pra "saber em que cidade estou", `LOW_POWER` e ideal.

**Nullable returns sao um sinal honesto.** Voce poderia ter feito `fetchOnce()` jogar excecao em caso de falha. Mas falha aqui e **esperada** (sem permissao, sem sinal, etc.) — nao e excepcional. Use `T?` em vez de exceptions pra erros esperados; reserve exceptions pra erros de programacao ou bugs.

**`@SuppressLint` com cautela.** Cada vez que voce desliga um warning do Lint, voce esta dizendo "confio". Se a confianca for justificada (voce checou condicao manualmente), ok. Se for so pra silenciar barulho, voce esta criando bug de produto.
