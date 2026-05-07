# Tarefa 16 — Calibracao final + criterios de aceite

## Visao geral

A funcionalidade ja esta toda implementada nas tarefas 1-15. Esta tarefa e sobre **fechar o ciclo**: rodar a checklist de aceite da spec no relogio real, ajustar o que precisa ser empirico (tipicamente o tamanho da fonte), atualizar o README, e fazer o commit final + push.

Aqui voce aprende uma disciplina de engenharia que e pouco discutida mas extremamente importante: **definir criterios de aceite antes de implementar**, e usar esses criterios pra saber **quando parar**.

## Tecnologias e conceitos novos

**Criterios de aceite (acceptance criteria).** Sao a lista de "como saber que esta pronto" que voce escreveu na spec **antes** de implementar. Vantagens:
- Sao concretos e verificaveis (nao "deve funcionar bem" mas "deve mostrar 60 tracos quando bateria 100%")
- Evitam scope creep ("ja que estou aqui, adiciono mais X")
- Te dao um "stop signal" — quando todos os criterios passam, voce parou.

A spec tem 9 criterios. Voce roda cada um manualmente no Watch 8 Classic, marca check, e segue.

**Validacao manual em watchface.** Testes automatizados de UI (Espresso, etc.) nao funcionam bem em watchface — o pipeline de render passa pelo sistema, vc nao tem acesso direto aos pixels. Pra esse projeto pessoal, validacao manual no relogio real e o caminho certo. Voce coloca o relogio no pulso, usa pelo dia, e observa.

**Polish creep.** O risco oposto a "scope creep" e ficar polindo demais. Ajustar 5px aqui, 2px ali, "ah, mas se eu mudasse a cor um pouco...". Sem criterios claros, voce gasta horas em melhorias incrementais que ninguem vai notar. **Os criterios da spec sao o limite**. Se passa, ta pronto. Liberte-se da urgencia de continuar polindo.

**Calibracao empirica de fonte.** Algumas decisoes so podem ser feitas no dispositivo real:
- Tamanho que parece "certo" sem cortar bordas
- Contraste suficiente em luz forte (ao ar livre)
- Legibilidade de relance vs leitura focada

A spec ja deixou um range (240-300px) — voce ajusta uma vez no fim, comparando varios valores. Em ~10 minutos voce tem uma resposta.

**Drenagem de bateria como criterio.** O criterio 9 ("drenagem dentro de ±10% da padrao em 8h") e o mais subjetivo. Voce precisa medir:
1. Carregar o relogio a 100%.
2. Selecionar a watchface padrao do sistema, observar bateria depois de 8h.
3. Resetar pra 100%.
4. Selecionar a sua watchface, observar 8h.
5. Comparar.

Trabalhoso, mas vale: e o criterio que voce **menos quer falhar** num produto pessoal.

## Passo a passo do que voce executa

**Step 1: Rodar os 9 criterios.** Lista da spec, cada um marcado checkbox:

1. Instalado e selecionavel
2. Hora em Caesar Dressing 24h
3. Anel reflete bateria (testar 100, ~50, baixo)
4. DIA: claro topo, escuro embaixo
5. NOITE: escuro topo, claro embaixo
6. Transicao sunrise/sunset visivelmente suave
7. Sem GPS / negada → fallback 06h/18h
8. Ambient mode → tela apagada
9. Bateria ±10% da padrao em 8h

Pra alguns voce vai ter que **forcar condicoes**: mudar hora do dispositivo pra testar transicoes, desligar GPS pra testar fallback, etc. Pra criterio 6 e 9 voce precisa testar **em tempo real** durante o dia inteiro.

**Step 2: Calibrar fonte se necessario.** Se a hora ficou pequena ou cortando bordas:

```kotlin
const val HOUR_FONT_SIZE_PX = 280f  // ajuste 240..300 conforme necessario
```

Cada deploy e ~10s — voce testa varios valores rapido.

**Step 3: Atualizar README.** Substituir secao `## Status` por instrucoes de instalacao concretas:

```markdown
## Status

Implementacao completa. Watchface funcional via sideload ADB.

### Como instalar

1. Habilitar modo desenvolvedor + ADB Wi-Fi no relogio
2. adb pair / adb connect
3. ./gradlew installDebug
4. Settings > Watch faces > "Minimal BW"
5. Abrir launcher "Permissoes" pra conceder ACCESS_COARSE_LOCATION
```

**Step 4: Commit + push final.**

```bash
git add .
git commit -m "calibracao final e atualizacao do readme"
git push
```

## Pegadinhas

- **Nao "ajeitar uma coisa de ultima hora" sem revisar o ciclo.** Se calibrar tamanho de fonte significa mudar o renderer, voce **revalida o criterio 2** ("hora em Caesar Dressing 24h") — passa? Se sim, segue. Se quebrou outro criterio sem voce notar, esta voltando atras.
- **Bateria depende de muitas variaveis.** Temperatura, idade da bateria, sinal Bluetooth, app de saude rodando, etc. ±10% e generoso — voce pode ver mais de 10% em dias diferentes mesmo com a mesma watchface. Faca media de 2-3 dias.
- **Polish creep sutil.** Voce vai sentir vontade de adicionar "uma sombra fininha aqui", "um anti-aliasing extra ali". **Anote pra V2** num arquivo separado e SIGA. Lance V1.
- **`git push` final.** O remote e publico. Confira que nada sensivel vazou (procurar por chaves de API, paths absolutos com seu usuario, etc.) antes de rodar push.

## Conexao com a proxima tarefa

Nao ha proxima tarefa — voce **terminou**. Entrega completa, watchface no pulso, repo publico.

Se mais tarde voce quiser uma V2, comeca um novo ciclo brainstorm → spec → plano → implementacao. As licoes que voce aprendeu aqui aceleram tudo.

## Pra sua proxima watchface

**Criterios de aceite na spec.** Comece **toda** spec com a secao "Validacao manual" antes de pensar na implementacao. Se voce nao consegue articular como saber que esta pronto, voce nao entendeu o que esta construindo.

**Calibracao em separado dos features.** Trate calibracao como tarefa final, nao como ajuste continuo durante o desenvolvimento. Voce escreve o codigo com um valor inicial razoavel (palpite), e na hora de polir voce ajusta com olho no aparelho. Evita o ciclo "ajusta 2px → recompila → testa → ajusta 1px → recompila..." durante features.

**Lance V1, anote V2.** Apaixonado pelo projeto? Otimo. Vai ter ideias todo dia. Anote num `IDEAS.md` ou num issue do GitHub. Nao implemente todas. V1 sai pelos criterios; V2 sai quando voce tiver vontade e tempo.

**Documentacao no fim, nao no comeco.** Voce **escreve** a spec no comeco (necessario pra alinhar mente). Mas a documentacao do README ("como instalar", "como usar") so faz sentido depois que o produto existe — voce pode tirar print real, escrever com confianca de que aquilo funciona, etc. Ate la, README so tem skeleton.

**Quando esta pronto, esta pronto.** Aprenda a sentir o "pronto". Watchface funciona, criterios passam, documentacao escrita, commit feito. Levante da cadeira, abra outra coisa. Nao revisite por dias. Quando voltar, vai notar com olhos novos o que precisa de V2.
