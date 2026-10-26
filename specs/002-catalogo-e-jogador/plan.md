# Plano técnico: S02 Catálogo de enigmas e jogador

Spec: [spec.md](spec.md) · Tarefas: [tasks.md](tasks.md) · Constituição 1.0.0: [docs/constitution.md](../../docs/constitution.md)

## Resumo

`Enigma` vira uma classe com `private set` em `conteudo` e `status` e três comandos que verificam o status antes de mudar. O conteúdo é um valor imutável criado só pela fábrica `ConteudoEnigma.de`, que normaliza e valida. A seleção é uma função pura sobre uma lista. `Player` é imutável. As portas ficam em `domain.<agregado>.port`; os repositórios em memória e o seed ficam em `infrastructure.persistence`, com suítes de contrato em `testFixtures` para a S09 reaproveitar.

## Contexto técnico

| Item | Escolha |
|---|---|
| Linguagem e build | Os da S00: Kotlin JVM, Gradle com Kotlin DSL, plugin de convenção `quebugesse.kotlin-module` |
| Testes | Kotest: `FunSpec`, `assertions-core`, `withData` |
| Fixtures de teste compartilhadas | Plugin `java-test-fixtures` do Gradle no módulo `infrastructure` |
| Cobertura | Kover |
| Dependências de produção | `domain`: nenhuma além do Kotlin. `infrastructure`: só `project(":domain")` |

## Portão da constituição

| Princípio | Como a S02 cumpre |
|---|---|
| P1 Domínio puro | Sem relógio nem aleatoriedade: a seleção não sorteia (EL2) e os IDs chegam prontos. O `DominioPuroTest` da S01 e a regra de arquitetura da S00 já varrem todo `domain/src/main` |
| P2 Invariantes protegidas | `private set` em `Enigma`; construtores privados em `ConteudoEnigma` e `Player`, com validação no `init`; `deveRejeitar` compara fotografias |
| P3 Sigilo da solução | `toString` sem a solução (E8, CA-06). A visão por papel fica na S03 |
| P4 Regras como testes | 32 CAs, cada teste nomeado pelo ID |
| P5 Teste primeiro | Cada tarefa de [tasks.md](tasks.md) começa pelo teste falhando |
| P6 Fakes | `EnigmaRepositoryEmMemoria` e `PlayerRepositoryEmMemoria`, com suíte de contrato |
| P7 Camadas | `enigma` e `player` só importam `shared`; `port` importa só o próprio agregado; `infrastructure` só importa `domain` |
| P8 ADR | Q1 a Q6 com decisão provisória na spec |
| P9 Diagramas | A tarefa 7 atualiza as fontes Mermaid com D1 a D9 |
| P10 Linguagem ubíqua | `Enigma`, `Player`, `PlayerType`, `EstatisticasJogador`, `partidasJogadas` e `partidasResolvidas` como no diagrama; termos novos no glossário |

## Decisões de projeto

**DP1. Rejeição por exceção tipada, uma por agregado.** `EnigmaException(ViolacaoEnigma)` e `JogadorException(ViolacaoJogador)`, no mesmo padrão da `PartidaException` da S01. Cada agregado tem a sua para não criar dependência entre pacotes (P7).

**DP2. Conteúdo com construtor privado.** `ConteudoEnigma` é `data class` com construtor privado e `@ConsistentCopyVisibility` (o `copy` fica privado também). A única entrada é `de(...)`, que apara os textos, normaliza as tags numa coleção nova e chama o construtor; o `init` valida os valores já aparados. Assim nenhum conteúdo inválido existe, nem com tags apontando para uma lista que quem chamou ainda pode mudar (CA-05).

**DP3. Revisão troca o conteúdo inteiro.** `revisar(novo: ConteudoEnigma)` em vez de um método por campo: a validação mora num lugar só e uma revisão nunca fica pela metade.

**DP4. `toString` sem a solução.** `ConteudoEnigma.toString()` lista título, categoria, dificuldade e tags; `Enigma.toString()` acrescenta ID, origem e status. O `toString` gerado pela `data class` mostraria a solução em log e em mensagem de teste.

**DP5. Enigma com igualdade de identidade.** `Enigma` é `class`, sem `equals` próprio. Testes e repositórios comparam a fotografia (ID, origem, conteúdo, status). `Player` é `data class` com construtor privado, porque é imutável nesta spec; a S12, que lhe dá comandos, revê isso.

**DP6. Seleção pura e ordenada.** `elegiveis` filtra a lista recebida sem reordenar (EL2). O resultado é determinístico; a aplicação sorteia com um sorteador injetado (S03).

**DP7. Portas em `domain.<agregado>.port`** (M16). Para a regra entre agregados da S00, `domain.enigma.port` pertence ao agregado `enigma`.

**DP8. Repositório em memória guarda cópia.** Na entrada e na saída, `Enigma.reconstituir(id, conteudo, origem, status)`; `ConteudoEnigma` e `Player` são imutáveis e podem ser compartilhados. O mapa interno é um `LinkedHashMap` acessado dentro de `synchronized` (RNF4). `publicados()` devolve uma lista nova.

**DP9. Suítes de contrato abstratas em `testFixtures`.** `abstract class EnigmaRepositoryContrato(novo: () -> EnigmaRepository) : FunSpec({ ... })`; o Kotest não instancia classe abstrata, então a suíte só roda pelas subclasses (`EnigmaRepositoryEmMemoriaTest` agora, a do banco na S09). Cada teste chama `novo()` para começar com repositório vazio.

**DP10. Seed em Kotlin, idempotente por ID.** `SeedInicial.carregar` usa IDs fixos e grava cada item só se `buscar` devolver nulo (SD2). O texto fica em constantes Kotlin; um teste confere que ele aparece igual em `docs/enigmas/seed.md` (SD3), para a revisão da equipe e o código não se separarem.

## Modelo

Assinaturas que as tarefas implementam. Corpos ficam para a implementação.

```kotlin
// com.quebugesse.domain.shared (da S01; a S02 só usa)
@JvmInline value class EnigmaId(val value: UUID)
@JvmInline value class PlayerId(val value: UUID)

// com.quebugesse.domain.enigma
enum class Categoria { SINTAXE, DEBUG, GESTAO_PROJETO, DESIGN, DISCOVERY }
enum class Dificuldade { FACIL, MEDIA, DIFICIL }
enum class OrigemEnigma { MANUAL, IA }
enum class StatusEnigma { RASCUNHO, PUBLICADO, ARQUIVADO }

enum class ViolacaoEnigma {
    TITULO_INVALIDO, ENUNCIADO_INVALIDO, SOLUCAO_INVALIDA, TRANSICAO_INVALIDA, ENIGMA_NAO_EDITAVEL,
}
class EnigmaException(val violacao: ViolacaoEnigma) : RuntimeException(violacao.name)

@ConsistentCopyVisibility
data class ConteudoEnigma private constructor(
    val titulo: String, val enunciado: String, val solucaoCompleta: String,
    val categoria: Categoria, val dificuldade: Dificuldade, val tags: Set<String>,
) {
    init { /* E1 sobre os valores já aparados, na ordem título, enunciado, solução */ }
    override fun toString(): String   // sem solucaoCompleta (E8)

    companion object {
        const val MAX_TITULO = 100
        const val MAX_ENUNCIADO = 2000
        const val MAX_SOLUCAO = 2000
        fun de(
            titulo: String, enunciado: String, solucaoCompleta: String,
            categoria: Categoria, dificuldade: Dificuldade, tags: Collection<String> = emptyList(),
        ): ConteudoEnigma
    }
}

class Enigma private constructor(
    val id: EnigmaId, val origem: OrigemEnigma, conteudo: ConteudoEnigma, status: StatusEnigma,
) {
    var conteudo: ConteudoEnigma = conteudo; private set
    var status: StatusEnigma = status; private set

    fun revisar(novo: ConteudoEnigma)
    fun publicar()
    fun arquivar()
    override fun toString(): String   // sem solucaoCompleta (E8)

    companion object {
        fun criar(id: EnigmaId, conteudo: ConteudoEnigma, origem: OrigemEnigma): Enigma
        fun reconstituir(id: EnigmaId, conteudo: ConteudoEnigma, origem: OrigemEnigma, status: StatusEnigma): Enigma
    }
}

data class CriterioDeSelecao(val categoria: Categoria? = null, val dificuldade: Dificuldade? = null) {
    fun elegiveis(candidatos: List<Enigma>, jaJogados: Set<EnigmaId>): List<Enigma>
}

// com.quebugesse.domain.enigma.port
interface EnigmaRepository {
    fun buscar(id: EnigmaId): Enigma?
    fun salvar(enigma: Enigma)
    fun publicados(): List<Enigma>
}

// com.quebugesse.domain.player
enum class PlayerType { HUMANO, IA }
enum class ViolacaoJogador { NOME_INVALIDO, ESTATISTICAS_INVALIDAS }
class JogadorException(val violacao: ViolacaoJogador) : RuntimeException(violacao.name)

data class EstatisticasJogador(val partidasJogadas: Int = 0, val partidasResolvidas: Int = 0) {
    init { /* J2 */ }
}

@ConsistentCopyVisibility
data class Player private constructor(
    val id: PlayerId, val nome: String, val tipo: PlayerType, val stats: EstatisticasJogador,
) {
    init { /* J1 sobre o nome já aparado */ }

    companion object {
        const val MAX_NOME = 40
        fun criar(id: PlayerId, nome: String, tipo: PlayerType): Player
        fun reconstituir(id: PlayerId, nome: String, tipo: PlayerType, stats: EstatisticasJogador): Player
    }
}

// com.quebugesse.domain.player.port
interface PlayerRepository {
    fun buscar(id: PlayerId): Player?
    fun salvar(player: Player)
}

// com.quebugesse.infrastructure.persistence.memoria
class EnigmaRepositoryEmMemoria : EnigmaRepository
class PlayerRepositoryEmMemoria : PlayerRepository

// com.quebugesse.infrastructure.persistence.seed
object SeedInicial {
    val JOGADOR_IA: PlayerId          // 00000000-0000-0000-0000-0000000000a1
    val ENIGMAS: List<Enigma>         // IDs ...0e1 a ...0e5, na ordem de seed.md
    fun carregar(enigmas: EnigmaRepository, jogadores: PlayerRepository)
}
```

`Player.criar` e `reconstituir` aparam o nome antes de chamar o construtor; `criar` usa `EstatisticasJogador()`. Como `ENIGMAS` devolve objetos mutáveis, `carregar` grava cópias novas a cada chamada.

## Estrutura de arquivos

```
domain/src/main/kotlin/com/quebugesse/domain/
  enigma/Enums.kt                     # Categoria, Dificuldade, OrigemEnigma, StatusEnigma
  enigma/ViolacaoEnigma.kt            # ViolacaoEnigma e EnigmaException
  enigma/ConteudoEnigma.kt
  enigma/Enigma.kt
  enigma/CriterioDeSelecao.kt
  enigma/port/EnigmaRepository.kt
  player/PlayerType.kt
  player/ViolacaoJogador.kt           # ViolacaoJogador e JogadorException
  player/EstatisticasJogador.kt
  player/Player.kt
  player/port/PlayerRepository.kt
domain/src/test/kotlin/com/quebugesse/domain/
  enigma/FixturesEnigma.kt            # conteúdos e enigmas de exemplo
  enigma/FotografiaEnigma.kt          # fotografia() e deveRejeitar()
  enigma/ConteudoEnigmaTest.kt
  enigma/CicloDeVidaEnigmaTest.kt
  enigma/CriterioDeSelecaoTest.kt
  player/PlayerTest.kt
infrastructure/build.gradle.kts       # + java-test-fixtures
infrastructure/src/main/kotlin/com/quebugesse/infrastructure/persistence/
  memoria/EnigmaRepositoryEmMemoria.kt
  memoria/PlayerRepositoryEmMemoria.kt
  seed/SeedInicial.kt
infrastructure/src/testFixtures/kotlin/com/quebugesse/infrastructure/persistence/contrato/
  EnigmaRepositoryContrato.kt
  PlayerRepositoryContrato.kt
infrastructure/src/test/kotlin/com/quebugesse/infrastructure/persistence/
  memoria/EnigmaRepositoryEmMemoriaTest.kt
  memoria/PlayerRepositoryEmMemoriaTest.kt
  seed/SeedInicialTest.kt
docs/enigmas/seed.md                  # já existe; revisado pela equipe
docs/diagramas/classes-dominio.md     # atualizado na tarefa 7
docs/diagramas/sequencia-iniciar-partida.md
docs/diagramas/pacotes.md
```

## Estratégia de testes

- **Fixtures.** `conteudo(...)` com parâmetros nomeados e valores padrão válidos; IDs `E1`, `E2`, `E3` (`EnigmaId(UUID(3, n))`); `rascunho()`, `publicado()`, `arquivado()`.
- **Rejeição.** `Enigma.deveRejeitar(violacao) { comando }` confere a violação e compara a fotografia antes e depois (CA-15), como na S01.
- **Tabelas.** `withData` para os textos inválidos de cada campo e para os status que rejeitam cada comando.
- **Contrato.** Uma suíte por porta, em `testFixtures`, nomeada pelos CAs 25 a 29. A subclasse em memória só passa a fábrica.
- **Seed.** Repositórios em memória reais; o teste de SD3 lê `../docs/enigmas/seed.md` a partir do diretório do módulo.
- **Cobertura.** `koverVerify` com mínimo de 90% de linhas nos pacotes da RNF2.

## Riscos

| Risco | Mitigação |
|---|---|
| `@ConsistentCopyVisibility` ausente na versão do Kotlin fixada na S00 | A anotação existe desde o Kotlin 2.0.20; a S00 escolhe versão mais nova. Sem ela, trocar a `data class` por `class` com `equals` e `hashCode` escritos à mão |
| `\u000a` no texto do seed vira quebra de linha no Kotlin | Em string Kotlin, `\u000a` é escape; escrever `\\u000a`. O CA-32 pega a diferença |
| `testFixtures` não enxerga o `domain` | Declarar `testFixturesImplementation(project(":domain"))`; a dependência `implementation` do código principal não passa para as fixtures |
| A equipe muda o texto do seed | O CA-32 falha até o Kotlin acompanhar o `seed.md` |
| A S02 é implementada antes da S01 | A tarefa 1 cria `shared/Ids.kt` só com `EnigmaId` e `PlayerId`, e a S01 completa o arquivo |

## Fora deste plano

Casos de uso de curadoria (S11), sorteio e "já jogados" (S03), estatísticas (S12), repositórios em banco (S09).
