# Tarefa 1 — Bootstrap do projeto Gradle

## Visao geral

Antes de uma linha de Kotlin existir, voce precisa do **scaffolding de build**. No mundo Android, isso significa Gradle. Esta tarefa cria os arquivos que ensinam o Gradle a montar seu APK: como compilar Kotlin, qual SDK do Android usar, quais bibliotecas baixar, e que o projeto tem um modulo unico chamado `app`.

Se voce vem de Node.js, Gradle e o equivalente do `package.json` + `webpack.config.js` + `tsconfig.json` consolidados. Se vem de Python, e o `pyproject.toml` + `setup.py`. E uma ferramenta de build com sistema de plugins; o "ecossistema Android" e basicamente um conjunto de plugins Gradle oficiais do Google.

## Tecnologias e conceitos novos

**Gradle.** Ferramenta de build automatizado da JVM. Le arquivos `.gradle.kts` (script Kotlin) ou `.gradle` (Groovy). Funciona por **tasks** que voce invoca via `./gradlew <task>` (ex.: `./gradlew assembleDebug`).

**Kotlin DSL para Gradle.** Os arquivos `.gradle.kts` sao codigo Kotlin de verdade. Te da autocomplete e checagem de tipo na IDE — ao contrario do Groovy classico.

**Estrutura tres-arquivos.** Todo projeto Android Gradle tem essa hierarquia:
- `settings.gradle.kts` — declara quais modulos existem (aqui so `:app`) e onde buscar plugins.
- `build.gradle.kts` (raiz) — declara plugins disponiveis no projeto inteiro, sem aplica-los.
- `app/build.gradle.kts` — configura o modulo: SDK alvo, versao, dependencias, plugins efetivos.

**applicationId vs namespace.** O `applicationId` e o ID unico do app na Play Store/sistema (`dev.felipe.watchface`). O `namespace` e o pacote Java/Kotlin onde a classe `R` (recursos gerados) vai aparecer. Eles podem ser iguais ou diferentes; aqui sao iguais.

**SDK levels.**
- `compileSdk` — versao do Android contra a qual o codigo compila. Define que APIs voce pode chamar.
- `minSdk` — versao minima onde o app vai instalar (30 = Android 11; Wear OS 3+).
- `targetSdk` — versao alvo declarada. Afeta comportamentos de compatibilidade.

**AndroidX.** Conjunto moderno de libs de suporte do Google (substitui o velho `android.support.*`). Habilitado via `android.useAndroidX=true` no `gradle.properties`.

**Java Toolchain.** Voce pinou Java 17 como compatibilidade. AGP 8.x ja exige Java 17 pra rodar; sem isso, o build quebra.

**Gradle Wrapper.** Arquivos `gradlew`, `gradlew.bat` e `gradle/wrapper/*` sao um script + jar que baixam a versao certa do Gradle automaticamente. Garantia: qualquer pessoa que clonar o repo vai usar a mesma versao do Gradle que voce, sem precisar instalar nada.

## Passo a passo do que voce escreveu

**`settings.gradle.kts`** define `pluginManagement` (de onde Gradle baixa plugins — Google Maven, Maven Central, Gradle Plugin Portal) e `dependencyResolutionManagement` (de onde baixa libs do app). Inclui o modulo `:app`. Sem esse arquivo, Gradle nao monta o projeto.

**`build.gradle.kts` raiz** lista plugins disponiveis com `apply false` — quer dizer "esses plugins existem no classpath, mas nao se aplicam ainda". Modulos individuais aplicam quando precisam.

**`gradle.properties`** sao flags globais. `org.gradle.jvmargs=-Xmx2048m` da memoria pra JVM do build (Android builds comem RAM). `android.useAndroidX=true` ativa AndroidX.

**`app/build.gradle.kts`** e o coracao: declara que esse modulo e uma `application` Android Kotlin, configura SDK levels, applicationId, e lista todas as dependencias. As dependencias usam coordenadas Maven `grupo:artifact:versao`, ex.: `androidx.wear.watchface:watchface:1.2.1`.

## Pegadinhas

- **Wrapper precisa ser gerado uma vez.** Se voce nao tem Gradle 8.5+ instalado, `gradle wrapper --gradle-version 8.5` nao funciona. A saida pratica: importar o projeto no Android Studio uma vez — ele detecta a falta do wrapper e oferece "Use Gradle wrapper task". Aceita. Pronto.
- **`sourceSets` mapeia kotlin/.** Por padrao, Android espera `src/main/java/`. Como queremos `src/main/kotlin/` (mais idiomatic), precisa redirecionar via `sourceSets { getByName("main") { java.srcDirs("src/main/kotlin") } }`.
- **Versoes de libs envelhecem rapido.** As versoes que voce pinou (1.2.1 da watchface, 21.3.0 da location, etc.) eram correntes em 2025-2026. Se rebuilds quebrarem por incompatibilidade, suba uma de cada vez verificando o changelog.

## Conexao com a proxima tarefa

Voce tem o esqueleto Gradle pronto, mas ainda nao ha `AndroidManifest.xml` nem nenhum `.kt`. A proxima tarefa cria o manifest, o servico do watchface e o renderer minimo — e o primeiro `./gradlew installDebug` que vai pra um relogio real.

## Pra sua proxima watchface

Esse mesmo padrao **`settings.gradle.kts` + raiz + modulo`**`** se repete em **todo** projeto Android Gradle, do mais simples ao mais complexo. Voce nao escreve isso do zero todo projeto — copia daqui, ajusta o `applicationId`/`namespace`, troca dependencias, e segue. O Android Studio tambem oferece "New project" com varios templates (incluindo "Empty Wear OS app") que geram esses arquivos pra voce.

Conhecer **a estrutura** importa mais que decorar a sintaxe: quando algum build quebrar no futuro, voce vai saber em qual dos tres arquivos o problema esta.
