# Tarefa 3 — Adicionar fonte Caesar Dressing como asset

## Visao geral

Esta e uma tarefa quase administrativa: voce baixa o arquivo `.ttf` da Caesar Dressing do Google Fonts e coloca dentro de `app/src/main/assets/`. Nada de Kotlin, nada de XML, so um arquivo binario na pasta certa. Mas tem **conceito importante** por tras: a diferenca entre `assets/` e `res/`, como o Android empacota arquivos no APK e como o codigo carrega esses arquivos depois.

Sem ela, a fonte da hora ia cair no padrao do sistema (Roboto), e voce perderia toda a personalidade que escolheu. Caesar Dressing tem licenca **OFL (SIL Open Font License)** — pode redistribuir embedada no app sem problema legal.

## Tecnologias e conceitos novos

**`assets/` vs `res/`.** Sao duas formas de embutir arquivos no APK, com filosofias diferentes:

- **`res/`** (resources): cada arquivo gera um ID na classe `R` (`R.drawable.foo`, `R.raw.bar`). Sao processados pelo build do Android — XMLs sao otimizados, imagens podem ser densificadas. Bom pra coisas que voce quer referenciar declarativamente em XML (tipo no manifest ou em layouts).
- **`assets/`** e uma pasta "raw": tudo que voce poe la vai pro APK **sem processamento**, com a mesma estrutura de pastas. Voce abre os arquivos por **caminho relativo** atraves do `AssetManager`. Bom pra arquivos grandes, opacos ao sistema (TTF de fonte, modelos ML, JSON estatico, midia bruta).

Pra fontes em watchface, `assets/` e o caminho mais direto. (Existe tambem o sistema "downloadable fonts" e o `res/font/`, mas pra um caso simples assim, `assets/` e menos burocratico.)

**`Typeface.createFromAsset(context.assets, "CaesarDressing-Regular.ttf")`.** API do Android pra carregar uma fonte de um caminho relativo dentro de `assets/`. Retorna um `Typeface` que voce passa a um `Paint` pra desenhar texto. Voce vai usar isso na proxima tarefa.

**OFL — SIL Open Font License.** Licenca permissiva criada pela SIL International pra fontes. Permite usar, modificar e redistribuir a fonte (incluindo embedada em software) **desde que** voce nao venda a fonte sozinha (sem o app). Pra apps comerciais e nao-comerciais, e segura. Caesar Dressing tem essa licenca, entao voce pode tranquilamente fazer commit do `.ttf` no repo.

**Tamanho do arquivo no APK.** Fontes TTF tipicas tem entre 20KB e 200KB. Caesar Dressing fica em torno de 30-40KB. Isso nao e nada num APK Wear OS (limite pratico ~5MB pra distribuicao via Play Store; sideload sem limite). Voce pode embutir varias fontes sem se preocupar com peso.

## Passo a passo do que voce executou

1. Foi em [fonts.google.com/specimen/Caesar+Dressing](https://fonts.google.com/specimen/Caesar+Dressing).
2. Clicou em "Download family". Veio um ZIP com `CaesarDressing-Regular.ttf` e `OFL.txt` (a licenca).
3. Extraiu o ZIP, copiou o `.ttf` pra `app/src/main/assets/`. (Se a pasta `assets/` nao existir, crie.)
4. Verificou com `ls -la app/src/main/assets/` que o arquivo esta la.
5. Rodou `./gradlew assembleDebug` so pra confirmar que o build empacota o asset sem reclamar.
6. Commit.

E so isso.

## Pegadinhas

- **Path case-sensitive.** No Android (Linux por baixo), `CaesarDressing-Regular.ttf` e `caesardressing-regular.ttf` sao arquivos **diferentes**. Se voce digitar errado em `Typeface.createFromAsset(...)`, da `RuntimeException` em runtime — o build nao avisa. Pra evitar, copie o nome exato do arquivo.
- **OFL.txt junto.** Se for distribuir o app na Play Store, **inclua a licenca OFL** em algum lugar acessivel ao usuario (geralmente uma tela "Sobre" com creditos). Pra sideload pessoal, nao e juridicamente obrigatorio mas e boa pratica. Pode commitar o `OFL.txt` junto com o `.ttf` no `assets/` (vai junto no APK).
- **Versoes variaveis (variable fonts).** Caesar Dressing nao tem variantes (so Regular). Pra fontes variaveis (axes de peso, largura), o `.ttf` e maior e o Android API 26+ suporta acessar axes. Nao e o caso aqui.
- **Cache da fonte.** O `Typeface.createFromAsset()` faz IO disco a cada chamada. Voce **nao** quer chamar isso a cada `render()` (60 frames por minuto se necessario). Carregue uma vez no `init` ou na primeira chamada e guarde em propriedade. (E exatamente o que a tarefa 4 vai fazer.)

## Conexao com a proxima tarefa

A tarefa 4 vai fazer:
1. Carregar essa fonte com `Typeface.createFromAsset` no construtor do Renderer.
2. Criar um `Paint` que usa essa Typeface.
3. Desenhar a hora atual com `canvas.drawText(...)`.

Sem o asset que voce adicionou aqui, o `Typeface.createFromAsset` ia jogar `RuntimeException`.

## Pra sua proxima watchface

**Padrao geral:** todos os recursos opacos (fontes, sons, imagens nao-vetoriais grandes, dados JSON estaticos) entram em `assets/`. Voce os le via `context.assets.open(path)` ou APIs especializadas (`Typeface.createFromAsset`, `MediaPlayer.setDataSource`, etc).

**Decisao `assets/` vs `res/raw/`:** ambos funcionam pra arquivos opacos, mas:
- `res/raw/` referencia por `R.raw.id` no codigo. Mais "android-idiomatic". Use quando voce quer compile-time check do nome.
- `assets/` referencia por string. Use quando o conjunto de arquivos pode crescer dinamicamente, ou pra preservar estrutura de pastas.

Pra fontes especificamente, ha um terceiro caminho moderno: `res/font/` + `<font-family>` XML. Permite usar `R.font.caesardressing` em XML de layout. Mas em watchfaces voce trabalha quase tudo em codigo (Canvas), entao `assets/` ou `res/font/` sao igualmente validos. `assets/` ganha pela menor burocracia.
