# Plano técnico: S03 Casos de uso da partida

Spec: [spec.md](spec.md) · Tarefas: [tasks.md](tasks.md) · Constituição 1.0.0: [docs/constitution.md](../../docs/constitution.md)

## Resumo

A S03 liga os agregados da S01 e da S02 em doze casos de uso na camada `application`, cria as portas da partida (`GameSessionRepository`, `Narrador`, `VerificadorDeSolucao`) em `domain.gamesession.port`, entrega o repositório em memória com suíte de contrato em `infrastructure` e fecha a fronteira de entrada/saída com DTOs puros, visões por papel (`VisaoNarrador` / `VisaoInvestigador`) e `CodigoErro`. Aplica também as emendas 1 e 2 à S01 (`aplicarVeredito` com autor, ADR 0001 A, e `GameSession.reconstituir`).

## Contexto técnico

| Item | Escolha |
|---|---|
| Repositório | `Que-Bug-E-Esse/back` (ADR 0009 A) |
| Linguagem e build | Kotlin/JVM (Temurin JDK 25 LTS), Gradle 9.8.2 com Kotlin DSL e plugin `quebugesse.kotlin-module` (S00) |
| Módulos tocados | `domain` (emendas 1–2 e portas), `application` (DTOs, mapeadores, 12 casos de uso), `infrastructure` (`GameSessionRepositoryEmMemoria` e `testFixtures` de contrato) |
| Testes | Kotest (`FunSpec`, `assertions-core`, `withData`) |
| Cobertura | Kover (≥ 90% em `com.quebugesse.application` e `com.quebugesse.infrastructure.persistence`) |
| Dependências de produção | `application` → `:domain`; `infrastructure` → `:domain`. Nos testes de `application`: `testImplementation(project(":infrastructure"))` e `testFixtures(project(":infrastructure"))` (Q4) |

## Portão da constituição

| Princípio | Como a S03 cumpre |
|---|---|
| P1 Domínio puro | Portas em `domain.gamesession.port` usam só tipos de `domain.gamesession`, `domain.shared` e stdlib; `Clock`, `GeradorDeIds` e `Sorteador` vivem em `application` |
| P2 Invariantes protegidas | `GameSession.reconstituir` valida I1–I8 e recusa estado corrompido (CA-26); `GameSessionRepositoryEmMemoria` salva e devolve cópias reconstituídas (CA-25); comandos rejeitados não chamam `salvar` (A5) |
| P3 Sigilo da solução | `VisaoInvestigador.solucaoCompleta` é `null` até `RESOLVIDO` ou `ABANDONADO`; `VisaoNarrador` só vai para o narrador da partida; CA-23 varre `toString()` de toda `VisaoInvestigador` antes do fim |
| P4 Regras como testes | 27 CAs da S03 + CA-33 da emenda 1 à S01, cada teste nomeado por `"CA-NN "` |
| P5 Teste primeiro | Cada tarefa de [tasks.md](tasks.md) começa pelos testes falhando (vermelho → verde → refatorar) |
| P6 Fakes | `GameSessionRepositoryEmMemoria` com suíte de contrato em `testFixtures`; `FakeNarrador` e `FakeVerificadorDeSolucao` programáveis (inclusive para lançar exceção e testar H4) |
| P7 Camadas | `ContextoDaNarracao` usa só `String` e `Answer` (sem importar `domain.enigma`); `application` não importa `infrastructure` em `src/main`; DTOs públicos não expõem tipos de `domain` (A10, S00 Q1) |
| P8 ADR | Segue ADRs aceitas: 0001 A (narrador humano julga no grupo, IA só no solo), 0002 A (`Narrador` síncrono só para IA), 0007 A (`RegistrarJogador` convidado), 0009 A (código em `back`) |
| P9 Diagramas | A tarefa 8 atualiza `docs/diagramas/` com as divergências D1 a D10 |
| P10 Linguagem ubíqua | `GameSession`, `Question`, `Answer`, `Player`, `Veredito`, `Tentativa`, `Narrador`, `Investigador` mantidos conforme glossário |

## Decisões de projeto

**DP1. Retorno explícito com `Resultado<T>`.** Todo caso de uso devolve `Resultado<T>` (`Sucesso(valor)` ou `Erro(codigo: CodigoErro)`), sem usar exceções para fluxos esperados de domínio ou validação. Erros de invariante interna (ex.: partida apontando para enigma ausente no repositório) continuam como `IllegalStateException` (conforme spec).

**DP2. Enums espelho e mapeamento exaustivo.** `application.dto` define enums espelho com os mesmos nomes dos enums do domínio (`ModoDeJogoDto`, `StatusPartidaDto`, `FaseDaInvestigacaoDto`, `RespostaDto`, `SituacaoTentativaDto`, `CategoriaDto`, `DificuldadeDto`). Funções `internal` em `Mapeadores.kt` usam `when` exaustivo (sem `else`), garantindo que qualquer novo valor quebre compilação ou o CA-24.

**DP3. `ContextoDaNarracao` sem acoplamento entre agregados.** `ContextoDaNarracao(enunciado, solucaoCompleta, historico)` fica em `domain.gamesession.port` carregando apenas `String` e `Answer` (`PerguntaRespondidaResumo(texto, resposta)`), evitando que `domain.gamesession` importe `domain.enigma` (P7).

**DP4. `MontarVisao` centraliza P3.** Uma única classe interna `MontarVisao(enigmas, jogadores)` transforma `(GameSession, solicitante)` em `Resultado<VisaoPartida>`: devolve `VisaoNarrador` (com `solucaoCompleta`) quando `solicitante == partida.narrador`, `VisaoInvestigador` (com `solucaoCompleta` apenas em `RESOLVIDO`/`ABANDONADO`, `null` antes) quando `solicitante in partida.investigadores`, e `NAO_PARTICIPA` caso contrário.

**DP5. Isolamento por cópia no repositório em memória.** `GameSessionRepositoryEmMemoria` usa `LinkedHashMap` protegido por `synchronized` (RNF4) e chama `GameSession.reconstituir(...)` tanto ao salvar quanto ao buscar, descartando eventos pendentes. `enigmasJogadosPor` percorre todas as partidas salvas (em qualquer status) e retorna os `EnigmaId` das partidas em que qualquer jogador do conjunto informado aparece em `investigadores` (ignorando quem só narrou, Q5).

**DP6. Infraestrutura pronta para a S04.** A execução de comando sobre partida existente segue um fluxo único (`buscar` → executar comando → `salvar` → montar visão) encapsulado em auxiliar interno, facilitando o encaixe de `PublicadorDeEventos` e `FALHA_AO_SALVAR` na S04 sem alterar assinaturas públicas de entrada/saída.

**DP7. `GeradorDeIds` e `Sorteador` na aplicação.** Interfaces pequenas em `com.quebugesse.application.port` injetadas por construtor. Nos testes, implementações determinísticas (IDs sequenciais e primeiro elemento); no `bootstrap` (S05), `UUID.randomUUID()` e `Random.Default`.

## Modelo

```kotlin
// Emendas em com.quebugesse.domain.gamesession.GameSession (S01)
fun aplicarVeredito(autor: PlayerRef, veredito: Veredito, agora: Instant)
companion object {
    fun reconstituir(
        id: GameSessionId, enigmaId: EnigmaId, modo: ModoDeJogo,
        narrador: PlayerRef, investigadores: List<PlayerRef>, criadaEm: Instant,
        status: GameStatus, resolvidaEm: Instant?,
        perguntas: List<Question>, tentativas: List<Tentativa>,
    ): GameSession
}

// com.quebugesse.domain.gamesession.port
interface GameSessionRepository {
    fun buscar(id: GameSessionId): GameSession?
    fun salvar(partida: GameSession)
    fun enigmasJogadosPor(jogadores: Set<PlayerRef>): Set<EnigmaId>
}

data class PerguntaRespondidaResumo(val texto: String, val resposta: Answer)
data class ContextoDaNarracao(
    val enunciado: String, val solucaoCompleta: String,
    val historico: List<PerguntaRespondidaResumo>,
) {
    override fun toString(): String // sem solucaoCompleta (P3)
}

interface Narrador {
    fun responder(pergunta: String, contexto: ContextoDaNarracao): Answer
}

interface VerificadorDeSolucao {
    fun avaliar(proposta: String, solucaoCompleta: String): Veredito
}

// com.quebugesse.application.port
interface GeradorDeIds {
    fun novoIdPartida(): GameSessionId
    fun novoIdJogador(): PlayerId
}
interface Sorteador {
    fun <T> escolher(itens: List<T>): T
}

// com.quebugesse.application.dto
sealed interface Resultado<out T> {
    data class Sucesso<T>(val valor: T) : Resultado<T>
    data class Erro(val codigo: CodigoErro) : Resultado<Nothing>
}

enum class ModoDeJogoDto { SOLO, GRUPO }
enum class StatusPartidaDto { AGUARDANDO_INICIO, EM_ANDAMENTO, RESOLVIDO, ABANDONADO }
enum class FaseDaInvestigacaoDto { AGUARDANDO_PERGUNTA, PERGUNTA_PENDENTE, SOLUCAO_EM_AVALIACAO }
enum class RespostaDto { SIM, NAO, IRRELEVANTE }
enum class SituacaoTentativaDto { PENDENTE, CORRETA, INCORRETA, RETIRADA }
enum class CategoriaDto { SINTAXE, DEBUG, GESTAO_PROJETO, DESIGN, DISCOVERY }
enum class DificuldadeDto { FACIL, MEDIA, DIFICIL }

enum class CodigoErro {
    PARTIDA_ENCERRADA, PARTIDA_NAO_INICIADA, PARTIDA_JA_INICIADA, CONFIGURACAO_INVALIDA,
    NAO_E_INVESTIGADOR, NAO_E_NARRADOR, NAO_AUTORIZADO, TEXTO_INVALIDO,
    PERGUNTA_INEXISTENTE, PERGUNTA_JA_FECHADA, PERGUNTA_EM_ABERTO,
    PROPOSTA_PENDENTE, SEM_PROPOSTA_PENDENTE,
    NOME_INVALIDO,
    PARTIDA_INEXISTENTE, JOGADOR_INEXISTENTE, SEM_ENIGMA_ELEGIVEL,
    MODO_INCOMPATIVEL, SEM_PERGUNTA_ABERTA, IA_INDISPONIVEL, NAO_PARTICIPA,
}

data class JogadorDto(val id: UUID, val nome: String)
data class ParticipanteDto(val id: UUID, val nome: String)
data class PerguntaDto(
    val id: Int, val autor: ParticipanteDto, val texto: String,
    val resposta: RespostaDto?, val cancelada: Boolean,
)
data class TentativaDto(
    val id: Int, val autor: ParticipanteDto, val texto: String,
    val situacao: SituacaoTentativaDto,
)

sealed interface VisaoPartida {
    val partidaId: UUID
    val modo: ModoDeJogoDto
    val status: StatusPartidaDto
    val fase: FaseDaInvestigacaoDto?
    val titulo: String
    val enunciado: String
    val categoria: CategoriaDto
    val dificuldade: DificuldadeDto
    val narrador: ParticipanteDto
    val investigadores: List<ParticipanteDto>
    val perguntas: List<PerguntaDto>
    val tentativas: List<TentativaDto>
}

data class VisaoNarrador(
    override val partidaId: UUID, override val modo: ModoDeJogoDto,
    override val status: StatusPartidaDto, override val fase: FaseDaInvestigacaoDto?,
    override val titulo: String, override val enunciado: String,
    override val categoria: CategoriaDto, override val dificuldade: DificuldadeDto,
    override val narrador: ParticipanteDto, override val investigadores: List<ParticipanteDto>,
    override val perguntas: List<PerguntaDto>, override val tentativas: List<TentativaDto>,
    val solucaoCompleta: String,
) : VisaoPartida

data class VisaoInvestigador(
    override val partidaId: UUID, override val modo: ModoDeJogoDto,
    override val status: StatusPartidaDto, override val fase: FaseDaInvestigacaoDto?,
    override val titulo: String, override val enunciado: String,
    override val categoria: CategoriaDto, override val dificuldade: DificuldadeDto,
    override val narrador: ParticipanteDto, override val investigadores: List<ParticipanteDto>,
    override val perguntas: List<PerguntaDto>, override val tentativas: List<TentativaDto>,
    val solucaoCompleta: String?,
) : VisaoPartida

// com.quebugesse.application.usecase (12 casos de uso)
class RegistrarJogador(jogadores: PlayerRepository, ids: GeradorDeIds) {
    fun executar(nome: String): Resultado<JogadorDto>
}
class IniciarPartida(
    partidas: GameSessionRepository, enigmas: EnigmaRepository, jogadores: PlayerRepository,
    ids: GeradorDeIds, sorteador: Sorteador, clock: Clock, jogadorIaId: PlayerId,
) {
    fun executar(
        modo: ModoDeJogoDto, iniciador: UUID, investigadores: List<UUID> = emptyList(),
        categoria: CategoriaDto? = null, dificuldade: DificuldadeDto? = null,
    ): Resultado<VisaoPartida>
}
class FazerPergunta(partidas: GameSessionRepository, montador: MontarVisao, clock: Clock) {
    fun executar(partidaId: UUID, autor: UUID, texto: String): Resultado<VisaoPartida>
}
class ResponderPergunta(partidas: GameSessionRepository, montador: MontarVisao, clock: Clock) {
    fun executar(partidaId: UUID, autor: UUID, perguntaId: Int, resposta: RespostaDto): Resultado<VisaoPartida>
}
class CancelarPergunta(partidas: GameSessionRepository, montador: MontarVisao, clock: Clock) {
    fun executar(partidaId: UUID, autor: UUID, perguntaId: Int): Resultado<VisaoPartida>
}
class ResponderComIA(
    partidas: GameSessionRepository, enigmas: EnigmaRepository,
    narrador: Narrador, montador: MontarVisao, clock: Clock,
) {
    fun executar(partidaId: UUID): Resultado<VisaoInvestigador>
}
class ProporSolucao(partidas: GameSessionRepository, montador: MontarVisao, clock: Clock) {
    fun executar(partidaId: UUID, autor: UUID, texto: String): Resultado<VisaoPartida>
}
class RetirarProposta(partidas: GameSessionRepository, montador: MontarVisao, clock: Clock) {
    fun executar(partidaId: UUID, autor: UUID): Resultado<VisaoPartida>
}
class JulgarProposta(partidas: GameSessionRepository, montador: MontarVisao, clock: Clock) {
    fun executar(partidaId: UUID, autor: UUID, correta: Boolean): Resultado<VisaoPartida>
}
class AvaliarComIA(
    partidas: GameSessionRepository, enigmas: EnigmaRepository,
    verificador: VerificadorDeSolucao, montador: MontarVisao, clock: Clock,
) {
    fun executar(partidaId: UUID): Resultado<VisaoInvestigador>
}
class Abandonar(partidas: GameSessionRepository, montador: MontarVisao, clock: Clock) {
    fun executar(partidaId: UUID, autor: UUID): Resultado<VisaoPartida>
}
class ConsultarPartida(partidas: GameSessionRepository, montador: MontarVisao) {
    fun executar(partidaId: UUID, solicitante: UUID): Resultado<VisaoPartida>
}
```

## Estrutura de arquivos

```
domain/src/main/kotlin/com/quebugesse/domain/gamesession/
  GameSession.kt                                  # emendas 1 (autor em aplicarVeredito) e 2 (reconstituir)
  port/GameSessionRepository.kt
  port/Narrador.kt                                # Narrador, ContextoDaNarracao, PerguntaRespondidaResumo
  port/VerificadorDeSolucao.kt
domain/src/test/kotlin/com/quebugesse/domain/gamesession/
  PropostasTest.kt                                # + CA-33 (emenda 1)
  ReconstituirTest.kt                             # CA-26 (emenda 2)
infrastructure/src/main/kotlin/com/quebugesse/infrastructure/persistence/memoria/
  GameSessionRepositoryEmMemoria.kt
infrastructure/src/testFixtures/kotlin/com/quebugesse/infrastructure/persistence/contrato/
  GameSessionRepositoryContrato.kt                # CA-25
infrastructure/src/test/kotlin/com/quebugesse/infrastructure/persistence/memoria/
  GameSessionRepositoryEmMemoriaTest.kt
application/build.gradle.kts                      # testImplementation(project(":infrastructure"))
application/src/main/kotlin/com/quebugesse/application/
  port/GeradorDeIds.kt
  port/Sorteador.kt
  dto/Resultado.kt
  dto/EnumsDto.kt
  dto/CodigoErro.kt
  dto/VisoesDto.kt
  dto/Mapeadores.kt
  usecase/MontarVisao.kt
  usecase/RegistrarJogador.kt
  usecase/IniciarPartida.kt
  usecase/FazerPergunta.kt
  usecase/ResponderPergunta.kt
  usecase/CancelarPergunta.kt
  usecase/ResponderComIA.kt
  usecase/ProporSolucao.kt
  usecase/RetirarProposta.kt
  usecase/JulgarProposta.kt
  usecase/AvaliarComIA.kt
  usecase/Abandonar.kt
  usecase/ConsultarPartida.kt
application/src/test/kotlin/com/quebugesse/application/
  FixturesAplicacao.kt                            # fakes, repositório espião, relógio fixo, seed
  dto/MapeamentoTest.kt                           # CA-24
  usecase/RegistrarJogadorTest.kt                 # CA-27
  usecase/IniciarPartidaTest.kt                   # CA-01 a CA-09
  usecase/ComandosDaPartidaTest.kt                # CA-10 a CA-13
  usecase/JulgarPropostaTest.kt                   # CA-14, CA-15
  usecase/ResponderComIATest.kt                   # CA-16 a CA-18
  usecase/AvaliarComIATest.kt                     # CA-19, CA-20
  usecase/ConsultarPartidaTest.kt                 # CA-21 a CA-23
docs/diagramas/
  componentes.md
  classes-dominio.md
  estados-partida.md
  pacotes.md
  sequencia-iniciar-partida.md
  sequencia-investigar-e-resolver.md
```

## Estratégia de testes

- **Fixtures de aplicação (`FixturesAplicacao.kt`).** Monta repositórios em memória com o `SeedInicial` (e1..e5, jogador IA), registra `N`, `Ana`, `Beto` (deixando `Zé` sem registro), `Clock.fixed`, `GeradorDeIdsSequencial`, `SorteadorPrimeiro` (com registro dos elegíveis recebidos), `FakeNarrador` e `FakeVerificadorDeSolucao` programáveis (retornos ou exceção) e um wrapper espião de `GameSessionRepository` que conta chamadas a `salvar`.
- **Contrato de repositório (`GameSessionRepositoryContrato.kt`).** Classe abstrata em `infrastructure/src/testFixtures`, garantindo que a mesma suíte (CA-25) rode depois no SQLite da S09 (ADR 0004 B).
- **Sigilo P3 (CA-23).** Roteiro percorre grupo e solo por todas as fases não terminais verificando `visao.toString() shouldNotContain enigma.conteudo.solucaoCompleta` em toda `VisaoInvestigador`.
- **Cobertura e arquitetura.** `koverVerify` ≥ 90% em `com.quebugesse.application` e `com.quebugesse.infrastructure.persistence`; Konsist garante P1, P7 e ausência de tipos do domínio nas assinaturas públicas de `application.usecase` e `application.dto`.

## Riscos

| Risco | Mitigação |
|---|---|
| Emenda 1 quebra testes antigos da S01 que chamam `aplicarVeredito(veredito, agora)` | Atualizar chamadas na tarefa 1 e rodar `:domain:test` completo imediatamente |
| `ContextoDaNarracao` vazar `solucaoCompleta` em log ou mensagem de erro | Sobrescrever `toString()` ocultando `solucaoCompleta` |
| `GameSession.reconstituir` aceitar estado corrompido (ex.: duas perguntas abertas ou `resolvidaEm` em partida ativa) | Validar todas as invariantes I1–I8 no `reconstituir` e cobrir no CA-26 |
| `interfaces` depender de tipos de `domain` por acidente | Assinaturas públicas de `application` usam só `UUID`, `String`, primitivos e `application.dto` (A10), conferido no CA-24 e pelo Konsist |

## Fora deste plano

Publicação de eventos pós-persistência e ouvintes automáticos do solo (S04), CLI local (S05), REST/WebSocket Ktor e autenticação por token (S06, ADRs 0003/0007), adaptadores HTTP chamando `Que-Bug-E-Esse/ai` (`/narrar` e `/verificar`, S07/S08, ADR 0009), persistência SQLite + Flyway + outbox (S09, ADR 0004 B).
