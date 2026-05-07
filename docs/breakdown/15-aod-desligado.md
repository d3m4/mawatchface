# Tarefa 15 — AOD desligado

## Visao geral

Voce desabilita o **Always-On Display** (AOD) — a tela "fantasma" simplificada que permanece ativa em baixo brilho enquanto o relogio nao esta sendo olhado ativamente. No nosso watchface, AOD = tela apagada totalmente. Quando o usuario levantar o pulso, a tela volta ao watchface completo.

A spec define isso como decisao de design (economia de bateria + esteticamente queremos a tela "limpa" so quando ativa). Implementacao e literalmente 4 linhas de codigo, mas voce conhece o conceito de **DrawMode** que e fundamental pra qualquer watchface sria.

## Tecnologias e conceitos novos

**Always-On Display (AOD).** Funcionalidade de telas OLED onde uma versao reduzida da watchface continua visivel mesmo quando o relogio nao esta em uso ativo. Vantagens: usuario ve a hora a qualquer momento sem virar o pulso. Desvantagens: gasta bateria (2-5x mais que tela apagada), precisa ser bem desenhada (limite de pixels acesos pra nao queimar OLED).

Em Wear OS, AOD pode ser ligado ou desligado por watchface. Quando ligado, a Watch Face Library chama `render()` com `DrawMode.AMBIENT`. Em DOZE/AMBIENT mode, voce deve desenhar uma versao **simplificada** (preto-fundo, monochrome, ≤15% pixels acesos).

**`RenderParameters` e `DrawMode`.** A `Renderer` (classe base que voce extende) expoe `renderParameters: RenderParameters` que tem `drawMode: DrawMode`. Os modos:
- `INTERACTIVE` — relogio olhado pelo user, render normal
- `AMBIENT` — AOD ligado, render simplificado
- `LOW_BATTERY_INTERACTIVE` — interactive em baixa bateria
- `MUTE` — modo nao-perturbe

Voce verifica `renderParameters.drawMode` no inicio do `render()` e decide o que fazer.

**Early return no `render`.** O jeito mais simples de "desligar AOD" e detectar `AMBIENT` e desenhar so preto:

```kotlin
if (renderParameters.drawMode == androidx.wear.watchface.DrawMode.AMBIENT) {
    canvas.drawColor(Color.BLACK)
    return
}
// resto do render normal...
```

A Watch Face Library nao "sabe" que voce desligou AOD — ela continua chamando render. Voce so devolve preto e segue.

**Por que nao "configurar AOD off no manifest"?** A Wear OS espera que watchfaces tenham AOD habilitada por default. Existem capability flags que voce poderia setar (`HasAlwaysOnDisplayProperty`), mas o caminho mais simples e mais defensivo e renderizar preto em ambient. Pixels apagados em OLED = literalmente sem consumo de energia. Resultado equivalente a "desligado".

**OLED e pixel preto.** Em telas OLED (Watch 8 Classic), pixels pretos sao **nao acesos** — energia zero. Pixels brancos consomem o maximo. Por isso "preto puro" em ambient mode e equivalente a tela desligada. Em telas LCD (relogios baratos), tela preta ainda gasta energia da backlight — mas a Wear OS nao recomenda LCD pra Wear.

## Passo a passo do que voce escreveu

No comeco do `render`:

```kotlin
override fun render(
    canvas: Canvas, bounds: Rect, zonedDateTime: ZonedDateTime, sharedAssets: NoOpSharedAssets
) {
    if (renderParameters.drawMode == androidx.wear.watchface.DrawMode.AMBIENT) {
        canvas.drawColor(Color.BLACK)
        return
    }

    // resto do render INTERACTIVE: gradiente, anel, hora
}
```

E so. O `renderParameters` e propriedade herdada de `Renderer` — sempre disponivel sem voce passar.

## Pegadinhas

- **Em alguns dispositivos, ambient ainda chama render multiplas vezes.** Wear OS pode chamar `render` no minuto inicial em ambient pra "preview". Voce so devolve preto, ai e logo desligado. Ok.
- **Diferenca entre `AMBIENT` e `INTERACTIVE_ONLY`.** Existe tambem o flag `WatchState.isAmbient` (StateFlow que voce poderia observar). Pro caso simples, checar `renderParameters.drawMode` no `render` e suficiente.
- **Nao tente desativar a chamada de render.** Voce nao tem como dizer "Sistema, nao me chame em ambient". Voce so devolve preto cedo. E ok — a chamada e barata.
- **Animacoes paradas em ambient.** Em interactive, voce pode ter animacoes (segundeiros, transicoes). Em ambient, voce **deve** parar tudo: render simplificado, frame rate baixissimo, nada de movimento. Como nosso watchface nao anima ja em interactive (so tick por minuto), nao temos nada a desligar — a regra aplica trivialmente.
- **OLED burn-in.** Mesmo voce pintando preto, se voce um dia mudar a logica de ambient pra mostrar a hora estatica, atencao com burn-in: deslocar a hora alguns pixels a cada minuto evita queimar uma marca permanente na tela. Watch 8 Classic ja tem proteccoes mas e bom saber.

## Conexao com a proxima tarefa

A tarefa 16 e a final: validacao de aceite + calibracao fina + atualizacao do README. Sem novas features.

## Pra sua proxima watchface

**Sempre considere ambient mode no design.** Watchfaces ficam ~90% do tempo em ambient. O que voce mostra la **e** o produto, na pratica. Se voce desliga AOD (como fizemos), assume essa tradeoff conscientemente.

**Padrao "early return em ambient":**

```kotlin
override fun render(...) {
    if (renderParameters.drawMode == DrawMode.AMBIENT) {
        renderAmbient(canvas, bounds, zonedDateTime)  // ou drawColor(BLACK) e return
        return
    }
    renderInteractive(canvas, bounds, zonedDateTime)
}
```

Separa codigo de render em duas funcoes — fica claro o que e cada modo. Pra watchfaces que sustentam AOD, a `renderAmbient` e onde voce desenha a versao "linha so".

**Pixels pretos = bateria zero (em OLED).** Lembrete eterno pra design de qualquer UI Wear OS / OLED. Se voce esta com pixel branco no fundo, voce esta queimando bateria toda hora. Use preto como cor base sempre.

**RenderParameters tem mais que drawMode.** Tem tambem `watchFaceLayers` (camadas a desenhar — base, complications, overlay), `highlightLayer`, etc. Pra customizacoes avancadas (animar so durante setting picker, por exemplo), explore.
