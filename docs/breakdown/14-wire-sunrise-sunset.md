# Tarefa 14 — Wire sunrise/sunset → Renderer

## Visao geral

Voce **conecta** todas as pecas que construiu nas tarefas 8-13 ao Renderer. Substitui a logica boba `isDayHour(hour)` da tarefa 6 pelo pipeline real:

```
SunCache.read(zone) → SunWindow → GradientInterpolator.stops(now, sunrise, sunset) → IntArray cores
                                                                                        ↓
                                                                              LinearGradient → Paint.shader
                                                                                        ↓
                                                                              canvas.drawRect(bounds, paint)
```

Tudo o que voce ja escreveu vira um pipeline coeso. Esta e a tarefa de **integracao**.

## Tecnologias e conceitos novos

**Composicao de unidades.** Cada componente que voce escreveu tem responsabilidade unica:
- `SunCache` — persistencia
- `SunCalculator` — calculo
- `LocationProvider` — busca de coords
- `LocationWorker` — agendamento
- `GradientInterpolator` — interpolacao matematica
- `BatteryProvider` — leitura de bateria
- `Renderer` — desenho

Cada unidade testavel/raciociavel sozinha. **Wiring** e o ato de pluga-las umas nas outras com codigo glue. Boa arquitetura mantem o glue **fino** — se voce escrever 200 linhas de logica nova no Renderer durante o wiring, alguma coisa esta no lugar errado. No nosso caso, o wiring e literalmente uma dezena de linhas.

**Fallback gracioso (`?:`).** Se o `SunCache.read(zone)` retornar null (cache vazio na primeira execucao, antes do worker rodar), voce nao quer crashar. Voce cai num `SunWindow` fictico com horarios fixos:

```kotlin
val window = sunCache.read(zone) ?: SunWindow(
    sunrise = zonedDateTime.toLocalDate().atTime(6, 0).atZone(zone),
    sunset = zonedDateTime.toLocalDate().atTime(18, 0).atZone(zone)
)
```

Isso e o **fallback 06h/18h** que a spec previu. Implementado em uma linha.

**Nao otimizar prematuramente.** Voce constroi um novo `LinearGradient` a cada chamada de `render()` (1x/min). Cada construcao aloca um pequeno objeto Java. **Nao se preocupe.** A JVM lida com aloacoes pequenas com facilidade, e relogios modernos tem GC otimizado pra coisas assim. Se voce **medir** que o GC esta causando jitter, ai sim cacheia. Antes disso, simplicidade > performance especulativa.

**`zonedDateTime.zone`.** O `ZonedDateTime` que a Watch Face Library te passa ja vem com o zone do dispositivo. Voce extrai pra passar pro `SunCache.read(zone)` — que devolve um `SunWindow` com os instantes ajustados pro mesmo zone (importante pra que `now.isBefore(sunrise)` faca sentido).

**`zonedDateTime.toLocalDate()`.** Extrai so a data (sem hora, sem zone) do ZonedDateTime. Usado pra construir o fallback `LocalDate.atTime(6, 0).atZone(zone)`.

**`atTime(hour, min)` + `.atZone(zone)`.** Builder fluente da `java.time` pra construir um ZonedDateTime de um LocalDate. `LocalDate(2026,5,8).atTime(6,0)` = `LocalDateTime` (sem zone). `.atZone(saoPauloZone)` = `ZonedDateTime`.

## Passo a passo do que voce escreveu

**Adicao da propriedade no Renderer:**

```kotlin
private val sunCache = SunCache(context)
```

Cria o wrapper do SharedPreferences. Cheap — so abre o XML do prefs.

**Substituicao da logica antiga:**

Antes (tarefa 6):
```kotlin
val isDay = isDayHour(zonedDateTime.hour)
backgroundPaint.shader = buildGradient(width, height, isDay)
```

Depois:
```kotlin
val zone = zonedDateTime.zone
val window = sunCache.read(zone) ?: SunWindow(
    sunrise = zonedDateTime.toLocalDate().atTime(6, 0).atZone(zone),
    sunset = zonedDateTime.toLocalDate().atTime(18, 0).atZone(zone)
)
val stops = GradientInterpolator.stops(zonedDateTime, window.sunrise, window.sunset)

backgroundPaint.shader = LinearGradient(
    0f, 0f, 0f, bounds.height().toFloat(),
    stops,
    floatArrayOf(0f, 0.5f, 1f),
    Shader.TileMode.CLAMP
)
canvas.drawRect(bounds, backgroundPaint)
```

A funcao `buildGradient` e `isDayHour` da tarefa 6 sao **removidas**. Codigo morto.

## Pegadinhas

- **Cache vazio na primeira instalacao.** Antes do `LocationWorker` rodar a primeira vez, `sunCache.read` retorna null. Por isso o fallback. Sem ele, `GradientInterpolator.stops(now, null, null)` ia jogar NPE.
- **Zones precisam combinar.** Se voce armazena sunrise em zone "America/Sao_Paulo" e le esperando "UTC", o `now.isBefore(sunrise)` pode dar resposta errada. O `SunCache.read(zone)` ja faz `withZoneSameInstant(zone)` pra normalizar.
- **`sunCache.read(zone)` e I/O.** Cada chamada le SharedPreferences. SharedPreferences cacheia em memoria apos primeira leitura, entao chamadas seguintes sao baratas — mas a primeira pode ter ~1-5ms de latencia. Pra `render` rodando 1x/min, isso e fine. Se voce um dia precisar de leitura mais frequente, cacheia o resultado por algumas chamadas.
- **`zonedDateTime.toLocalDate()` e date no zone do `zonedDateTime`.** Se o ZonedDateTime vem em UTC mas o local e em SP, a `LocalDate` extraida e a data UTC, nao a local. No nosso caso a Watch Face Library ja entrega no zone do dispositivo, entao tudo certo — mas saiba dessa armadilha geral.
- **Logica antiga deletada.** Voce tem que **remover** as funcoes `buildGradient` e `isDayHour`. Deixa-las como "morta" suja o codigo. Se um dia precisar de logica fixa de novo, a logica esta no fallback do `?:` — nao precisa de funcao auxiliar.

## Conexao com a proxima tarefa

A tarefa 15 mexe num aspecto totalmente separado: AOD off. Voce desabilita renderizacao em ambient mode. Independente da logica de gradiente.

## Pra sua proxima watchface

**Wiring fica numa funcao so.** Quando voce escreve componentes pequenos e bem definidos, integra-los e simples. Se o wiring estiver crescendo demais, e sinal de que a abstracao esta errada — algum componente esta vazando responsabilidade.

**Fallback de uma linha com Elvis.** O padrao `result ?: defaultBuilder` e onipresente em Kotlin. Use sem medo. Substitui blocos `if-else if` longos.

**Apague codigo morto imediatamente.** Quando uma logica deixa de ser usada, remova **na mesma tarefa**. Codigo morto cresce, atrai bugs, atrapalha leitura. Voce vai sentir um "arrependimento bom" toda vez que apagar 30 linhas que nao precisava.

**Fallbacks no caminho feliz.** Quando seu app pode operar **sem** alguma feature (no caso, sem GPS), os fallbacks devem ser **transparentes**. Usuario nao deve ver mensagem de erro nem comportamento estranho — so um modo "menos preciso" que ainda funciona. Esse e o tipo de polish que diferencia app pessoal de app profissional.
