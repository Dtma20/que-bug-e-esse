# Plano técnico: S01 Domínio da partida

Spec: [spec.md](spec.md) · Tarefas: [tasks.md](tasks.md) · Constituição 1.0.0: [docs/constitution.md](../../docs/constitution.md)

## Resumo

`GameSession` vira uma classe Kotlin com estado privado e um método por comando. Cada método verifica todas as guardas na ordem da spec antes de mudar qualquer coisa, aplica a mudança e acumula um evento. Perguntas e tentativas são valores imutáveis que a raiz substitui na própria lista. Nada fora da biblioteca padrão do Kotlin.

## Contexto técnico

| Item | Escolha |
|---|---|
| Linguagem | Kotlin na JVM; versões do Kotlin e do JDK (LTS) fixadas na S00 |
| Build | Gradle com Kotlin DSL, módulo `domain`, catálogo `gradle/libs.versions.toml` |
| Testes | Kotest: runner JUnit 5, `assertions-core`, `property` e testes por dados (`withData`) |
| Cobertura | Kover |
| Dependências de produção | Nenhuma além do Kotlin |

## Portão da constituição

| Princípio | Como a S01 cumpre |
|---|---|
| P1 Domínio puro | Sem dependências; `agora: Instant` em todo comando; IDs de agregado vêm de fora; IDs locais são sequenciais. A tarefa 9 adiciona um teste que proíbe `Instant.now(`, `Clock`, `UUID.randomUUID(` e `Random` em `src/main` |
| P2 Invariantes protegidas | `private set`; listas expostas como cópia somente leitura de valores imutáveis; guardas antes de qualquer mutação; testes de rejeição comparam fotografias |
| P3 Sigilo da solução | Não se aplica: a partida não guarda a solução |
| P4 Regras como testes | 32 CAs, cada teste nomeado pelo ID |
| P5 Teste primeiro | Cada tarefa de [tasks.md](tasks.md) começa pelo teste falhando |
| P6 Fakes | Não se aplica: a S01 não define portas |
| P7 Camadas | `domain.gamesession` só importa `domain.shared` |
| P8 ADR | Q1 a Q5 com decisão provisória registrada na spec |
| P9 Diagramas | A tarefa 9 atualiza as fontes Mermaid de `docs/diagramas/` com as divergências D1 a D10 |
| P10 Linguagem ubíqua | Nomes do diagrama mantidos; termos novos no glossário da spec |

## Decisões de projeto

**DP1. Rejeição por exceção tipada.** Comando inválido lança `PartidaException(violacao: Violacao)`. Alternativa descartada: `Result` ou `Either` (Arrow), que acrescentam dependência ou cerimônia sem ganho no domínio. A aplicação (S03) traduz `Violacao` em resposta HTTP ou mensagem de WebSocket.

**DP2. Guardas antes de mutação.** Todo comando tem duas partes: primeiro a sequência de verificações na ordem da seção "Ordem de verificação" da spec, depois as mutações. Nenhuma linha que muda estado aparece entre verificações. É isso que garante R3 sem transação.

**DP3. Entidades locais imutáveis.** `Question` e `Tentativa` são `data class` imutáveis. Responder, cancelar ou julgar troca o elemento da lista por uma cópia. Os getters `perguntas`, `tentativas` e `eventosPendentes` devolvem `toList()`, nunca a lista interna: sem isso, a fotografia dos testes seria uma visão viva e a verificação de R3 passaria sempre.

**DP4. Identidade local.** `QuestionId` e `TentativaId` envolvem um `Int` igual ao tamanho da lista mais um. `GameSessionId`, `EnigmaId` e `PlayerId` envolvem `UUID` e são gerados pela aplicação.

**DP5. Eventos acumulados na raiz.** `eventosPendentes` permite ler sem consumir; `puxarEventos()` devolve e esvazia. A aplicação puxa depois de salvar (H6, S04). `DomainEvent` fica em `shared` como interface comum; os eventos da partida ficam sob `sealed interface EventoDePartida : DomainEvent` em `gamesession`, porque o Kotlin exige que subtipos diretos de uma interface selada estejam no mesmo pacote.

**DP6. Fase derivada.** `fase`, `perguntaAberta` e `propostaPendente` são calculadas a partir de `status`, `perguntas` e `tentativas`, sem campo próprio. Assim I7 vale por construção.

**DP7. Criar sem validar.** `criar` aceita qualquer configuração e copia a lista de investigadores; H5 é verificada em `iniciar`, como no diagrama de estados. Isso deixa espaço para a sala de espera da S06.

**DP8. Texto.** `trim()`, depois `codePointCount` entre 1 e 500; o texto guardado é o aparado.

**DP9. Nome do parâmetro de tempo.** `agora` em todos os comandos, inclusive `aplicarVeredito` (o diagrama usa `instante`; ajustado por consistência, D1).

## Modelo

Assinaturas que as tarefas implementam. Corpos ficam para a implementação.

```kotlin
// com.quebugesse.domain.shared
@JvmInline value class GameSessionId(val value: UUID)
@JvmInline value class EnigmaId(val value: UUID)
@JvmInline value class PlayerId(val value: UUID)
@JvmInline value class QuestionId(val value: Int)
@JvmInline value class TentativaId(val value: Int)
data class PlayerRef(val id: PlayerId)
interface DomainEvent { val ocorridoEm: Instant }

// com.quebugesse.domain.gamesession
enum class GameStatus { AGUARDANDO_INICIO, EM_ANDAMENTO, RESOLVIDO, ABANDONADO }
enum class ModoDeJogo { SOLO, GRUPO }
enum class Answer { SIM, NAO, IRRELEVANTE }
enum class FaseDaInvestigacao { AGUARDANDO_PERGUNTA, PERGUNTA_PENDENTE, SOLUCAO_EM_AVALIACAO }
enum class SituacaoTentativa { PENDENTE, CORRETA, INCORRETA, RETIRADA }
data class Veredito(val correta: Boolean)

enum class Violacao {
    PARTIDA_ENCERRADA, PARTIDA_NAO_INICIADA, PARTIDA_JA_INICIADA, CONFIGURACAO_INVALIDA,
    NAO_E_INVESTIGADOR, NAO_E_NARRADOR, NAO_AUTORIZADO,
    TEXTO_INVALIDO, PERGUNTA_INEXISTENTE,
    PERGUNTA_JA_FECHADA, PERGUNTA_EM_ABERTO, PROPOSTA_PENDENTE, SEM_PROPOSTA_PENDENTE,
}
class PartidaException(val violacao: Violacao) : RuntimeException(violacao.name)

data class Question(
    val id: QuestionId, val autor: PlayerRef, val texto: String, val criadaEm: Instant,
    val resposta: Answer? = null, val cancelada: Boolean = false,
) { val aberta: Boolean get() = resposta == null && !cancelada }

data class Tentativa(
    val id: TentativaId, val autor: PlayerRef, val texto: String, val criadaEm: Instant,
    val situacao: SituacaoTentativa = SituacaoTentativa.PENDENTE,
)

sealed interface EventoDePartida : DomainEvent { val partidaId: GameSessionId }
data class PartidaIniciada(override val partidaId: GameSessionId, override val ocorridoEm: Instant) : EventoDePartida
data class PerguntaFeita(override val partidaId: GameSessionId, override val ocorridoEm: Instant,
    val perguntaId: QuestionId, val autor: PlayerRef, val texto: String) : EventoDePartida
data class PerguntaRespondida(override val partidaId: GameSessionId, override val ocorridoEm: Instant,
    val perguntaId: QuestionId, val resposta: Answer) : EventoDePartida
data class PerguntaCancelada(override val partidaId: GameSessionId, override val ocorridoEm: Instant,
    val perguntaId: QuestionId, val canceladaPor: PlayerRef) : EventoDePartida
data class SolucaoProposta(override val partidaId: GameSessionId, override val ocorridoEm: Instant,
    val tentativaId: TentativaId, val autor: PlayerRef, val texto: String) : EventoDePartida
data class PropostaRetirada(override val partidaId: GameSessionId, override val ocorridoEm: Instant,
    val tentativaId: TentativaId, val retiradaPor: PlayerRef) : EventoDePartida
data class SolucaoRecusada(override val partidaId: GameSessionId, override val ocorridoEm: Instant,
    val tentativaId: TentativaId) : EventoDePartida
data class PartidaResolvida(override val partidaId: GameSessionId, override val ocorridoEm: Instant,
    val tentativaId: TentativaId) : EventoDePartida
data class PartidaAbandonada(override val partidaId: GameSessionId, override val ocorridoEm: Instant,
    val abandonadaPor: PlayerRef) : EventoDePartida

class GameSession private constructor(
    val id: GameSessionId, val enigmaId: EnigmaId, val modo: ModoDeJogo,
    val narrador: PlayerRef, val investigadores: List<PlayerRef>, val criadaEm: Instant,
) {
    var status: GameStatus; private set
    var resolvidaEm: Instant?; private set
    val perguntas: List<Question>
    val tentativas: List<Tentativa>
    val perguntaAberta: Question?
    val propostaPendente: Tentativa?
    val fase: FaseDaInvestigacao?
    val eventosPendentes: List<EventoDePartida>
    fun puxarEventos(): List<EventoDePartida>

    fun iniciar(agora: Instant)
    fun fazerPergunta(autor: PlayerRef, texto: String, agora: Instant): Question
    fun responderPergunta(autor: PlayerRef, id: QuestionId, resposta: Answer, agora: Instant)
    fun cancelarPergunta(autor: PlayerRef, id: QuestionId, agora: Instant)
    fun proporSolucao(autor: PlayerRef, texto: String, agora: Instant): Tentativa
    fun retirarProposta(autor: PlayerRef, agora: Instant)
    fun aplicarVeredito(veredito: Veredito, agora: Instant)
    fun abandonar(autor: PlayerRef, agora: Instant)

    companion object {
        fun criar(id: GameSessionId, enigmaId: EnigmaId, modo: ModoDeJogo, narrador: PlayerRef,
                  investigadores: List<PlayerRef>, agora: Instant): GameSession
    }
}
```

Forma de cada comando (DP2), com auxiliares privados `exigir(condicao: Boolean, violacao: Violacao)`, `exigirNaoEncerrada()`, `exigirIniciada()` e `textoValido(texto: String): String`:

```kotlin
fun fazerPergunta(autor: PlayerRef, texto: String, agora: Instant): Question {
    exigirNaoEncerrada()
    exigirIniciada()
    exigir(autor in investigadores, Violacao.NAO_E_INVESTIGADOR)
    val aparado = textoValido(texto)
    exigir(perguntaAberta == null, Violacao.PERGUNTA_EM_ABERTO)
    exigir(propostaPendente == null, Violacao.PROPOSTA_PENDENTE)
    // daqui para baixo: mutações e evento
}
```

## Estrutura de arquivos

```
settings.gradle.kts                       # include("domain"), se a S00 ainda não existir
gradle/libs.versions.toml
domain/
  build.gradle.kts
  src/main/kotlin/com/quebugesse/domain/
    shared/Ids.kt                         # value classes de ID
    shared/PlayerRef.kt
    shared/DomainEvent.kt
    gamesession/Enums.kt                  # GameStatus, ModoDeJogo, Answer, FaseDaInvestigacao, SituacaoTentativa
    gamesession/Veredito.kt
    gamesession/Violacao.kt               # Violacao e PartidaException
    gamesession/Question.kt
    gamesession/Tentativa.kt
    gamesession/EventosDePartida.kt
    gamesession/GameSession.kt
  src/test/kotlin/com/quebugesse/domain/
    gamesession/Fixtures.kt               # participantes, instantes, cenários prontos
    gamesession/Fotografia.kt             # fotografia() e deveRejeitar()
    gamesession/IniciarPartidaTest.kt
    gamesession/PerguntasTest.kt
    gamesession/CancelarPerguntaTest.kt
    gamesession/PropostasTest.kt
    gamesession/RetirarPropostaTest.kt
    gamesession/AbandonoTest.kt
    gamesession/EstadosEPrecedenciaTest.kt
    gamesession/InvariantesPropertyTest.kt
    DominioPuroTest.kt
docs/diagramas/
  estados-partida.md
  classes-dominio.md
```

## Estratégia de testes

- **Fixtures.** Participantes `N`, `ANA`, `BETO`, `ZE`; instante `T0` e `t(n)` (n minutos depois); `novaPartida(...)` e cenários prontos (`emAndamento`, `comPerguntaAberta`, `comPropostaPendente`, `resolvida`, `abandonada`) com os eventos já retirados, para que cada teste veja só os seus.
- **Rejeição.** `deveRejeitar(violacao) { comando }` confere a violação e compara a fotografia (status, fase, `resolvidaEm`, perguntas, tentativas, eventos pendentes) antes e depois. Cobre CA-27 em todo teste de rejeição.
- **Tabelas.** `withData` para H1 (8 comandos × 2 estados terminais) e para comandos antes do início.
- **Propriedade.** Comandos aleatórios (autor entre os quatro participantes, IDs de 1 a 3, texto vazio ou válido, veredito verdadeiro ou falso), até 50 por sequência, 1000 sequências. Depois de cada comando: R3 se foi rejeitado e I1 a I8 sempre. Na falha o Kotest mostra a sequência reduzida e a seed; a sequência vira teste de regressão.
- **Cobertura.** `koverVerify` com mínimo de 90% de linhas em `com.quebugesse.domain.gamesession`.

## Riscos

| Risco | Mitigação |
|---|---|
| A equipe rejeita alguma decisão provisória (Q1 a Q5) | A spec lista os CAs afetados; cada regra mora numa guarda só |
| Código e diagramas divergem | Tarefa 9 atualiza as fontes Mermaid antes de fechar a spec |
| Teste de propriedade lento ou difícil de reproduzir | 1000 sequências de até 50 comandos; seed impressa na falha |
| A S00 ainda não existe | A tarefa 1 cria o mínimo do módulo `domain` |

## Fora deste plano

`GameSession.reconstituir` vem com a S03 (emenda 2), porque o repositório em memória precisa dela para guardar cópias. Versão para bloqueio otimista e serialização dos eventos ficam para a S06 (DTOs de evento) e a S09 (banco e outbox).
