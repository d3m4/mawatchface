# Tarefa 7 — BatteryProvider + anel de bateria

## Visao geral

Voce introduz duas coisas grandes nessa tarefa:

1. **Leitura de estado do sistema via BroadcastReceiver** — o jeito padrao Android de "escutar eventos" como mudanca de bateria, conexao de rede, alarme de bateria fraca, telefone tocando, etc.
2. **Geometria polar pra desenhar o anel** — converter "angulo + raio" em "x, y" pra desenhar tracos ao redor do centro.

O resultado: um anel de 60 tracos brancos na borda do canvas, refletindo a porcentagem real da bateria.

## Tecnologias e conceitos novos

**`BroadcastReceiver`.** Componente que recebe **intents broadcast** — mensagens enviadas pelo sistema ou outros apps. Voce registra o receiver com um `IntentFilter` que diz "me chama quando essa acao acontece". Pra bateria, a acao e `Intent.ACTION_BATTERY_CHANGED`. Toda vez que o nivel da bateria muda, o sistema dispara um broadcast e seu receiver recebe via `onReceive(context, intent)`.

**Sticky broadcast.** `ACTION_BATTERY_CHANGED` e um broadcast "sticky" (legado) — o sistema **lembra** o ultimo intent enviado. Quando voce chama `context.registerReceiver(receiver, filter)`, ele te devolve **imediatamente** o ultimo intent gravado (ou null se nunca houve um). Voce aproveita pra inicializar o estado sem esperar o proximo broadcast.

```kotlin
val intent = context.registerReceiver(receiver, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
intent?.let { receiver.onReceive(context, it) }
```

**`BatteryManager.EXTRA_LEVEL` e `EXTRA_SCALE`.** O intent traz dois extras: `level` (corrente, ex.: 87) e `scale` (maximo, geralmente 100). A porcentagem real e `level * 100 / scale`, pra robustez (em alguns dispositivos o `scale` nao e 100).

**`@Volatile`.** Anotacao Kotlin que vira `volatile` em Java/JVM. Garante que leituras da propriedade sempre veem o ultimo valor escrito por outras threads. Sem `@Volatile`, a JVM pode cachear o valor em registrador. Como o `BroadcastReceiver` recebe na main thread mas o `render()` (que le `percentage`) tambem roda na main thread, neste codigo `@Volatile` e mais "cinto e suspensorio". Mas e bom habito quando ha duvida sobre threading.

**`runCatching`.** Versao Kotlin de `try-catch` que retorna um `Result<T>`. Voce usa em `unregisterReceiver` porque o sistema pode jogar `IllegalArgumentException` se voce desregistrar duas vezes ou se o receiver nunca foi registrado. Voce nao quer crash no shutdown — `runCatching { context.unregisterReceiver(receiver) }` engole.

**Geometria polar.** Pra distribuir 60 tracos em volta de um centro, voce pensa em **angulos**. O traco N esta no angulo `N * 6°` (porque 360°/60 = 6°). Em coordenadas polares:

```
x = centroX + raio * cos(angulo)
y = centroY + raio * sin(angulo)
```

Mas no sistema de coordenadas do Android, **y cresce pra baixo** (oposto da matematica classica), e angulos comecam no leste (3 horas) e crescem no sentido **horario**. Pra comecar no topo (12 horas) e ir clockwise, voce subtrai 90° do angulo:

```
val angleDeg = -90.0 + i * 6.0
val angleRad = Math.toRadians(angleDeg)
val sinA = sin(angleRad)
val cosA = cos(angleRad)
```

Como o Android ja inverte y, o `sin` positivo aponta pra baixo. Combinado com `-90°`, voce comeca no topo e gira clockwise — exatamente o que a spec pede.

**Math.toRadians + sin/cos do `kotlin.math`.** Trigonometria pede radianos, nao graus. Voce converte com `Math.toRadians()`. As funcoes `sin/cos` voce importa de `kotlin.math` (em vez do `java.lang.Math.sin` que tambem existe — qualquer um serve, mas `kotlin.math` e mais idiomatico).

**Alpha em Paint.** `paint.alpha = (0.95 * 255).toInt()` define opacidade 0-255. Voce alterna entre `0.95 * 255 ≈ 242` (tracos ativos) e `0.15 * 255 ≈ 38` (tracos faded). Atribuir alpha **NAO** muda a `color` da Paint — substitui o canal alpha mas mantem RGB. Mais barato que recriar a Paint.

**`onDestroy()` do Renderer.** A Watch Face Library chama `onDestroy()` quando o renderer e descartado (usuario trocou de watchface, sistema reciclando). Voce sobrescreve pra parar o BroadcastReceiver — sem isso, vaza memoria.

## Passo a passo do que voce escreveu

**`BatteryProvider.kt`** encapsula a leitura: `start()` registra o receiver e captura o sticky broadcast inicial; `stop()` desregistra com `runCatching`; `percentage` e a propriedade publica que sempre tem o ultimo valor.

**No `WatchFaceRenderer.kt`:**

```kotlin
private val batteryProvider = BatteryProvider(context).also { it.start() }
```

`also { ... }` e similar a `apply`, mas dentro do bloco `it` aponta pro objeto (em vez de `this`). Aqui voce so chama `start()` no objeto recem-criado.

**`tickPaint`:**

```kotlin
private val tickPaint = Paint().apply {
    color = Color.WHITE
    strokeWidth = 3f
    strokeCap = Paint.Cap.ROUND
    isAntiAlias = true
}
```

Paint pra desenhar linhas. `strokeCap = ROUND` arredonda as pontas dos tracos.

**`drawBatteryRing`** itera de 0 a 59, calcula angulo, posiciona ponto inicial (raio interno) e final (raio externo), desenha. O alpha e ajustado por traco: ativos `0.95 * 255`, faded `0.15 * 255`.

**No `render`:** chamada antes de desenhar a hora, depois do gradiente.

## Pegadinhas

- **Esquecer `unregisterReceiver` vaza memoria.** O sistema mantem referencia ao receiver enquanto registrado, e o receiver tem referencia ao Renderer (via closure), que tem referencia ao Context. Resultado: o Renderer "morto" continua na memoria.
- **Calculo de tracos ativos com floor.** `(60 * batteryPct / 100)` em Int Kotlin ja faz divisao inteira (truncates). Garante que 99% nao mostra "60 tracos brilhantes" mas 59. Em outros idiomas voce usaria `Math.floor`.
- **`Paint.Cap.ROUND` muda visual significativamente.** `BUTT` (default) tem ponta quadrada, mas pode parecer cortado com strokeWidth fino. `ROUND` adiciona ~strokeWidth/2 em cada ponta — entao o traco visual e mais longo que `(outerR - innerR)`. Tens que ajustar geometria se quiser ponta exata.
- **Angulos em radianos.** `Math.cos(60.0)` te da o cosseno de **60 radianos**, nao 60 graus. Se vier de matematica de calculadora, atenta com isso.

## Conexao com a proxima tarefa

A tarefa 8 muda de assunto pra logica solar — voce comeca a calcular sunrise/sunset com a biblioteca commons-suncalc. Bateria fica intacta.

## Pra sua proxima watchface

**Padrao BroadcastReceiver:** se voce quer reagir a estado do sistema (bateria, rede, hora trocando), voce vai criar um receiver, registrar com filter, desregistrar no fim do lifecycle. Same template every time.

**Polar coordinates pra anel/arc:** sempre que voce desenhar elementos circulares ao redor de um centro (anel de complications, marcas de hora analogica, halos), `cos`/`sin` com angulo `i * 360/N` e o jeito. Decore o offset de `-90°` pra comecar no topo.

**Sticky broadcasts:** legado mas util. Pra novos apps, prefira `BatteryManager.getIntProperty(BATTERY_PROPERTY_CAPACITY)` em vez do receiver — mais moderno. Mas o receiver sticky funciona em todas as versoes Android desde sempre.
