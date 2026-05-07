# Tarefa 6 — Logica DIA/NOITE com horario fixo

## Visao geral

Voce introduz a **alternancia entre estados** com a logica mais simples possivel: hora 6-17 = DIA, resto = NOITE. Em DIA o gradiente vai de claro (topo) pra escuro (base); em NOITE inverte. Voce ainda nao tem GPS nem sunrise real; a tarefa serve pra estabelecer o **andaime** da feature antes de ligar a logica completa na tarefa 14.

Esta e uma tecnica importante de programacao incremental: **substituir a complexidade por um stub**, fazer todo o pipeline visual funcionar com o stub, e so depois trocar o stub pela logica real. Voce evita "implementar tudo de uma vez e descobrir que algo nao integra".

## Tecnologias e conceitos novos

**Refatoracao incremental.** Voce nao escreve codigo novo — voce **transforma** o que ja existe. A `setDayGradient(width, height)` da tarefa 5 vira `buildGradient(width, height, isDay)`, e voce decide `isDay` em runtime. Esse padrao de "extrair parametros" e o feijao com arroz da refatoracao.

**Range Kotlin com `in`.** A expressao `hour in 6..17` checa se `hour` (Int) esta no intervalo fechado `[6, 17]`. Idiomatico Kotlin. O equivalente Java seria `hour >= 6 && hour <= 17`. Cuidado: `6..17` e **inclusivo** dos dois lados; `6 until 17` exclui o 17.

**Inversao de cores via condicional.** O truque pra inverter o gradiente sem duplicar codigo:

```kotlin
val (topColor, bottomColor) = if (isDay) {
    Color.parseColor("#F0F0F0") to Color.parseColor("#000000")
} else {
    Color.parseColor("#000000") to Color.parseColor("#F0F0F0")
}
```

`if` em Kotlin **e expressao**, nao statement — retorna valor. `to` cria uma `Pair<A, B>`. A destructuring declaration `val (topColor, bottomColor) = ...` desempacota. Fica enxuto e single-source-of-truth.

**Recriar Shader a cada frame.** Diferente da tarefa 5 (onde voce cacheou o shader na primeira chamada), aqui voce constroi o gradiente **toda hora que `render` for chamado**, porque o estado DIA/NOITE pode mudar entre frames. Isso e perfeitamente OK — `LinearGradient` e barato pra construir; o que pesa e renderizar pixels.

**`zonedDateTime.hour`.** A `ZonedDateTime` que a Watch Face Library te entrega ja vem no fuso local do dispositivo. `.hour` retorna 0-23.

## Passo a passo do que voce escreveu

**Substituicao de `setDayGradient` por `buildGradient`:**

```kotlin
private fun buildGradient(width: Int, height: Int, isDay: Boolean): LinearGradient {
    val (topColor, bottomColor) = if (isDay) {
        Color.parseColor("#F0F0F0") to Color.parseColor("#000000")
    } else {
        Color.parseColor("#000000") to Color.parseColor("#F0F0F0")
    }
    return LinearGradient(
        0f, 0f, 0f, height.toFloat(),
        intArrayOf(topColor, Color.parseColor("#666666"), bottomColor),
        floatArrayOf(0f, 0.5f, 1f),
        Shader.TileMode.CLAMP
    )
}
```

A funcao agora retorna `LinearGradient` em vez de mutar uma propriedade. E uma **funcao pura** (mesma entrada → mesma saida, sem side effects), facil de raciocinar.

**Funcao auxiliar `isDayHour`:**

```kotlin
private fun isDayHour(hour: Int): Boolean = hour in 6..17
```

Sintaxe de funcao expression-bodied (`= expressao` em vez de `{ return expressao }`). Mais conciso pra one-liners.

**Atualizacao do `render`:**

```kotlin
val isDay = isDayHour(zonedDateTime.hour)
backgroundPaint.shader = buildGradient(bounds.width(), bounds.height(), isDay)
canvas.drawRect(bounds, backgroundPaint)
// ... resto: hora central
```

A cada frame, decide DIA/NOITE, monta o gradiente apropriado, pinta.

## Pegadinhas

- **A regra 6-17 e arbitraria.** No Brasil, `6..17` aproxima razoavelmente, mas nao reflete o ciclo real. Verao no sul, sol nasce ~5h45 e poe ~19h30. Esse stub vai parecer "errado" perto desses horarios; mas voce vai trocar pelo sunrise real na tarefa 14.
- **Stub = codigo descartavel?** Nao. O stub vira o **fallback** quando a logica real nao tiver dados (ex.: usuario negou permissao de localizacao). Mantenha-o testavel. A funcao `isDayHour` e exatamente o que voce vai chamar como fallback.
- **Comentarios explicando "por que stub" ajudam.** Mas a spec global dele e disciplinada com comentarios — o codigo deveria ser claro o suficiente. O nome `isDayHour` ja sugere "logica simples por hora", em contraste com algo que viria a ser `isDaytime(now, sunrise, sunset)` na versao final.

## Conexao com a proxima tarefa

A tarefa 7 muda de assunto temporariamente — vai construir o anel de bateria. Sem mexer no gradiente ainda. Mais tarde (tarefas 8-14) voce volta a esse codigo pra trocar `isDayHour` pela logica baseada em sunrise/sunset reais com transicao suave.

## Pra sua proxima watchface

**Padrao "stub-then-replace":** quando uma feature tem multiplos componentes (UI + logica + dados externos), escreva primeiro a UI com **dados fake hardcoded** (ou logica boba). Quando a UI estiver pronta, troque a logica boba pela real. Voce vai detectar bugs visuais antes de embaralhar com bugs de logica.

**Funcoes puras pra logica de transformacao.** `buildGradient(width, height, isDay)` nao acessa propriedades nem chama I/O — so pega entradas e devolve saida. Isso e a forma mais facil de testar, debugar, e raciocinar. Sempre que uma funcao puder ser pura, deixe-a pura. Side effects ficam pra `render()`, callbacks, etc.

**Evitar duplicacao via destructuring + if-expression.** Kotlin (e tambem Swift, Rust) te dao ferramentas pra escrever logica binaria com **uma** declaracao. Em vez de duplicar a construcao do `LinearGradient` em dois branches, voce constroi uma vez parametrizado. Quanto menos duplicacao, menos lugar pra divergir no futuro.
