# Tarefa 4 — Renderizar hora central em Caesar Dressing

## Visao geral

Esta e a primeira tarefa onde voce realmente desenha algo no Canvas. Voce:
1. Carrega a fonte Caesar Dressing como `Typeface`.
2. Cria um `Paint` configurado com essa fonte, cor branca, tamanho 280px.
3. Calcula a hora atual e centraliza ela visualmente.
4. Desenha com `canvas.drawText(...)`.

E a primeira interacao real com o **Canvas API do Android**, que e o sistema de desenho 2D imediato (parecido com o Canvas HTML5 ou com o `<canvas>` do Skia). Toda a parte visual do watchface daqui pra frente vai sair de combinacoes de chamadas Canvas: `drawRect`, `drawText`, `drawLine`, `drawCircle`, `drawPath`.

## Tecnologias e conceitos novos

**`android.graphics.Paint`.** Configuracao de "como desenhar". Cor, tamanho de texto, estilo (FILL/STROKE), antialiasing, typeface, alinhamento, alpha, shader (gradiente), tudo mora num `Paint`. O Canvas nao tem "estado de cor" — voce sempre passa um `Paint` em cada chamada de desenho.

**`isAntiAlias = true`.** Liga suavizacao das bordas. Sem isso, texto e linhas inclinadas ficam serrilhadas (aliased). Voce **sempre** quer ligar pra texto e formas curvas.

**`Typeface`.** Representa uma familia/peso de fonte carregada. Imutavel. Atribuir num `Paint.typeface` faz a Paint usar essa fonte pra qualquer `drawText`.

**`textAlign = Paint.Align.CENTER`.** Define o ponto de ancoragem horizontal do texto: o `x` que voce passa pra `drawText` vira o centro do texto, em vez do canto esquerdo (default).

**`Canvas.drawText(text, x, y, paint)`.** O `y` aqui nao e o topo nem o centro — e a **baseline** do texto, ou seja, a linha imaginaria onde a base de letras como "T", "M", "x" se apoiam. Letras com descender (como "g", "y", "p") descem abaixo da baseline. Isso e fundamental pra centralizar verticalmente.

**`FontMetrics`.** Estrutura com 5 valores que descrevem proporcoes da fonte no tamanho atual:
- `top` (topo absoluto da metrica, mais alto que a maior letra)
- `ascent` (negativo — distancia da baseline ate o topo das letras maiusculas, tipo o cap height)
- `descent` (positivo — distancia da baseline ate a base das letras com descender)
- `bottom` (positivo — base absoluta)
- `leading` (espaco extra entre linhas)

Pra centralizar visualmente uma linha unica de texto no eixo Y, a formula classica e:

```
baseline = centroY - (ascent + descent) / 2
```

Como `ascent` e negativo e geralmente maior em magnitude que `descent`, o resultado tipico e mover a baseline um pouco PRA BAIXO do centro geometrico — exatamente o que a percepcao visual espera.

**`Renderer` recebendo `Context`.** Voce mudou o construtor pra aceitar `context: android.content.Context`. Por que? Porque `Typeface.createFromAsset()` precisa de um `AssetManager`, que voce so consegue via `Context`. O `WatchFaceService` te da `applicationContext` (escopo do app inteiro, vivo enquanto o processo vive).

**`ZonedDateTime`.** Da `java.time` (Java 8+, hoje padrao em Android via desugaring). Representa um instante com timezone. A Watch Face Library passa um `ZonedDateTime` ja no fuso correto do dispositivo a cada `render()`. Voce le `.hour` (0-23), `.minute`, `.second`, etc.

## Passo a passo do que voce escreveu

**No construtor do Renderer:**

```kotlin
private val typeface: Typeface = Typeface.createFromAsset(
    context.assets,
    "CaesarDressing-Regular.ttf"
)
```

Carrega a fonte uma vez. Vira propriedade `private val` — imutavel, vivida pelo tempo de vida do Renderer.

**O Paint da hora:**

```kotlin
private val hourPaint = Paint().apply {
    color = Color.WHITE
    typeface = this@WatchFaceRenderer.typeface
    textSize = HOUR_FONT_SIZE_PX  // 280f
    textAlign = Paint.Align.CENTER
    isAntiAlias = true
}
```

`apply { ... }` e idiomatica Kotlin: chama a funcao no objeto recem-criado e devolve esse objeto. Te da uma "configuracao em bloco" sem variavel temporaria. O `this@WatchFaceRenderer.typeface` precisa de qualifier porque dentro de `apply` o `this` aponta pro `Paint`, nao pra classe externa.

**Dentro de `render()`:**

```kotlin
val cx = bounds.exactCenterX()
val cy = bounds.exactCenterY()
val hour = zonedDateTime.hour.toString()
val fm = hourPaint.fontMetrics
val baseline = cy - (fm.ascent + fm.descent) / 2f
canvas.drawText(hour, cx, baseline, hourPaint)
```

`bounds` e o `Rect` da area de desenho (aqui, 480x480 inteiro). `exactCenterX()` e `exactCenterY()` retornam `Float` precisos. `zonedDateTime.hour` e Int 0-23. `.toString()` vira "0", "1", ..., "23".

## Pegadinhas

- **Hora sem zero a esquerda.** `zonedDateTime.hour.toString()` da "9" pras nove da manha, nao "09". Se voce quiser zero-padded, use `String.format("%02d", zonedDateTime.hour)`. A spec diz **sem** zero — entao tudo certo.
- **`Paint` declarado dentro do `render()` = desastre de performance.** Cada chamada a `Paint()` aloca objeto. Multiplica por frames por minuto e voce gera GC pressure no relogio (que ja tem pouca RAM). Sempre declare `Paint`, `Typeface`, `Shader` como propriedades fora do `render`.
- **Fonte custom + textSize muito grande.** Algumas fontes (especialmente display fonts como Caesar Dressing) tem proporcoes diferentes de fontes "neutras". 280px na Caesar Dressing pode parecer muito maior que 280px em Inter. A spec previu isso — calibracao final empirica de 240-300px na tarefa 16.

## Conexao com a proxima tarefa

Voce ve a hora em branco sobre fundo preto. A tarefa 5 troca o fundo preto por um gradiente vertical (claro topo → escuro base). A hora vai sobreviver intacta.

## Pra sua proxima watchface

**Padrao de Paint cacheado:** declare TODO `Paint` como propriedade `private val`, configurado com `apply { ... }`. Nunca crie no `render()`. Mesma logica vale pra `Typeface`, `Shader`, `Bitmap`, `Path` — qualquer objeto reutilizavel.

**Centralizacao vertical de texto:** a formula `baseline = cy - (ascent + descent) / 2` e o canivete suico. Decore. Funciona pra qualquer fonte, qualquer tamanho.

**Construtor com Context:** sempre que precisar carregar resources (assets, strings, drawables) num componente Android, voce vai querer um `Context`. O `applicationContext` (escopo do app) e o seguro pra cachear; o `Activity` context vaza memoria se cacheado por muito tempo.
