# Tarefa 13 — GradientInterpolator (transicao suave dia/noite)

## Visao geral

Voce escreve a **matematica** que decide, pra cada momento do dia, qual gradiente renderizar. Em vez do switch binario "DIA ou NOITE" da tarefa 6, agora ha uma **janela de transicao** de 60 minutos centrada em sunrise e sunset, onde as cores morpham gradualmente entre os dois estados.

Esta tarefa e logica pura: nao toca em Canvas, Context, ou disco. Recebe `(now, sunrise, sunset)`, retorna 3 cores. Sem side effects. Voce vai adorar isso depois de 12 tarefas mexendo com APIs Android.

## Tecnologias e conceitos novos

**Lerp (Linear Interpolation).** Tecnica matematica fundamental pra animar entre dois estados. Dado:
- Estado A (cor inicial)
- Estado B (cor final)
- t = fracao do progresso, 0..1

A interpolacao e:

```
resultado = A + (B - A) * t
```

Quando `t=0`, resultado = A. Quando `t=1`, resultado = B. Em valores intermedios, resultado e a mistura linear. Funciona pra qualquer espaco vetorial — voce aplica componente a componente em RGB:

```
r = aR + (bR - aR) * t
g = aG + (bG - aG) * t
b = aB + (bB - aB) * t
```

Isso e o que `lerpColor(a, b, t)` faz no codigo.

**`Color.red(int)`, `.green`, `.blue`.** Funcoes utility pra extrair componentes 0-255 de uma cor packed Int. `Color.red(0xFFFF0000.toInt())` retorna 255. `Color.rgb(r, g, b)` faz o caminho inverso — empacota em Int.

**`coerceIn(0, 255)`.** Metodo de Int em Kotlin que clampa o valor entre min e max. `(-3).coerceIn(0, 255) == 0`. `300.coerceIn(0, 255) == 255`. Util quando a matematica de lerp pode estourar por arredondamento.

**`java.time.Duration`.** Representa um intervalo de tempo (positivo ou negativo). `Duration.ofMinutes(30)` = 30 min. Voce subtrai/soma de `ZonedDateTime`:

```kotlin
val sunriseStart = sunrise.minus(WINDOW)  // 30 min antes
val sunriseEnd = sunrise.plus(WINDOW)     // 30 min depois
```

Tambem da pra calcular diferencas: `Duration.between(start, end).toMillis()`.

**Comparacoes de `ZonedDateTime`.** Suporta `.isBefore(other)`, `.isAfter(other)`, `.isEqual(other)`. Voce constroi a ordem temporal das janelas com sequencia de `if/else` checando `isBefore`.

**Logica em sequence checks.** A funcao `stops` segue uma sequencia ordenada:

```
if now < sunriseStart       → noite plena
else if now < sunriseEnd    → janela amanhecer (lerp noite→dia)
else if now < sunsetStart   → dia pleno
else if now < sunsetEnd     → janela anoitecer (lerp dia→noite)
else                        → noite plena (resto da noite ate o proximo sunrise)
```

Cada caso e mutuamente exclusivo. A ordem importa — voce nao quer cair no segundo caso quando o primeiro ja deveria ter pegado.

**Funcao `fraction(start, end, now)`.** Calcula `t = (now - start) / (end - start)`. Resultado em 0..1 quando `now` esta entre `start` e `end`. Voce usa `Duration.between(...).toMillis().toFloat()` pra ter milisegundos como Float (resolucao mais que suficiente). Clampa com `coerceIn(0f, 1f)` por seguranca.

**Por que linear (lerp) e nao easing curve?** Pra um gradiente de 60 minutos, lerp e suficientemente suave a olho nu. Se voce quisesse algo mais "natural" (acelerar no meio, desacelerar nas pontas), poderia usar `t * t * (3 - 2*t)` (smoothstep), mas e detalhe que ninguem nota numa transicao tao lenta.

## Passo a passo do que voce escreveu

**Constantes de cor:**

```kotlin
private val DAY_TOP = Color.parseColor("#F0F0F0")
private val DAY_MID = Color.parseColor("#666666")
// ...
```

Estados fixos. Hardcoded — em uma versao com `UserStyleSchema` (configuravel pelo usuario), voce poderia parametrizar. Pra essa V1, fixos.

**Funcao `stops(now, sunrise, sunset): IntArray`:**

```kotlin
return when {
    now.isBefore(sunriseStart) -> intArrayOf(NIGHT_TOP, NIGHT_MID, NIGHT_BOTTOM)
    now.isBefore(sunriseEnd) -> {
        val t = fraction(sunriseStart, sunriseEnd, now)
        lerpStops(NIGHT_..., DAY_..., t)
    }
    // ...
}
```

`when {}` em Kotlin sem argumento e equivalente ao if/else if encadeado, mas mais legivel.

**`lerpStops(aTop, aMid, aBot, bTop, bMid, bBot, t)`** — interpola componente a componente. Retorna `IntArray(3)`.

**`lerpColor(a, b, t)`** — extrai r/g/b de cada cor, lerps cada componente, empacota de volta com `Color.rgb`.

**`fraction(start, end, now)`** — calcula posicao normalizada de `now` no intervalo `[start, end]`.

## Pegadinhas

- **Edge case sunrise apos sunset.** Em circulos polares, em alguns dias `sunrise` pode ser apos `sunset` (sol da meia-noite invertido). Sua logica assume ordenacao normal — em locais nao-polares isso nunca acontece, mas o codigo silenciosamente falha em condicoes extremas. Pra essa watchface focada no Brasil/Watch 8 Classic do user, irrelevante.
- **`now` exatamente em `sunrise`.** `t` vira ~0.5 dentro da janela do amanhecer (porque a janela e `[sunrise-30, sunrise+30]`, e now esta no meio). Esperado e correto.
- **`fraction` antes do clamp.** Se `now < start`, a divisao da negativo. Se `now > end`, da > 1. O `coerceIn(0f, 1f)` salva — mas voce nunca deve chamar `lerpStops` fora da janela. A logica de `when` garante isso, mas e bom ter o clamp como cinto e suspensorio.
- **Performance.** A funcao e chamada uma vez por frame (1x/min). Cada chamada faz ~10 operacoes aritmeticas + 4 comparacoes de ZonedDateTime. Custo desprezivel.

## Conexao com a proxima tarefa

A tarefa 14 conecta esse interpolator no Renderer. Substitui a logica binaria `isDayHour` da tarefa 6 pelo dinamico do interpolator + dados do `SunCache`.

## Pra sua proxima watchface

**Logica pura sem side effects e ouro.** Voce pode testar no REPL, no playground Kotlin, sem nada de Android, e validar cada caso. Sempre que possivel, isole logica deste tipo numa `object` ou `class` autocontida que recebe inputs simples e devolve outputs simples.

**Lerp e a base de toda animacao.** Decore. `result = a + (b-a) * t` aparece em **tudo**: animacao de UI, mistura de cores, interpolacao de coordenadas, blending de sons, smoothing de sensores. E a operacao mais reutilizavel da computacao grafica/animacao.

**`Duration` e `ZonedDateTime` da `java.time` sao o jeito certo.** Esqueca `java.util.Date` (legado, mutavel, confuso). Use a API moderna em todo lugar.

**`when {}` sobre `if/else if`.** Mais escalavel, mais idiomatico Kotlin. Quando voce tiver 5+ branches, fica visualmente mais limpo. Bonus: o Kotlin compiler pode te avisar se voce esquecer um caso quando `when` se baseia em sealed class.
