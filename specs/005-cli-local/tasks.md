# Tarefas: S05 CLI local

> **Para agentes:** execute tarefa a tarefa em TDD (vermelho → verde → refatorar). Marque as caixas (`- [x]`) conforme concluir cada passo. Código em `Que-Bug-E-Esse/back` (ADR 0009 A).

**Objetivo:** implementar a CLI local do modo grupo no mesmo dispositivo (`interfaces.cli`) e o ponto de entrada `main` (`bootstrap`), fechando o marco M1 com os 15 CAs verdes e cobertura ≥ 90% em `com.quebugesse.interfaces.cli`.

**Arquitetura:** `interfaces.cli` depende apenas de `application` (DTOs, `CodigoErro` e os 10 casos de uso de `CasosDeUsoCli`, sem `ResponderComIA`/`AvaliarComIA`); `CliLocal` recebe `BufferedReader` e `PrintWriter` para testes determinísticos sem terminal real; `bootstrap` monta adaptadores em memória, `SeedInicial`, `PublicadorDeEventosEmMemoria` e expõe `AplicacaoCli` + `Main.kt`.

**Stack:** Kotlin/JVM (JDK 25 LTS), Gradle 9.8.2 (Kotlin DSL + plugin `application`), Kotest (`FunSpec`, `assertions-core`), Konsist, Kover.

**Spec:** [spec.md](spec.md) · Plano técnico: [plan.md](plan.md) · Constituição: [docs/constitution.md](../../docs/constitution.md)

## Restrições globais

- RNF1 / P7: `interfaces/src/main` importa exclusivamente `com.quebugesse.application.*` e biblioteca padrão (zero import de `domain` ou `infrastructure`).
- P3 / CL6: tela inicial e `ver` nunca exibem `solucaoCompleta` antes do fim, nem na vez do narrador; `solucao` pede confirmação `Mostrar? (s/n) `.
- CL12: a CLI nunca referencia nem chama `ResponderComIA` ou `AvaliarComIA`.
- CL9 / RNF3: entrada e saída em UTF-8; linhas vazias ignoradas; EOF em qualquer ponto encerra com código 0 sem escrever em stderr.
- Todo teste de CA começa por `"CA-NN "`.
- Cobertura de linhas ≥ 90% em `com.quebugesse.interfaces.cli`.
- Testes: `.\gradlew.bat :interfaces:test`, `.\gradlew.bat :bootstrap:test`, `.\gradlew.bat check`.
- Commits em `Que-Bug-E-Esse/back`: `S05: <verbo no imperativo>`.

Convenções de caminho:
- `cli/` = `interfaces/src/main/kotlin/com/quebugesse/interfaces/cli/`
- `cli-test/` = `interfaces/src/test/kotlin/com/quebugesse/interfaces/cli/`
- `boot/` = `bootstrap/src/main/kotlin/com/quebugesse/bootstrap/`
- `boot-test/` = `bootstrap/src/test/kotlin/com/quebugesse/cli/`

## Foco de revisão

1. **Ordem de validação na abertura (CL5):** perguntar todos os 4 campos (ou 2 na abertura curta) antes de validar nomes → categoria → dificuldade → `IniciarPartida`; no erro, recomeçar a abertura inteira (CA-04, CA-05, CA-12).
2. **Normalização e grafia original (CL3, CL4, CL7):** `"Ana"` e `"ana"` são o mesmo jogador, mas o prompt `[Ana]> ` preserva a grafia da primeira digitação; `" Gestão de projeto "` e `"gestao-projeto"` selecionam `GESTAO_PROJETO` (CA-03, CA-05, CA-07).
3. **Sigilo P3 em tela compartilhada:** nem tela inicial nem `ver` (mesmo como narrador) podem mostrar a solução antes de `RESOLVIDO`/`ABANDONADO`; `solucao` só revela após `s`/`sim` (CA-01, CA-10).
4. **Ordem de erro em `responder`:** validar `sim|nao|irrelevante` (`ARGUMENTO_INVALIDO`) antes de checar se existe pergunta aberta (`SEM_PERGUNTA_ABERTA`, CA-15).
5. **Exaustividade de mensagens (CA-14):** todos os 22 `CodigoErro` e os 3 `CodigoCli` têm mensagem única e não vazia em português.

---

### Tarefa 1: `Normalizador`, parsers, exibição e `Mensagens` (CA-14)

**Files:**
- Create: `cli/Normalizador.kt`, `cli/Mensagens.kt`
- Create: `cli-test/MensagensTest.kt`

**Interfaces:**
- Produz: `Normalizador.normalizar`, `CodigoCli`, `Mensagens` (mensagens dos 22 `CodigoErro` + 3 `CodigoCli`, nomes de exibição de `CategoriaDto`/`DificuldadeDto`/`RespostaDto`, parsers normalizados).

- [ ] **Passo 1: escrever `MensagensTest.kt` (vermelho)**
  - `CA-14`: iterar `CodigoErro.entries` (22) e `CodigoCli.entries` (3); garantir mensagem não vazia e todas distintas entre si; conferir textos chave da tabela da spec.
  - Testes unitários para `Normalizador` e parsers (`" Gestão de projeto "`, `"gestao-projeto"`, `"MÉDIA"`, `"FÁCIL"`, `"Não"`, `"irrelevante"`, `"certa"`, `"errada"`, `"sim"`, `"n"`).
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :interfaces:test --tests "*MensagensTest"`
- [ ] **Passo 3: implementar `Normalizador.kt` e `Mensagens.kt`**
  - Usar `when` exaustivo (sem `else`) para todos os enums.
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :interfaces:test --tests "*MensagensTest"`
- [ ] **Passo 5: commit**
  - `git commit -m "S05: implementa Normalizador, parsers e tabela de Mensagens da CLI"`

---

### Tarefa 2: abertura, sessão de jogadores e tela inicial (CA-01 a CA-06)

**Files:**
- Create: `cli/CasosDeUsoCli.kt`, `cli/CliLocal.kt`
- Create: `boot/GeradorDeIdsUuid.kt`, `boot/SorteadorAleatorio.kt`, `boot/AplicacaoCli.kt`
- Create: `boot-test/RoteirosTest.kt` (CAs 01 a 06)

**Interfaces:**
- Produz: `CasosDeUsoCli`, `CliLocal` (fluxo de abertura e tela inicial), `AplicacaoCli.criar` (fábrica testável com `Clock`, `GeradorDeIds`, `Sorteador` injetáveis).

- [ ] **Passo 1: escrever `RoteirosTest.kt` para CA-01 a CA-06 (vermelho)**
  - `CA-01`: `N`, `Ana, Beto`, Enter, Enter → 4 prompts na ordem, título/categoria/dificuldade/enunciado/participantes, prompt `[Ana]> `, sem texto da solução.
  - `CA-02`: `debug`, `MÉDIA` → e1, saída contém `"Férias fatais"` e `"Média"`.
  - `CA-03`: `" Gestão de projeto "` + `"facil"` e `"gestao-projeto"` + `"FÁCIL"` → e3 nos dois.
  - `CA-04`: categoria `"redes"` → `ARGUMENTO_INVALIDO` listando as 5 categorias e abertura recomeça em `Narrador: `.
  - `CA-05`: narrador `Ana` + investigadores `ana, Beto` → `CONFIGURACAO_INVALIDA`; narrador Enter → `NOME_INVALIDO`; recomeça nas duas vezes.
  - `CA-06`: investigadores `Ana, , Beto,` → apenas `Ana` e `Beto`, nessa ordem.
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :bootstrap:test --tests "*RoteirosTest"`
- [ ] **Passo 3: implementar `CasosDeUsoCli`, `AplicacaoCli` e a abertura em `CliLocal`**
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :bootstrap:test --tests "*RoteirosTest"`
- [ ] **Passo 5: commit**
  - `git commit -m "S05: implementa abertura, registro de sessao e tela inicial da CLI"`

---

### Tarefa 3: loop de comandos, vez, julgamento, sigilo P3 e abandono (CA-07 a CA-11, CA-15)

**Files:**
- Modify: `cli/CliLocal.kt`
- Modify: `boot-test/RoteirosTest.kt` (+ CAs 07 a 11 e 15)

**Interfaces:**
- Completa todos os 12 comandos da tabela da spec (`vez`, `perguntar`, `responder`, `cancelar`, `propor`, `retirar`, `julgar`, `solucao`, `ver`, `abandonar`, `ajuda`, `sair`).

- [ ] **Passo 1: adicionar testes CA-07 a CA-11 e CA-15 em `RoteirosTest.kt` (vermelho)**
  - `CA-07`: `perguntar`, `vez n`, `responder nao`, `vez BETO`, `perguntar` → saídas exatas e prompts `[Ana]> `, `[N]> `, `[Beto]> `.
  - `CA-08`: `propor`, `julgar errada`, nova proposta, `julgar certa` → `Resolvido!`, `Solução: ...` e `Jogar outra? (s/n) `.
  - `CA-09`: `cancelar` sem pergunta (`SEM_PERGUNTA_ABERTA`), `perguntar` + `cancelar` (`Pergunta 1 cancelada.`), `retirar` sem proposta (`SEM_PROPOSTA_PENDENTE`).
  - `CA-10`: `solucao` por Ana (`NAO_E_NARRADOR`); `vez N`, `solucao` + `n` (aviso sem solução); `ver` (sem solução); `solucao` + `s` (aviso e solução). Garantir ausência da solução em todo `ver` antes do fim (P3).
  - `CA-11`: `abandonar` + `s` por Ana (`NAO_AUTORIZADO`); `vez N`, `abandonar` + `n` (nada muda); `abandonar` + `s` (`Partida abandonada por N.`, solução e `Jogar outra? (s/n) `).
  - `CA-15`: `xyz`, `vez Zé`, `vez`, `responder talvez`, `PERGUNTAR é rede?`, `ajuda` (lista os 12 comandos).
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :bootstrap:test --tests "*RoteirosTest"`
- [ ] **Passo 3: implementar o despacho dos comandos em `CliLocal.kt`**
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :bootstrap:test --tests "*RoteirosTest"`
- [ ] **Passo 5: commit**
  - `git commit -m "S05: implementa comandos da partida, sigilo P3 e ajuda na CLI"`

---

### Tarefa 4: `Jogar outra?`, EOF, `sair` e `Main.kt` no `bootstrap` (CA-12, CA-13)

**Files:**
- Modify: `cli/CliLocal.kt` (desfecho, abertura curta, linhas vazias, EOF, `sair`)
- Modify: `bootstrap/build.gradle.kts` (plugin `application`, `mainClass`, `UTF-8`, `standardInput`)
- Create: `boot/Main.kt`
- Modify: `boot-test/RoteirosTest.kt` (+ CA-12, CA-13)

**Interfaces:**
- Produz: ponto de entrada executável `.\gradlew.bat :bootstrap:run` em UTF-8.

- [ ] **Passo 1: adicionar CA-12 e CA-13 em `RoteirosTest.kt` (vermelho)**
  - `CA-12`: após resolver e1, responder `talvez` (repete `Jogar outra?`), `s`, `debug`, Enter → `SEM_ENIGMA_ELEGIVEL` e recomeça abertura curta em `Categoria`; depois Enter, Enter → novo enigma, mesmos participantes e prompt `[Ana]> `.
  - `CA-13`: linhas vazias/espaços durante a partida repetem o prompt; EOF na abertura, EOF na partida, EOF em `Jogar outra?` e `sair` encerram com código 0 sem escrever em stderr.
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :bootstrap:test --tests "*RoteirosTest"`
- [ ] **Passo 3: completar `CliLocal.kt`, configurar `bootstrap/build.gradle.kts` e criar `Main.kt`**
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :interfaces:test :bootstrap:test`
- [ ] **Passo 5: commit**
  - `git commit -m "S05: implementa abertura curta, tratamento de EOF e Main no bootstrap"`

---

### Tarefa 5: revisão final, cobertura, arquitetura, diagramas e marco M1

**Files:**
- Modify: `docs/diagramas/implantacao.md`, `pacotes.md`, `casos-de-uso.md`, `componentes.md`, `docs/diagramas/README.md`
- Modify: `docs/pendencias.md` (marcar `- [x] S05 implementada`)

- [ ] **Passo 1: rodar build completo, cobertura e Konsist**
  - `.\gradlew.bat check`
  - Conferir: 0 falhas em todos os módulos (`domain`, `application`, `infrastructure`, `interfaces`, `bootstrap`), `koverVerify` ≥ 90% em `com.quebugesse.interfaces.cli` (RNF2), Konsist confirmando RNF1 (`interfaces.cli` sem imports de `domain`/`infrastructure`).
- [ ] **Passo 2: teste rápido de fumaça da tarefa `:bootstrap:run`**
  - Executar uma partida curta redirecionando entrada UTF-8 com acentos para `.\gradlew.bat :bootstrap:run --console=plain -q` e verificar saída legível e exit code 0 (RNF3).
- [ ] **Passo 3: atualizar diagramas (P9)**
  - Refletir divergências D1 a D4 nas fontes Mermaid e atualizar `docs/diagramas/README.md`.
- [ ] **Passo 4: atualizar `docs/pendencias.md` (marco M1 concluído) e conferir LF por bytes**
- [ ] **Passo 5: commit**
  - `git commit -m "S05: atualiza diagramas e conclui marco M1"`
