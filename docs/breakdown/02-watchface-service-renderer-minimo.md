# Tarefa 2 — WatchFaceService + Renderer minimo (canvas preto)

## Visao geral

Aqui voce coloca o primeiro pixel do projeto na tela do relogio. A tarefa cria o **manifest** (que ensina ao sistema Android o que esse app sabe fazer), o **servico de watchface** (a "porta de entrada" que o sistema invoca quando o usuario seleciona seu watchface), e um **renderer** que so preenche a tela de preto. E pequeno, mas a partir desse ponto voce ja tem um app instalavel via ADB que aparece na lista de watchfaces.

## Tecnologias e conceitos novos

**`AndroidManifest.xml`.** Documento XML que descreve o app pro sistema operacional: quais "components" (Activities, Services, Receivers, Providers) o app oferece, quais permissoes pede, com quais `<intent-filter>` ele responde a chamadas do sistema. Sem manifest, o sistema nao sabe que seu app existe.

**Service.** No Android, um `Service` e um componente sem UI direta que executa em segundo plano. Watchfaces sao implementadas via servicos especiais que o sistema "chama" quando o usuario quer renderizar. Especificamente, a Wear OS espera um servico que extende `androidx.wear.watchface.WatchFaceService`.

**`<intent-filter>` + categoria.** Um intent-filter declara: "responda a intents desse tipo". Pra watchfaces, voce filtra:
- Acao: `android.service.wallpaper.WallpaperService` (porque tecnicamente um watchface e um WallpaperService especializado)
- Categoria: `com.google.android.wearable.watchface.category.WATCH_FACE` (marca isso como Wear-specific)

**Meta-data.** Pares chave-valor que o sistema le pra detalhes adicionais. Voce usa duas:
- `android:name="com.google.android.wearable.watchface.preview"` aponta pra um drawable (a thumbnail mostrada no seletor)
- `android:name="android.service.wallpaper"` aponta pro `xml/watch_face.xml` (declara o "tipo" de wallpaper)

**`android:exported="true"`.** Diz ao sistema que outros apps (no caso, o sistema seletor de watchface) podem invocar esse servico. Sem isso, o servico fica invisivel pra Wear OS e seu watchface nunca aparece na lista.

**`uses-library`.** Empresta classes do Wear OS pro classpath. Marcado `required="false"` pra nao quebrar em dispositivos sem Wear.

**Resources e drawables.** Tudo que vai dentro de `res/` e gerenciado pelo Android: vai pro APK comprimido, fica acessivel via ID gerado (a classe `R`). `res/drawable/watch_face_preview.png` e o thumbnail. `res/values/strings.xml` centraliza textos pra suportar idiomas.

**`Renderer.CanvasRenderer2`.** Da Watch Face Library AndroidX. Voce extende dela, sobrescreve `render()`, e o sistema chama esse metodo a cada frame. Voce desenha o que quiser num `android.graphics.Canvas`. O `2` no nome indica a v2 da API (a v1 esta deprecada).

**`SharedAssets`.** Conceito da Watch Face Library: assets caros (Bitmaps grandes, fontes) que voce quer carregar uma vez e reusar por toda vida do renderer. Como nao temos nada caro ainda, declaramos `NoOpSharedAssets`.

**`WatchFace`.** Wrapper que combina o renderer com o tipo de watchface (`DIGITAL` ou `ANALOG`).

## Passo a passo do que voce escreveu

**`strings.xml`** centraliza textos do app — `app_name`, `watchface_name`, etc. Quando precisar internacionalizar, voce cria `values-pt-BR/strings.xml` e o sistema escolhe automaticamente.

**`xml/watch_face.xml`** e um arquivo praticamente vazio com `<wallpaper />`. So existe pra cumprir o contrato do `<meta-data>` no manifest. Em watchfaces avancadas voce poderia configurar coisas la.

**`AndroidManifest.xml`** declara as permissoes (`PROVIDE_BACKGROUND`, `WAKE_LOCK`), o `<application>`, e dentro dele o `<service>` com o nome `.WatchFaceService` (o ponto e atalho pro `namespace` do Gradle). O servico tem `directBootAware="true"` pra rodar antes do usuario destravar o relogio (Wear OS quer isso pra mostrar a hora cedo).

**`WatchFaceRenderer.kt`** estende `Renderer.CanvasRenderer2<NoOpSharedAssets>`. O construtor recebe `surfaceHolder` (a superficie de pixels onde voce desenha), `currentUserStyleRepository` (configuracoes do usuario, vazio aqui), `watchState` (estado do relogio: ambient, visibility), `CanvasType.HARDWARE` (acelerado por GPU) e `INTERACTIVE_UPDATE_RATE_MS = 60_000L` (atualiza a cada 1 minuto). O `render()` so faz `canvas.drawColor(Color.BLACK)`.

**`WatchFaceService.kt`** estende `WatchFaceService` da Watch Face Library e sobrescreve tres metodos: schema de estilo de usuario (vazio), gerenciador de complications (vazio) e `createWatchFace` (instancia o renderer e empacota num `WatchFace` digital).

## Pegadinhas

- **Crash silencioso por falta de `ic_launcher`.** O manifest aponta pra `@mipmap/ic_launcher`. Se nao existir, o app nem instala. Crie um PNG quadrado preto bem simples em todas as densidades (mdpi, hdpi, xhdpi, xxhdpi, xxxhdpi) ou use o "Image Asset Studio" do Android Studio.
- **Watchface nao aparece na lista.** Mais comum: `exported="true"` faltando no servico, ou o intent-filter incompleto, ou o drawable de preview faltando. `adb logcat | grep watchface` te ajuda a depurar.
- **`createWatchFace` e suspend.** Voce nao pode chamar codigo bloqueante dentro dela sem cuidado, mas pode chamar outras suspends. Se voce nunca viu coroutines Kotlin, fica tranquilo: aqui voce so retorna o `WatchFace`, sem precisar entender muito mais por enquanto.

## Conexao com a proxima tarefa

Tela preta no pulso. Agora a proxima tarefa adiciona o asset da fonte Caesar Dressing, e a tarefa 4 finalmente desenha algo (a hora) por cima.

## Pra sua proxima watchface

A trindade **manifest + service + renderer** e o esqueleto que voce vai recriar em **todo** watchface. As variantes aparecem em:
- O que vai dentro do `render()`
- Que `SharedAssets` voce carrega
- Se voce expoe `UserStyleSchema` (pra usuario customizar cores/elementos)
- Se voce tem complications

Mas a estrutura externa muda muito pouco. Voce vai aprender a copiar essa porcao pronta e focar nas duas funcoes que importam: `render()` e talvez `createUserStyleSchema()`.
