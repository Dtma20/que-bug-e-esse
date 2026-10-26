# Tarefas: S01 Domínio da partida

> **Para agentes:** SUB-SKILL OBRIGATÓRIA: `superpowers:subagent-driven-development` ou `superpowers:executing-plans`, tarefa a tarefa. Os passos usam caixas de seleção (`- [ ]`) para acompanhamento.

**Objetivo:** implementar o agregado `GameSession` da spec S01, com os 32 CAs como testes Kotest verdes.

**Arquitetura:** classe Kotlin com estado privado e um método por comando. Cada método verifica as guardas na ordem da spec, depois muda o estado e acumula um evento (DP2). Perguntas e tentativas são `data class` imutáveis trocadas na lista; fase, pergunta aberta e proposta pendente são derivadas (DP6).

**Stack:** Kotlin/JVM, Gradle com Kotlin DSL e `gradle/libs.versions.toml`, Kotest (runner JUnit 5, `assertions-core`, `property`, `withData`), Kover.

**Spec:** [spec.md](spec.md) · Plano técnico: [plan.md](plan.md) (seções "Modelo" e "Estrutura de arquivos") · Constituição: [docs/constitution.md](../../docs/constitution.md)

## Restrições globais

- P1: nada em `domain/src/main` usa `Instant.now(`, `Clock`, `UUID.randomUUID(` ou `Random`; todo comando recebe `agora: Instant`.
- Dependências de produção: só a biblioteca padrão do Kotlin, `java.time` e `java.util.UUID`.
- Pacotes `com.quebugesse.domain.shared` e `com.quebugesse.domain.gamesession`; `gamesession` só importa `shared` (P7).
- Nomes, tipos e assinaturas: exatamente os do bloco "Modelo" do plan.md.
- `status` e `resolvidaEm` com `private set`; `perguntas`, `tentativas` e `eventosPendentes` devolvem `toList()` da lista interna (DP3).
- Todas as guardas antes da primeira mutação, na "Ordem de verificação" da spec (DP2). Rejeição: `throw PartidaException(violacao)`.
- Texto (R4, DP8): `trim()`, depois `codePointCount` de 1 a 500; guarda-se o texto aparado.
- `QuestionId(perguntas.size + 1)` e `TentativaId(tentativas.size + 1)`: numerações separadas, a partir de 1 (DP4).
- Todo teste de CA tem nome começando por `"CA-NN "`; todo teste de rejeição usa `deveRejeitar` (CA-27).
- Cobertura de linhas ≥ 90% em `com.quebugesse.domain.gamesession` (RNF2); suíte do `domain` em menos de 10 s (RNF3).
- Versões de Kotlin, JDK (LTS), Kotest e Kover ficam em `gradle/libs.versions.toml`; este plano não fixa números.
- Testes: `./gradlew :domain:test --tests "com.quebugesse.domain.gamesession.<Classe>"` (no PowerShell, `.\gradlew.bat`).
- Commits: `git add` só com os caminhos da tarefa; mensagem no formato da constituição, `S01: <verbo no presente>`.

Convenções deste documento:
- Nos blocos **Files**, `main/` abrevia `domain/src/main/kotlin/com/quebugesse/domain/` e `test/` abrevia `domain/src/test/kotlin/com/quebugesse/domain/`. Os comandos `git add` usam o caminho completo.
- Os testes são `FunSpec` no pacote `com.quebugesse.domain.gamesession`. Os trechos omitem `package` e imports, inclusive os imports estáticos das constantes de enum (`SIM`, `GRUPO`, `PARTIDA_ENCERRADA`...).

## Foco de revisão

1. Texto com emoji: `"🐛".repeat(500)` tem 1000 unidades UTF-16 e 500 code points; deve ser aceito, e `"🐛".repeat(501)` rejeitado. Contar com `length` quebraria isso. Teste na tarefa 2.
2. Lista de investigadores alterada por quem chamou `criar`: a partida não muda junto. Teste na tarefa 1.
3. Veredito duplicado: o segundo `aplicarVeredito` dá `SEM_PROPOSTA_PENDENTE` depois de um incorreto e `PARTIDA_ENCERRADA` depois de um correto. Teste na tarefa 4.
4. Responder pergunta cancelada: `PERGUNTA_JA_FECHADA`, nunca resposta aceita nem `PERGUNTA_INEXISTENTE`. Teste na tarefa 3.
5. Nova proposta depois de veredito incorreto: vira a tentativa 2, `PENDENTE`, e a tentativa 1 continua `INCORRETA`. Teste na tarefa 4.

---

### Tarefa 1: tipos do domínio, criação e início da partida

**Files:**
- Create (só se a S00 não existir): `settings.gradle.kts`, `gradle/libs.versions.toml`, `domain/build.gradle.kts`, `.gitignore`, wrapper do Gradle
- Create: `main/shared/Ids.kt`, `main/shared/PlayerRef.kt`, `main/shared/DomainEvent.kt`, `main/gamesession/Enums.kt`, `main/gamesession/Veredito.kt`, `main/gamesession/Violacao.kt`, `main/gamesession/Question.kt`, `main/gamesession/Tentativa.kt`, `main/gamesession/EventosDePartida.kt`, `main/gamesession/GameSession.kt`
- Test: `test/gamesession/Fixtures.kt`, `test/gamesession/Fotografia.kt`, `test/gamesession/IniciarPartidaTest.kt`

**Interfaces:**
- Consome: nada.
- Produz: todos os tipos do "Modelo" do plan.md. Em `GameSession`: `criar`, `status`, `resolvidaEm`, `perguntas`, `tentativas`, `perguntaAberta`, `propostaPendente`, `fase`, `eventosPendentes`, `puxarEventos()`, `iniciar(agora: Instant)` e os auxiliares privados `exigir(condicao: Boolean, violacao: Violacao)` e `exigirNaoEncerrada()`. Nos testes: `T0`, `t(n: Long)`, `N`, `ANA`, `BETO`, `ZE`, `novaPartida(...)`, `emAndamento(modo)`, `Fotografia`, `GameSession.fotografia()`, `GameSession.deveRejeitar(esperada, comando)`.

- [ ] **Passo 1: garantir o build mínimo (só se a S00 ainda não existir)**

Na pasta que contém `docs/` e `specs/`:
- Se não for repositório: `git init`, `.gitignore` com `.gradle/` e `build/`, e um primeiro commit só com a documentação (`git add docs specs planejamento-sdd.md diagramas .gitignore`, mensagem `S01: adiciona constituição, spec e plano`).
- `settings.gradle.kts`: `rootProject.name = "que-bug-e-esse"` e `include("domain")`.
- `gradle/libs.versions.toml`: versões estáveis atuais de Kotlin, Kotest e Kover; bibliotecas `kotest-runner-junit5`, `kotest-assertions-core`, `kotest-property` e, se a versão do Kotest ainda publicar o `withData` em módulo separado, `kotest-framework-datatest`; plugins `kotlin-jvm` e `kover`.
- `domain/build.gradle.kts`: plugins `kotlin-jvm` e `kover`; `kotlin { jvmToolchain(<JDK LTS>) }`; Kotest só em `testImplementation`; nenhuma dependência em `implementation`; `tasks.test { useJUnitPlatform() }`.
- Wrapper: `gradle wrapper` (exige o Gradle instalado uma vez).

Run: `./gradlew :domain:test`
Expected: `BUILD SUCCESSFUL` (ainda sem testes).

- [ ] **Passo 2: escrever as fixtures, a fotografia e os testes de início**

`Fixtures.kt`:

```kotlin
val T0: Instant = Instant.parse("2026-01-01T10:00:00Z")
fun t(n: Long): Instant = T0.plusSeconds(60 * n)

val N = PlayerRef(PlayerId(UUID(0, 1)))
val ANA = PlayerRef(PlayerId(UUID(0, 2)))
val BETO = PlayerRef(PlayerId(UUID(0, 3)))
val ZE = PlayerRef(PlayerId(UUID(0, 4)))   // não participa

fun novaPartida(
    modo: ModoDeJogo = GRUPO,
    narrador: PlayerRef = N,
    investigadores: List<PlayerRef> = if (modo == SOLO) listOf(ANA) else listOf(ANA, BETO),
) = GameSession.criar(GameSessionId(UUID(1, 1)), EnigmaId(UUID(2, 1)), modo, narrador, investigadores, T0)

fun emAndamento(modo: ModoDeJogo = GRUPO) = novaPartida(modo).apply { iniciar(t(1)); puxarEventos() }
```

`Fotografia.kt`:

```kotlin
data class Fotografia(
    val status: GameStatus, val fase: FaseDaInvestigacao?, val resolvidaEm: Instant?,
    val perguntas: List<Question>, val tentativas: List<Tentativa>, val eventos: List<EventoDePartida>,
)

fun GameSession.fotografia() = Fotografia(status, fase, resolvidaEm, perguntas, tentativas, eventosPendentes)

/** Confere a violação e R3 (CA-27): nada muda, nem os eventos acumulados. */
fun GameSession.deveRejeitar(esperada: Violacao, comando: GameSession.() -> Unit) {
    val antes = fotografia()
    shouldThrow<PartidaException> { this.comando() }.violacao shouldBe esperada
    fotografia() shouldBe antes
}
```

`IniciarPartidaTest.kt`:

```kotlin
class IniciarPartidaTest : FunSpec({
    test("CA-01 rejeita iniciar partida em grupo sem investigadores") {
        val p = novaPartida(investigadores = emptyList())
        p.deveRejeitar(CONFIGURACAO_INVALIDA) { iniciar(t(1)) }
        p.status shouldBe AGUARDANDO_INICIO
    }
    test("CA-02 inicia com narrador e uma investigadora") {
        val p = novaPartida(investigadores = listOf(ANA))
        p.iniciar(t(1))
        p.status shouldBe EM_ANDAMENTO
        p.fase shouldBe AGUARDANDO_PERGUNTA
        p.puxarEventos() shouldBe listOf(PartidaIniciada(p.id, t(1)))
    }
    test("CA-16 rejeita iniciar partida em andamento") {
        emAndamento().deveRejeitar(PARTIDA_JA_INICIADA) { iniciar(t(2)) }
    }
    context("CA-19 rejeita narrador entre os investigadores ou investigador repetido") {
        withData(mapOf("N investigador" to listOf(N, ANA), "Ana repetida" to listOf(ANA, ANA))) { lista ->
            novaPartida(investigadores = lista).deveRejeitar(CONFIGURACAO_INVALIDA) { iniciar(t(1)) }
        }
    }
    test("CA-20 modo solo exige exatamente um investigador") {
        novaPartida(SOLO, investigadores = listOf(ANA, BETO)).deveRejeitar(CONFIGURACAO_INVALIDA) { iniciar(t(1)) }
        novaPartida(SOLO).apply { iniciar(t(1)) }.status shouldBe EM_ANDAMENTO
    }
    test("CA-27 comando rejeitado preserva os eventos ainda não puxados") {
        val p = novaPartida().apply { iniciar(t(1)) }
        p.deveRejeitar(PARTIDA_JA_INICIADA) { iniciar(t(2)) }
        p.eventosPendentes shouldBe listOf(PartidaIniciada(p.id, t(1)))
    }
    test("criar copia a lista de investigadores") {
        val lista = mutableListOf(ANA)
        val p = novaPartida(investigadores = lista)
        lista += BETO
        p.investigadores shouldBe listOf(ANA)
    }
})
```

- [ ] **Passo 3: rodar e ver falhar**

Run: `./gradlew :domain:test --tests "com.quebugesse.domain.gamesession.IniciarPartidaTest"`
Expected: FAIL na compilação, com `Unresolved reference` para `GameSession`, `PlayerRef` e os demais tipos.

- [ ] **Passo 4: criar os tipos e, em `GameSession`, `criar`, as propriedades, `puxarEventos()` e `iniciar(agora: Instant)`**

- Cada tipo do "Modelo" no arquivo indicado na "Estrutura de arquivos" do plan.md.
- Estado interno: três `MutableList` privadas (perguntas, tentativas, eventos). `criar` guarda `investigadores.toList()`, começa em `AGUARDANDO_INICIO` e não gera evento (DP7).
- `perguntaAberta` é a pergunta com `aberta`; `propostaPendente` é a tentativa `PENDENTE`; `fase` segue I7 e é nula fora de `EM_ANDAMENTO`.
- `puxarEventos()` devolve uma cópia e esvazia a lista interna.
- `iniciar`: `exigirNaoEncerrada()` (`PARTIDA_ENCERRADA` em `RESOLVIDO` ou `ABANDONADO`); `PARTIDA_JA_INICIADA` fora de `AGUARDANDO_INICIO`; `CONFIGURACAO_INVALIDA` se a lista estiver vazia, contiver o narrador, tiver repetidos ou, no `SOLO`, não tiver exatamente um investigador. Depois: `EM_ANDAMENTO` e `PartidaIniciada(id, agora)`.

- [ ] **Passo 5: rodar e ver passar**

Run: o mesmo do passo 3.
Expected: PASS (CA-19 com 2 casos).

- [ ] **Passo 6: commit**

```bash
git add settings.gradle.kts gradle gradlew gradlew.bat domain/build.gradle.kts domain/src
git commit -m "S01: cria tipos do domínio e inicia partida"
```

Se a S00 já existia, só `domain/src`.

---

### Tarefa 2: fazer e responder perguntas

**Files:**
- Modify: `main/gamesession/GameSession.kt`, `test/gamesession/Fixtures.kt`
- Test: `test/gamesession/PerguntasTest.kt`

**Interfaces:**
- Consome: tarefa 1.
- Produz: `fazerPergunta(autor: PlayerRef, texto: String, agora: Instant): Question`, `responderPergunta(autor: PlayerRef, id: QuestionId, resposta: Answer, agora: Instant)`; auxiliares privados `exigirIniciada()` e `textoValido(texto: String): String`. Fixtures `comPerguntaAberta(autor: PlayerRef = ANA)` e `textosInvalidos: Map<String, String>`.

- [ ] **Passo 1: escrever as fixtures e os testes**

Em `Fixtures.kt`:

```kotlin
fun comPerguntaAberta(autor: PlayerRef = ANA) =
    emAndamento().apply { fazerPergunta(autor, "foi no deploy?", t(2)); puxarEventos() }

/** Chaves descritivas porque o Kotest recusa caso de teste com nome em branco. */
val textosInvalidos = mapOf(
    "vazio" to "", "só espaços" to "   ", "501 letras" to "a".repeat(501), "501 emojis" to "🐛".repeat(501),
)
```

`PerguntasTest.kt`:

```kotlin
class PerguntasTest : FunSpec({
    test("CA-03 rejeita pergunta de quem não é investigador") {
        listOf(N, ZE).forEach { quem ->
            emAndamento().deveRejeitar(NAO_E_INVESTIGADOR) { fazerPergunta(quem, "foi no deploy?", t(2)) }
        }
    }
    test("CA-04 rejeita segunda pergunta com uma em aberto") {
        comPerguntaAberta(ANA).deveRejeitar(PERGUNTA_EM_ABERTO) { fazerPergunta(BETO, "foi na migração?", t(3)) }
    }
    test("CA-05 rejeita resposta de quem não é o narrador") {
        comPerguntaAberta().deveRejeitar(NAO_E_NARRADOR) { responderPergunta(ANA, QuestionId(1), SIM, t(3)) }
    }
    test("CA-06 rejeita resposta a pergunta já respondida ou inexistente") {
        val p = comPerguntaAberta().apply { responderPergunta(N, QuestionId(1), NAO, t(3)) }
        p.deveRejeitar(PERGUNTA_JA_FECHADA) { responderPergunta(N, QuestionId(1), SIM, t(4)) }
        p.deveRejeitar(PERGUNTA_INEXISTENTE) { responderPergunta(N, QuestionId(99), SIM, t(4)) }
    }
    test("CA-07 narrador responde SIM e a partida volta a aguardar pergunta") {
        val p = comPerguntaAberta()
        p.responderPergunta(N, QuestionId(1), SIM, t(2))
        p.perguntas.single().resposta shouldBe SIM
        p.perguntas.single().aberta shouldBe false
        p.fase shouldBe AGUARDANDO_PERGUNTA
        p.puxarEventos() shouldBe listOf(PerguntaRespondida(p.id, t(2), QuestionId(1), SIM))
    }
    context("CA-18 rejeita texto de pergunta inválido") {
        withData(textosInvalidos) { texto ->
            emAndamento().deveRejeitar(TEXTO_INVALIDO) { fazerPergunta(ANA, texto, t(2)) }
        }
    }
    test("CA-18 aceita pergunta de 500 caracteres, conta emoji como um e apara as pontas") {
        emAndamento().fazerPergunta(ANA, "a".repeat(500), t(2)).texto shouldBe "a".repeat(500)
        emAndamento().fazerPergunta(ANA, "🐛".repeat(500), t(2)).texto shouldBe "🐛".repeat(500)
        emAndamento().fazerPergunta(ANA, "  oi  ", t(2)).texto shouldBe "oi"
    }
    test("CA-26 puxarEventos devolve os eventos em ordem e esvazia a lista") {
        val p = novaPartida()
        p.iniciar(t(1))
        p.fazerPergunta(ANA, "foi no deploy?", t(2))
        p.responderPergunta(N, QuestionId(1), NAO, t(3))
        p.eventosPendentes shouldHaveSize 3
        p.puxarEventos() shouldBe listOf(
            PartidaIniciada(p.id, t(1)),
            PerguntaFeita(p.id, t(2), QuestionId(1), ANA, "foi no deploy?"),
            PerguntaRespondida(p.id, t(3), QuestionId(1), NAO),
        )
        p.puxarEventos().shouldBeEmpty()
    }
    test("CA-31 perguntas numeradas a partir de 1") {
        val p = emAndamento()
        p.fazerPergunta(ANA, "foi no deploy?", t(2))
        p.responderPergunta(N, QuestionId(1), NAO, t(3))
        p.fazerPergunta(BETO, "foi na migração?", t(4))
        p.perguntas.map { it.id } shouldBe listOf(QuestionId(1), QuestionId(2))
    }
    test("CA-32 investigadora faz pergunta") {
        val p = emAndamento()
        val q = p.fazerPergunta(ANA, "foi no deploy?", t(2))
        q shouldBe Question(QuestionId(1), ANA, "foi no deploy?", t(2))
        p.perguntaAberta shouldBe q
        p.fase shouldBe PERGUNTA_PENDENTE
        p.puxarEventos() shouldBe listOf(PerguntaFeita(p.id, t(2), QuestionId(1), ANA, "foi no deploy?"))
    }
    test("fotografia não acompanha mudanças posteriores da partida") {
        val p = emAndamento()
        val antes = p.fotografia()
        p.fazerPergunta(ANA, "foi no deploy?", t(2))
        antes.perguntas.shouldBeEmpty()
        antes.eventos.shouldBeEmpty()
    }
})
```

- [ ] **Passo 2: rodar e ver falhar**

Run: `./gradlew :domain:test --tests "com.quebugesse.domain.gamesession.PerguntasTest"`
Expected: FAIL na compilação, `Unresolved reference: fazerPergunta`.

- [ ] **Passo 3: implementar `fazerPergunta` e `responderPergunta` em `GameSession.kt`**

- `fazerPergunta`: a forma do plan.md ("Forma de cada comando"); depois acrescenta `Question(QuestionId(perguntas.size + 1), autor, aparado, agora)`, gera `PerguntaFeita` e devolve a pergunta.
- `responderPergunta`: guardas 1, 2, `NAO_E_NARRADOR` (`autor != narrador`), `PERGUNTA_INEXISTENTE`, `PERGUNTA_JA_FECHADA`; troca a pergunta por `copy(resposta = resposta)`; gera `PerguntaRespondida`.
- `exigirIniciada()`: `PARTIDA_NAO_INICIADA` em `AGUARDANDO_INICIO`.
- `textoValido`: `trim()`; `TEXTO_INVALIDO` se `codePointCount(0, length)` ficar fora de 1..500; devolve o aparado.

- [ ] **Passo 4: rodar e ver passar**

Run: `./gradlew :domain:test`
Expected: PASS em `IniciarPartidaTest` e `PerguntasTest`.

- [ ] **Passo 5: commit**

```bash
git add domain/src/main/kotlin/com/quebugesse/domain/gamesession/GameSession.kt domain/src/test/kotlin/com/quebugesse/domain/gamesession/Fixtures.kt domain/src/test/kotlin/com/quebugesse/domain/gamesession/PerguntasTest.kt
git commit -m "S01: faz e responde perguntas"
```

---

### Tarefa 3: cancelar pergunta

**Files:**
- Modify: `main/gamesession/GameSession.kt`
- Test: `test/gamesession/CancelarPerguntaTest.kt`

**Interfaces:**
- Consome: tarefas 1 e 2 (`comPerguntaAberta`, `responderPergunta`).
- Produz: `cancelarPergunta(autor: PlayerRef, id: QuestionId, agora: Instant)`.

- [ ] **Passo 1: escrever os testes**

```kotlin
class CancelarPerguntaTest : FunSpec({
    context("CA-21 autora ou narrador cancelam a pergunta aberta") {
        withData(mapOf("Ana" to ANA, "N" to N)) { quem ->
            val p = comPerguntaAberta(ANA)
            p.cancelarPergunta(quem, QuestionId(1), t(3))
            p.perguntas.single().cancelada shouldBe true
            p.perguntas.single().aberta shouldBe false
            p.fase shouldBe AGUARDANDO_PERGUNTA
            p.puxarEventos() shouldBe listOf(PerguntaCancelada(p.id, t(3), QuestionId(1), quem))
            p.fazerPergunta(BETO, "foi na migração?", t(4)).id shouldBe QuestionId(2)
        }
    }
    test("CA-22 rejeita cancelamento por terceiro e de pergunta já respondida") {
        comPerguntaAberta(ANA).deveRejeitar(NAO_AUTORIZADO) { cancelarPergunta(BETO, QuestionId(1), t(3)) }
        val respondida = comPerguntaAberta().apply { responderPergunta(N, QuestionId(1), SIM, t(3)) }
        respondida.deveRejeitar(PERGUNTA_JA_FECHADA) { cancelarPergunta(N, QuestionId(1), t(4)) }
    }
    test("cancelar confere se a pergunta existe antes da autoria") {
        emAndamento().deveRejeitar(PERGUNTA_INEXISTENTE) { cancelarPergunta(ZE, QuestionId(99), t(3)) }
    }
    test("rejeita resposta a pergunta cancelada") {
        val p = comPerguntaAberta().apply { cancelarPergunta(ANA, QuestionId(1), t(3)) }
        p.deveRejeitar(PERGUNTA_JA_FECHADA) { responderPergunta(N, QuestionId(1), SIM, t(4)) }
    }
})
```

- [ ] **Passo 2: rodar e ver falhar**

Run: `./gradlew :domain:test --tests "com.quebugesse.domain.gamesession.CancelarPerguntaTest"`
Expected: FAIL na compilação, `Unresolved reference: cancelarPergunta`.

- [ ] **Passo 3: implementar `cancelarPergunta` em `GameSession.kt`**

Ordem da exceção da spec: guardas 1, 2, `PERGUNTA_INEXISTENTE`, `PERGUNTA_JA_FECHADA` e só então `NAO_AUTORIZADO` (autor que não é o da pergunta nem o narrador, R5). Troca a pergunta por `copy(cancelada = true)`; gera `PerguntaCancelada` com `canceladaPor = autor`.

- [ ] **Passo 4: rodar e ver passar**

Run: `./gradlew :domain:test`
Expected: PASS.

- [ ] **Passo 5: commit**

```bash
git add domain/src/main/kotlin/com/quebugesse/domain/gamesession/GameSession.kt domain/src/test/kotlin/com/quebugesse/domain/gamesession/CancelarPerguntaTest.kt
git commit -m "S01: cancela pergunta em aberto"
```

---

### Tarefa 4: propor solução e aplicar veredito

**Files:**
- Modify: `main/gamesession/GameSession.kt`, `test/gamesession/Fixtures.kt`
- Test: `test/gamesession/PropostasTest.kt`

**Interfaces:**
- Consome: tarefas 1 e 2 (`textosInvalidos`, `comPerguntaAberta`, `fazerPergunta`).
- Produz: `proporSolucao(autor: PlayerRef, texto: String, agora: Instant): Tentativa`, `aplicarVeredito(veredito: Veredito, agora: Instant)`. Fixtures `comPropostaPendente(autor: PlayerRef = ANA)` e `resolvida()`.

- [ ] **Passo 1: escrever as fixtures e os testes**

Em `Fixtures.kt`:

```kotlin
fun comPropostaPendente(autor: PlayerRef = ANA) =
    emAndamento().apply { proporSolucao(autor, "foi o cache", t(3)); puxarEventos() }

fun resolvida() = comPropostaPendente().apply { aplicarVeredito(Veredito(true), t(4)); puxarEventos() }
```

`PropostasTest.kt`:

```kotlin
class PropostasTest : FunSpec({
    test("CA-08 rejeita proposta com pergunta em aberto") {
        comPerguntaAberta().deveRejeitar(PERGUNTA_EM_ABERTO) { proporSolucao(ANA, "foi o cache", t(3)) }
    }
    test("CA-09 proposta vira tentativa pendente") {
        val p = emAndamento()
        val tentativa = p.proporSolucao(ANA, "foi o cache", t(3))
        tentativa shouldBe Tentativa(TentativaId(1), ANA, "foi o cache", t(3))
        p.propostaPendente shouldBe tentativa
        p.fase shouldBe SOLUCAO_EM_AVALIACAO
        p.puxarEventos() shouldBe listOf(SolucaoProposta(p.id, t(3), TentativaId(1), ANA, "foi o cache"))
    }
    test("CA-10 veredito correto resolve a partida") {
        val p = comPropostaPendente()
        p.aplicarVeredito(Veredito(true), t(4))
        p.status shouldBe RESOLVIDO
        p.resolvidaEm shouldBe t(4)
        p.tentativas.single().situacao shouldBe CORRETA
        p.puxarEventos() shouldBe listOf(PartidaResolvida(p.id, t(4), TentativaId(1)))
    }
    test("CA-11 veredito incorreto volta a aguardar pergunta") {
        val p = comPropostaPendente()
        p.aplicarVeredito(Veredito(false), t(4))
        p.status shouldBe EM_ANDAMENTO
        p.fase shouldBe AGUARDANDO_PERGUNTA
        p.resolvidaEm shouldBe null
        p.tentativas.single().situacao shouldBe INCORRETA
        p.puxarEventos() shouldBe listOf(SolucaoRecusada(p.id, t(4), TentativaId(1)))
        p.fazerPergunta(BETO, "foi na migração?", t(5)).id shouldBe QuestionId(1)
    }
    test("CA-12 rejeita veredito sem tentativa pendente") {
        emAndamento().deveRejeitar(SEM_PROPOSTA_PENDENTE) { aplicarVeredito(Veredito(true), t(4)) }
    }
    context("CA-18 rejeita texto de proposta inválido") {
        withData(textosInvalidos) { texto ->
            emAndamento().deveRejeitar(TEXTO_INVALIDO) { proporSolucao(ANA, texto, t(3)) }
        }
    }
    test("CA-18 aceita proposta de 500 caracteres e apara as pontas") {
        emAndamento().proporSolucao(ANA, "a".repeat(500), t(3)).texto shouldBe "a".repeat(500)
        emAndamento().proporSolucao(ANA, "🐛".repeat(500), t(3)).texto shouldBe "🐛".repeat(500)
        emAndamento().proporSolucao(ANA, "  oi  ", t(3)).texto shouldBe "oi"
    }
    test("CA-24 rejeita pergunta com tentativa pendente") {
        comPropostaPendente(ANA).deveRejeitar(PROPOSTA_PENDENTE) { fazerPergunta(BETO, "foi na migração?", t(4)) }
    }
    test("CA-25 rejeita segunda proposta com tentativa pendente") {
        comPropostaPendente(ANA).deveRejeitar(PROPOSTA_PENDENTE) { proporSolucao(BETO, "foi o DNS", t(4)) }
    }
    test("CA-31 tentativas têm numeração própria, a partir de 1") {
        val p = comPerguntaAberta().apply { responderPergunta(N, QuestionId(1), NAO, t(3)) }
        p.proporSolucao(ANA, "foi o cache", t(4))
        p.aplicarVeredito(Veredito(false), t(5))
        p.proporSolucao(BETO, "foi o DNS", t(6))
        p.tentativas.map { it.id } shouldBe listOf(TentativaId(1), TentativaId(2))
    }
    test("nova proposta depois de veredito incorreto fica pendente como tentativa 2") {
        val p = comPropostaPendente().apply { aplicarVeredito(Veredito(false), t(4)) }
        val nova = p.proporSolucao(BETO, "foi o DNS", t(5))
        nova.id shouldBe TentativaId(2)
        p.tentativas.map { it.situacao } shouldBe listOf(INCORRETA, PENDENTE)
        p.propostaPendente shouldBe nova
    }
    test("rejeita veredito duplicado") {
        val recusada = comPropostaPendente().apply { aplicarVeredito(Veredito(false), t(4)) }
        recusada.deveRejeitar(SEM_PROPOSTA_PENDENTE) { aplicarVeredito(Veredito(true), t(5)) }
        resolvida().deveRejeitar(PARTIDA_ENCERRADA) { aplicarVeredito(Veredito(true), t(5)) }
    }
})
```

- [ ] **Passo 2: rodar e ver falhar**

Run: `./gradlew :domain:test --tests "com.quebugesse.domain.gamesession.PropostasTest"`
Expected: FAIL na compilação, `Unresolved reference: proporSolucao`.

- [ ] **Passo 3: implementar `proporSolucao` e `aplicarVeredito` em `GameSession.kt`**

- `proporSolucao`: guardas 1, 2, `NAO_E_INVESTIGADOR`, `textoValido`, `PERGUNTA_EM_ABERTO`, `PROPOSTA_PENDENTE`; acrescenta `Tentativa(TentativaId(tentativas.size + 1), autor, aparado, agora)`; gera `SolucaoProposta` e devolve a tentativa.
- `aplicarVeredito`: guardas 1, 2, `SEM_PROPOSTA_PENDENTE`; não verifica autor (Q3). Correto: tentativa `CORRETA`, `RESOLVIDO`, `resolvidaEm = agora`, `PartidaResolvida`. Incorreto: tentativa `INCORRETA`, `SolucaoRecusada`.

- [ ] **Passo 4: rodar e ver passar**

Run: `./gradlew :domain:test`
Expected: PASS.

- [ ] **Passo 5: commit**

```bash
git add domain/src/main/kotlin/com/quebugesse/domain/gamesession/GameSession.kt domain/src/test/kotlin/com/quebugesse/domain/gamesession/Fixtures.kt domain/src/test/kotlin/com/quebugesse/domain/gamesession/PropostasTest.kt
git commit -m "S01: propõe solução e aplica veredito"
```

---

### Tarefa 5: retirar proposta

**Files:**
- Modify: `main/gamesession/GameSession.kt`
- Test: `test/gamesession/RetirarPropostaTest.kt`

**Interfaces:**
- Consome: tarefa 4 (`comPropostaPendente`).
- Produz: `retirarProposta(autor: PlayerRef, agora: Instant)`.

- [ ] **Passo 1: escrever os testes**

```kotlin
class RetirarPropostaTest : FunSpec({
    context("CA-23 autora ou narrador retiram a tentativa pendente") {
        withData(mapOf("Ana" to ANA, "N" to N)) { quem ->
            val p = comPropostaPendente(ANA)
            p.retirarProposta(quem, t(4))
            p.tentativas.single().situacao shouldBe RETIRADA
            p.fase shouldBe AGUARDANDO_PERGUNTA
            p.puxarEventos() shouldBe listOf(PropostaRetirada(p.id, t(4), TentativaId(1), quem))
        }
    }
    test("CA-23 rejeita retirada por terceiro e sem tentativa pendente") {
        comPropostaPendente(ANA).deveRejeitar(NAO_AUTORIZADO) { retirarProposta(BETO, t(4)) }
        // Zé de propósito: a falta de tentativa vem antes da autoria
        emAndamento().deveRejeitar(SEM_PROPOSTA_PENDENTE) { retirarProposta(ZE, t(4)) }
    }
})
```

- [ ] **Passo 2: rodar e ver falhar**

Run: `./gradlew :domain:test --tests "com.quebugesse.domain.gamesession.RetirarPropostaTest"`
Expected: FAIL na compilação, `Unresolved reference: retirarProposta`.

- [ ] **Passo 3: implementar `retirarProposta` em `GameSession.kt`**

Guardas 1, 2, `SEM_PROPOSTA_PENDENTE` e só então `NAO_AUTORIZADO` (autor que não é o da tentativa nem o narrador, R5). Tentativa vira `RETIRADA`; gera `PropostaRetirada` com `retiradaPor = autor`.

- [ ] **Passo 4: rodar e ver passar**

Run: `./gradlew :domain:test`
Expected: PASS.

- [ ] **Passo 5: commit**

```bash
git add domain/src/main/kotlin/com/quebugesse/domain/gamesession/GameSession.kt domain/src/test/kotlin/com/quebugesse/domain/gamesession/RetirarPropostaTest.kt
git commit -m "S01: retira proposta pendente"
```

---

### Tarefa 6: abandonar partida

**Files:**
- Modify: `main/gamesession/GameSession.kt`, `test/gamesession/Fixtures.kt`
- Test: `test/gamesession/AbandonoTest.kt`

**Interfaces:**
- Consome: tarefas 1, 2 e 4 (cenários prontos).
- Produz: `abandonar(autor: PlayerRef, agora: Instant)`. Fixtures `abandonada()` e `naoTerminais: Map<String, () -> GameSession>` (a tarefa 7 usa `abandonada()`).

- [ ] **Passo 1: escrever as fixtures e os testes**

Em `Fixtures.kt`:

```kotlin
fun abandonada() = emAndamento().apply { abandonar(N, t(5)); puxarEventos() }

val naoTerminais: Map<String, () -> GameSession> = mapOf(
    "aguardando início" to { novaPartida() },
    "aguardando pergunta" to { emAndamento() },
    "pergunta pendente" to { comPerguntaAberta() },
    "solução em avaliação" to { comPropostaPendente() },
)
```

`AbandonoTest.kt`:

```kotlin
class AbandonoTest : FunSpec({
    context("CA-14 rejeita abandono por quem não participa") {
        withData(naoTerminais) { cenario -> cenario().deveRejeitar(NAO_AUTORIZADO) { abandonar(ZE, t(5)) } }
    }
    context("CA-15 narrador abandona partida não terminal") {
        withData(naoTerminais) { cenario ->
            val p = cenario()
            p.abandonar(N, t(5))
            p.status shouldBe ABANDONADO
            p.resolvidaEm shouldBe null
            p.fase shouldBe null
            p.puxarEventos() shouldBe listOf(PartidaAbandonada(p.id, t(5), N))
        }
    }
    test("CA-28 investigadora só abandona no modo solo") {
        emAndamento(GRUPO).deveRejeitar(NAO_AUTORIZADO) { abandonar(ANA, t(5)) }
        emAndamento(SOLO).apply { abandonar(ANA, t(5)) }.status shouldBe ABANDONADO
    }
})
```

- [ ] **Passo 2: rodar e ver falhar**

Run: `./gradlew :domain:test --tests "com.quebugesse.domain.gamesession.AbandonoTest"`
Expected: FAIL na compilação, `Unresolved reference: abandonar`.

- [ ] **Passo 3: implementar `abandonar` em `GameSession.kt`**

`exigirNaoEncerrada()`, sem a guarda de início; `NAO_AUTORIZADO`, salvo se `autor == narrador` ou se o modo for `SOLO` e `autor in investigadores` (H2, Q1). Depois `ABANDONADO` e `PartidaAbandonada` com `abandonadaPor = autor`. Pergunta aberta ou tentativa pendente ficam como estão.

- [ ] **Passo 4: rodar e ver passar**

Run: `./gradlew :domain:test`
Expected: PASS.

- [ ] **Passo 5: commit**

```bash
git add domain/src/main/kotlin/com/quebugesse/domain/gamesession/GameSession.kt domain/src/test/kotlin/com/quebugesse/domain/gamesession/Fixtures.kt domain/src/test/kotlin/com/quebugesse/domain/gamesession/AbandonoTest.kt
git commit -m "S01: abandona partida"
```

---

### Tarefa 7: estados terminais e precedência das guardas

**Files:**
- Test: `test/gamesession/EstadosEPrecedenciaTest.kt`

**Interfaces:**
- Consome: os 8 comandos e as fixtures `novaPartida`, `comPerguntaAberta`, `resolvida`, `abandonada`.
- Produz: nada.

- [ ] **Passo 1: escrever os testes**

```kotlin
private val todosOsComandos: Map<String, GameSession.() -> Unit> = mapOf(
    "iniciar" to { iniciar(t(9)) },
    "fazerPergunta" to { fazerPergunta(ANA, "foi no deploy?", t(9)) },
    "responderPergunta" to { responderPergunta(N, QuestionId(1), SIM, t(9)) },
    "cancelarPergunta" to { cancelarPergunta(N, QuestionId(1), t(9)) },
    "proporSolucao" to { proporSolucao(ANA, "foi o cache", t(9)) },
    "retirarProposta" to { retirarProposta(N, t(9)) },
    "aplicarVeredito" to { aplicarVeredito(Veredito(true), t(9)) },
    "abandonar" to { abandonar(N, t(9)) },
)

class EstadosEPrecedenciaTest : FunSpec({
    context("CA-13 estado terminal rejeita os 8 comandos") {
        withData(mapOf("resolvida" to { resolvida() }, "abandonada" to { abandonada() })) { cenario ->
            withData(todosOsComandos) { comando -> cenario().deveRejeitar(PARTIDA_ENCERRADA, comando) }
        }
    }
    context("CA-17 partida não iniciada rejeita os 6 comandos de jogo") {
        withData(todosOsComandos - "iniciar" - "abandonar") { comando ->
            novaPartida().deveRejeitar(PARTIDA_NAO_INICIADA, comando)
        }
    }
    test("CA-29 autoria vem antes do estado da investigação, e encerramento antes da autoria") {
        comPerguntaAberta().deveRejeitar(NAO_E_INVESTIGADOR) { fazerPergunta(ZE, "foi no deploy?", t(3)) }
        resolvida().deveRejeitar(PARTIDA_ENCERRADA) { abandonar(ZE, t(5)) }
    }
})
```

- [ ] **Passo 2: rodar**

Run: `./gradlew :domain:test --tests "com.quebugesse.domain.gamesession.EstadosEPrecedenciaTest"`
Expected: PASS (16 + 6 casos, mais CA-29). Os comandos já existem, então esta tabela é a rede de regressão de H1 e da ordem das guardas. Se algum caso falhar, corrigir a ordem das guardas do comando em `GameSession.kt`, nunca o teste.

- [ ] **Passo 3: commit**

```bash
git add domain/src/test/kotlin/com/quebugesse/domain/gamesession/EstadosEPrecedenciaTest.kt
git commit -m "S01: testa estados terminais e precedência das guardas"
```

---

### Tarefa 8: invariantes por propriedade

**Files:**
- Test: `test/gamesession/InvariantesPropertyTest.kt`

**Interfaces:**
- Consome: os 8 comandos, `novaPartida`, `Fotografia`, `fotografia()`.
- Produz: `verificarSequencia(modo: ModoDeJogo, comandos: List<Comando>)`, usada pelos testes de regressão.

- [ ] **Passo 1: escrever o teste**

`sealed interface Comando { fun aplicar(p: GameSession) }`, com um tipo por comando: `data object Iniciar`, `Perguntar(autor, texto)`, `Responder(autor, id: Int, resposta: Answer)`, `Cancelar(autor, id: Int)`, `Propor(autor, texto)`, `Retirar(autor)`, `Julgar(correta: Boolean)` e `Abandonar(autor)`. Cada um chama o método correspondente com `t(9)`. São `data` para o Kotest imprimir a sequência reduzida de forma legível.

```kotlin
private val autores = Arb.element(N, ANA, BETO, ZE)
private val ids = Arb.int(1..3)
private val textos = Arb.element("", "x")
private val comando: Arb<Comando> = Arb.choice(
    Arb.constant(Iniciar),
    Arb.bind(autores, textos, ::Perguntar),
    Arb.bind(autores, ids, Arb.enum<Answer>(), ::Responder),
    Arb.bind(autores, ids, ::Cancelar),
    Arb.bind(autores, textos, ::Propor),
    autores.map(::Retirar),
    Arb.boolean().map(::Julgar),
    autores.map(::Abandonar),
)
private val terminais = setOf(RESOLVIDO, ABANDONADO)

fun verificarSequencia(modo: ModoDeJogo, comandos: List<Comando>) {
    val p = novaPartida(modo)
    comandos.forEach { c ->
        val antes = p.fotografia()
        val rejeitado = try { c.aplicar(p); false } catch (e: PartidaException) { true }
        val depois = p.fotografia()
        if (rejeitado) depois shouldBe antes                                         // R3
        verificarInvariantes(p, antes, depois)
    }
}

fun verificarInvariantes(p: GameSession, antes: Fotografia, depois: Fotografia) {
    val abertas = p.perguntas.count { it.aberta }
    val pendentes = p.tentativas.count { it.situacao == PENDENTE }
    abertas shouldBeLessThanOrEqual 1                                                // I1
    pendentes shouldBeLessThanOrEqual 1                                              // I2
    (abertas > 0 && pendentes > 0) shouldBe false                                    // I3
    (p.resolvidaEm != null) shouldBe (p.status == RESOLVIDO)                         // I4
    if (antes.status in terminais) depois shouldBe antes                             // I5
    depois.perguntas.size shouldBeGreaterThanOrEqual antes.perguntas.size            // I6
    depois.tentativas.size shouldBeGreaterThanOrEqual antes.tentativas.size
    antes.perguntas.zip(depois.perguntas).forEach { (a, d) ->
        if (a.aberta) d.copy(resposta = null, cancelada = false) shouldBe a else d shouldBe a
    }
    antes.tentativas.zip(depois.tentativas).forEach { (a, d) ->
        if (a.situacao == PENDENTE) d.copy(situacao = PENDENTE) shouldBe a else d shouldBe a
    }
    (p.fase != null) shouldBe (p.status == EM_ANDAMENTO)                             // I7
    if (p.status == EM_ANDAMENTO) {
        (p.fase == PERGUNTA_PENDENTE) shouldBe (abertas > 0)
        (p.fase == SOLUCAO_EM_AVALIACAO) shouldBe (pendentes > 0)
    }
    if (p.status == RESOLVIDO) {                                                     // I8
        p.tentativas.count { it.situacao == CORRETA } shouldBe 1
        p.tentativas.last().situacao shouldBe CORRETA
    }
}

class InvariantesPropertyTest : FunSpec({
    test("CA-30 invariantes valem depois de cada comando de uma sequência aleatória") {
        checkAll(1000, Arb.list(comando, 0..50), Arb.enum<ModoDeJogo>()) { comandos, modo ->
            verificarSequencia(modo, comandos)
        }
    }
})
```

- [ ] **Passo 2: rodar**

Run: `./gradlew :domain:test --tests "com.quebugesse.domain.gamesession.InvariantesPropertyTest"`
Expected: PASS. Se falhar, o Kotest mostra a sequência reduzida e a seed: corrigir `GameSession.kt` e acrescentar `test("CA-30 regressão: <o que quebrou>") { verificarSequencia(<modo>, listOf(<sequência reduzida>)) }`.

- [ ] **Passo 3: conferir que o teste pega erro**

Comentar a guarda `PROPOSTA_PENDENTE` de `fazerPergunta` e rodar de novo.
Expected: FAIL em I3, com sequência reduzida curta (algo como `Iniciar`, `Propor`, `Perguntar`). Desfazer a mudança e rodar até voltar a PASS.

- [ ] **Passo 4: commit**

```bash
git add domain/src/test/kotlin/com/quebugesse/domain/gamesession/InvariantesPropertyTest.kt
git commit -m "S01: testa invariantes por propriedade"
```

---

### Tarefa 9: cobertura, pureza do domínio e diagramas

**Files:**
- Modify: `domain/build.gradle.kts`, `specs/001-dominio-partida/spec.md` (seção "Divergências")
- Modify: `docs/diagramas/estados-partida.md`, `docs/diagramas/classes-dominio.md`, `docs/diagramas/README.md` (tabela "Pendências por diagrama")
- Test: `test/DominioPuroTest.kt` (pacote `com.quebugesse.domain`)

**Interfaces:**
- Consome: o módulo completo.
- Produz: nada.

- [ ] **Passo 1: escrever o teste de pureza**

```kotlin
class DominioPuroTest : FunSpec({
    test("P1 código de produção do domínio não lê relógio nem gera aleatoriedade") {
        val proibidos = listOf("Instant.now(", "Clock", "UUID.randomUUID(", "Random")
        val fontes = File("src/main/kotlin").walkTopDown().filter { it.extension == "kt" }.toList()
        fontes.shouldNotBeEmpty()   // o diretório de trabalho do teste é domain/
        fontes.forEach { arquivo ->
            val codigo = arquivo.readText()
            proibidos.forEach { termo -> withClue("${arquivo.name} usa $termo") { codigo shouldNotContain termo } }
        }
    }
})
```

- [ ] **Passo 2: rodar, ver falhar de propósito e ver passar**

Acrescentar `private val relogio = java.time.Instant.now()` em `GameSession.kt` e rodar `./gradlew :domain:test --tests "com.quebugesse.domain.DominioPuroTest"`.
Expected: FAIL com `GameSession.kt usa Instant.now(`. Desfazer e rodar de novo: PASS.

- [ ] **Passo 3: exigir cobertura no Kover**

Em `domain/build.gradle.kts` (DSL do Kover 0.8 ou mais novo):

```kotlin
kover {
    reports {
        filters { includes { packages("com.quebugesse.domain.gamesession") } }
        verify { rule { minBound(90) } }   // % de linhas cobertas
    }
}
```

Run: `./gradlew :domain:koverVerify :domain:koverLog`
Expected: `BUILD SUCCESSFUL` e linhas ≥ 90%. Abaixo disso, a linha descoberta indica o caso sem teste: escrever o teste, nunca excluir o código do filtro.

- [ ] **Passo 4: atualizar as fontes dos diagramas (P9)**

Os dois arquivos já existem, com a transcrição dos PDFs de 14/10/2026. Trocar o bloco `mermaid` de cada um:

- `estados-partida.md` (`stateDiagram-v2`): `AGUARDANDO_INICIO`, `EM_ANDAMENTO` com os subestados `AGUARDANDO_PERGUNTA`, `PERGUNTA_PENDENTE` e `SOLUCAO_EM_AVALIACAO`, `RESOLVIDO` e `ABANDONADO`. Uma transição para cada "→" da tabela "Comportamento esperado" da spec, incluindo `cancelarPergunta` e `retirarProposta` (D4). Notas com H1, H2 (D7), H3, H5 estendida (D6) e R3.
- `classes-dominio.md` (`classDiagram`): os tipos do "Modelo" do plan.md, com `modo` (D2), `Tentativa` e `tentativas` (D3), os 9 eventos sob `EventoDePartida`, que estende `DomainEvent` (D5), `Question.aberta` considerando o cancelamento (D8), `fase` (D9), `QuestionId` e `TentativaId` sobre `Int` (D10) e `agora: Instant` em todos os comandos (D1).
- Cada item marcado com um comentário `%% Dn`.
- Em `docs/diagramas/README.md`, tirar da tabela "Pendências por diagrama" as linhas da S01 aplicadas e atualizar o campo "Estado" dos dois arquivos.

Verificar: os dois arquivos abrem sem erro de sintaxe no preview Mermaid do editor (ou no `mmdc`, se já estiver instalado; não baixar nada só para isso).

- [ ] **Passo 5: registrar na spec**

Logo abaixo da tabela de "Divergências em relação aos diagramas", acrescentar: "Fontes Mermaid atualizadas em `docs/diagramas/` (tarefa 9). Os PDFs em `diagramas/` seguem desatualizados até serem gerados de novo a partir delas."

- [ ] **Passo 6: rodar tudo**

Run: `./gradlew :domain:test :domain:koverVerify`
Expected: `BUILD SUCCESSFUL`; o tempo total em `domain/build/reports/tests/test/index.html` fica abaixo de 10 s (RNF3).

Run (Git Bash): `grep -rhoE '"CA-[0-9]{2}' domain/src/test | sort -u | wc -l`
Expected: `32`.

- [ ] **Passo 7: commit**

```bash
git add domain/build.gradle.kts domain/src/test/kotlin/com/quebugesse/domain/DominioPuroTest.kt docs/diagramas/estados-partida.md docs/diagramas/classes-dominio.md docs/diagramas/README.md specs/001-dominio-partida/spec.md
git commit -m "S01: exige cobertura e pureza e atualiza diagramas"
```

---

## Cobertura dos CAs

| Tarefa | CAs |
|---|---|
| 1 | 01, 02, 16, 19, 20, 27 |
| 2 | 03, 04, 05, 06, 07, 18 (pergunta), 26, 31 (perguntas), 32 |
| 3 | 21, 22 |
| 4 | 08, 09, 10, 11, 12, 18 (proposta), 24, 25, 31 (tentativas) |
| 5 | 23 |
| 6 | 14, 15, 28 |
| 7 | 13, 17, 29 |
| 8 | 30 |
| 9 | RNF1, RNF2, RNF3, D1 a D10 |

CA-27 também vale em todo teste de rejeição, pela comparação de fotografias em `deveRejeitar`.
