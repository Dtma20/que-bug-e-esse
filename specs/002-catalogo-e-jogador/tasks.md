# Tarefas: S02 Catálogo de enigmas e jogador

> **Para agentes:** SUB-SKILL OBRIGATÓRIA: `superpowers:subagent-driven-development` ou `superpowers:executing-plans`, tarefa a tarefa. Os passos usam caixas de seleção (`- [ ]`) para acompanhamento.

**Objetivo:** implementar `Enigma`, `CriterioDeSelecao`, `Player`, as portas de repositório, os repositórios em memória e o seed da spec S02, com os 32 CAs como testes Kotest verdes.

**Arquitetura:** valores imutáveis criados por fábrica (`ConteudoEnigma.de`, `Player.criar`) com validação no `init`; `Enigma` com `private set` e guardas antes da mutação; portas em `domain.<agregado>.port`; repositórios em memória que guardam cópias; suítes de contrato em `testFixtures` (DP2, DP8, DP9).

**Stack:** Kotlin/JVM, Gradle com Kotlin DSL e plugin de convenção da S00, Kotest (`FunSpec`, `assertions-core`, `withData`), `java-test-fixtures`, Kover.

**Spec:** [spec.md](spec.md) · Plano técnico: [plan.md](plan.md) (seções "Modelo" e "Estrutura de arquivos") · Constituição: [docs/constitution.md](../../docs/constitution.md)

## Restrições globais

- P1: nada em `domain/src/main` usa `Instant.now(`, `Clock`, `UUID.randomUUID(` ou `Random`; a seleção não sorteia.
- Pacotes `com.quebugesse.domain.enigma`, `.enigma.port`, `.player`, `.player.port`; `enigma` e `player` só importam `shared` (P7). `infrastructure` só importa `domain`.
- Nomes, tipos e assinaturas: exatamente os do bloco "Modelo" do plan.md.
- Texto (E1, J1): `trim()`, depois `codePointCount` no intervalo; guarda-se o texto aparado.
- Todo teste de CA tem nome começando por `"CA-NN "`; todo teste de rejeição do enigma usa `deveRejeitar` (CA-15).
- Cobertura de linhas ≥ 90% nos pacotes da RNF2.
- Testes: `./gradlew :domain:test --tests "com.quebugesse.domain.enigma.<Classe>"` e `./gradlew :infrastructure:test` (no PowerShell, `.\gradlew.bat`).
- Commits: `git add` só com os caminhos da tarefa; mensagem `S02: <verbo no presente>`.

Convenções deste documento:
- Nos blocos **Files**, `main/` abrevia `domain/src/main/kotlin/com/quebugesse/domain/`, `test/` abrevia `domain/src/test/kotlin/com/quebugesse/domain/`, `infra/` abrevia `infrastructure/src/main/kotlin/com/quebugesse/infrastructure/persistence/`, `infra-test/` abrevia `infrastructure/src/test/kotlin/com/quebugesse/infrastructure/persistence/` e `contrato/` abrevia `infrastructure/src/testFixtures/kotlin/com/quebugesse/infrastructure/persistence/contrato/`. Os comandos `git add` usam o caminho completo.
- Os trechos de teste omitem `package` e imports, inclusive os imports estáticos das constantes de enum.

## Foco de revisão

1. Lista de tags alterada por quem chamou `de`: o conteúdo não muda junto. Teste na tarefa 1 (CA-05).
2. Solução vazando por `toString` da `data class`: `ConteudoEnigma` e `Enigma` sobrescrevem. Testes nas tarefas 1 e 2 (CA-06).
3. Repositório devolvendo o próprio objeto guardado: mudar o enigma sem salvar não pode aparecer no `buscar`. Teste na tarefa 5 (CA-27).
4. `\u000a` no enigma de SINTAXE: em string Kotlin é escape de quebra de linha; o texto precisa de `\\u000a`. Teste na tarefa 6 (CA-32).
5. Seed que sobrescreve: depois de arquivar um enigma do seed, carregar de novo não o republica. Teste na tarefa 6 (CA-31).

---

### Tarefa 1: conteúdo do enigma

**Files:**
- Create (só se a S01 ainda não existir): `main/shared/Ids.kt` com `EnigmaId` e `PlayerId`
- Create: `main/enigma/Enums.kt`, `main/enigma/ViolacaoEnigma.kt`, `main/enigma/ConteudoEnigma.kt`
- Test: `test/enigma/FixturesEnigma.kt`, `test/enigma/ConteudoEnigmaTest.kt`

**Interfaces:**
- Consome: `EnigmaId` (S01).
- Produz: `Categoria`, `Dificuldade`, `OrigemEnigma`, `StatusEnigma`, `ViolacaoEnigma`, `EnigmaException`, `ConteudoEnigma` com `de(...)` e os limites `MAX_*`. Nos testes: `conteudo(...)` e `textosInvalidos(max: Int)`.

- [ ] **Passo 1: escrever as fixtures e os testes**

`FixturesEnigma.kt`:

```kotlin
fun conteudo(
    titulo: String = "Férias fatais",
    enunciado: String = "Toda vez que o Rafael sai de férias, a produção falha.",
    solucao: String = "o notebook hospedava o endpoint",
    categoria: Categoria = DEBUG,
    dificuldade: Dificuldade = MEDIA,
    tags: Collection<String> = listOf("rede"),
) = ConteudoEnigma.de(titulo, enunciado, solucao, categoria, dificuldade, tags)

/** Chaves descritivas porque o Kotest recusa caso de teste com nome em branco. */
fun textosInvalidos(max: Int) = mapOf(
    "vazio" to "", "só espaços" to "   ",
    "${max + 1} letras" to "a".repeat(max + 1), "${max + 1} emojis" to "🐛".repeat(max + 1),
)
```

`ConteudoEnigmaTest.kt`:

```kotlin
private fun deveRejeitarConteudo(esperada: ViolacaoEnigma, criar: () -> ConteudoEnigma) {
    shouldThrow<EnigmaException> { criar() }.violacao shouldBe esperada
}

class ConteudoEnigmaTest : FunSpec({
    test("CA-01 de apara os textos e normaliza as tags") {
        val c = ConteudoEnigma.de("  Férias fatais ", " enunciado  ", "  solução ", DEBUG, MEDIA,
            listOf(" Rede ", "", "rede", "PRODUCAO"))
        c.titulo shouldBe "Férias fatais"
        c.enunciado shouldBe "enunciado"
        c.solucaoCompleta shouldBe "solução"
        c.tags shouldBe setOf("rede", "producao")
    }
    context("CA-02 rejeita título inválido") {
        withData(textosInvalidos(ConteudoEnigma.MAX_TITULO)) { t -> deveRejeitarConteudo(TITULO_INVALIDO) { conteudo(titulo = t) } }
    }
    test("CA-02 aceita título de 100 caracteres, contando emoji como um") {
        conteudo(titulo = "a".repeat(100)).titulo shouldBe "a".repeat(100)
        conteudo(titulo = "🐛".repeat(100)).titulo shouldBe "🐛".repeat(100)
    }
    context("CA-03 rejeita enunciado inválido") {
        withData(textosInvalidos(ConteudoEnigma.MAX_ENUNCIADO)) { t -> deveRejeitarConteudo(ENUNCIADO_INVALIDO) { conteudo(enunciado = t) } }
    }
    test("CA-03 aceita enunciado de 2000 caracteres") {
        conteudo(enunciado = "a".repeat(2000)).enunciado shouldBe "a".repeat(2000)
    }
    context("CA-04 rejeita solução inválida") {
        withData(textosInvalidos(ConteudoEnigma.MAX_SOLUCAO)) { t -> deveRejeitarConteudo(SOLUCAO_INVALIDA) { conteudo(solucao = t) } }
    }
    test("CA-04 aceita solução de 2000 caracteres") {
        conteudo(solucao = "a".repeat(2000)).solucaoCompleta shouldBe "a".repeat(2000)
    }
    test("CA-05 conteúdo não acompanha a lista de tags de quem chamou") {
        val tags = mutableListOf("rede")
        val c = conteudo(tags = tags)
        tags += "nova"
        c.tags shouldBe setOf("rede")
    }
    test("CA-06 toString do conteúdo não mostra a solução") {
        conteudo(solucao = "o notebook hospedava o endpoint").toString() shouldNotContain "notebook"
    }
    test("CA-07 título é verificado antes do enunciado") {
        deveRejeitarConteudo(TITULO_INVALIDO) { conteudo(titulo = "", enunciado = "") }
    }
})
```

- [ ] **Passo 2: rodar e ver falhar**

Run: `./gradlew :domain:test --tests "com.quebugesse.domain.enigma.ConteudoEnigmaTest"`
Expected: FAIL na compilação, com `Unresolved reference` para `ConteudoEnigma`.

- [ ] **Passo 3: implementar**

- Enums e exceção conforme o "Modelo".
- `de`: apara os três textos; tags com `map { it.trim().lowercase() }.filter { it.isNotEmpty() }.toSet()`; chama o construtor privado.
- `init`: verifica, nesta ordem, título, enunciado e solução com `codePointCount(0, length)` entre 1 e o limite; lança `EnigmaException` com a violação do campo.
- `toString()`: `"ConteudoEnigma(titulo=..., categoria=..., dificuldade=..., tags=...)"`.

- [ ] **Passo 4: rodar e ver passar**

Run: o mesmo do passo 2.
Expected: PASS.

- [ ] **Passo 5: commit**

```bash
git add domain/src/main/kotlin/com/quebugesse/domain/enigma domain/src/test/kotlin/com/quebugesse/domain/enigma
git commit -m "S02: cria conteúdo do enigma"
```

Se criou `shared/Ids.kt`, incluí-lo no `git add`.

---

### Tarefa 2: ciclo de vida do enigma

**Files:**
- Create: `main/enigma/Enigma.kt`
- Modify: `test/enigma/FixturesEnigma.kt`
- Test: `test/enigma/FotografiaEnigma.kt`, `test/enigma/CicloDeVidaEnigmaTest.kt`

**Interfaces:**
- Consome: tarefa 1.
- Produz: `Enigma` com `criar`, `reconstituir`, `revisar`, `publicar`, `arquivar`, `toString`. Nos testes: `E1`, `E2`, `E3`, `rascunho(id)`, `publicado(id)`, `arquivado(id)`, `FotografiaEnigma`, `Enigma.fotografia()`, `Enigma.deveRejeitar(esperada, comando)`.

- [ ] **Passo 1: escrever as fixtures e os testes**

Em `FixturesEnigma.kt`:

```kotlin
val E1 = EnigmaId(UUID(3, 1))
val E2 = EnigmaId(UUID(3, 2))
val E3 = EnigmaId(UUID(3, 3))

fun rascunho(id: EnigmaId = E1, c: ConteudoEnigma = conteudo()) = Enigma.criar(id, c, MANUAL)
fun publicado(id: EnigmaId = E1, c: ConteudoEnigma = conteudo()) = rascunho(id, c).apply { publicar() }
fun arquivado(id: EnigmaId = E1, c: ConteudoEnigma = conteudo()) = rascunho(id, c).apply { arquivar() }
```

`FotografiaEnigma.kt`:

```kotlin
data class FotografiaEnigma(val id: EnigmaId, val origem: OrigemEnigma, val conteudo: ConteudoEnigma, val status: StatusEnigma)

fun Enigma.fotografia() = FotografiaEnigma(id, origem, conteudo, status)

/** Confere a violação e E7 (CA-15): nada muda. */
fun Enigma.deveRejeitar(esperada: ViolacaoEnigma, comando: Enigma.() -> Unit) {
    val antes = fotografia()
    shouldThrow<EnigmaException> { this.comando() }.violacao shouldBe esperada
    fotografia() shouldBe antes
}
```

`CicloDeVidaEnigmaTest.kt`:

```kotlin
private val novo = conteudo(titulo = "Férias fatais, revisado")

class CicloDeVidaEnigmaTest : FunSpec({
    test("CA-06 toString do enigma não mostra a solução") {
        rascunho().toString() shouldNotContain "notebook"
    }
    test("CA-08 enigma nasce em rascunho") {
        val e = Enigma.criar(E1, conteudo(), IA)
        e.fotografia() shouldBe FotografiaEnigma(E1, IA, conteudo(), RASCUNHO)
    }
    test("CA-09 revisar em rascunho troca o conteúdo") {
        val e = rascunho()
        e.revisar(novo)
        e.conteudo shouldBe novo
        e.status shouldBe RASCUNHO
    }
    context("CA-10 só rascunho é editável") {
        withData(mapOf("publicado" to { publicado() }, "arquivado" to { arquivado() })) { cenario ->
            cenario().deveRejeitar(ENIGMA_NAO_EDITAVEL) { revisar(novo) }
        }
    }
    test("CA-11 publicar rascunho") {
        rascunho().apply { publicar() }.status shouldBe PUBLICADO
    }
    context("CA-12 publicar fora de rascunho é transição inválida") {
        withData(mapOf("publicado" to { publicado() }, "arquivado" to { arquivado() })) { cenario ->
            cenario().deveRejeitar(TRANSICAO_INVALIDA) { publicar() }
        }
    }
    context("CA-13 arquivar rascunho ou publicado") {
        withData(mapOf("rascunho" to { rascunho() }, "publicado" to { publicado() })) { cenario ->
            cenario().apply { arquivar() }.status shouldBe ARQUIVADO
        }
    }
    test("CA-14 arquivado é terminal") {
        arquivado().deveRejeitar(TRANSICAO_INVALIDA) { arquivar() }
    }
    test("CA-16 reconstituir mantém os campos e segue a tabela de status") {
        val e = Enigma.reconstituir(E1, conteudo(), MANUAL, PUBLICADO)
        e.fotografia() shouldBe FotografiaEnigma(E1, MANUAL, conteudo(), PUBLICADO)
        e.deveRejeitar(TRANSICAO_INVALIDA) { publicar() }
        e.arquivar()
        e.status shouldBe ARQUIVADO
    }
})
```

- [ ] **Passo 2: rodar e ver falhar**

Run: `./gradlew :domain:test --tests "com.quebugesse.domain.enigma.CicloDeVidaEnigmaTest"`
Expected: FAIL na compilação, `Unresolved reference: Enigma`.

- [ ] **Passo 3: implementar `Enigma.kt`**

- Forma da S01 (DP2 de lá): guardas primeiro, mutação depois.
- `revisar`: `ENIGMA_NAO_EDITAVEL` se `status != RASCUNHO`; troca `conteudo`.
- `publicar`: `TRANSICAO_INVALIDA` se `status != RASCUNHO`; `PUBLICADO`.
- `arquivar`: `TRANSICAO_INVALIDA` se `status == ARQUIVADO`; `ARQUIVADO`.
- `toString()`: `"Enigma(id=..., origem=..., status=..., conteudo=$conteudo)"`, que já esconde a solução.

- [ ] **Passo 4: rodar e ver passar**

Run: `./gradlew :domain:test`
Expected: PASS em `ConteudoEnigmaTest` e `CicloDeVidaEnigmaTest`.

- [ ] **Passo 5: commit**

```bash
git add domain/src/main/kotlin/com/quebugesse/domain/enigma/Enigma.kt domain/src/test/kotlin/com/quebugesse/domain/enigma
git commit -m "S02: cria ciclo de vida do enigma"
```

---

### Tarefa 3: critério de seleção

**Files:**
- Create: `main/enigma/CriterioDeSelecao.kt`
- Test: `test/enigma/CriterioDeSelecaoTest.kt`

**Interfaces:**
- Consome: tarefas 1 e 2 (`publicado`, `rascunho`, `arquivado`, `conteudo`).
- Produz: `CriterioDeSelecao.elegiveis(candidatos: List<Enigma>, jaJogados: Set<EnigmaId>): List<Enigma>`.

- [ ] **Passo 1: escrever os testes**

```kotlin
private fun ids(lista: List<Enigma>) = lista.map { it.id }

class CriterioDeSelecaoTest : FunSpec({
    test("CA-17 só publicado é elegível") {
        val candidatos = listOf(rascunho(E1), publicado(E2), arquivado(E3))
        ids(CriterioDeSelecao().elegiveis(candidatos, emptySet())) shouldBe listOf(E2)
    }
    test("CA-18 filtra por categoria, por dificuldade, pelas duas ou por nenhuma") {
        val debugFacil = publicado(E1, conteudo(categoria = DEBUG, dificuldade = FACIL))
        val debugDificil = publicado(E2, conteudo(categoria = DEBUG, dificuldade = DIFICIL))
        val designFacil = publicado(E3, conteudo(categoria = DESIGN, dificuldade = FACIL))
        val todos = listOf(debugFacil, debugDificil, designFacil)
        ids(CriterioDeSelecao(categoria = DEBUG).elegiveis(todos, emptySet())) shouldBe listOf(E1, E2)
        ids(CriterioDeSelecao(dificuldade = FACIL).elegiveis(todos, emptySet())) shouldBe listOf(E1, E3)
        ids(CriterioDeSelecao(DEBUG, FACIL).elegiveis(todos, emptySet())) shouldBe listOf(E1)
        ids(CriterioDeSelecao().elegiveis(todos, emptySet())) shouldBe listOf(E1, E2, E3)
    }
    test("CA-19 exclui os já jogados") {
        ids(CriterioDeSelecao().elegiveis(listOf(publicado(E1), publicado(E2)), setOf(E1))) shouldBe listOf(E2)
    }
    test("CA-20 mantém a ordem recebida e devolve vazio sem elegíveis") {
        val candidatos = listOf(publicado(E3), publicado(E1), publicado(E2))
        ids(CriterioDeSelecao().elegiveis(candidatos, emptySet())) shouldBe listOf(E3, E1, E2)
        CriterioDeSelecao(categoria = SINTAXE).elegiveis(candidatos, emptySet()).shouldBeEmpty()
    }
})
```

- [ ] **Passo 2: rodar e ver falhar**

Run: `./gradlew :domain:test --tests "com.quebugesse.domain.enigma.CriterioDeSelecaoTest"`
Expected: FAIL na compilação, `Unresolved reference: CriterioDeSelecao`.

- [ ] **Passo 3: implementar**

Um `filter` sobre `candidatos`: `status == PUBLICADO`, `id !in jaJogados`, e categoria e dificuldade iguais às do critério quando não nulas.

- [ ] **Passo 4: rodar e ver passar**

Run: `./gradlew :domain:test`
Expected: PASS.

- [ ] **Passo 5: commit**

```bash
git add domain/src/main/kotlin/com/quebugesse/domain/enigma/CriterioDeSelecao.kt domain/src/test/kotlin/com/quebugesse/domain/enigma/CriterioDeSelecaoTest.kt
git commit -m "S02: seleciona enigmas elegíveis"
```

---

### Tarefa 4: jogador

**Files:**
- Create: `main/player/PlayerType.kt`, `main/player/ViolacaoJogador.kt`, `main/player/EstatisticasJogador.kt`, `main/player/Player.kt`
- Test: `test/player/PlayerTest.kt`

**Interfaces:**
- Consome: `PlayerId` (S01).
- Produz: `PlayerType`, `ViolacaoJogador`, `JogadorException`, `EstatisticasJogador`, `Player` com `criar`, `reconstituir` e `MAX_NOME`.

- [ ] **Passo 1: escrever os testes**

```kotlin
private val ID = PlayerId(UUID(0, 2))
private val nomesInvalidos = mapOf("vazio" to "", "só espaços" to "   ", "41 letras" to "a".repeat(41), "41 emojis" to "🐛".repeat(41))

private fun deveRejeitarJogador(esperada: ViolacaoJogador, bloco: () -> Unit) {
    shouldThrow<JogadorException> { bloco() }.violacao shouldBe esperada
}

class PlayerTest : FunSpec({
    test("CA-21 criar apara o nome e zera as estatísticas") {
        val ana = Player.criar(ID, "  Ana  ", HUMANO)
        ana.nome shouldBe "Ana"
        ana.tipo shouldBe HUMANO
        ana.stats shouldBe EstatisticasJogador(0, 0)
    }
    context("CA-22 rejeita nome inválido") {
        withData(nomesInvalidos) { nome -> deveRejeitarJogador(NOME_INVALIDO) { Player.criar(ID, nome, HUMANO) } }
    }
    test("CA-22 aceita nome de 40 caracteres, contando emoji como um") {
        Player.criar(ID, "a".repeat(40), HUMANO).nome shouldBe "a".repeat(40)
        Player.criar(ID, "🐛".repeat(40), HUMANO).nome shouldBe "🐛".repeat(40)
    }
    context("CA-23 rejeita estatísticas inválidas") {
        withData(mapOf("jogadas negativas" to (-1 to 0), "resolvidas negativas" to (0 to -1), "mais resolvidas que jogadas" to (1 to 2))) { (j, r) ->
            deveRejeitarJogador(ESTATISTICAS_INVALIDAS) { EstatisticasJogador(j, r) }
        }
    }
    test("CA-23 aceita estatísticas válidas") {
        EstatisticasJogador(2, 1).partidasResolvidas shouldBe 1
        EstatisticasJogador().partidasJogadas shouldBe 0
    }
    test("CA-24 reconstituir mantém os campos e valida o nome") {
        val p = Player.reconstituir(ID, "Ana", HUMANO, EstatisticasJogador(5, 2))
        listOf(p.id, p.nome, p.tipo, p.stats) shouldBe listOf(ID, "Ana", HUMANO, EstatisticasJogador(5, 2))
        deveRejeitarJogador(NOME_INVALIDO) { Player.reconstituir(ID, "", HUMANO, EstatisticasJogador()) }
    }
})
```

- [ ] **Passo 2: rodar e ver falhar**

Run: `./gradlew :domain:test --tests "com.quebugesse.domain.player.PlayerTest"`
Expected: FAIL na compilação, `Unresolved reference: Player`.

- [ ] **Passo 3: implementar**

- `EstatisticasJogador.init`: `ESTATISTICAS_INVALIDAS` salvo se `0 <= partidasResolvidas <= partidasJogadas`.
- `Player`: `criar` e `reconstituir` aparam o nome e chamam o construtor privado; o `init` lança `NOME_INVALIDO` se `codePointCount` ficar fora de 1..40.

- [ ] **Passo 4: rodar e ver passar**

Run: `./gradlew :domain:test`
Expected: PASS.

- [ ] **Passo 5: commit**

```bash
git add domain/src/main/kotlin/com/quebugesse/domain/player domain/src/test/kotlin/com/quebugesse/domain/player
git commit -m "S02: cria jogador"
```

---

### Tarefa 5: portas, repositórios em memória e suítes de contrato

**Files:**
- Create: `main/enigma/port/EnigmaRepository.kt`, `main/player/port/PlayerRepository.kt`
- Create: `infra/memoria/EnigmaRepositoryEmMemoria.kt`, `infra/memoria/PlayerRepositoryEmMemoria.kt`
- Create: `contrato/EnigmaRepositoryContrato.kt`, `contrato/PlayerRepositoryContrato.kt`
- Modify: `infrastructure/build.gradle.kts`
- Test: `infra-test/memoria/EnigmaRepositoryEmMemoriaTest.kt`, `infra-test/memoria/PlayerRepositoryEmMemoriaTest.kt`

**Interfaces:**
- Consome: tarefas 2 e 4.
- Produz: as duas portas, os dois repositórios em memória e as classes abstratas `EnigmaRepositoryContrato(novo: () -> EnigmaRepository)` e `PlayerRepositoryContrato(novo: () -> PlayerRepository)`, publicadas como `testFixtures(project(":infrastructure"))`.

- [ ] **Passo 1: preparar o build**

Em `infrastructure/build.gradle.kts`: plugin `` `java-test-fixtures` ``; `testFixturesImplementation(project(":domain"))`, `testFixturesImplementation(libs.kotest.runner.junit5)` e `testFixturesImplementation(libs.kotest.assertions.core)`.

- [ ] **Passo 2: escrever as portas, os contratos e os testes**

As portas são as do "Modelo". `EnigmaRepositoryContrato.kt`:

```kotlin
abstract class EnigmaRepositoryContrato(novo: () -> EnigmaRepository) : FunSpec({
    fun conteudo(titulo: String = "Férias fatais") = ConteudoEnigma.de(titulo, "enunciado", "solução", DEBUG, MEDIA)
    fun enigma(n: Long, status: StatusEnigma = RASCUNHO) = Enigma.reconstituir(EnigmaId(UUID(3, n)), conteudo(), MANUAL, status)
    fun Enigma.campos() = listOf(id, origem, conteudo, status)

    test("CA-25 buscar enigma inexistente devolve nulo") {
        novo().buscar(EnigmaId(UUID(3, 99))) shouldBe null
    }
    test("CA-26 salvar insere e, com o mesmo ID, substitui") {
        val repo = novo()
        val e1 = enigma(1)
        repo.salvar(e1)
        repo.buscar(e1.id)!!.campos() shouldBe e1.campos()
        repo.salvar(enigma(1).apply { revisar(conteudo("Férias fatais, revisado")) })
        repo.buscar(e1.id)!!.conteudo.titulo shouldBe "Férias fatais, revisado"
    }
    test("CA-27 mudança sem salvar não chega ao repositório") {
        val repo = novo()
        val e1 = enigma(1)
        repo.salvar(e1)
        e1.publicar()
        repo.buscar(e1.id)!!.status shouldBe RASCUNHO
        repo.buscar(e1.id)!!.publicar()
        repo.buscar(e1.id)!!.status shouldBe RASCUNHO
    }
    test("CA-28 publicados devolve só os publicados") {
        val repo = novo()
        listOf(enigma(1, RASCUNHO), enigma(2, PUBLICADO), enigma(3, ARQUIVADO)).forEach(repo::salvar)
        repo.publicados().map { it.id } shouldBe listOf(EnigmaId(UUID(3, 2)))
    }
})
```

`PlayerRepositoryContrato.kt`:

```kotlin
abstract class PlayerRepositoryContrato(novo: () -> PlayerRepository) : FunSpec({
    val id = PlayerId(UUID(0, 2))

    test("CA-25 buscar jogador inexistente devolve nulo") {
        novo().buscar(id) shouldBe null
    }
    test("CA-29 salvar insere e, com o mesmo ID, substitui") {
        val repo = novo()
        val ana = Player.criar(id, "Ana", HUMANO)
        repo.salvar(ana)
        repo.buscar(id) shouldBe ana
        val outra = Player.reconstituir(id, "Ana Maria", HUMANO, EstatisticasJogador(3, 1))
        repo.salvar(outra)
        repo.buscar(id) shouldBe outra
    }
})
```

Testes concretos:

```kotlin
class EnigmaRepositoryEmMemoriaTest : EnigmaRepositoryContrato({ EnigmaRepositoryEmMemoria() }) {
    init {
        test("RNF4 salvar de várias threads não perde enigma") {
            val repo = EnigmaRepositoryEmMemoria()
            val pool = Executors.newFixedThreadPool(8)
            (1L..800L).map { n ->
                pool.submit { repo.salvar(Enigma.reconstituir(EnigmaId(UUID(4, n)), ConteudoEnigma.de("t", "e", "s", DEBUG, MEDIA), MANUAL, PUBLICADO)) }
            }.forEach { it.get() }
            pool.shutdown()
            repo.publicados() shouldHaveSize 800
        }
    }
}

class PlayerRepositoryEmMemoriaTest : PlayerRepositoryContrato({ PlayerRepositoryEmMemoria() })
```

- [ ] **Passo 3: rodar e ver falhar**

Run: `./gradlew :infrastructure:test`
Expected: FAIL na compilação, `Unresolved reference: EnigmaRepositoryEmMemoria`.

- [ ] **Passo 4: implementar os repositórios em memória**

DP8: `private val dados = LinkedHashMap<EnigmaId, Enigma>()`; cada método dentro de `synchronized(dados)`; `salvar` guarda `Enigma.reconstituir(id, conteudo, origem, status)`; `buscar` devolve outra cópia; `publicados()` filtra e copia. `PlayerRepositoryEmMemoria` igual, sem cópia (o `Player` é imutável).

- [ ] **Passo 5: rodar e ver passar**

Run: `./gradlew :domain:test :infrastructure:test`
Expected: PASS, com os CAs 25 a 29 listados sob as classes `EmMemoriaTest`.

- [ ] **Passo 6: commit**

```bash
git add domain/src/main/kotlin/com/quebugesse/domain/enigma/port domain/src/main/kotlin/com/quebugesse/domain/player/port infrastructure/build.gradle.kts infrastructure/src
git commit -m "S02: cria portas e repositórios em memória com suíte de contrato"
```

---

### Tarefa 6: seed

**Files:**
- Create: `infra/seed/SeedInicial.kt`
- Test: `infra-test/seed/SeedInicialTest.kt`
- Read: `docs/enigmas/seed.md` (fonte do texto)

**Interfaces:**
- Consome: tarefa 5.
- Produz: `SeedInicial.JOGADOR_IA`, `SeedInicial.ENIGMAS`, `SeedInicial.carregar(enigmas, jogadores)`. A S03 usa `JOGADOR_IA` como narrador do solo; a S05 chama `carregar` ao subir.

- [ ] **Passo 1: escrever os testes**

```kotlin
private val ID_IA = PlayerId(UUID.fromString("00000000-0000-0000-0000-0000000000a1"))

class SeedInicialTest : FunSpec({
    test("CA-30 seed grava o jogador IA e um enigma publicado por categoria") {
        val enigmas = EnigmaRepositoryEmMemoria()
        val jogadores = PlayerRepositoryEmMemoria()
        SeedInicial.carregar(enigmas, jogadores)
        jogadores.buscar(ID_IA)!!.tipo shouldBe IA
        val publicados = enigmas.publicados()
        publicados.map { it.origem }.toSet() shouldBe setOf(MANUAL)
        publicados.associate { it.conteudo.categoria to it.conteudo.dificuldade } shouldBe mapOf(
            DEBUG to MEDIA, SINTAXE to DIFICIL, GESTAO_PROJETO to FACIL, DESIGN to MEDIA, DISCOVERY to FACIL,
        )
        publicados shouldHaveSize 5
    }
    test("CA-31 carregar de novo não duplica nem desfaz mudanças") {
        val enigmas = EnigmaRepositoryEmMemoria()
        val jogadores = PlayerRepositoryEmMemoria()
        SeedInicial.carregar(enigmas, jogadores)
        val primeiro = SeedInicial.ENIGMAS.first().id
        enigmas.salvar(enigmas.buscar(primeiro)!!.apply { arquivar() })
        SeedInicial.carregar(enigmas, jogadores)
        enigmas.buscar(primeiro)!!.status shouldBe ARQUIVADO
        enigmas.publicados() shouldHaveSize 4
    }
    test("CA-32 textos do seed são os de docs/enigmas/seed.md") {
        val md = File("../docs/enigmas/seed.md").readText()   // diretório de trabalho do teste: infrastructure/
        SeedInicial.ENIGMAS.forEach { e ->
            listOf(e.conteudo.titulo, e.conteudo.enunciado, e.conteudo.solucaoCompleta).forEach { texto ->
                withClue(e.conteudo.titulo) { md shouldContain texto }
            }
        }
    }
})
```

- [ ] **Passo 2: rodar e ver falhar**

Run: `./gradlew :infrastructure:test --tests "com.quebugesse.infrastructure.persistence.seed.SeedInicialTest"`
Expected: FAIL na compilação, `Unresolved reference: SeedInicial`.

- [ ] **Passo 3: implementar `SeedInicial.kt`**

- `JOGADOR_IA = PlayerId(UUID.fromString("00000000-0000-0000-0000-0000000000a1"))`, nome "Narrador IA", tipo `IA`.
- `ENIGMAS`: `get()` que cria a cada chamada os cinco enigmas de `seed.md`, com IDs `00000000-0000-0000-0000-0000000000e1` a `...0e5`, via `Enigma.reconstituir(id, ConteudoEnigma.de(...), MANUAL, PUBLICADO)`. Copiar título, enunciado, solução e tags de `seed.md` sem mudar uma letra; no enigma de SINTAXE, escrever `\\u000a` na string Kotlin.
- `carregar`: para cada item, `salvar` só se `buscar(id) == null`.

- [ ] **Passo 4: rodar e ver passar**

Run: `./gradlew :infrastructure:test`
Expected: PASS.

- [ ] **Passo 5: commit**

```bash
git add infrastructure/src/main/kotlin/com/quebugesse/infrastructure/persistence/seed infrastructure/src/test/kotlin/com/quebugesse/infrastructure/persistence/seed
git commit -m "S02: carrega catálogo inicial e jogador IA"
```

---

### Tarefa 7: cobertura e diagramas

**Files:**
- Modify: `domain/build.gradle.kts`, `infrastructure/build.gradle.kts`
- Modify: `docs/diagramas/classes-dominio.md`, `docs/diagramas/sequencia-iniciar-partida.md`, `docs/diagramas/pacotes.md`, `specs/002-catalogo-e-jogador/spec.md` (seção "Divergências")

**Interfaces:**
- Consome: o módulo completo.
- Produz: nada.

- [ ] **Passo 1: exigir cobertura no Kover**

- `domain/build.gradle.kts`: acrescentar `"com.quebugesse.domain.enigma"` e `"com.quebugesse.domain.player"` ao `packages(...)` do filtro da S01 (ou criar o bloco da tarefa 9 da S01, se ela ainda não existir), com `minBound(90)`.
- `infrastructure/build.gradle.kts`: o mesmo bloco, com `packages("com.quebugesse.infrastructure.persistence")`.

Run: `./gradlew :domain:koverVerify :infrastructure:koverVerify :domain:koverLog :infrastructure:koverLog`
Expected: `BUILD SUCCESSFUL` e linhas ≥ 90%. Abaixo disso, escrever o teste que falta; nunca tirar código do filtro.

- [ ] **Passo 2: atualizar as fontes dos diagramas (P9)**

- `classes-dominio.md`: em `domain.enigma`, `ConteudoEnigma` (D2) com `solucaoCompleta` pública (D3), `status` e `origem` com seus enums (D1), `CriterioDeSelecao` (D7) e a nota de D5; em `domain.player`, a restrição de `EstatisticasJogador` (D8) e a nota "sem comandos até a S12" (D6); as portas `EnigmaRepository` e `PlayerRepository` com `«port»` nos pacotes `.port` (D4).
- `sequencia-iniciar-partida.md`: passo 03 vira `publicados()`, seguido de `elegiveis(...)` e do sorteio na aplicação (D5).
- `pacotes.md`: `persistence.memoria` e `persistence.seed` (D9) e os subpacotes `port` (D4).
- Cada mudança marcada com o comentário `%% S02 Dn`.

Verificar: os três diagramas renderizam sem erro no preview Mermaid do GitHub ou do editor.

- [ ] **Passo 3: registrar na spec**

Logo abaixo da tabela de "Divergências em relação aos diagramas", acrescentar: "Fontes Mermaid atualizadas em `docs/diagramas/` (tarefa 7). Os PDFs em `diagramas/` seguem desatualizados até serem gerados de novo a partir delas."

- [ ] **Passo 4: rodar tudo**

Run: `./gradlew build`
Expected: `BUILD SUCCESSFUL`, inclusive o teste de arquitetura da S00.

Run (Git Bash): `grep -rhoE '"CA-[0-9]{2}' domain/src/test/kotlin/com/quebugesse/domain/enigma domain/src/test/kotlin/com/quebugesse/domain/player infrastructure/src | sort -u | wc -l`
Expected: `31` (todos menos o CA-15, que vive dentro de `deveRejeitar`).

- [ ] **Passo 5: commit**

```bash
git add domain/build.gradle.kts infrastructure/build.gradle.kts docs/diagramas/classes-dominio.md docs/diagramas/sequencia-iniciar-partida.md docs/diagramas/pacotes.md specs/002-catalogo-e-jogador/spec.md
git commit -m "S02: exige cobertura e atualiza diagramas"
```

---

## Cobertura dos CAs

| Tarefa | CAs |
|---|---|
| 1 | 01, 02, 03, 04, 05, 06 (conteúdo), 07 |
| 2 | 06 (enigma), 08, 09, 10, 11, 12, 13, 14, 16 |
| 3 | 17, 18, 19, 20 |
| 4 | 21, 22, 23, 24 |
| 5 | 25, 26, 27, 28, 29, RNF4 |
| 6 | 30, 31, 32 |
| 7 | RNF1, RNF2, RNF3, D1 a D9 |

CA-15 vale em todo teste de rejeição do enigma, pela comparação de fotografias em `deveRejeitar`.
