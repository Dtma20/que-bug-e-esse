# Spec 001: Domínio da partida

| Campo | Valor |
|---|---|
| ID | S01 |
| Status | Rascunho |
| Depende de | S00 (módulo `domain` com Kotest rodando) |
| Casos de uso | UC3 Iniciar partida, UC4 Fazer pergunta, UC5 Responder pergunta, UC6 Propor solução, UC11 Abandonar partida (novo, M10) |
| Diagramas de origem | `diagrama_classes.pdf` (pacotes `domain.gamesession` e `domain.shared`), `diagrama_estados.pdf` |
| Melhorias incorporadas | M3, M4, M5, M10 (parte de domínio), M20 do [planejamento](../../planejamento-sdd.md) |
| Plano | [plan.md](plan.md) · Tarefas: [tasks.md](tasks.md) |

## Contexto

A partida (`GameSession`) é o núcleo do jogo: guarda quem narra, quem investiga, as perguntas, as tentativas de solução e o estado. Esta spec cobre só o agregado e suas regras. Casos de uso, persistência, IA, eventos distribuídos e telas ficam para outras specs (ver "Fora de escopo").

## Histórias

- **US1.** Como narrador, quero que a partida só comece com narrador e investigadores válidos, para não começar um jogo quebrado.
- **US2.** Como investigador, quero fazer uma pergunta por vez e receber SIM, NAO ou IRRELEVANTE, para investigar o caso.
- **US3.** Como narrador, quero responder apenas à pergunta em aberto da minha partida.
- **US4.** Como investigador, quero propor uma solução quando não houver pergunta pendente e saber se acertei.
- **US5.** Como participante, quero desfazer uma pergunta ou proposta travada, para a partida não ficar presa quando o narrador ou a IA não respondem (M3).
- **US6.** Como narrador, ou como jogador no modo solo, quero abandonar a partida.
- **US7.** Como sistema, quero que cada mudança de estado gere um evento de domínio, para avisar os outros participantes (consumido na S04).

## Glossário

| Termo | Significado |
|---|---|
| Partida (`GameSession`) | Agregado raiz desta spec |
| Modo (`ModoDeJogo`) | `SOLO` (a IA narra, um investigador) ou `GRUPO` (narrador humano, um ou mais investigadores). O modo "offline" do UC1 é um `GRUPO` jogado num só dispositivo e não muda o domínio |
| Narrador | Participante que responde às perguntas. No solo é o jogador IA |
| Investigador | Participante que pergunta e propõe soluções |
| Pergunta (`Question`) | Texto de um investigador; fica **aberta** até ser respondida ou cancelada |
| Resposta (`Answer`) | `SIM`, `NAO` ou `IRRELEVANTE` |
| Tentativa | Proposta de solução de um investigador e o destino dela: `PENDENTE`, `CORRETA`, `INCORRETA` ou `RETIRADA` (novo, M4) |
| Veredito | Resultado da avaliação de uma tentativa: correta ou não |
| Fase da investigação | Subestado de `EM_ANDAMENTO` no diagrama: `AGUARDANDO_PERGUNTA`, `PERGUNTA_PENDENTE`, `SOLUCAO_EM_AVALIACAO` |
| Violação | Motivo de rejeição de um comando |
| Evento de domínio | Fato registrado pela partida após um comando aceito |

## Regras

| ID | Regra |
|---|---|
| H1 | Estados terminais (`RESOLVIDO`, `ABANDONADO`) não aceitam comandos |
| H2 | Só participante autorizado abandona. **Decisão provisória (Q1):** o narrador sempre pode; um investigador só no modo `SOLO` |
| H3 | Proposta só sem pergunta em aberto |
| H4 | Falha da IA não muda estado. Fica fora do domínio: o domínio só é chamado depois que a IA responde (S03/S07) |
| H5 | Para iniciar: narrador definido e ao menos um investigador. **Estendida (Q5):** o narrador não está entre os investigadores, não há investigador repetido e o modo `SOLO` tem exatamente um investigador |
| H8 | Toda proposta fica registrada como tentativa, com o veredito que recebeu |
| R1 | No máximo uma pergunta aberta por vez |
| R2 | No máximo uma tentativa pendente por vez; pergunta aberta e tentativa pendente nunca coexistem |
| R3 | Comando rejeitado não altera nada e não gera evento |
| R4 | Texto de pergunta e de proposta: espaços nas pontas são removidos; o resultado tem de 1 a 500 caracteres, contados em code points (**Q4**) |
| R5 | Cancelar pergunta ou retirar proposta: só o autor dela ou o narrador (**Q2**, M3) |

## Comportamento esperado

Estado ou fase × comando. "→" indica o novo estado ou fase; um nome em maiúsculas sozinho indica a violação que rejeita o comando.

| | `iniciar` | `fazerPergunta` | `responderPergunta` | `cancelarPergunta` | `proporSolucao` | `retirarProposta` | `aplicarVeredito` | `abandonar` |
|---|---|---|---|---|---|---|---|---|
| `AGUARDANDO_INICIO` | → `AGUARDANDO_PERGUNTA` (H5) | `PARTIDA_NAO_INICIADA` | `PARTIDA_NAO_INICIADA` | `PARTIDA_NAO_INICIADA` | `PARTIDA_NAO_INICIADA` | `PARTIDA_NAO_INICIADA` | `PARTIDA_NAO_INICIADA` | → `ABANDONADO` (H2) |
| `AGUARDANDO_PERGUNTA` | `PARTIDA_JA_INICIADA` | → `PERGUNTA_PENDENTE` | `PERGUNTA_INEXISTENTE` ou `PERGUNTA_JA_FECHADA` | idem | → `SOLUCAO_EM_AVALIACAO` | `SEM_PROPOSTA_PENDENTE` | `SEM_PROPOSTA_PENDENTE` | → `ABANDONADO` |
| `PERGUNTA_PENDENTE` | `PARTIDA_JA_INICIADA` | `PERGUNTA_EM_ABERTO` | → `AGUARDANDO_PERGUNTA` | → `AGUARDANDO_PERGUNTA` | `PERGUNTA_EM_ABERTO` (H3) | `SEM_PROPOSTA_PENDENTE` | `SEM_PROPOSTA_PENDENTE` | → `ABANDONADO` |
| `SOLUCAO_EM_AVALIACAO` | `PARTIDA_JA_INICIADA` | `PROPOSTA_PENDENTE` | `PERGUNTA_INEXISTENTE` ou `PERGUNTA_JA_FECHADA` | idem | `PROPOSTA_PENDENTE` | → `AGUARDANDO_PERGUNTA` | correta → `RESOLVIDO`; incorreta → `AGUARDANDO_PERGUNTA` | → `ABANDONADO` |
| `RESOLVIDO`, `ABANDONADO` | `PARTIDA_ENCERRADA` (H1) | idem | idem | idem | idem | idem | idem | idem |

`iniciar` com configuração inválida rejeita com `CONFIGURACAO_INVALIDA`.

### Ordem de verificação

Quando um comando viola mais de uma regra, a violação devolvida é a primeira desta ordem:

1. `PARTIDA_ENCERRADA`
2. `PARTIDA_NAO_INICIADA` ou `PARTIDA_JA_INICIADA` (`abandonar` pula este passo)
3. Autoria: `NAO_E_INVESTIGADOR`, `NAO_E_NARRADOR`, `NAO_AUTORIZADO`
4. Conteúdo do comando: `TEXTO_INVALIDO`, `PERGUNTA_INEXISTENTE`
5. Estado da investigação: `PERGUNTA_EM_ABERTO`, `PROPOSTA_PENDENTE`, `PERGUNTA_JA_FECHADA`, `SEM_PROPOSTA_PENDENTE`

Exceções:
- `iniciar`: passos 1 e 2, depois `CONFIGURACAO_INVALIDA`.
- `cancelarPergunta` e `retirarProposta`: a autoria depende do item, então a ordem é 1, 2, item existe (`PERGUNTA_INEXISTENTE` ou `SEM_PROPOSTA_PENDENTE`), item aberto (`PERGUNTA_JA_FECHADA`) e só então autoria (`NAO_AUTORIZADO`).

### Eventos

| Evento | Quando | Dados além de `partidaId` e `ocorridoEm` |
|---|---|---|
| `PartidaIniciada` | `iniciar` aceito | — |
| `PerguntaFeita` | `fazerPergunta` aceito | `perguntaId`, `autor`, `texto` |
| `PerguntaRespondida` | `responderPergunta` aceito | `perguntaId`, `resposta` |
| `PerguntaCancelada` (novo) | `cancelarPergunta` aceito | `perguntaId`, `canceladaPor` |
| `SolucaoProposta` | `proporSolucao` aceito | `tentativaId`, `autor`, `texto` |
| `PropostaRetirada` (novo) | `retirarProposta` aceito | `tentativaId`, `retiradaPor` |
| `SolucaoRecusada` (novo) | veredito incorreto | `tentativaId` |
| `PartidaResolvida` | veredito correto | `tentativaId` |
| `PartidaAbandonada` | `abandonar` aceito | `abandonadaPor` |

Criar a partida não gera evento. Os eventos ficam acumulados na partida, na ordem em que ocorreram, até alguém retirá-los.

### Invariantes

Valem depois de qualquer sequência de comandos, aceitos ou rejeitados:

| ID | Invariante |
|---|---|
| I1 | No máximo uma pergunta aberta |
| I2 | No máximo uma tentativa `PENDENTE` |
| I3 | Pergunta aberta e tentativa pendente nunca coexistem |
| I4 | `resolvidaEm` está preenchido se e somente se o status é `RESOLVIDO` |
| I5 | Em estado terminal, nenhum comando muda a partida |
| I6 | Perguntas e tentativas só são acrescentadas; resposta, cancelamento e destino final de uma tentativa não mudam depois de definidos |
| I7 | A fase existe se e somente se o status é `EM_ANDAMENTO`; `PERGUNTA_PENDENTE` se e somente se há pergunta aberta; `SOLUCAO_EM_AVALIACAO` se e somente se há tentativa pendente |
| I8 | Status `RESOLVIDO` implica exatamente uma tentativa `CORRETA`, e ela é a última |

## Critérios de aceitação

Os CAs 01 a 15 mantêm a numeração do planejamento. Em todo CA de rejeição vale R3, verificado pela comparação da partida antes e depois (CA-27). Participantes de exemplo: narrador N, investigadores Ana e Beto, e Zé, que não participa.

| CA | Regra | Dado | Quando | Então |
|---|---|---|---|---|
| 01 | H5 | partida em grupo sem investigadores | `iniciar` | `CONFIGURACAO_INVALIDA`; segue `AGUARDANDO_INICIO` |
| 02 | H5 | narrador N e investigadora Ana | `iniciar` em t1 | `EM_ANDAMENTO`, fase `AGUARDANDO_PERGUNTA`, eventos `[PartidaIniciada(t1)]` |
| 03 | autoria | partida em andamento | N ou Zé fazem pergunta | `NAO_E_INVESTIGADOR` |
| 04 | R1 | pergunta de Ana aberta | Beto faz pergunta | `PERGUNTA_EM_ABERTO` |
| 05 | autoria | pergunta aberta | Ana responde | `NAO_E_NARRADOR` |
| 06 | — | pergunta 1 já respondida | N responde à pergunta 1 ou à pergunta 99 | `PERGUNTA_JA_FECHADA` e `PERGUNTA_INEXISTENTE` |
| 07 | — | pergunta 1 aberta | N responde `SIM` em t2 | pergunta com resposta `SIM` e fechada; fase `AGUARDANDO_PERGUNTA`; evento `PerguntaRespondida(1, SIM)` |
| 08 | H3 | pergunta aberta | Ana propõe solução | `PERGUNTA_EM_ABERTO` |
| 09 | H8 | sem pergunta aberta | Ana propõe "foi o cache" em t3 | tentativa 1 `PENDENTE` com autor e texto; fase `SOLUCAO_EM_AVALIACAO`; evento `SolucaoProposta` |
| 10 | — | tentativa pendente | veredito correto em t4 | `RESOLVIDO`; `resolvidaEm = t4`; tentativa `CORRETA`; evento `PartidaResolvida` |
| 11 | — | tentativa pendente | veredito incorreto | segue `EM_ANDAMENTO`, fase `AGUARDANDO_PERGUNTA`; tentativa `INCORRETA`; evento `SolucaoRecusada`; nova pergunta é aceita |
| 12 | — | sem tentativa pendente | `aplicarVeredito` | `SEM_PROPOSTA_PENDENTE` |
| 13 | H1 | partida `RESOLVIDO` ou `ABANDONADO` | cada um dos 8 comandos | `PARTIDA_ENCERRADA` (16 casos) |
| 14 | H2 | partida não terminal | Zé abandona | `NAO_AUTORIZADO` |
| 15 | H2 | partida em `AGUARDANDO_INICIO` e partida em andamento | N abandona em t5 | `ABANDONADO`; `resolvidaEm` vazio; evento `PartidaAbandonada(N)` |
| 16 | — | partida em andamento | `iniciar` | `PARTIDA_JA_INICIADA` |
| 17 | — | partida em `AGUARDANDO_INICIO` | cada comando exceto `iniciar` e `abandonar` | `PARTIDA_NAO_INICIADA` (6 casos) |
| 18 | R4 | partida em andamento | pergunta ou proposta com `""`, `"   "` ou 501 caracteres | `TEXTO_INVALIDO`; com 500 caracteres é aceita; `"  oi  "` é guardada como `"oi"` |
| 19 | H5 | N também na lista de investigadores, ou Ana repetida | `iniciar` | `CONFIGURACAO_INVALIDA` |
| 20 | H5 | modo `SOLO` | `iniciar` com Ana e Beto; `iniciar` só com Ana | `CONFIGURACAO_INVALIDA`; com Ana sozinha, `EM_ANDAMENTO` |
| 21 | R5 | pergunta de Ana aberta | Ana ou N cancelam | pergunta cancelada e fechada; fase `AGUARDANDO_PERGUNTA`; evento `PerguntaCancelada`; nova pergunta é aceita |
| 22 | R5 | pergunta de Ana aberta; pergunta já respondida | Beto cancela a de Ana; N cancela a respondida | `NAO_AUTORIZADO`; `PERGUNTA_JA_FECHADA` |
| 23 | R5 | tentativa de Ana pendente | Ana ou N retiram; Beto retira; alguém retira sem tentativa pendente | tentativa `RETIRADA`, fase `AGUARDANDO_PERGUNTA`, evento `PropostaRetirada`; `NAO_AUTORIZADO`; `SEM_PROPOSTA_PENDENTE` |
| 24 | R2 | tentativa pendente | Beto faz pergunta | `PROPOSTA_PENDENTE` |
| 25 | R2 | tentativa pendente | Beto propõe | `PROPOSTA_PENDENTE` |
| 26 | — | três comandos aceitos | retirar os eventos duas vezes | a 1ª retirada traz os três eventos, na ordem; a 2ª vem vazia |
| 27 | R3 | qualquer CA de rejeição | — | partida idêntica antes e depois, inclusive os eventos acumulados |
| 28 | H2 | modo `GRUPO`; modo `SOLO` | Ana abandona | `NAO_AUTORIZADO` no grupo; `ABANDONADO` no solo |
| 29 | ordem | partida com pergunta aberta; partida resolvida | Zé faz pergunta; Zé abandona a resolvida | `NAO_E_INVESTIGADOR`; `PARTIDA_ENCERRADA` |
| 30 | I1–I8 | partida nova | sequência aleatória de até 50 comandos | todas as invariantes valem depois de cada comando |
| 31 | — | partida em andamento | duas perguntas e duas tentativas criadas | perguntas com IDs 1 e 2, tentativas com IDs 1 e 2 (numeração própria de cada lista) |
| 32 | R1 | partida em andamento, sem pergunta aberta | Ana pergunta "foi no deploy?" em t2 | pergunta 1 aberta, com autora Ana e o texto; fase `PERGUNTA_PENDENTE`; evento `PerguntaFeita(1, Ana, "foi no deploy?")` |

## Requisitos não funcionais

- **RNF1.** Domínio puro conforme P1 da [constituição](../../docs/constitution.md).
- **RNF2.** Cobertura de linhas de pelo menos 90% em `com.quebugesse.domain.gamesession`.
- **RNF3.** A suíte do módulo `domain` roda em menos de 10 segundos numa máquina de desenvolvimento.

## Fora de escopo

- `Enigma`, `Player` e o sigilo de `solucaoCompleta` (S02, S03).
- Casos de uso, H4 (falha da IA e retentativa) e autorização por sessão autenticada (S03, S06).
- Publicação dos eventos depois de persistir, H6 (S04).
- Entrada de investigadores depois da criação, sala de espera (S06, M11).
- Persistência, reconstrução da partida a partir do banco e bloqueio otimista (S09).
- Revelar a solução ao abandonar, pontuação e limites de perguntas ou tentativas (M24).

## Divergências em relação aos diagramas

Pelo princípio P9, `diagrama_estados` e `diagrama_classes` precisam ser atualizados com estes pontos (tarefa 9 de [tasks.md](tasks.md)).

| ID | Diagrama hoje | Esta spec | Origem |
|---|---|---|---|
| D1 | Só `aplicarVeredito` recebe instante; `Question.criadaEm` não tem origem | Todo comando recebe `agora: Instant` | P1 |
| D2 | `GameSession` não guarda o modo | `modo: ModoDeJogo` | M5 |
| D3 | Proposta não é registrada | Entidade `Tentativa` e lista `tentativas` | M4, H8 |
| D4 | Pergunta pendente e solução em avaliação só saem por resposta, veredito ou abandono | `cancelarPergunta` e `retirarProposta` voltam a `AGUARDANDO_PERGUNTA` | M3 |
| D5 | Seis eventos, fora do diagrama de classes | Nove eventos sob `sealed interface EventoDePartida`, que estende a interface comum `DomainEvent` de `domain.shared`; novos: `PerguntaCancelada`, `PropostaRetirada` e `SolucaoRecusada` | M20 |
| D6 | H5: narrador e ao menos um investigador | H5 estendida | Q5 |
| D7 | H2: "participante autorizado" sem definição | Narrador sempre; investigador só no solo | Q1, M10 |
| D8 | `/aberta = resposta == null` | `aberta = resposta == null && !cancelada` | M3 |
| D9 | Subestados só no diagrama de estados | `fase: FaseDaInvestigacao?` exposta pela partida | — |
| D10 | Tipo de `QuestionId` não definido | `QuestionId` e `TentativaId` numerados a partir de 1 dentro de cada partida | — |

## Questões abertas

Todas têm decisão provisória para não travar a implementação. Se a equipe decidir diferente, mudam os CAs indicados.

| Q | Pergunta | Decisão provisória | CAs afetados |
|---|---|---|---|
| Q1 | Quem pode abandonar (H2)? | Narrador sempre; investigador só no `SOLO`. No grupo, a saída de um investigador será "sair da partida" (S06) | 14, 15, 28 |
| Q2 | Quem cancela pergunta ou retira proposta? | O autor do item ou o narrador | 21, 22, 23 |
| Q3 | Quem aplica o veredito no grupo? ([ADR 0001](../../docs/adr/0001-veredito-no-modo-grupo.md)) | O domínio não verifica autor no `aplicarVeredito`, como no diagrama. Se a ADR 0001 decidir "narrador humano", o comando ganha `autor` e um CA novo | 10, 11, 12 |
| Q4 | Limite de texto | 1 a 500 code points depois de remover espaços das pontas | 18 |
| Q5 | Regras extras de H5 | Narrador fora da lista de investigadores, sem repetidos, `SOLO` com exatamente um | 19, 20 |

**Q3, atualização de 26/10/2026:** a [S03](../003-casos-de-uso-da-partida/spec.md) adota a opção A da [ADR 0001](../../docs/adr/0001-veredito-no-modo-grupo.md), ainda em `Proposta`. Pela emenda 1 da S03, `aplicarVeredito` ganha `autor` e rejeita quem não é o narrador com `NAO_E_NARRADOR`, coberto pelo novo CA-33.

## Rastreabilidade

| CA | Arquivo de teste |
|---|---|
| 01, 02, 16, 19, 20 | `IniciarPartidaTest.kt` |
| 03 a 07, 18 (pergunta), 26, 31 (perguntas), 32 | `PerguntasTest.kt` |
| 21, 22 | `CancelarPerguntaTest.kt` |
| 08 a 12, 18 (proposta), 24, 25, 31 (tentativas) | `PropostasTest.kt` |
| 23 | `RetirarPropostaTest.kt` |
| 14, 15, 28 | `AbandonoTest.kt` |
| 13, 17, 29 | `EstadosEPrecedenciaTest.kt` |
| 27 | todos (via `deveRejeitar`) |
| 30 | `InvariantesPropertyTest.kt` |
