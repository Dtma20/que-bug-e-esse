# Tarefas: S04 Eventos depois de persistir

> **Para agentes:** execute tarefa a tarefa em TDD (vermelho → verde → refatorar). Marque as caixas (`- [x]`) conforme concluir cada passo. Código em `Que-Bug-E-Esse/back` (ADR 0009 A).

**Objetivo:** implementar `PublicadorDeEventos`, `PublicadorDeEventosEmMemoria` (síncrono, FIFO por thread, isolando falhas), as emendas 1 e 2 nos casos de uso da S03 (`salvar` → `publicar` → reler visão, + `FALHA_AO_SALVAR`) e os ouvintes `AoFazerPerguntaNoSolo` e `AoProporSolucaoNoSolo`, com os 12 CAs verdes.

**Arquitetura:** porta em `domain.shared.port`, adaptador sem threads extras em `infrastructure.messaging` com `CopyOnWriteArrayList` + `ThreadLocal<ArrayDeque>`, auxiliar interno de persistência/publicação em `application.usecase`, ouvintes do solo em `application.ouvinte`.

**Stack:** Kotlin/JVM (JDK 25 LTS), Gradle 9.8.2 (Kotlin DSL), Kotest (`FunSpec`, `assertions-core`), Konsist, Kover.

**Spec:** [spec.md](spec.md) · Plano técnico: [plan.md](plan.md) · Constituição: [docs/constitution.md](../../docs/constitution.md)

## Restrições globais

- P1: `domain.shared.port.PublicadorDeEventos` só usa `DomainEvent` e `List`.
- P7: `infrastructure.messaging` só importa `domain`; `application.ouvinte` não importa `infrastructure` em `src/main`.
- RNF1: `PublicadorDeEventosEmMemoria` não cria threads, não usa temporizador e limpa o `ThreadLocal` ao esvaziar a fila.
- Todo teste de CA começa por `"CA-NN "`.
- Cobertura de linhas ≥ 90% em `com.quebugesse.application.ouvinte` e `com.quebugesse.infrastructure.messaging`.
- Testes: `.\gradlew.bat :infrastructure:test`, `.\gradlew.bat :application:test`, `.\gradlew.bat check`.
- Commits em `Que-Bug-E-Esse/back`: `S04: <verbo no imperativo>`.

Convenções de caminho:
- `dom/` = `domain/src/main/kotlin/com/quebugesse/domain/`
- `infra/` = `infrastructure/src/main/kotlin/com/quebugesse/infrastructure/`
- `infra-test/` = `infrastructure/src/test/kotlin/com/quebugesse/infrastructure/`
- `app/` = `application/src/main/kotlin/com/quebugesse/application/`
- `app-test/` = `application/src/test/kotlin/com/quebugesse/application/`

## Foco de revisão

1. **Ordem H6:** `salvar` sempre termina antes de `publicar`; se `salvar` falhar, `publicar` nunca é chamado (CA-01, CA-03).
2. **Fila sem recursão:** 10.000 republicações encadeadas devem passar sem `StackOverflowError` e o `publicar` interno deve retornar antes de entregar o evento recém-enfileirado (CA-07).
3. **Isolamento entre threads:** duas threads publicando 100 eventos simultâneos não misturam entregas (CA-08).
4. **Releitura (EV5):** no solo, quem chama `FazerPergunta` ou `ProporSolucao` já recebe a visão com a resposta ou o veredito da IA aplicado (CA-09, CA-12).
5. **Falha da IA (OS3, H4):** `IA_INDISPONIVEL` no ouvinte vai para o registro de falhas, não muda a visão de retorno do jogador e mantém a pergunta aberta / proposta pendente (CA-11).

---

### Tarefa 1: porta `PublicadorDeEventos` e `PublicadorDeEventosEmMemoria`

**Files:**
- Create: `dom/shared/port/PublicadorDeEventos.kt`
- Create: `infra/messaging/PublicadorDeEventosEmMemoria.kt`
- Create: `infra-test/messaging/PublicadorDeEventosEmMemoriaTest.kt`

**Interfaces:**
- Produz: `PublicadorDeEventos`, `OuvinteDeEvento`, `RegistroDeFalhaDeEvento`, `PublicadorDeEventosEmMemoria`.

- [ ] **Passo 1: criar a porta e escrever `PublicadorDeEventosEmMemoriaTest.kt` (vermelho)**
  - `CA-05`: ouvintes O1, O2 (lança exceção), O3 → O1 e O3 recebem, registro de falhas anota `(evento, O2, excecao)`, `publicar` não lança.
  - `CA-06`: ouvintes A, B, C com `publicar([e1, e2])` → ordem exata `A(e1), B(e1), C(e1), A(e2), B(e2), C(e2)`.
  - `CA-07`: ouvinte A publica `[e3]` ao receber `e1` → ordem `A(e1), B(e1), A(e2), B(e2), A(e3), B(e3)`, retorno imediato do `publicar` interno; teste adicional de cadeia com 10.000 republicações sem estouro de pilha.
  - `CA-08`: duas threads concorrentes publicando 100 eventos distintos cada → cada evento entregue na própria thread, sem mistura.
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :infrastructure:test --tests "*PublicadorDeEventosEmMemoriaTest"`
- [ ] **Passo 3: implementar `PublicadorDeEventosEmMemoria`**
  - Usar `CopyOnWriteArrayList`, `ThreadLocal<EstadoDaThread>` com `ArrayDeque` e `remove()` no `finally`.
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :infrastructure:test --tests "*PublicadorDeEventosEmMemoriaTest"`
- [ ] **Passo 5: commit**
  - `git commit -m "S04: implementa PublicadorDeEventos e PublicadorDeEventosEmMemoria"`

---

### Tarefa 2: `FALHA_AO_SALVAR` e emendas 1 e 2 nos casos de uso da S03

**Files:**
- Modify: `app/dto/CodigoErro.kt` (+ `FALHA_AO_SALVAR`)
- Create: `app/usecase/PersistirEPublicar.kt`
- Modify: os 10 casos de uso de comando em `app/usecase/` + `RegistrarJogador.kt`
- Modify: `app-test/FixturesAplicacao.kt` (+ `PublicadorEspiao` e falha programável em `salvar`)
- Create: `app-test/usecase/PublicacaoAposSalvarTest.kt`

**Interfaces:**
- Modifica: `CodigoErro` (22 valores); os 10 casos de uso de comando recebem `PublicadorDeEventos` e seguem `salvar` → `puxarEventos()` → `publicar` → reler visão.

- [ ] **Passo 1: escrever `PublicacaoAposSalvarTest.kt` (vermelho)**
  - `CA-01`: linha do tempo compartilhada prova `salvar` antes da entrega de `PerguntaFeita`; ouvinte lê do repositório e vê pergunta 1 aberta.
  - `CA-02`: cada um dos 9 comandos humanos (`IniciarPartida`, `FazerPergunta`, `ResponderPergunta`, `CancelarPergunta`, `ProporSolucao`, `RetirarProposta`, `JulgarProposta` certa/errada, `Abandonar`) chama `publicar` uma vez com o evento exato; `ConsultarPartida` e `RegistrarJogador` não chamam `publicar`.
  - `CA-03`: falha em `salvar` em `IniciarPartida`, `FazerPergunta` e `RegistrarJogador` → `FALHA_AO_SALVAR` e zero chamadas a `publicar`.
  - `CA-04`: comandos rejeitados (`PERGUNTA_EM_ABERTO`, `PARTIDA_INEXISTENTE`, `NAO_E_NARRADOR`) não chamam `publicar`.
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :application:test --tests "*PublicacaoAposSalvarTest"`
- [ ] **Passo 3: implementar `FALHA_AO_SALVAR`, `PersistirEPublicar` e atualizar os casos de uso e fixtures**
  - Garantir que `MapeamentoTest` e todos os testes da S03 continuem verdes.
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :application:test`
- [ ] **Passo 5: commit**
  - `git commit -m "S04: publica eventos apos persistir e rele visao nos casos de uso"`

---

### Tarefa 3: ouvintes do modo solo (`AoFazerPerguntaNoSolo` e `AoProporSolucaoNoSolo`)

**Files:**
- Create: `app/ouvinte/FalhaOuvinteSoloException.kt`, `app/ouvinte/AoFazerPerguntaNoSolo.kt`, `app/ouvinte/AoProporSolucaoNoSolo.kt`
- Create: `app-test/ouvinte/OuvintesDoSoloTest.kt`

**Interfaces:**
- Produz: `FalhaOuvinteSoloException`, `AoFazerPerguntaNoSolo`, `AoProporSolucaoNoSolo`.

- [ ] **Passo 1: escrever `OuvintesDoSoloTest.kt` (vermelho)**
  - `CA-09`: solo em andamento, ouvintes registrados no `PublicadorDeEventosEmMemoria`, `FakeNarrador` responde `SIM` → Ana faz pergunta → publicados `[PerguntaFeita]` e `[PerguntaRespondida]`; partida salva e visão devolvida a Ana já têm pergunta 1 com `SIM`.
  - `CA-10`: modo grupo com ouvintes do solo registrados → Ana pergunta, N responde, Ana propõe → `FakeNarrador` e `FakeVerificadorDeSolucao` nunca chamados; pergunta e proposta aguardam N.
  - `CA-11`: solo com `FakeNarrador` lançando exceção → Ana faz pergunta → visão devolvida tem pergunta 1 aberta; publicado só `[PerguntaFeita]`; registro de falhas tem entrada com `FalhaOuvinteSoloException(IA_INDISPONIVEL)`; partida salva mantém pergunta aberta (H4).
  - `CA-12`: solo com `FakeVerificadorDeSolucao` dizendo correta → `[SolucaoProposta]` e `[PartidaResolvida]`, visão `RESOLVIDO` com solução; em outra partida dizendo incorreta → `[SolucaoProposta]` e `[SolucaoRecusada]`, tentativa `INCORRETA`, fase `AGUARDANDO_PERGUNTA`.
- [ ] **Passo 2: rodar e ver falhar**
  - `.\gradlew.bat :application:test --tests "*OuvintesDoSoloTest"`
- [ ] **Passo 3: implementar os dois ouvintes e `FalhaOuvinteSoloException`**
- [ ] **Passo 4: rodar e ver passar**
  - `.\gradlew.bat :application:test`
- [ ] **Passo 5: commit**
  - `git commit -m "S04: implementa ouvintes do solo para resposta e avaliacao automatica"`

---

### Tarefa 4: revisão final, cobertura, arquitetura e diagramas

**Files:**
- Modify: `docs/diagramas/componentes.md`, `estados-partida.md`, `pacotes.md`, `sequencia-iniciar-partida.md`, `sequencia-investigar-e-resolver.md`, `implantacao.md`, `docs/diagramas/README.md`
- Modify: `docs/pendencias.md` (marcar `- [x] S04 implementada`)

- [ ] **Passo 1: rodar build completo, cobertura e Konsist**
  - `.\gradlew.bat check`
  - Conferir: 0 falhas, `koverVerify` ≥ 90% em `com.quebugesse.application.ouvinte` e `com.quebugesse.infrastructure.messaging`, Konsist verde, suítes abaixo de 10 s (RNF4).
- [ ] **Passo 2: atualizar diagramas (P9)**
  - Refletir D1 a D7 (incluindo H6 no diagrama de estados e `application.ouvinte` no de pacotes).
- [ ] **Passo 3: atualizar `docs/pendencias.md` e conferir LF por bytes**
- [ ] **Passo 4: commit**
  - `git commit -m "S04: atualiza diagramas e fecha definicao de pronto"`
