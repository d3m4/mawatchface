# Tarefa 5 — Gradiente DIA fixo

## Visao geral

Voce troca o fundo preto solido por um **gradiente vertical** com 3 stops: cinza-claro no topo (`#F0F0F0`), cinza medio no meio (`#666666`), preto na base. E so o estado DIA por enquanto — a inversao pra NOITE chega na tarefa 6, e a interpolacao suave por sunrise/sunset chega na tarefa 13/14. Mas voce ja aprende o sistema de Shaders do Android Canvas, que e o mecanismo que vai permitir tudo isso depois.

## Tecnologias e conceitos novos

**`Shader`.** No Canvas API, um Shader e um "gerador de cor por pixel" anexado a um `Paint`. Em vez de usar uma cor solida, o Paint pergunta ao Shader: "qual e a cor no pixel (x, y)?". Existem varios tipos: `LinearGradient`, `RadialGradient`, `SweepGradient`, `BitmapShader`, `ComposeShader`. Quando voce setta `paint.shader = ...`, qualquer chamada de desenho que use esse Paint (drawRect, drawCircle, drawText) vai colorir os pixels via shader em vez de cor solida.

**`LinearGradient`.** Shader que interpola cores ao longo de uma linha. O construtor pega:
- `(x0, y0)` — ponto de inicio
- `(x1, y1)` — ponto de fim
- `colors: IntArray` — array de cores ARGB, no minimo 2
- `positions: FloatArray?` — posicoes 0..1 ao longo da linha onde cada cor "ancora" (null = espalhamento uniforme)
- `tile: TileMode` — o que fazer fora da linha (CLAMP estende a primeira/ultima cor; REPEAT repete; MIRROR espelha)

Pra um gradiente vertical no canvas inteiro:
```
LinearGradient(0f, 0f, 0f, height, [topColor, midColor, bottomColor], [0f, 0.5f, 1f], CLAMP)
```

X fixo em 0 = a interpolacao acontece so no eixo Y.

**Color stops com 3+ pontos.** Com `[0f, 0.5f, 1f]`, voce cria 2 segmentos: do topo ate o meio, do meio ate a base. Com 3 cores diferentes em cada stop, voce ve uma transicao "S" — cores intermediarias passam pelo cinza puro no meio.

**`Color.parseColor("#F0F0F0")`.** Converte hex string pra Int ARGB. Aceita `#RGB`, `#RRGGBB`, `#AARRGGBB`. Use isso pra cores definidas em hex; pra cores hardcoded, voce pode usar `Color.WHITE`, `Color.BLACK`, `Color.argb(a, r, g, b)` ou `Color.rgb(r, g, b)`.

**`Shader.TileMode.CLAMP`.** Quando o pixel esta fora do range do shader (ex.: y < 0 ou y > height), o pixel pega a cor do extremo mais proximo (a primeira ou ultima cor do array). Pra um gradiente cobrindo o canvas inteiro, isso evita artefatos.

**Performance: criar Shader uma vez ou a cada frame?** No caso ideal, voce cria o shader uma vez e reutiliza. Mas como o shader depende do `bounds` (que so vem em `render()`), uma estrategia comum e criar lazily na primeira chamada e cachear. Em renderers que mudam o gradiente dinamicamente (como o nosso na tarefa 14), voce vai recriar o shader em cada frame — e tudo bem; criar um LinearGradient e barato. O caro e renderizar pixels.

**`canvas.drawRect(bounds, paint)`.** Pinta um retangulo com o paint atual. Como o paint tem shader, o retangulo vira o gradiente. `bounds` aqui e um `Rect` (Int) — o Canvas aceita.

## Passo a passo do que voce escreveu

**Adicao da propriedade `backgroundPaint`:**

```kotlin
private val backgroundPaint = Paint().apply { isAntiAlias = true }
```

Paint compartilhado pro fundo. Sem cor explicita — sera o shader que define cor.

**Funcao `setDayGradient` que injeta o shader:**

```kotlin
private fun setDayGradient(width: Int, height: Int) {
    backgroundPaint.shader = LinearGradient(
        0f, 0f, 0f, height.toFloat(),
        intArrayOf(
            Color.parseColor("#F0F0F0"),
            Color.parseColor("#666666"),
            Color.parseColor("#000000")
        ),
        floatArrayOf(0f, 0.5f, 1f),
        Shader.TileMode.CLAMP
    )
}
```

`intArrayOf(...)` cria um `IntArray` (necessario pelo construtor — nao aceita `List<Int>`).

**No `render()`:**

```kotlin
if (backgroundPaint.shader == null) {
    setDayGradient(bounds.width(), bounds.height())
}
canvas.drawRect(bounds, backgroundPaint)
```

A primeira chamada cria o shader; chamadas seguintes reusam. O `canvas.drawRect(bounds, paint)` pinta o canvas inteiro com gradiente. Logo em seguida vem o `drawText` da hora, que pinta sobre o gradiente.

## Pegadinhas

- **Ordem de desenho importa.** Voce desenha **primeiro** o fundo, **depois** os elementos por cima. Inverter quebra o visual — o fundo cobriria a hora.
- **Shader = nao desenhe `drawColor()` antes.** Se voce chamar `canvas.drawColor(Color.BLACK)` antes do `drawRect(bounds, backgroundPaint)`, fica funcional (o gradiente sobrescreve), mas e desperdicio. Pinta de uma vez so com o shader.
- **`bounds` ≠ tamanho fixo de 480.** Mesmo que o Watch 8 Classic seja 480x480, **nao hardcode** 480 no codigo. Use `bounds.width()`/`height()`. Outros relogios tem outras resolucoes; emuladores, idem.
- **`floatArrayOf(0f, 0.5f, 1f)` precisa estar ordenado.** Stops fora de ordem geram comportamento indefinido visualmente.

## Conexao com a proxima tarefa

A tarefa 6 vai pegar essa funcao `setDayGradient` e generaliza-la pra `buildGradient(isDay)`, alternando entre estado DIA e estado NOITE com base na hora atual.

## Pra sua proxima watchface

**Mental model do Canvas:** voce esta sempre **pintando camadas em ordem**: fundo → backgrounds intermediarios → elementos principais → highlights → overlays. Cada `draw*()` deposita pixels que cobrem o que veio antes (a menos que voce use blending modes — assunto avancado).

**Gradientes sao proximo passo natural depois de cores solidas.** Quase todo watchface "polido" tem algum gradiente de fundo, anel ou destaque. Decore o construtor de `LinearGradient` — voce vai usar muito.

**Cache de shader vs recriar:** se o shader nao depende do tempo, **cache**. Se depende (gradiente animado, gradient que segue um valor dinamico), **recrie a cada frame**. O custo de construcao e baixo; o custo de pixel shading e o que importa.
