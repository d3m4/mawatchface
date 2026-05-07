# Watchface Galaxy Watch 8 Classic — Design

**Data:** 2026-05-08
**Autor:** Felipe
**Status:** Spec aprovada, aguardando plano de implementacao

## Objetivo

Construir um watchface minimalista pro Galaxy Watch 8 Classic (Wear OS 6) que mostra **apenas a hora atual** em fonte estilizada, com **anel de bateria** ao redor da tela e **fundo em gradiente preto-e-branco** que inverte conforme o ciclo dia/noite (sunrise/sunset reais via GPS).

Distribuicao: sideload pessoal via ADB. Sem publicacao na Play Store.

## Plataforma alvo

| Item | Valor |
|------|-------|
| Modelo | Galaxy Watch 8 Classic (46mm) |
| OS | Wear OS 6 (One UI Watch 8) |
| API alvo | Android 34 |
| API minima | Android 30 (Wear OS 3+, mantem compatibilidade) |
| Resolucao | 480 × 480 px (tela redonda) |
| Densidade | ~321 dpi |

## Visual

### Canvas

- Forma: circulo, 480 × 480
- Fundo: preto puro fora do circulo (irrelevante, OLED apaga)
- Paleta: somente preto puro (`#000000`), branco puro (`#FFFFFF`) e cinzas intermediarios

### Gradiente de fundo (dia/noite)

Gradiente vertical de 3 stops sobre o circulo:

```
Estado DIA   (sunrise + 30min  →  sunset - 30min):
  topo (0%)    → #F0F0F0 (cinza-claro)
  meio (50%)   → #666666
  base (100%)  → #000000

Estado NOITE (sunset + 30min  →  sunrise - 30min do dia seguinte):
  topo (0%)    → #000000
  meio (50%)   → #666666
  base (100%)  → #F0F0F0
```

**Transicao "amanhecer" e "anoitecer":** janela de 60 minutos centrada no evento (30min antes + 30min depois). Durante essa janela os tres stops sao interpolados linearmente entre os valores DIA e NOITE.

Exemplo: se sunrise = 06:00, das 05:30 ate 06:30 o gradiente morpha gradualmente de NOITE pra DIA. Idem para sunset.

### Anel de bateria

- Posicao: anel na borda externa do circulo, centrado em raio ~225px (entre 210 e 225)
- 60 tracos uniformes ao redor (1 traco a cada 6°)
- Comprimento de cada traco: 15px (de raio 210 a raio 225)
- Espessura: 3px
- Cor: branco (`#FFFFFF`)
- Cap: round
- **Inicio do preenchimento:** topo (12h, posicao 0°)
- **Sentido:** horario
- **Quantidade ativa:** `floor(60 × bateria_pct / 100)` tracos com opacidade 0.95
- **Restante:** tracos com opacidade 0.15 (visiveis bem fracos pra dar referencia visual de "espaco que falta")

Exemplos:
- 100% bateria → 60 tracos brilhantes
- 87% bateria → 52 tracos brilhantes + 8 faded
- 0% bateria → 60 tracos faded

### Hora central

- Fonte: **Caesar Dressing** (Google Fonts, OFL license)
- Asset: `app/src/main/assets/CaesarDressing-Regular.ttf`
- Tamanho: 280px (valor de partida do mockup; calibracao final feita via inspecao visual no Watch 8 real apos primeiro deploy — pode variar entre 240–300px)
- Cor: branco puro (`#FFFFFF`)
- Posicao: centro geometrico do canvas (240, 240)
- Alinhamento: horizontal centralizado, baseline no centro vertical
- Formato: 24h, **somente hora** (sem minutos, sem segundos, sem AM/PM)
- Valores possiveis: `0` a `23` (1 ou 2 digitos, sem zero a esquerda)

## Comportamento dinamico

### Tick rate

- Update interval: **1 minuto**
- Justificativa: como so exibimos hora (sem minutos/segundos), nao ha necessidade de tick por segundo. Reducao de tick rate economiza bateria.
- Excecao: ao redor das viradas de hora, o tick de 1 minuto ja garante atualizacao no momento certo (margem de erro ate 60s, aceitavel pro caso de uso)

### Bateria

- Provider: `BatteryManager` via `BroadcastReceiver` em `Intent.ACTION_BATTERY_CHANGED`
- Cache local do ultimo valor pra evitar query a cada redraw
- Trigger de redraw: ao receber broadcast com mudanca de >= 1%

### Sunrise/sunset

- Biblioteca: `org.shredzone.commons:commons-suncalc:3.x`
- Algoritmo: posicao solar real (NOAA-equivalent)
- Input: latitude/longitude obtidos via `FusedLocationProviderClient` (Google Play Services Location)
- **Leitura de localizacao: 4x por dia** (a cada 6 horas, em horarios fixos: 00:00, 06:00, 12:00, 18:00 local). Justificativa: leitura de GPS e o componente mais caro em bateria — limitar a 4x/dia mantem a localizacao razoavelmente fresca pra um usuario que se desloca durante o dia, sem o custo de leituras frequentes.
- Implementacao: agendamento via `WorkManager` com 4 `PeriodicWorkRequest` (ou 1 unitario que se reagenda).
- Cache: ultima `(lat, lon)` valida + par `(sunrise, sunset)` do dia atual, persistido em `SharedPreferences`.
- Recalculo de sunrise/sunset (operacao barata, so CPU): a cada leitura de localizacao bem-sucedida E na virada do dia (00:00 local).
- Se uma leitura agendada falhar (sem GPS, sem permissao), mantem o ultimo `(lat, lon)` valido conhecido. Sunrise/sunset continuam validos com a localizacao antiga.

### Permissao de localizacao

- Permissao requerida: `ACCESS_COARSE_LOCATION` (precisao ~1 km, suficiente pra calculo solar)
- Solicitacao: na primeira instalacao, atraves de Activity launcher dedicada
- **Fallback:** se permissao foi negada *e* nunca houve uma leitura bem-sucedida, usar horarios fixos:
  - Sunrise = 06:00 local
  - Sunset = 18:00 local
- Reconciliacao: as proprias 4 tentativas/dia agendadas servem de retry — se a permissao for concedida depois ou o GPS voltar, na proxima janela ja sai do fallback.

### Always-On Display (AOD)

- AOD **desativado**
- Implementacao: o `Renderer` retorna sem desenhar nada quando `renderParameters.drawMode == DrawMode.AMBIENT`
- Tela apaga totalmente em ambient. Acende somente em `DrawMode.INTERACTIVE` (raise-to-wake ou interacao)

## Arquitetura do codigo

### Estrutura de pastas

```
watchface/
├── app/
│   ├── src/main/
│   │   ├── kotlin/dev/felipe/watchface/
│   │   │   ├── WatchFaceService.kt        # entry point (extends WatchFaceService)
│   │   │   ├── WatchFaceRenderer.kt       # CanvasRenderer.Shared, faz o draw
│   │   │   ├── BatteryProvider.kt         # broadcast receiver + state
│   │   │   ├── SunCalculator.kt           # wraps commons-suncalc
│   │   │   ├── LocationProvider.kt        # fused location + permissao
│   │   │   ├── GradientInterpolator.kt    # logica de morphing dia/noite
│   │   │   └── PermissionActivity.kt      # solicita ACCESS_COARSE_LOCATION
│   │   ├── res/
│   │   │   └── xml/watch_face_info.xml    # metadata do watchface
│   │   ├── assets/
│   │   │   └── CaesarDressing-Regular.ttf
│   │   └── AndroidManifest.xml
│   └── build.gradle.kts
├── settings.gradle.kts
├── build.gradle.kts
├── gradle.properties
├── .gitignore
├── CLAUDE.md
└── README.md
```

### Componentes e responsabilidades

| Componente | Responsabilidade |
|------------|------------------|
| `WatchFaceService` | Bootstrap. Instancia `Renderer` e devolve `WatchFace` configurado. |
| `WatchFaceRenderer` | Recebe `RenderParameters`, ZonedDateTime, e desenha no Canvas. Composes: gradiente + anel + numero. |
| `BatteryProvider` | Registra/desregistra BroadcastReceiver. Expoe `StateFlow<Int>` com porcentagem. |
| `SunCalculator` | Recebe `lat, lon, date` → retorna `Pair<sunrise: ZonedDateTime, sunset: ZonedDateTime>`. |
| `LocationProvider` | Pede permissao, busca localizacao, persiste lat/lon em SharedPreferences. |
| `GradientInterpolator` | Recebe hora atual + sunrise + sunset → retorna 3 stops de cor RGB pra renderer. |
| `PermissionActivity` | UI minima na primeira run pra pedir ACCESS_COARSE_LOCATION. |

### Fluxo de dados (resumido)

1. **Init:** `WatchFaceService.onCreate` instancia o `Renderer`. `Renderer` instancia `BatteryProvider`, `SunCalculator`, `LocationProvider`, `GradientInterpolator`.
2. **Cada frame (1 vez/min ou em mudanca de bateria):**
   - `Renderer.render(canvas, zonedDateTime, ...)`:
     - Pega `batteryPct` do `BatteryProvider`
     - Pega `(sunrise, sunset)` do `SunCalculator` (cache)
     - Pede 3 stops do `GradientInterpolator(zonedDateTime, sunrise, sunset)`
     - Desenha: circulo com gradiente → anel de bateria → numero da hora
3. **Mudanca de localizacao ou data:** invalidacao do cache do `SunCalculator`.
4. **Ambient mode:** `Renderer.render` retorna cedo, canvas fica preto.

## Build e distribuicao

### Stack de build

- Gradle 8.x com Kotlin DSL
- Android Gradle Plugin 8.x
- Kotlin 1.9.x ou superior
- Java 17 (toolchain)

### Dependencias principais

```kotlin
// app/build.gradle.kts (resumo)
dependencies {
    implementation("androidx.wear.watchface:watchface:1.2.x")
    implementation("androidx.wear.watchface:watchface-data:1.2.x")
    implementation("androidx.wear.watchface:watchface-style:1.2.x")
    implementation("com.google.android.gms:play-services-location:21.x")
    implementation("org.shredzone.commons:commons-suncalc:3.x")
    implementation("androidx.core:core-ktx:1.13.x")
}
```

(Versoes exatas a fixar no plano de implementacao apos consultar Context7.)

### Sideload via ADB

```
1. Habilitar "Modo desenvolvedor" no Watch 8 Classic
2. Habilitar "Depuracao via ADB" e "Depuracao via Wi-Fi"
3. Parear ADB com o relogio: adb pair <ip>:<port>
4. Conectar: adb connect <ip>:<port>
5. Instalar: ./gradlew installDebug
6. Ativar o watchface: Settings > Watch faces > selecionar
```

## Nao-objetivos (escopo fora desta entrega)

- Publicacao na Play Store ou Galaxy Store
- AOD (Always-On Display) — explicitamente desligado
- Complications (data, passos, clima, etc) — fora do escopo, requisito e minimalismo
- Configuracoes de usuario (cor, peso de fonte, paleta) — primeira versao e fixa
- Localizacao em tempo real / acompanhar usuario em movimento — 4 leituras/dia bastam
- Suporte a outros modelos de relogio Wear OS — focado no Watch 8 Classic 480×480
- Testes automatizados — projeto pessoal, validacao via uso real

## Notas de implementacao

- **Canvas vs Compose:** usar `CanvasRenderer.Shared` (mais leve e direto pra desenho 2D). Compose for Wear seria overkill pro escopo.
- **Anti-aliasing:** ativar em todos os Paints pra evitar serrilhado nos numeros e nos tracos.
- **Tipografia:** carregar a TTF uma vez no `init` do Renderer e reusar o `Typeface`. Nao recarregar a cada frame.
- **Performance:** o tick a cada minuto + redraws pontuais de bateria mantem CPU baixa. Sem AOD, ainda mais.
- **Privacidade:** localizacao COARSE so pra calculo solar. Nao mandar pra lugar nenhum, nao logar.

## Validacao manual (criterios de aceite)

A implementacao esta completa quando, no Watch 8 Classic real:

1. Instalado via ADB e selecionavel nas watch faces do sistema
2. Mostra a hora atual em Caesar Dressing, formato 24h sem minutos
3. Anel de bateria reflete corretamente a porcentagem (validar com 100%, ~50%, baixa)
4. Gradiente em DIA mostra claro topo → escuro embaixo
5. Gradiente em NOITE mostra escuro topo → claro embaixo
6. Transicao no horario do sunrise/sunset e visivelmente suave (nao um corte seco)
7. Sem GPS / permissao negada, fallback de 06:00/18:00 funciona
8. Em ambient mode, tela apaga (zero pixels acesos)
9. Apos 8 horas de uso continuo com a watchface custom, drenagem de bateria fica dentro de ±10% da watchface padrao do sistema (medicao manual via Settings > Battery)
