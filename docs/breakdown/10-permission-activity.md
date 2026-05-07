# Tarefa 10 — PermissionActivity (ACCESS_COARSE_LOCATION)

## Visao geral

Pra ler GPS, voce precisa de **permissao de runtime** — declarar no manifest nao basta a partir de Android 6.0 (API 23). Voce precisa explicitamente pedir ao usuario, em uma tela, e o sistema mostra um dialog. Se ele negar, voce trabalha sem GPS.

Como watchfaces sao um tipo "headless" de app (nao tem janela visivel propria), voce nao tem onde pedir permissao **dentro** do watchface. Solucao: criar uma **Activity** comum, com icone no launcher do relogio, que o usuario abre uma vez na primeira instalacao pra conceder. Depois nunca mais precisa abrir.

## Tecnologias e conceitos novos

**`Activity` em Android.** Tela com UI. Ciclo de vida `onCreate → onStart → onResume → ...`. Mais antiga que Service e mais usada — quase todo app Android tem pelo menos uma. Mesmo um watchface "puro" se beneficia de uma Activity de configuracao/permissao.

**`ComponentActivity`.** A classe base moderna pra Activities (do AndroidX). Sucessora de `Activity` direta e da `AppCompatActivity`. Suporta `registerForActivityResult`, ViewModel, lifecycle scopes.

**`<activity>` no manifest.** Declarar uma `<activity>` registra ela com o sistema. Adicionar um `<intent-filter>` com `MAIN` + `LAUNCHER` faz a activity virar o "ponto de entrada" — aparece no menu de apps do relogio (com nome `permission_activity_label` da `strings.xml`).

**Permissoes runtime (API 23+).** Desde Android 6, permissoes "perigosas" (localizacao, microfone, camera, contatos, etc.) precisam ser:
1. Declaradas no manifest (`<uses-permission android:name="..." />`).
2. Pedidas em runtime via API moderna.
3. Verificadas antes de cada uso (`ContextCompat.checkSelfPermission`).

Permissoes "normais" (internet, vibrate) so precisam do (1).

**`registerForActivityResult` + `ActivityResultContracts.RequestPermission`.** API moderna (substituiu o velho `onRequestPermissionsResult`). Voce **registra** um launcher na inicializacao da Activity:

```kotlin
private val launcher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { granted: Boolean ->
    // callback chamado depois do dialog do sistema
}
```

E **lanca** quando precisar:

```kotlin
launcher.launch(Manifest.permission.ACCESS_COARSE_LOCATION)
```

A registracao **precisa** acontecer antes de `onStart` (em `onCreate` ou como inicializador de field). Isso permite que o sistema reconecte o callback se a Activity for destruida e recriada (rotacao, etc.).

**`Manifest.permission.ACCESS_COARSE_LOCATION`.** String constante `"android.permission.ACCESS_COARSE_LOCATION"`. Usar a constante evita typo.

**`COARSE` vs `FINE` location.**
- `ACCESS_COARSE_LOCATION` — precisao ~1km. Usa Wi-Fi/celula. Permissao mais aceitavel pelo usuario.
- `ACCESS_FINE_LOCATION` — precisao ~metros. Usa GPS. Permissao mais sensivel.

Pra calculo de sunrise/sunset, COARSE e mais que suficiente: 1km de erro causa ~1 segundo de imprecisao no horario do nascer do sol — invisivel a olho nu. Sempre escolha o **menos privilegiado** que resolve.

**`setContentView` com TextView dinamico.** Em vez de criar um layout XML, voce cria um TextView em codigo e setta como conteudo da Activity. Pra UI minima de 2 linhas, e mais simples que XML. Pra UI complexa, voce escreveria um layout em `res/layout/activity_permission.xml` e faria `setContentView(R.layout.activity_permission)`.

## Passo a passo do que voce escreveu

**No manifest:**

```xml
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

Declara que o app **pode** pedir essa permissao.

```xml
<activity android:name=".PermissionActivity" ...>
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

Declara a activity e marca como entry point. O sistema gera o icone no launcher.

**`PermissionActivity.kt`:**

```kotlin
private val launcher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { granted ->
    statusView?.text = if (granted) "Permissao concedida." else "Permissao negada — fallback 06h/18h ativo."
}
```

Registrado como **field**, antes de `onCreate` rodar. O callback recebe `Boolean`.

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    val tv = TextView(this).apply { ... }
    statusView = tv
    setContentView(tv)
    launcher.launch(Manifest.permission.ACCESS_COARSE_LOCATION)
}
```

Cria a TextView, mostra na tela, dispara o request. O sistema interrompe com o dialog padrao "Permitir localizacao aproximada?". Quando o usuario decide, o callback dispara, atualizando o texto.

## Pegadinhas

- **Registrar launcher dentro de `onCreate` quebra.** A API exige que `registerForActivityResult` aconteca **antes** do `onStart` lifecycle. Boa pratica: declarar como property field (init eager) ou em `onCreate` antes de chamar `super.onCreate`. Se voce errar, a app crasha em rotacao com excecao "LifecycleOwner ... in unsupported state".
- **Permissao "negar e nunca mais perguntar".** Em Android 11+, se o usuario clicar "Don't ask again" duas vezes, o sistema bloqueia o request — `launcher.launch` nao mostra dialog mais, callback retorna `false` instantaneamente. Voce deveria detectar com `shouldShowRequestPermissionRationale` e direcionar pro Settings. Pra essa watchface pessoal, nao implementamos esse fluxo.
- **`statusView?` nullable.** Voce declara `private var statusView: TextView? = null`. Como o callback pode disparar antes de `onCreate` terminar (em teoria), e como Activities podem ser recriadas, e mais seguro tornar a referencia nullable e usar `?.` pra acessar.
- **Manifest precisa do `MAIN`+`LAUNCHER`** ou a activity nao vira icone no launcher do relogio. Se voce so tem MAIN sem LAUNCHER, a activity existe mas nao pode ser aberta pelo usuario.

## Conexao com a proxima tarefa

A tarefa 11 cria o `LocationProvider` que **usa** essa permissao concedida pra ler GPS. Sem o usuario abrir essa Activity uma vez e clicar "Permitir", o `LocationProvider.fetchOnce()` sempre retorna null.

## Pra sua proxima watchface

**Padrao Activity de permissao em watchface:** sempre que sua watchface precisar de permissao runtime, voce vai criar uma Activity dedicada com MAIN+LAUNCHER. O usuario instala, abre uma vez, concede, fecha. Daqui em diante a watchface trabalha sozinha.

**`registerForActivityResult` substituiu o velho onRequestPermissionsResult.** Memorize o padrao. Funciona pra outros "results" tambem: pegar foto da camera, pedir contato, escolher arquivo, retorno de OAuth.

**Sempre prefira COARSE a FINE quando ambos resolvem.** Usuario aceita mais facil, e e mais respeitoso com privacidade. So pegue FINE se o caso de uso realmente exigir precisao de metros.

**Active design: tela minima.** Sua Activity de permissao pode ser feiosa. Ninguem vai voltar nela. Foque em ser direta — texto clarinho descrevendo por que voce precisa, botao se preciso, fim.
