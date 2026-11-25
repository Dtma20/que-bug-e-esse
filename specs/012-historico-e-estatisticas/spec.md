# Spec 012: Histórico da partida e estatísticas do jogador

| Campo | Valor |
|---|---|
| ID | S12 |
| Status | Aprovada |
| Depende de | S02 (`Player`, `EstatisticasJogador`, `PlayerRepository`, regra J2), S04 (`PublicadorDeEventos`, `OuvinteDeEvento`, `application.ouvinte`, H6 e EV1), S06 (`GET /api/perfil`, `GET /api/historico`, `ConsultarPerfil`, `ConsultarHistorico`), S09 (`PlayerRepositorySqlite`, coluna `partidas_jogadas` e `partidas_resolvidas` em `players`) |
| Casos de uso | UC7 Consultar histórico da partida (histórico completo do jogador e estatísticas atualizadas por evento) |
| Diagramas de origem | [classes-dominio.md](../../docs/diagramas/classes-dominio.md), [componentes.md](../../docs/diagramas/componentes.md), [implantacao.md](../../docs/diagramas/implantacao.md), [casos-de-uso.md](../../docs/diagramas/casos-de-uso.md) |
| Decisões que assume | [ADR 0004](../../docs/adr/0004-banco-de-dados.md) opção B (SQLite em arquivo persistindo `players.partidas_jogadas` e `players.partidas_resolvidas`), [ADR 0006](../../docs/adr/0006-modo-offline.md) opção A (CLI local no mesmo dispositivo), [ADR 0007](../../docs/adr/0007-identidade-e-autenticacao.md) opção A (token HMAC identificando o jogador em `/api/perfil` e `/api/historico`), todas `Aceita` |
| Melhorias incorporadas | M13 (UC7 ligado diretamente ao Investigador), M17 (atualização de `EstatisticasJogador` por evento de domínio em `PartidaResolvida` e `PartidaAbandonada`), M24 (contadores simples sem pontuação complexa no MVP) do [planejamento](../../planejamento-sdd.md) |
| Plano | Plano e tarefas: ainda não escritos (implementação do marco M4 em `Que-Bug-E-Esse/back`) |

## Contexto

Na S02, o agregado `Player` nasceu com `EstatisticasJogador(partidasJogadas, partidasResolvidas)` sempre zerado e sem comandos para atualizá-lo (regra J3, M17). Na S06, `GET /api/perfil` e `GET /api/historico` já foram expostos, mas sem atualização orientada a eventos no agregado `Player` persistido em banco.

Esta spec fecha o requisito M17 e conclui o UC7: adiciona comandos puros de atualização em `EstatisticasJogador` e `Player`, implementa o ouvinte de domínio `AoEncerrarPartidaAtualizarEstatisticas` em `application.ouvinte` reagindo a `PartidaResolvida` e `PartidaAbandonada` após a persistência (H6), liga o ouvinte no `bootstrap` (`ServidorHttp` e `AplicacaoCli`), persiste os contadores em SQLite e em memória, e expõe o resultado de forma consistente em `GET /api/perfil` e `GET /api/historico` preservando P3.

## Histórias

- **US1.** Como investigador, quero que minhas estatísticas (`partidasJogadas` e `partidasResolvidas`) sejam atualizadas automaticamente sempre que uma partida em que participei for resolvida ou abandonada (UC7, M17).
- **US2.** Como jogador, quero consultar meu perfil em `GET /api/perfil` vendo os contadores reais persistidos mesmo após reiniciar o servidor (ADR 0004 B, UC7).
- **US3.** Como jogador, quero consultar meu histórico em `GET /api/historico` ordenado da partida mais recente para a mais antiga, sem vazamento de `solucaoCompleta` (UC7, P3).
- **US4.** Como equipe, quero que a atualização de estatísticas ocorra de forma desacoplada por evento de domínio pós-persistência (H6), sem poluir os casos de uso de comando da partida e sem quebrar a partida se o ouvinte falhar (EV1).

## Glossário

| Termo | Significado |
|---|---|
| `registrarPartidaResolvida()` | Método puro em `EstatisticasJogador` e `Player` que devolve nova instância com `partidasJogadas + 1` e `partidasResolvidas + 1` (novo, M17) |
| `registrarPartidaAbandonada()` | Método puro em `EstatisticasJogador` e `Player` que devolve nova instância com `partidasJogadas + 1` e `partidasResolvidas` inalterado (novo, M17) |
| `AoEncerrarPartidaAtualizarEstatisticas` | Ouvinte em `application.ouvinte` inscrito no `PublicadorDeEventos` que atualiza `Player.stats` dos investigadores humanos ao receber `PartidaResolvida` ou `PartidaAbandonada` (novo, M17) |

## Regras

| ID | Regra |
|---|---|
| EST1 | Comandos puros no domínio (`domain.player`, P1, P2, J2): `EstatisticasJogador.registrarPartidaResolvida()` retorna nova instância com `(partidasJogadas + 1, partidasResolvidas + 1)`; `EstatisticasJogador.registrarPartidaAbandonada()` retorna nova instância com `(partidasJogadas + 1, partidasResolvidas)`. `Player.registrarPartidaResolvida()` e `Player.registrarPartidaAbandonada()` retornam nova instância de `Player` com `stats` atualizadas, sem mutar a instância original e preservando sempre a invariante J2 (`0 <= partidasResolvidas <= partidasJogadas`) |
| EST2 | Ouvinte reativo pós-persistência (H6, M17): `AoEncerrarPartidaAtualizarEstatisticas` (em `com.quebugesse.application.ouvinte`) implementa `OuvinteDeEvento` e reage exclusivamente a `PartidaResolvida` e `PartidaAbandonada`, ignorando todos os demais eventos de `EventoDePartida`. Como eventos só são publicados após `GameSessionRepository.salvar` (S04 H6), comandos rejeitados ou falhas de persistência nunca alteram estatísticas |
| EST3 | Alvo da atualização: ao receber `PartidaResolvida` ou `PartidaAbandonada`, o ouvinte carrega a partida em `GameSessionRepository.buscar(evento.partidaId)`. Para cada `PlayerRef` em `partida.investigadores`, carrega o `Player` via `PlayerRepository.buscar`; se existir e for `PlayerType.HUMANO`, aplica `registrarPartidaResolvida()` ou `registrarPartidaAbandonada()` e persiste com `PlayerRepository.salvar`. Jogadores `PlayerType.IA` e o narrador não têm estatísticas de investigação alteradas |
| EST4 | Isolamento de falhas (S04 EV1): qualquer exceção lançada ao atualizar um jogador no ouvinte é tratada pelo registro de falhas do `PublicadorDeEventos`, sem reverter a partida já encerrada nem impedir a entrega do evento aos demais ouvintes (como a difusão WebSocket) |
| EST5 | Ligação obrigatória no `bootstrap`: `AoEncerrarPartidaAtualizarEstatisticas` é registrado no `PublicadorDeEventos` tanto em `ServidorHttp` (API Ktor + SQLite) quanto em `AplicacaoCli` (CLI local, ADR 0006 A) |
| EST6 | Persistência e `GET /api/perfil`: `PlayerRepositorySqlite` (tabela `players`, colunas `partidas_jogadas` e `partidas_resolvidas`, S09) e `PlayerRepositoryEmMemoria` (S02) persistem as estatísticas atualizadas. `ConsultarPerfil` / `GET /api/perfil` mantém o contrato JSON da S06 `{ "playerId", "nome", "jogadas", "resolvidas", "abandonadas", "emAndamento" }`, com `jogadas` e `resolvidas` vindos de `Player.stats` persistido combinados com o detalhamento de partidas (`abandonadas`, `emAndamento`) |
| EST7 | Histórico em `GET /api/historico` (UC7, P3, M24): `ConsultarHistorico` devolve a lista de partidas das quais o jogador autenticado participa ou participou (como investigador ou narrador), ordenada da mais recente para a mais antiga, nunca contendo `solucaoCompleta` em nenhum item. Sem sistema de pontuação ou limites de perguntas/tempo no MVP (M24) |

### Contrato REST (`GET /api/perfil` e `GET /api/historico`)

Ambas as rotas exigem `Authorization: Bearer <token>` (ADR 0007 A; `401` sem token válido):

| Rota | Caso de uso | Resposta `200 OK` |
|---|---|---|
| `GET /api/perfil` | `ConsultarPerfil` | `{ "playerId": "UUID", "nome": "string", "jogadas": Int, "resolvidas": Int, "abandonadas": Int, "emAndamento": Int }` (`jogadas` e `resolvidas` de `Player.stats` persistido) |
| `GET /api/historico` | `ConsultarHistorico` | Lista ordenada (mais recente primeiro) com resumo das partidas do jogador, sem `solucaoCompleta` (P3) |

## Critérios de aceitação

Participantes de exemplo: narrador humano N, investigadoras Ana e Bia, e jogador IA. Testes de domínio puros (P1); testes de aplicação com repositórios em memória e `PublicadorDeEventosEmMemoria`; testes de integração com SQLite real + Flyway + Ktor `testApplication`.

| CA | Regra | Dado | Quando | Então |
|---|---|---|---|---|
| 01 | EST1, J2 | `Player` humano com `stats = (0, 0)` | chamar `registrarPartidaResolvida()` e depois `registrarPartidaAbandonada()` | instâncias retornadas têm `(1, 1)` e `(2, 1)`; a instância original permanece `(0, 0)` e `0 <= resolvidas <= jogadas` vale sempre |
| 02 | EST2, EST3 | partida em grupo com narrador N e investigadoras Ana e Bia; `AoEncerrarPartidaAtualizarEstatisticas` registrado | N julga proposta de Ana como `CORRETA` (`PartidaResolvida`) | Ana e Bia passam para `(jogadas=1, resolvidas=1)` no `PlayerRepository`; N permanece `(0, 0)` |
| 03 | EST2, EST3 | partida solo de Ana com narrador IA; ouvinte registrado | Ana abandona a partida (`PartidaAbandonada`) | Ana passa para `(jogadas=1, resolvidas=0)`; jogador IA permanece `(0, 0)` |
| 04 | EST2 | partida em andamento; ouvinte registrado | ocorrem `PartidaIniciada`, `PerguntaFeita`, `PerguntaRespondida`, `SolucaoProposta` e `SolucaoRecusada` | estatísticas de Ana permanecem `(0, 0)` em todos os eventos não terminais |
| 05 | EST2, H6 | `GameSessionRepository.salvar` lança exceção ao encerrar partida | comando de encerramento falha com `FALHA_AO_SALVAR` | nenhum evento é publicado e `Player.stats` não muda |
| 06 | EST4, EV1 | `PlayerRepository.salvar` falha dentro do ouvinte de estatísticas + segundo ouvinte espião registrado | partida é resolvida | a partida permanece salva como `RESOLVIDO`, a falha é registrada no publicador e o segundo ouvinte ainda recebe `PartidaResolvida` |
| 07 | EST5, EST6 | servidor HTTP com SQLite real (`DB_PATH` temporário) e Ana autenticada | Ana conclui uma partida solo (resolvida ou abandonada), servidor é reiniciado apontando para o mesmo arquivo SQLite e Ana chama `GET /api/perfil` | `200` com `jogadas >= 1` (e `resolvidas == 1` se resolvida, ou `abandonadas == 1` se abandonada) persistidos na tabela `players` |
| 08 | EST7, P3 | Ana tem uma partida concluída e outra em andamento | `GET /api/historico` | `200` com as duas partidas ordenadas da mais recente para a mais antiga e zero ocorrências de `solucaoCompleta` no corpo JSON |
| 09 | EST5 | composição `AplicacaoCli` e `ServidorHttp` no `bootstrap` | encerrar partida via CLI ou API HTTP | ambos têm `AoEncerrarPartidaAtualizarEstatisticas` registrado e atualizam `PlayerRepository` |
| 10 | P1, P7 | módulos do `back` | teste de arquitetura Konsist (`ArquiteturaTest`) | zero infrações; `domain.player` permanece puro e `interfaces` não importa `domain` nem `infrastructure` |

## Requisitos não funcionais

- **RNF1.** Domínio puro (P1) e imutabilidade de agregados/VOs preservada (P2).
- **RNF2.** Cobertura de linhas de pelo menos 90% nos pacotes tocados (`domain.player`, `application.ouvinte`, `application.usecase`, `bootstrap`) medida pelo Kover.
- **RNF3.** Suítes de domínio, aplicação e SQLite rodam offline sem Docker nem rede externa (P6, ADR 0004 B).

## Fora de escopo

- Sistema de pontuação, XP persistido no banco ou ranking global entre jogadores (M24 decidida: contadores simples no MVP).
- Estatísticas de partidas como narrador ou métricas granulares por categoria/tempo médio no modelo de domínio.
- Reprocessamento distribuído do outbox em múltiplos nós (ADR 0009 A).

## Divergências em relação aos diagramas

Pelo princípio P9, [classes-dominio.md](../../docs/diagramas/classes-dominio.md), [componentes.md](../../docs/diagramas/componentes.md), [implantacao.md](../../docs/diagramas/implantacao.md), [casos-de-uso.md](../../docs/diagramas/casos-de-uso.md) e [README.md](../../docs/diagramas/README.md) são atualizados nesta mesma entrega com `registrarPartidaResolvida()`, `registrarPartidaAbandonada()` e o ouvinte `AoEncerrarPartidaAtualizarEstatisticas`. Nenhuma divergência fica pendente.

## Questões abertas

Nenhuma questão em aberto: M13, M17 e M24 foram fechadas pela equipe em [docs/pendencias.md](../../docs/pendencias.md).

## Rastreabilidade

| CA | Arquivo de teste |
|---|---|
| 01 | `Que-Bug-E-Esse/back`: `domain/.../player/PlayerTest.kt` |
| 02 a 06 | `Que-Bug-E-Esse/back`: `application/.../ouvinte/AoEncerrarPartidaAtualizarEstatisticasTest.kt` |
| 07, 08, 09 | `Que-Bug-E-Esse/back`: `bootstrap/.../ApiHttpIntegracaoTest.kt` e testes de SQLite/CLI |
| 10 | `Que-Bug-E-Esse/back`: `bootstrap/src/test/kotlin/com/quebugesse/arquitetura/ArquiteturaTest.kt` |
