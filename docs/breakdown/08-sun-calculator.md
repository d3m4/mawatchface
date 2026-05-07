# Tarefa 8 — SunCalculator (wrapper de commons-suncalc)

## Visao geral

Voce nao quer reescrever o algoritmo de posicao solar do zero — e matematica complicada de astronomia. A biblioteca **commons-suncalc** ja faz isso, e voce so adiciona um **wrapper fino** que esconde a API dela atras de uma fachada limpa que retorna `(sunrise, sunset)` pra um par lat/lon e uma data.

Isso introduz dois conceitos importantes: **integrar uma biblioteca externa** e **encapsular API estranha** atras de uma interface conveniente.

## Tecnologias e conceitos novos

**`commons-suncalc`.** Biblioteca Java pure-Java (sem dependencias Android) escrita por Richard Körber. Calcula posicao do sol, lua, eclipses, estacoes — tudo offline, sem precisar de API externa. Distribuida via Maven Central (`org.shredzone.commons:commons-suncalc:3.10`). Voce ja adicionou nas dependencias na tarefa 1.

**Builder pattern.** A API da `SunTimes` segue builder fluente:

```kotlin
val times = SunTimes.compute()      // cria builder
    .on(date)                       // define data
    .at(lat, lon)                   // define coordenadas
    .timezone(zone)                 // define timezone
    .execute()                      // executa, retorna SunTimes
```

Cada `.on/.at/.timezone` retorna o proprio builder — voce encadeia chamadas sem variaveis temporarias. `execute()` finaliza e retorna o resultado. Padrao **muito** comum em libs Java/Kotlin.

**`object` em Kotlin = singleton.** A declaracao `object SunCalculator { ... }` cria uma classe com **uma unica instancia**, criada na primeira vez que voce a referencia (lazy). E o equivalente do Singleton pattern em Java mas sem boilerplate.

```kotlin
object SunCalculator {
    fun compute(...) { ... }
}

// uso:
SunCalculator.compute(lat, lon, date, zone)
```

Voce nao instancia. Chama o nome direto. Util pra "stateless utilities" — funcoes puras agrupadas.

**`java.time.LocalDate / ZonedDateTime / ZoneId`.** API moderna de data/hora em Java 8+ (e Android via "core library desugaring", o AGP 8.x faz isso automaticamente). Conceitos:
- **`LocalDate`** — uma data sem hora nem fuso (ex.: 2026-05-08). "8 de maio em algum lugar".
- **`ZonedDateTime`** — instante exato no tempo + fuso. Tem ano, mes, dia, hora, min, seg, e timezone.
- **`ZoneId`** — identificador de fuso (`America/Sao_Paulo`, `UTC`, etc). `ZoneId.systemDefault()` pega o do dispositivo.

A diferenca e crucial: `LocalDate` nao representa "instante" no tempo. Pra saber "que momento exato e meia-noite no dia X em SP?", voce faz `localDate.atStartOfDay(saoPauloZone)` que vira um `ZonedDateTime`.

**Wrapper como facade.** O codigo cliente nao precisa conhecer a API do commons-suncalc. Ve so:

```kotlin
val window: SunWindow = SunCalculator.compute(lat, lon, date, zone)
```

Se um dia voce trocar de biblioteca (digamos por uma mais leve), voce so muda o **interior** de `compute`. O codigo cliente nao se mexe. Esse e o principio da **dependency inversion** + **encapsulation**.

**`data class SunWindow`.** Estrutura simples com dois campos:

```kotlin
data class SunWindow(val sunrise: ZonedDateTime, val sunset: ZonedDateTime)
```

`data class` em Kotlin auto-gera `equals`, `hashCode`, `toString`, `copy`, e destructuring (`val (rise, set) = window`). Equivalente ao "POJO-com-Lombok" do Java, mas built-in. **Use sempre que quiser uma tupla com nomes.**

**Fallback gracioso.** Em regioes polares no inverno/verao, o sol nao nasce ou nao se poe (o "sol da meia-noite"). Nesses casos, `times.rise` ou `times.set` podem vir `null`. Voce poe um fallback razoavel:

```kotlin
val rise = times.rise ?: date.atTime(6, 0).atZone(zone)
val set = times.set ?: date.atTime(18, 0).atZone(zone)
```

`?:` e o Elvis operator: "use o valor da esquerda se nao-null, senao use o da direita".

## Passo a passo do que voce escreveu

A funcao `compute(lat, lon, date, zone)` segue o builder, depois extrai `rise` e `set`. Cada um pode ser null — voce protege com Elvis. Empacota num `SunWindow` e devolve. Codigo enxuto, ~10 linhas.

A `data class SunWindow` fica como uma estrutura nomeada acessivel ao mundo — qualquer codigo no projeto pode importar e usar.

## Pegadinhas

- **API de commons-suncalc mudou entre 2.x e 3.x.** Em 2.x, `times.rise` retorna `Date?` (legado). Em 3.x, retorna `ZonedDateTime?`. Voce esta em 3.10, entao tudo certo. Se buildar com 2.x por engano, vai dar erro de tipo.
- **Timezone matters.** Se voce passar `ZoneId.of("UTC")` em vez do fuso local, a biblioteca calcula o sunrise em UTC (ainda preciso, mas o `ZonedDateTime` retornado fica em UTC). Voce **quer** o fuso local pra que comparar com `ZonedDateTime.now()` faca sentido. Sempre passe `ZoneId.systemDefault()`.
- **`object` vs `class` vs `companion object`.** Se voce quer estado por instancia, use `class`. Se quer uma utility singleton, use `object`. Se quer "metodos de classe" estilo Java static, use `class Foo { companion object { fun bar() {} } }`.

## Conexao com a proxima tarefa

A tarefa 9 cria o `SunCache` que persiste em SharedPreferences a ultima `(lat, lon)` e o ultimo par `(sunrise, sunset)`. Junto, eles formam o pipeline: `LocationProvider` busca → `SunCalculator.compute` calcula → `SunCache.store` persiste → o Renderer le do cache.

## Pra sua proxima watchface

**Wrappers magros sao baratos e valem a pena.** Mesmo que sua "facade" tenha so uma funcao chamando uma API externa, ela te da:
1. Lugar pra trocar dependencia sem cascatear refactor.
2. Nome dominante (`compute`) em vez do nome da biblioteca.
3. Lugar pra adicionar logging, cache, fallback, sem invadir o resto do codigo.

**`object` pra utilities.** Sempre que voce tiver "um conjunto de funcoes relacionadas que nao guardam estado", `object` e a forma mais limpa em Kotlin.

**`data class` pra tuplas com nome.** Em vez de `Pair<ZonedDateTime, ZonedDateTime>` (sem clareza qual e qual), `SunWindow(sunrise, sunset)` e auto-documentavel.

**`?:` e seu amigo.** Em codigo Android, voce vai lidar com **muitos** valores nullable (do sistema, da API, de I/O). O Elvis operator + `?.` (safe call) sao a forma idiomatica de Kotlin de lidar com isso sem exception-fests.
