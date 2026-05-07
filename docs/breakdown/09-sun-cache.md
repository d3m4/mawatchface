# Tarefa 9 — SunCache (persistencia em SharedPreferences)

## Visao geral

Voce nao quer chamar o GPS toda hora — caro de bateria, lento, e pode falhar. Tambem nao quer perder os dados quando o relogio reinicia. Solucao: **persistir** a ultima `(lat, lon)` e o par `(sunrise, sunset)` calculado, e ler do cache na maioria dos frames. So atualiza quando o WorkManager (tarefa 12) buscar nova localizacao.

`SunCache` e a camada de persistencia. Ela usa **SharedPreferences**, o jeito mais simples do Android de salvar dados chave-valor leves no disco interno do app.

## Tecnologias e conceitos novos

**`SharedPreferences`.** API de persistencia chave-valor key-value muito simples no Android. Cada arquivo de prefs e um XML em `/data/data/<package>/shared_prefs/<nome>.xml`. So o app tem acesso (sandboxed). Suporta tipos primitivos: `String, Int, Long, Float, Boolean, Set<String>`. **Nao suporta** `Double` (workaround: armazene como `Float` ou `Long`-bits).

E pequeno e rapido pra dados ate ~kilobytes. Pra dados maiores ou estruturados, voce sobe pra `DataStore` (substituto moderno) ou `Room` (banco SQLite). Pra um cache de ~5 valores, SharedPrefs e perfeito.

**`Context.getSharedPreferences("nome", MODE_PRIVATE)`.** Abre (ou cria) o arquivo de prefs com nome dado. `MODE_PRIVATE` e o unico modo seguro (outros sao deprecated).

**Read API (sem boilerplate):**
```kotlin
val value: String? = prefs.getString("key", null)
val int: Int = prefs.getInt("key", -1)
```
Sempre tem um valor default pra quando a chave nao existe.

**Write API (via Editor):**
```kotlin
prefs.edit()
    .putString("key", "value")
    .putFloat("lat", 12.34f)
    .apply()  // assincrono, fire-and-forget
```
Voce pega um `Editor`, encadeia `put*`, e chama `apply()`. **`apply()` e assincrono** (vai pro disco em background); **`commit()` e sincrono** mas bloqueia a thread chamadora. Use `apply()` 99% das vezes.

**Properties customizadas em Kotlin.** Em vez de getters/setters em Java, Kotlin te da **properties** com `get()` e `set()` customizaveis:

```kotlin
var lat: Double?
    get() = if (prefs.contains(KEY_LAT)) prefs.getFloat(KEY_LAT, 0f).toDouble() else null
    set(value) {
        prefs.edit().apply {
            if (value != null) putFloat(KEY_LAT, value.toFloat()) else remove(KEY_LAT)
        }.apply()
    }
```

O codigo cliente le e escreve como se fosse propriedade comum:

```kotlin
val l = cache.lat       // chama get
cache.lat = 45.6        // chama set
cache.lat = null        // chama set com null → remove a chave
```

Limpa, sem `getLat()`/`setLat(value)`/`clearLat()`.

**Conversao Double → Float pra SharedPreferences.** Como SharedPrefs nao tem `putDouble`, voce armazena como `Float` (perde precisao depois da ~7a casa decimal). Pra coordenadas geograficas, isso e ~1cm de imprecisao na superficie da Terra — irrelevante pra calculo solar (que e preciso so a kilometros).

**`apply { ... }` no Editor.** Cuidado: tem dois `apply` aqui. Um e o `apply { ... }` do Kotlin (escope function que retorna o objeto), outro e o `editor.apply()` que escreve as prefs. Confuso. Voce escreve:

```kotlin
prefs.edit().apply {        // escope function: dentro deste bloco, "this" = Editor
    if (value != null) putFloat(...)
    else remove(...)
}.apply()                   // <- esse apply chama Editor.apply() pra escrever no disco
```

**`prefs.contains(KEY_LAT)`.** Verifica se a chave existe **sem** retornar valor default. Util quando o "default" (`0f`) seria um valor valido — voce nao saberia distinguir "ausente" de "armazenado como 0".

**ISO 8601 strings pra ZonedDateTime.** SharedPrefs tem `putString` mas nao `putZonedDateTime`. Solucao classica: serializar como ISO string (`"2026-05-08T05:42:13-03:00[America/Sao_Paulo]"`) com `.toString()`, e desserializar com `ZonedDateTime.parse(str)`. Round-trip perfeito.

## Passo a passo do que voce escreveu

**Chaves** declaradas como `private const val` num `companion object`. Constantes privadas, escopo da classe, sem alocacao por instancia.

**5 properties:** `lat`, `lon`, `cachedDate`, `sunrise`, `sunset`. Cada uma com getter/setter que le/escreve a SharedPreferences. Setters aceitam null pra remover a chave.

**`store(window, lat, lon, date)`** — convenience method que escreve **tudo de uma vez** em uma transacao Editor. Mais eficiente que setar cada propriedade separadamente (uma chamada de `apply()` so).

**`read(zone)`** — convenience method que monta o `SunWindow` se ambos sunrise e sunset estiverem presentes; retorna `null` se algum faltar. Tambem ajusta o zone do ZonedDateTime pra match com o que o caller espera (`withZoneSameInstant(zone)`).

## Pegadinhas

- **`apply()` e async.** Se voce escrever e ler imediatamente em outra thread, pode ler o valor antigo. Em pratica isso quase nunca acontece em apps Android porque a UI thread serializa tudo, mas vale saber.
- **SharedPrefs nao e seguro pra dados sensiveis.** Senha, token, info pessoal — nao guarde em SharedPrefs sem criptografia. Use `EncryptedSharedPreferences` da Jetpack Security pra isso.
- **`prefs.edit().apply { ... }` vs `prefs.edit().also { ... }`.** Funcional sao iguais. `apply` te da `this = Editor`; `also` te da `it = Editor`. Pra encadear puts curtos, `apply` fica mais limpo.
- **Race condition entre processos.** SharedPrefs nao e safe entre processos (se voce tiver um service em outro processo, etc). No nosso caso e tudo um processo so — sem problema.

## Conexao com a proxima tarefa

A tarefa 10 cria a Activity que pede permissao de localizacao. Sem permissao, nem rola buscar `lat/lon`, entao tarefas 10-11 vem antes da tarefa 12 (que e quem realmente popula o cache).

## Pra sua proxima watchface

**SharedPreferences pra cache leve.** Sempre que voce tiver < 50 valores simples pra persistir (settings de usuario, ultimo estado conhecido, flags de feature), SharedPrefs e a opcao zero-overhead.

**Properties customizadas escondem complexidade.** Em vez de espalhar `prefs.edit().putString(...).apply()` pelo codigo, voce centraliza no `SunCache`. O codigo cliente fica limpo. Esse padrao se aplica a qualquer wrapper de fonte de dados.

**`var prop: T? get() set()`** e Kotlin idiomatic pra "field-com-comportamento". E o equivalente de um Java POJO com getters/setters anotados, mas com 1/3 do codigo.

**Convenience methods pra operacoes em lote.** `store(...)` que faz tudo em uma transacao SharedPreferences economiza I/O. Padrao geral: se voce vai escrever varias chaves relacionadas, faca em **uma** chamada do Editor.
