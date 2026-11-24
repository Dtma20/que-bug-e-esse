# Tarefas: S03 Casos de uso da partida

> **Para agentes:** execute tarefa a tarefa em TDD (vermelho → verde → refatorar). Marque as caixas (`- [x]`) conforme concluir cada passo. Código em `Que-Bug-E-Esse/back` (ADR 0009 A).

**Objetivo:** implementar as emendas 1 e 2 da S01, as portas de `domain.gamesession.port`, `GameSessionRepositoryEmMemoria` (+ suíte de contrato em `testFixtures`), os DTOs e mapeadores de `application.dto` e os 12 casos de uso de `application.usecase`, com os 27 CAs da S03 e o CA-33 da S01 verdes.

**Arquitetura:** casos de uso retornam `Resultado<T>` (sem tipos do domínio na fronteira, A10); `MontarVisao` garante sigilo por papel (P3); `GameSessionRepositoryEmMemoria` persiste cópias reconstituídas com `synchronized`; `FakeNarrador` e `FakeVerificadorDeSolucao` viabilizam testes determinísticos de H4 sem rede/LLM (P6).

**Stack:** Kotlin/JVM (JDK 25 LTS), Gradle 9.8.2 (Kotlin DSL), Kotest (`FunSpec`, `assertions-core`, `withData`), `java-test-fixtures`, Konsist, Kover.

**Spec:** [spec.md](spec.md) · Plano técnico: [plan.md](plan.md) · Constituição: [docs/constitution.md](../../docs/constitution.md)

## Restrições globais

- P1: nada em `domain/src/main` usa `Instant.now(`, `Clock`, `UUID.randomUUID(` ou `Random`.
- P7: `domain.gamesession.port` não importa `domain.enigma` nem `domain.player`; `application/src/main` não importa `infrastructure`; entradas/saídas públicas de `application.usecase` e `application.dto` não expõem tipos de `domain` (A10).
- P3: `VisaoInvestigador.solucaoCompleta` é `null` antes de `RESOLVIDO`/`ABANDONADO`; `ContextoDaNarracao.toString()` não inclui `solucaoCompleta`.
- Todo teste de CA começa por `"CA-NN "`.
- Cobertura de linhas ≥ 90% em `com.quebugesse.application` e `com.quebugesse.infrastructure.persistence`.
- Testes: `.\gradlew.bat :domain:test`, `.\gradlew.bat :infrastructure:test`, `.\gradlew.bat :application:test`, `.\gradlew.bat check`.
- Commits em `Que-Bug-E-Esse/back`: `S03: <verbo no imperativo>`.

Convenções de caminho:
- `dom/` = `domain/src/main/kotlin/com/quebugesse/domain/`
- `dom-test/` = `domain/src/test/kotlin/com/quebugesse/domain/`
- `infra/` = `infrastructure/src/main/kotlin/com/quebugesse/infrastructure/persistence/`
- `contrato/` = `infrastructure/src/testFixtures/kotlin/com/quebugesse/infrastructure/persistence/contrato/`
- `infra-test/` = `infrastructure/src/test/kotlin/com/quebugesse/infrastructure/persistence/`
- `app/` = `application/src/main/kotlin/com/quebugesse/application/`
- `app-test/` = `application/src/test/kotlin/com/quebugesse/application/`

## Foco de revisão

1. **Ordem de verificação em `IniciarPartida`:** `JOGADOR_INEXISTENTE` → IA no lugar de humano (`CONFIGURACAO_INVALIDA`) → `SEM_ENIGMA_ELEGIVEL` → H5. Nenhuma falha antes do sorteio pode chamar `Sorteador` ou `salvar` (CA-04, CA-09).
2. **`enigmasJogadosPor`:** contar partidas em qualquer status em que algum investigador novo foi investigador; ignorar partidas em que ele apenas narrou (CA-08, CA-25).
3. **Falha da IA (H4):** exceção em `Narrador` ou `VerificadorDeSolucao` devolve `IA_INDISPONIVEL`, não chama `salvar` e mantém pergunta aberta / tentativa pendente (CA-18, CA-20).
4. **Sigilo P3:** `VisaoInvestigador` não contém `solucaoCompleta` antes de estado terminal nem em `toString()` (CA-21, CA-22, CA-23).
5. **Reconstituição íntegra:** `GameSession.reconstituir` não gera eventos pendentes e rejeita violações de I1–I8 (CA-26).

---

### Tarefa 1: emendas 1 e 2 à S01 (`aplicarVeredito` com autor e `GameSession.reconstituir`)

**Files:**
- Modify: `dom/gamesession/GameSession.kt`
- Modify: `dom-test/gamesession/PropostasTest.kt` (e demais testes de S01 que chamam `aplicarVeredito`)
- Create: `dom-test/gamesession/ReconstituirTest.kt`

**Interfaces:**
- Modifica: `GameSession.aplicarVeredito(autor: PlayerRef, veredito: Veredito, agora: Instant)` (guarda `NAO_E_NARRADOR` no passo 3, antes de `SEM_PROPOSTA_PENDENTE`).
- Produz: `GameSession.reconstituir(...)` validando I1–I8 e sem eventos pendentes.

- [ ] **Passo 1: escrever os testes (vermelho)**
  - Adicionar `CA-33` em `PropostasTest.kt`: tentativa pendente; Ana chama `aplicarVeredito(ANA, Veredito(true), t4)` → `deveRejeitar(NAO_E_NARRADOR)`. Garantir precedência de `NAO_E_NARRADOR` antes de `SEM_PROPOSTA_PENDENTE`.
  - Criar `ReconstituirTest.kt` com `CA-26`: reconstituir partida com perguntas, tentativas e eventos pendentes → fotografia idêntica, `eventosPendentes` vazio, aceita próximo comando; reconstituir com duas perguntas abertas, com `resolvidaEm` não nulo fora de `RESOLVIDO`, ou com pergunta aberta e proposta pendente simultâneas → lança `PartidaException`.
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :domain:test --tests "com.quebugesse.domain.gamesession.*"`
- [ ] **Passo 3: implementar**
  - Atualizar `aplicarVeredito` com `exigir(autor == narrador, Violacao.NAO_E_NARRADOR)` antes de `propostaPendente != null`.
  - Implementar `GameSession.reconstituir(...)` copiando listas, validando I1–I8 e iniciando com `eventosPendentes` vazio.
  - Atualizar chamadas de `aplicarVeredito` nos testes existentes da S01 (incluindo `InvariantesPropertyTest`).
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :domain:test`
- [ ] **Passo 5: commit**
  - `git commit -m "S03: aplica emendas 1 e 2 em GameSession (autor no veredito e reconstituir)"`

---

### Tarefa 2: portas da partida e `GameSessionRepositoryEmMemoria` com contrato

**Files:**
- Create: `dom/gamesession/port/GameSessionRepository.kt`, `dom/gamesession/port/Narrador.kt`, `dom/gamesession/port/VerificadorDeSolucao.kt`
- Create: `infra/memoria/GameSessionRepositoryEmMemoria.kt`
- Create: `contrato/GameSessionRepositoryContrato.kt`
- Create: `infra-test/memoria/GameSessionRepositoryEmMemoriaTest.kt`

**Interfaces:**
- Produz: `GameSessionRepository`, `Narrador`, `ContextoDaNarracao` (`toString()` sem `solucaoCompleta`), `PerguntaRespondidaResumo`, `VerificadorDeSolucao`, `GameSessionRepositoryEmMemoria`, `GameSessionRepositoryContrato`.

- [ ] **Passo 1: escrever as portas e a suíte de contrato (vermelho)**
  - Criar as três interfaces em `dom/gamesession/port/`.
  - Criar `GameSessionRepositoryContrato` (`abstract class : FunSpec`) cobrindo `CA-25`: buscar inexistente → `null`; salvar e substituir; mutação externa antes/depois não altera o guardado (cópia via `reconstituir`, eventos pendentes não guardados); `enigmasJogadosPor` em partidas `EM_ANDAMENTO`, `RESOLVIDO`, `ABANDONADO`, `AGUARDANDO_INICIO`, ignorando partidas onde o jogador foi apenas narrador.
  - Criar `GameSessionRepositoryEmMemoriaTest` estendendo o contrato.
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :infrastructure:test --tests "*GameSessionRepository*"`
- [ ] **Passo 3: implementar `GameSessionRepositoryEmMemoria`**
  - `LinkedHashMap<GameSessionId, GameSession>` com `synchronized`, reconstituindo na entrada e na saída.
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :domain:test :infrastructure:test`
- [ ] **Passo 5: commit**
  - `git commit -m "S03: adiciona portas da partida e GameSessionRepositoryEmMemoria"`

---

### Tarefa 3: DTOs, `Resultado`, `CodigoErro` e mapeadores

**Files:**
- Modify: `application/build.gradle.kts` (adicionar `testImplementation(project(":infrastructure"))` e `testImplementation(testFixtures(project(":infrastructure")))` se necessário)
- Create: `app/port/GeradorDeIds.kt`, `app/port/Sorteador.kt`
- Create: `app/dto/Resultado.kt`, `app/dto/EnumsDto.kt`, `app/dto/CodigoErro.kt`, `app/dto/VisoesDto.kt`, `app/dto/Mapeadores.kt`
- Create: `app-test/dto/MapeamentoTest.kt`

**Interfaces:**
- Produz: todos os DTOs públicos de `application.dto`, `CodigoErro` (21 valores), `GeradorDeIds`, `Sorteador` e funções internas de mapeamento.

- [ ] **Passo 1: escrever `MapeamentoTest.kt` (vermelho)**
  - Cobrir `CA-24`: percorrer `entries` de `Violacao`, `Answer`, `GameStatus`, `ModoDeJogo`, `FaseDaInvestigacao`, `SituacaoTentativa`, `Categoria`, `Dificuldade` e verificar mapeamento 1:1 com o mesmo `name`; verificar que toda `Violacao` mapeia para o `CodigoErro` de mesmo `name`.
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :application:test --tests "com.quebugesse.application.dto.MapeamentoTest"`
- [ ] **Passo 3: implementar DTOs, portas auxiliares e `Mapeadores.kt`**
  - Garantir `when` exaustivo em todos os mapeadores e nenhum tipo de `domain` nas assinaturas públicas de `application.dto`.
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :application:test --tests "com.quebugesse.application.dto.MapeamentoTest"`
- [ ] **Passo 5: commit**
  - `git commit -m "S03: adiciona DTOs, CodigoErro e mapeadores da aplicacao"`

---

### Tarefa 4: `MontarVisao`, `RegistrarJogador` e `IniciarPartida`

**Files:**
- Create: `app/usecase/MontarVisao.kt`, `app/usecase/RegistrarJogador.kt`, `app/usecase/IniciarPartida.kt`
- Create: `app-test/FixturesAplicacao.kt`, `app-test/usecase/RegistrarJogadorTest.kt`, `app-test/usecase/IniciarPartidaTest.kt`

**Interfaces:**
- Produz: `MontarVisao` (internal), `RegistrarJogador`, `IniciarPartida` e fixtures reutilizáveis (`CenarioAplicacao`, `RepositorioPartidaEspiao`, `GeradorDeIdsSequencial`, `SorteadorGravador`, `FakeNarrador`, `FakeVerificadorDeSolucao`).

- [ ] **Passo 1: escrever fixtures e testes (vermelho)**
  - `RegistrarJogadorTest.kt`: `CA-27` (`"  Ana  "` salva como `"Ana"` com ID do gerador; `""` → `NOME_INVALIDO`).
  - `IniciarPartidaTest.kt`: `CA-01` a `CA-09` (grupo devolve `VisaoNarrador` com solução; solo devolve `VisaoInvestigador` sem solução com narrador IA; solo com investigador extra → `CONFIGURACAO_INVALIDA`; jogador inexistente → `JOGADOR_INEXISTENTE` sem chamar sorteador nem salvar; IA como humano → `CONFIGURACAO_INVALIDA`; H5 duplicado/narrador-investigador → `CONFIGURACAO_INVALIDA`; filtro por categoria/dificuldade; exclusão de já jogados ignorando narrador; sem elegível → `SEM_ENIGMA_ELEGIVEL`).
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :application:test --tests "*RegistrarJogadorTest" --tests "*IniciarPartidaTest"`
- [ ] **Passo 3: implementar `MontarVisao`, `RegistrarJogador` e `IniciarPartida`**
  - Ler `clock.instant()` uma única vez por execução (A4); respeitar ordem estrita de verificação (seção "Ordem de verificação" da spec).
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :application:test --tests "*RegistrarJogadorTest" --tests "*IniciarPartidaTest"`
- [ ] **Passo 5: commit**
  - `git commit -m "S03: implementa MontarVisao, RegistrarJogador e IniciarPartida"`

---

### Tarefa 5: comandos humanos da partida e `JulgarProposta`

**Files:**
- Create: `app/usecase/FazerPergunta.kt`, `app/usecase/ResponderPergunta.kt`, `app/usecase/CancelarPergunta.kt`, `app/usecase/ProporSolucao.kt`, `app/usecase/RetirarProposta.kt`, `app/usecase/JulgarProposta.kt`, `app/usecase/Abandonar.kt`
- Create: `app-test/usecase/ComandosDaPartidaTest.kt`, `app-test/usecase/JulgarPropostaTest.kt`

**Interfaces:**
- Produz: os 7 casos de uso de comando humano da partida.

- [ ] **Passo 1: escrever os testes (vermelho)**
  - `ComandosDaPartidaTest.kt`: `CA-10` (`FazerPergunta` salva em `t2` e retorna `VisaoInvestigador`), `CA-11` (`PARTIDA_INEXISTENTE` em todos os 10 casos de uso que recebem `partidaId` — completar com `ResponderComIA`, `AvaliarComIA` e `ConsultarPartida` nas tarefas 6–7), `CA-12` (rejeição não chama `salvar` nem altera estado guardado), `CA-13` (`ResponderPergunta`, `CancelarPergunta`, `ProporSolucao`, `RetirarProposta`, `Abandonar` salvando estado correto e devolvendo visão do autor).
  - `JulgarPropostaTest.kt`: `CA-14` (N julga certa → `RESOLVIDO` e visão de Ana ganha solução; errada → `INCORRETA`, `AGUARDANDO_PERGUNTA`), `CA-15` (Ana julga no grupo, Ana responde no solo, Ana julga no solo → `NAO_E_NARRADOR`).
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :application:test --tests "*ComandosDaPartidaTest" --tests "*JulgarPropostaTest"`
- [ ] **Passo 3: implementar os 7 casos de uso**
  - Traduzir `PartidaException` via `toCodigoErro()` sem chamar `salvar` em caso de rejeição.
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :application:test --tests "*ComandosDaPartidaTest" --tests "*JulgarPropostaTest"`
- [ ] **Passo 5: commit**
  - `git commit -m "S03: implementa casos de uso de comando e julgamento humano"`

---

### Tarefa 6: casos de uso da IA no solo (`ResponderComIA` e `AvaliarComIA`)

**Files:**
- Create: `app/usecase/ResponderComIA.kt`, `app/usecase/AvaliarComIA.kt`
- Create: `app-test/usecase/ResponderComIATest.kt`, `app-test/usecase/AvaliarComIATest.kt`

**Interfaces:**
- Produz: `ResponderComIA` e `AvaliarComIA`, chamando `Narrador` e `VerificadorDeSolucao` só no `SOLO`.

- [ ] **Passo 1: escrever os testes (vermelho)**
  - `ResponderComIATest.kt`: `CA-16` (solo com pergunta 1 respondida `NAO` e pergunta 2 aberta → fake recebe texto da pergunta 2 + contexto com enunciado, solução e histórico `[p1=NAO]` e salva resposta `SIM`), `CA-17` (`MODO_INCOMPATIVEL`, `SEM_PERGUNTA_ABERTA`, `PARTIDA_ENCERRADA`, `PARTIDA_INEXISTENTE` sem chamar a porta), `CA-18` (exceção no fake → `IA_INDISPONIVEL`, nada salvo, pergunta segue aberta; retentativa seguinte responde `NAO`, H4).
  - `AvaliarComIATest.kt`: `CA-19` (correta → `RESOLVIDO`, incorreta → `INCORRETA`, verificador recebe texto da proposta e solução completa), `CA-20` (`MODO_INCOMPATIVEL`, `SEM_PROPOSTA_PENDENTE`, `PARTIDA_ENCERRADA`, `PARTIDA_INEXISTENTE` sem chamar porta; exceção no fake → `IA_INDISPONIVEL`, tentativa segue pendente e nada salvo).
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :application:test --tests "*ResponderComIATest" --tests "*AvaliarComIATest"`
- [ ] **Passo 3: implementar `ResponderComIA` e `AvaliarComIA`**
  - Validar pré-condições na ordem estrita (`PARTIDA_INEXISTENTE` → `MODO_INCOMPATIVEL` → `PARTIDA_ENCERRADA` → `SEM_PERGUNTA_ABERTA`/`SEM_PROPOSTA_PENDENTE`) antes de invocar a porta; capturar exceção da porta retornando `IA_INDISPONIVEL` sem salvar.
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :application:test --tests "*ResponderComIATest" --tests "*AvaliarComIATest"`
- [ ] **Passo 5: commit**
  - `git commit -m "S03: implementa ResponderComIA e AvaliarComIA com protecao H4"`

---

### Tarefa 7: `ConsultarPartida` e garantia de sigilo P3

**Files:**
- Create: `app/usecase/ConsultarPartida.kt`
- Create: `app-test/usecase/ConsultarPartidaTest.kt`
- Modify: `app-test/usecase/ComandosDaPartidaTest.kt` (garantir todos os 10 casos de uso no `CA-11`)

**Interfaces:**
- Produz: `ConsultarPartida`.

- [ ] **Passo 1: escrever `ConsultarPartidaTest.kt` (vermelho)**
  - `CA-21`: em andamento, N recebe `VisaoNarrador` com solução, Ana recebe `VisaoInvestigador` com `solucaoCompleta == null`, Zé recebe `NAO_PARTICIPA`.
  - `CA-22`: em `RESOLVIDO` e em `ABANDONADO`, Ana recebe `VisaoInvestigador` com a solução.
  - `CA-23`: roteiro completo no grupo e no solo passando por todas as fases não terminais; verificar que `visao.toString()` de nenhuma `VisaoInvestigador` contém o texto da solução (P3).
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :application:test --tests "*ConsultarPartidaTest"`
- [ ] **Passo 3: implementar `ConsultarPartida`**
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :application:test`
- [ ] **Passo 5: commit**
  - `git commit -m "S03: implementa ConsultarPartida e testes de sigilo P3"`

---

### Tarefa 8: revisão final, cobertura, arquitetura e diagramas

**Files:**
- Modify: `docs/diagramas/componentes.md`, `classes-dominio.md`, `estados-partida.md`, `pacotes.md`, `sequencia-iniciar-partida.md`, `sequencia-investigar-e-resolver.md`, `docs/diagramas/README.md` (no repositório de docs)
- Modify: `docs/pendencias.md` (marcar `- [x] S03 implementada`)

- [ ] **Passo 1: rodar build completo, cobertura e Konsist**
  - `.\gradlew.bat check`
  - Conferir: 0 falhas, `koverVerify` ≥ 90% em `com.quebugesse.application` e `com.quebugesse.infrastructure.persistence`, testes de arquitetura P1/P7 verdes, suíte de `application` abaixo de 10 s (RNF3).
- [ ] **Passo 2: atualizar diagramas (P9)**
  - Refletir divergências D1 a D10 nas fontes Mermaid e atualizar `docs/diagramas/README.md`.
- [ ] **Passo 3: atualizar `docs/pendencias.md` e conferir LF por bytes**
  - Verificar zero CRLF em todos os arquivos tocados e commitar.
- [ ] **Passo 4: commit**
  - `git commit -m "S03: atualiza diagramas e fecha definicao de pronto"`
