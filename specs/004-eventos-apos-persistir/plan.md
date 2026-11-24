# Plano técnico: S04 Eventos depois de persistir

Spec: [spec.md](spec.md) · Tarefas: [tasks.md](tasks.md) · Constituição 1.0.0: [docs/constitution.md](../../docs/constitution.md)

## Resumo

A S04 fecha a regra H6 ("eventos só saem depois de persistir") e automatiza o modo solo (UC10). Cria a porta `PublicadorDeEventos` em `domain.shared.port`, implementa `PublicadorDeEventosEmMemoria` em `infrastructure.messaging` com despacho síncrono e fila FIFO por thread (sem recursão e com isolamento de falhas), atualiza os casos de uso da S03 para salvar → publicar → reler a visão (com `FALHA_AO_SALVAR`) e adiciona os ouvintes `AoFazerPerguntaNoSolo` e `AoProporSolucaoNoSolo` em `application.ouvinte`.

## Contexto técnico

| Item | Escolha |
|---|---|
| Repositório | `Que-Bug-E-Esse/back` (ADR 0009 A) |
| Linguagem e build | Kotlin/JVM (Temurin JDK 25 LTS), Gradle 9.8.2 com Kotlin DSL e plugin `quebugesse.kotlin-module` (S00) |
| Módulos tocados | `domain` (`domain.shared.port`), `infrastructure` (`infrastructure.messaging`), `application` (`CodigoErro`, casos de uso da S03 e `application.ouvinte`) |
| Testes | Kotest (`FunSpec`, `assertions-core`) |
| Cobertura | Kover (≥ 90% em `com.quebugesse.application.ouvinte` e `com.quebugesse.infrastructure.messaging`) |
| Dependências de produção | Nenhuma biblioteca externa (apenas stdlib Kotlin e `java.util.concurrent.CopyOnWriteArrayList`) |

## Portão da constituição

| Princípio | Como a S04 cumpre |
|---|---|
| P1 Domínio puro | `PublicadorDeEventos` depende só de `DomainEvent` em `domain.shared`; nenhuma thread, log ou IO no domínio |
| P2 Invariantes protegidas | `puxarEventos()` só é chamado após `salvar` ter sucesso; falha em `salvar` retorna `FALHA_AO_SALVAR` sem publicar; falha em ouvinte não desfaz estado salvo |
| P3 Sigilo da solução | Releitura após publicação usa `MontarVisao`: no solo, se a IA resolve a proposta, a visão devolvida traz a solução só porque o estado virou `RESOLVIDO` (CA-12) |
| P4 Regras como testes | 12 CAs (H6, EV1–EV5, OS1–OS3, H4), cada teste nomeado por `"CA-NN "` |
| P5 Teste primeiro | Cada tarefa de [tasks.md](tasks.md) começa com o teste falhando |
| P6 Fakes | `PublicadorEspiao` para testes unitários de casos de uso; `PublicadorDeEventosEmMemoria` + fakes da S03 para `OuvintesDoSoloTest` |
| P7 Camadas | `domain.shared.port` é a porta; `infrastructure.messaging` implementa; `application.ouvinte` conhece só `application` e `domain` |
| P8 ADR | Segue ADRs aceitas: 0001 A, 0002 A (sem reação a `PerguntaFeita` no grupo), 0004 B (outbox em tabela SQLite na S09), 0009 A (`back`) |
| P9 Diagramas | A tarefa 4 atualiza `docs/diagramas/` com D1 a D7 (incluindo H6 no diagrama de estados) |
| P10 Linguagem ubíqua | `PublicadorDeEventos`, `AoFazerPerguntaNoSolo`, `AoProporSolucaoNoSolo` conforme glossário da spec |

## Decisões de projeto

**DP1. Fila FIFO iterativa por thread (`ThreadLocal`).** `PublicadorDeEventosEmMemoria` mantém um `ThreadLocal<EstadoDaThread>` com `ArrayDeque<DomainEvent>` e `despachando: Boolean`. Chamada interna (`despachando == true`) apenas enfileira (`addAll`) e retorna (EV3). A chamada externa drena o `while (fila.isNotEmpty())` iterativamente, suportando 10.000 republicações em cadeia sem estourar a pilha (CA-07), e remove o `ThreadLocal` no bloco `finally` ao terminar (RNF1, sem vazamento de memória).

**DP2. Registro thread-safe sem bloqueio na entrega.** Os ouvintes ficam numa `CopyOnWriteArrayList<OuvinteDeEvento>` (RNF3). Registrar ouvintes após a construção quebra o ciclo de dependências no `bootstrap`: cria-se o publicador, injeta-se nos casos de uso, criam-se os ouvintes com esses casos de uso e registram-se os ouvintes.

**DP3. Isolamento total de falhas (EV1).** Cada invocação `ouvinte(evento)` fica dentro de `try / catch (e: Exception)` (sem engolir `Error` fatal da JVM como `OutOfMemoryError`), repassando `(evento, ouvinte, erro)` ao callback `aoFalhar`. O laço segue para o próximo ouvinte e o próximo evento, e `publicar` nunca lança por falha de ouvinte.

**DP4. Fluxo único `salvar` → `publicar` → reler (H6, EV5).** Um auxiliar interno em `application.usecase` executa:
1. `try { partidas.salvar(partida) } catch (_: Exception) { return Resultado.Erro(CodigoErro.FALHA_AO_SALVAR) }`
2. `val eventos = partida.puxarEventos(); if (eventos.isNotEmpty()) publicador.publicar(eventos)`
3. `val atualizada = partidas.buscar(partida.id) ?: error(...)`
4. `montador.montar(atualizada, solicitante)`

`RegistrarJogador` também envolve `jogadores.salvar(jogador)` no mesmo tratamento de `FALHA_AO_SALVAR`, sem publicar eventos.

**DP5. Exceção tipada de falha no ouvinte do solo (OS3).** Quando `ResponderComIA` ou `AvaliarComIA` retorna `Resultado.Erro(codigo)`, o ouvinte lança `FalhaOuvinteSoloException(val codigo: CodigoErro)`. O publicador a captura e entrega ao registro de falhas (CA-11), preservando H4 e mantendo o resultado do comando original do investigador intacto.

## Modelo

```kotlin
// com.quebugesse.domain.shared.port
typealias OuvinteDeEvento = (DomainEvent) -> Unit

interface PublicadorDeEventos {
    fun publicar(eventos: List<DomainEvent>)
}

// com.quebugesse.infrastructure.messaging
typealias RegistroDeFalhaDeEvento = (evento: DomainEvent, ouvinte: OuvinteDeEvento, erro: Exception) -> Unit

class PublicadorDeEventosEmMemoria(
    private val aoFalhar: RegistroDeFalhaDeEvento = { _, _, _ -> },
) : PublicadorDeEventos {
    fun registrar(ouvinte: OuvinteDeEvento)
    override fun publicar(eventos: List<DomainEvent>)
}

// com.quebugesse.application.dto.CodigoErro (22º valor)
// + FALHA_AO_SALVAR

// com.quebugesse.application.ouvinte
class FalhaOuvinteSoloException(val codigo: CodigoErro) : RuntimeException(codigo.name)

class AoFazerPerguntaNoSolo(
    private val partidas: GameSessionRepository,
    private val responderComIA: ResponderComIA,
) : OuvinteDeEvento {
    override fun invoke(evento: DomainEvent)
}

class AoProporSolucaoNoSolo(
    private val partidas: GameSessionRepository,
    private val avaliarComIA: AvaliarComIA,
) : OuvinteDeEvento {
    override fun invoke(evento: DomainEvent)
}
```

Os 10 casos de uso de comando da S03 (`IniciarPartida`, `FazerPergunta`, `ResponderPergunta`, `CancelarPergunta`, `ResponderComIA`, `ProporSolucao`, `RetirarProposta`, `JulgarProposta`, `AvaliarComIA`, `Abandonar`) recebem `publicador: PublicadorDeEventos` no construtor.

## Estrutura de arquivos

```
domain/src/main/kotlin/com/quebugesse/domain/shared/port/
  PublicadorDeEventos.kt
infrastructure/src/main/kotlin/com/quebugesse/infrastructure/messaging/
  PublicadorDeEventosEmMemoria.kt
infrastructure/src/test/kotlin/com/quebugesse/infrastructure/messaging/
  PublicadorDeEventosEmMemoriaTest.kt             # CA-05 a CA-08
application/src/main/kotlin/com/quebugesse/application/
  dto/CodigoErro.kt                               # + FALHA_AO_SALVAR
  usecase/PersistirEPublicar.kt                   # auxiliar interno H6 + EV5
  usecase/*.kt                                    # 10 casos de uso de comando + RegistrarJogador
  ouvinte/FalhaOuvinteSoloException.kt
  ouvinte/AoFazerPerguntaNoSolo.kt
  ouvinte/AoProporSolucaoNoSolo.kt
application/src/test/kotlin/com/quebugesse/application/
  FixturesAplicacao.kt                            # + PublicadorEspiao, repositórios com falha programável
  usecase/PublicacaoAposSalvarTest.kt             # CA-01 a CA-04
  ouvinte/OuvintesDoSoloTest.kt                   # CA-09 a CA-12
docs/diagramas/
  componentes.md
  estados-partida.md
  pacotes.md
  sequencia-iniciar-partida.md
  sequencia-investigar-e-resolver.md
  implantacao.md
```

## Estratégia de testes

- **Linha do tempo (CA-01).** Lista única compartilhada por um `GameSessionRepository` espião e um ouvinte: ao receber `PerguntaFeita`, o ouvinte confirma que `salvar` já foi registrado antes na lista e que `buscar(id)` já retorna a pergunta 1 aberta.
- **Publicador em memória (CA-05 a CA-08).** Testes unitários puros em `infrastructure`: isolamento de exceção (CA-05), ordem estrita `A(e1), B(e1), C(e1), A(e2)...` (CA-06), reentrância FIFO + cadeia de 10.000 republicações sem `StackOverflowError` (CA-07) e duas threads concorrentes publicando 100 eventos cada com `CountDownLatch` (CA-08).
- **Ouvintes do solo (CA-09 a CA-12).** Montados com `PublicadorDeEventosEmMemoria` real e fakes de `Narrador`/`VerificadorDeSolucao` (Q4 da S03), cobrindo resposta automática + visão atualizada (CA-09), inércia no modo grupo (CA-10), falha da IA preservando H4 com registro de `IA_INDISPONIVEL` (CA-11) e veredito correto/incorreto no solo (CA-12).
- **Cobertura.** `koverVerify` ≥ 90% em `com.quebugesse.application.ouvinte` e `com.quebugesse.infrastructure.messaging`, e suítes abaixo de 10 s (RNF4).

## Riscos

| Risco | Mitigação |
|---|---|
| `ThreadLocal` retido após a entrega vazar referências em pool de threads da S06 | Sempre chamar `estadoThread.remove()` no `finally` da chamada externa |
| Quebra dos testes da S03 ao adicionar `PublicadorDeEventos` nos construtores | Atualizar `FixturesAplicacao` na tarefa 2 injetando `PublicadorEspiao` e rodar toda a suíte da S03 |
| Loop infinito acidental se um ouvinte publicar o mesmo evento que consome | Cada ouvinte do solo só reage a `PerguntaFeita`/`SolucaoProposta` e dispara `PerguntaRespondida`/`PartidaResolvida`/`SolucaoRecusada` (tipos distintos) |
| Exceção fora de `Exception` no teste ou ouvinte | Capturar `Exception` (não engolir `Error` da JVM como `OutOfMemoryError`) |

## Fora deste plano

Difusão de eventos por WebSocket/SSE (S06), tabela outbox em SQLite (S09, ADR 0004 B), retentativa com backoff nos adaptadores HTTP para `Que-Bug-E-Esse/ai` (S07/S08, ADR 0009), ouvinte de estatísticas do jogador (S12).
