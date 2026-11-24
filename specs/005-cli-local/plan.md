# Plano técnico: S05 CLI local

Spec: [spec.md](spec.md) · Tarefas: [tasks.md](tasks.md) · Constituição 1.0.0: [docs/constitution.md](../../docs/constitution.md)

## Resumo

A S05 fecha o marco M1 (jogo inteiro no terminal no mesmo dispositivo, sem rede nem IA, ADR 0006 A). Cria o pacote `com.quebugesse.interfaces.cli` no módulo `interfaces` — falando apenas com casos de uso, DTOs e `CodigoErro` de `application` (P7, Q1 da S00) — e o ponto de entrada `main` + fábrica testável no módulo `bootstrap` (M16), que monta os repositórios em memória com `SeedInicial` (S02), `PublicadorDeEventosEmMemoria` (S04), relógio do sistema, gerador de UUIDs e sorteador aleatório.

## Contexto técnico

| Item | Escolha |
|---|---|
| Repositório | `Que-Bug-E-Esse/back` (ADR 0009 A) |
| Linguagem e build | Kotlin/JVM (Temurin JDK 25 LTS), Gradle 9.8.2 com Kotlin DSL, plugin `application` no `bootstrap` (`-Dfile.encoding=UTF-8`, RNF3) |
| Módulos tocados | `interfaces` (`com.quebugesse.interfaces.cli`), `bootstrap` (`com.quebugesse.bootstrap` e testes integrados em `com.quebugesse.cli`) |
| Testes | Kotest (`FunSpec`, `assertions-core`), sem terminal real nem subprocesso (entrada via `StringReader`, saída via `StringWriter`, RNF4) |
| Cobertura | Kover (≥ 90% em `com.quebugesse.interfaces.cli`) |
| Dependências de produção | `interfaces` → apenas `:application` (nunca `:domain` nem `:infrastructure`, RNF1). `bootstrap` → todos os módulos |

## Portão da constituição

| Princípio | Como a S05 cumpre |
|---|---|
| P1 Domínio puro | Nenhum código novo em `domain`; relógio do sistema, UUID e aleatoriedade ficam restritos ao `bootstrap` |
| P2 Invariantes protegidas | Toda regra do jogo continua nos casos de uso/domínio; a CLI só interpreta linhas e exibe resultados/erros |
| P3 Sigilo da solução | Tela inicial (CL6) e comando `ver` nunca imprimem `solucaoCompleta` antes do fim, nem na vez do narrador; `solucao` exige ser narrador, avisa e pede confirmação `Mostrar? (s/n) ` (CA-01, CA-10) |
| P4 Regras como testes | 15 CAs cobrindo CL1–CL12, P3, H2 e os 25 códigos de erro/CLI, cada teste nomeado por `"CA-NN "` |
| P5 Teste primeiro | Cada tarefa de [tasks.md](tasks.md) começa pelos testes falhando |
| P6 Fakes | `RoteirosTest` injeta `Clock.fixed`, `GeradorDeIds` sequencial e `Sorteador` que escolhe o primeiro elegível sobre repositórios em memória + seed |
| P7 Camadas | `interfaces.cli` importa apenas `com.quebugesse.application.*` e stdlib; validado pelo teste Konsist da S00 (RNF1) |
| P8 ADR | Segue ADRs aceitas: 0001 A (narrador julga no grupo), 0006 A (modo offline = mesmo dispositivo), 0007 A (registro por nome sem senha), 0009 A (`back`) |
| P9 Diagramas | A tarefa 5 atualiza `docs/diagramas/` com D1 a D4 (remover SSH em produção, acrescentar `interfaces.cli` e `main` no `bootstrap`, atualizar UC1 para "mesmo dispositivo") |
| P10 Linguagem ubíqua | Comandos, prompts, nomes de exibição e mensagens de erro 100% em português conforme a spec |

## Decisões de projeto

**DP1. `CliLocal` recebe `BufferedReader`, `PrintWriter` e `CasosDeUsoCli`.** Nada de `readln()` ou `println()` acoplado a `System.in`/`System.out` dentro de `interfaces.cli`. Assim os 14 CAs de roteiro rodam em milissegundos com `StringReader`/`StringWriter` (RNF4), e o `main` no `bootstrap` apenas configura UTF-8 explícito em `System.in`/`System.out` (RNF3).

**DP2. `CasosDeUsoCli` agrupa os 10 casos de uso do modo grupo (CL12).** Exclui deliberadamente `ResponderComIA` e `AvaliarComIA` do tipo injetado na CLI, tornando impossível chamá-los por engano no modo grupo local.

**DP3. Normalização determinística via `java.text.Normalizer` (CL3).** `normalizar(texto)` faz `Normalizer.normalize(texto.trim(), NFD)`, remove marcas diacríticas (`\p{Mn}+`), passa para `lowercase(Locale.ROOT)` e troca sequências de espaços/hífens por `_`. Assim `" Gestão de projeto "` normaliza para `"gestao_de_projeto"` (nome de exibição normalizado) e `"gestao-projeto"` normaliza para `"gestao_projeto"` (nome do enum normalizado); `Mensagens.parseCategoria` aceita ambas as formas e devolve `CategoriaDto.GESTAO_PROJETO`, assim como `"MÉDIA"`, `"FÁCIL"` e `"Não"` batem com as tabelas da spec (CL3, CA-02, CA-03, CA-07).

**DP4. `Mensagens.kt` com `when` exaustivo (CA-14).** `mensagem(codigo: CodigoErro)` e `mensagem(codigo: CodigoCli, opcoes: String)` usam `when` sem `else`, cobrindo exatamente os 22 valores de `CodigoErro` (S03 + `FALHA_AO_SALVAR` da S04) e os 3 valores de `CodigoCli`. `MensagensTest` garante que cada valor produz texto não vazio e único.

**DP5. Memória de sessão por nome normalizado (CL4, Q5).** `CliLocal` mantém `private val jogadoresDaSessao = mutableMapOf<String, ParticipanteDto>()`. Na primeira aparição de um nome normalizado, chama `RegistrarJogador(nomeAparado)` e guarda o `ParticipanteDto` com a grafia original da primeira digitação (usada no prompt `[Nome]> `, CA-07). Reaparições na mesma partida ou em "Jogar outra?" reutilizam o mesmo `UUID`, garantindo detecção de duplicidade (CA-05) e respeito a "já jogados" entre partidas (M18, CA-12).

**DP6. Validação da abertura na ordem estrita (CL5).** A abertura sempre lê os 4 prompts (`Narrador: `, `Investigadores (separados por vírgula): `, `Categoria (Enter = qualquer): `, `Dificuldade (Enter = qualquer): `) — ou os 2 da abertura curta — antes de validar: (1) nomes de narrador e investigadores via `RegistrarJogador` (ignorando pedaços vazios após aparar, CA-06), (2) categoria, (3) dificuldade, (4) `IniciarPartida`. O primeiro erro imprime a mensagem e reinicia o laço da abertura correspondente; EOF em qualquer leitura devolve `0` imediatamente sem escrever em stderr (CA-13).

**DP7. Leitura segura de `perguntaAberta` pela visão mais recente (CL8).** Para `responder` e `cancelar`, a CLI valida primeiro o argumento (`sim`/`nao`/`irrelevante` no caso de `responder`, dando `ARGUMENTO_INVALIDO` antes de olhar a pergunta), consulta a visão atual para localizar a pergunta aberta (`resposta == null && !cancelada`) e, se não houver, imprime `SEM_PERGUNTA_ABERTA` sem chamar o caso de uso (CA-09, CA-15).

**DP8. `AplicacaoCli` no `bootstrap` para testes de ponta a ponta.** O módulo `bootstrap` expõe `AplicacaoCli.criar(entrada, saida, clock, geradorDeIds, sorteador)` com os adaptadores reais em memória + `SeedInicial` + `PublicadorDeEventosEmMemoria`. O `main()` apenas chama essa fábrica com `Clock.systemUTC()`, `GeradorDeIdsUuid` e `SorteadorAleatorio`.

## Modelo

```kotlin
// com.quebugesse.interfaces.cli
enum class CodigoCli { COMANDO_DESCONHECIDO, JOGADOR_DESCONHECIDO, ARGUMENTO_INVALIDO }

object Normalizador {
    fun normalizar(texto: String): String
}

object Mensagens {
    fun mensagem(codigo: CodigoErro): String
    fun mensagem(codigo: CodigoCli, opcoes: String = ""): String
    fun exibir(categoria: CategoriaDto): String
    fun exibir(dificuldade: DificuldadeDto): String
    fun exibir(resposta: RespostaDto): String
    fun parseCategoria(texto: String): CategoriaDto?
    fun parseDificuldade(texto: String): DificuldadeDto?
    fun parseResposta(texto: String): RespostaDto?
    fun parseVeredito(texto: String): Boolean?
    fun parseConfirmacao(texto: String): Boolean?
}

data class CasosDeUsoCli(
    val registrarJogador: RegistrarJogador,
    val iniciarPartida: IniciarPartida,
    val fazerPergunta: FazerPergunta,
    val responderPergunta: ResponderPergunta,
    val cancelarPergunta: CancelarPergunta,
    val proporSolucao: ProporSolucao,
    val retirarProposta: RetirarProposta,
    val julgarProposta: JulgarProposta,
    val abandonar: Abandonar,
    val consultarPartida: ConsultarPartida,
)

class CliLocal(
    private val entrada: BufferedReader,
    private val saida: PrintWriter,
    private val casos: CasosDeUsoCli,
) {
    fun executar(): Int
}

// com.quebugesse.bootstrap
class GeradorDeIdsUuid : GeradorDeIds
class SorteadorAleatorio(private val random: Random = Random.Default) : Sorteador

object AplicacaoCli {
    fun criar(
        entrada: BufferedReader,
        saida: PrintWriter,
        clock: Clock = Clock.systemUTC(),
        geradorDeIds: GeradorDeIds = GeradorDeIdsUuid(),
        sorteador: Sorteador = SorteadorAleatorio(),
    ): CliLocal
}

fun main()
```

## Estrutura de arquivos

```
interfaces/build.gradle.kts                       # implementation(project(":application")) apenas
interfaces/src/main/kotlin/com/quebugesse/interfaces/cli/
  Normalizador.kt
  Mensagens.kt                                    # CodigoCli, mensagens dos 22+3 códigos, parsers e exibição
  CasosDeUsoCli.kt
  CliLocal.kt
interfaces/src/test/kotlin/com/quebugesse/interfaces/cli/
  MensagensTest.kt                                # CA-14 (+ testes unitários de Normalizador/parsers)
bootstrap/build.gradle.kts                        # plugin application, mainClass, UTF-8, standardInput
bootstrap/src/main/kotlin/com/quebugesse/bootstrap/
  GeradorDeIdsUuid.kt
  SorteadorAleatorio.kt
  AplicacaoCli.kt
  Main.kt
bootstrap/src/test/kotlin/com/quebugesse/cli/
  RoteirosTest.kt                                 # CA-01 a CA-13, CA-15
docs/diagramas/
  implantacao.md
  pacotes.md
  casos-de-uso.md
  componentes.md
```

## Estratégia de testes

- **`MensagensTest.kt` (`interfaces`).** Verifica CA-14: todos os 22 valores de `CodigoErro.entries` e os 3 de `CodigoCli.entries` têm mensagens não vazias e distintas entre si; verifica normalização de acentos, hífens e espaços nos parsers de categoria, dificuldade, resposta e veredito.
- **`RoteirosTest.kt` (`bootstrap`).** Helper `rodarRoteiro(vararg linhas: String): Pair<Int, String>` instancia `AplicacaoCli.criar` com `Clock.fixed`, `GeradorDeIdsSequencial` e `SorteadorPrimeiro` (escolhe o primeiro elegível), usando `StringReader` e `StringWriter`, e valida os CAs 01–13 e 15 (incluindo sigilo P3 em CA-01 e CA-10, confirmação de `solucao` e `abandonar`, `Jogar outra?` com M18 em CA-12, linhas vazias/EOF/`sair` com código 0 e stderr vazio em CA-13).
- **Arquitetura e cobertura.** Konsist confirma que `com.quebugesse.interfaces.cli` não importa `com.quebugesse.domain` nem `com.quebugesse.infrastructure` (RNF1); `koverVerify` exige ≥ 90% em `com.quebugesse.interfaces.cli` (RNF2).

## Riscos

| Risco | Mitigação |
|---|---|
| Acentos quebrados no PowerShell/Windows Terminal (RNF3) | Usar `UTF_8` explícito em `InputStreamReader`/`OutputStreamWriter` no `Main.kt` e `-Dfile.encoding=UTF-8` na tarefa `run` do `bootstrap` |
| Solução aparecer acidentalmente no `ver` ou na tela inicial quando a vez é do narrador (P3) | Formatação de tela inicial e `ver` lê apenas campos comuns de `VisaoPartida` (nunca `VisaoNarrador.solucaoCompleta`); testado no CA-01 e CA-10 |
| `CliLocal` ficar grande demais ou difícil de ler | Extrair formatação (`Mensagens.kt`), normalização (`Normalizador.kt`) e métodos privados coesos (`executarAbertura`, `executarPartida`, `executarComando`) |
| `bootstrap` com plugin `application` travar entrada interativa no Gradle | Configurar `standardInput = System.`in`` na task `run` do `bootstrap/build.gradle.kts` |

## Fora deste plano

Modo solo na CLI (entra após S07/S08), servidor HTTP/WebSocket Ktor e autenticação por token (S06, ADRs 0003/0007), persistência em SQLite entre execuções (S09, ADR 0004 B), front-end web em `Que-Bug-E-Esse/front` (S10, ADRs 0008/0009).
