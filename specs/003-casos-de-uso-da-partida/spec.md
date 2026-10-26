# Spec 003: Casos de uso da partida

| Campo | Valor |
|---|---|
| ID | S03 |
| Status | Rascunho |
| Depende de | S01 (`GameSession`), S02 (`Enigma`, `Player`, repositórios em memória e seed) |
| Casos de uso | UC2 Selecionar enigma (sorteio e "já jogados"), UC3 Iniciar partida, UC4 Fazer pergunta, UC5 Responder pergunta, UC6 Propor solução, UC10 Jogar sozinho (caminho da IA, com fakes), UC11 Abandonar partida |
| Diagramas de origem | `diagrama_seq_iniciar.pdf`, `diagrama_seq_jogar.pdf`, `Diagrama_comp.pdf` (camada de aplicação e portas) |
| Decisões que assume | [ADR 0001](../../docs/adr/0001-veredito-no-modo-grupo.md) e [ADR 0002](../../docs/adr/0002-contrato-da-porta-narrador.md), ambas em `Proposta`: esta spec segue a opção A das duas |
| Melhorias incorporadas | M1, M2, M3 (parte de aplicação), M5, M10, M12 e M18 (cálculo de "já jogados") do [planejamento](../../planejamento-sdd.md) |
| Plano | Plano e tarefas: ainda não escritos |

## Contexto

A S01 entrega a partida e a S02 entrega o catálogo e o jogador, mas ninguém ainda carrega um agregado, executa um comando e salva o resultado. Esta spec cria a camada de aplicação: um caso de uso por comando, as portas que a partida precisa (repositório, `Narrador` e `VerificadorDeSolucao`), a visão da partida por papel (M1) e os códigos de erro que a camada de entrada vai traduzir.

Tudo roda com fakes e repositórios em memória (P6). A publicação dos eventos fica para a S04, a rede para a S06 e os adaptadores de LLM para a S07 e a S08.

## Histórias

- **US1.** Como quem inicia a partida, quero escolher o modo, a categoria e a dificuldade e receber um enigma que o grupo ainda não jogou (UC2, UC3, M18).
- **US2.** Como investigador, quero perguntar, propor, cancelar a minha pergunta e retirar a minha proposta (UC4, UC6, M3).
- **US3.** Como narrador humano, quero responder às perguntas e julgar as propostas da minha partida (UC5, ADR 0001).
- **US4.** Como jogador solo, quero que a IA responda e avalie; se ela falhar, quero que nada mude e que eu possa pedir de novo (UC10, H4).
- **US5.** Como participante, quero consultar a partida vendo só o que o meu papel permite (M1, P3).
- **US6.** Como camada de entrada (CLI, REST, WebSocket), quero receber DTOs e códigos de erro estáveis, sem tipos do domínio (S00, Q1).
- **US7.** Como desenvolvedor, quero testar todos os casos de uso sem rede nem banco (P6).

## Glossário

| Termo | Significado |
|---|---|
| Caso de uso | Classe em `application.usecase` com uma operação: carrega o que precisa, chama o agregado e salva |
| Iniciador | Jogador que pede a partida. No `GRUPO` ele é o narrador; no `SOLO` ele é o investigador (M12) |
| Jogador IA | O `Player` do tipo `IA` que narra no solo. O ID chega ao caso de uso por configuração; no `bootstrap`, é `SeedInicial.JOGADOR_IA` da S02 |
| Visão (`VisaoNarrador`, `VisaoInvestigador`) | DTO de saída montado conforme o papel de quem pede (novo, M1) |
| Código de erro (`CodigoErro`) | Enum da aplicação que diz por que um caso de uso não foi executado (novo). A camada de entrada só conhece este enum |
| Já jogados | Enigmas das partidas, em qualquer status, em que algum investigador da partida nova já foi investigador |
| Sorteador | Interface da aplicação que escolhe um item de uma lista não vazia. Em produção, sorteio uniforme; no teste, determinístico (novo) |
| Gerador de IDs (`GeradorDeIds`) | Interface da aplicação que cria `GameSessionId` e `PlayerId` novos (novo) |
| Relógio | `java.time.Clock` injetado no caso de uso; o domínio continua recebendo só `agora: Instant` (P1) |
| Contexto da narração (`ContextoDaNarracao`) | O que a IA recebe para responder: enunciado, solução completa e as perguntas já respondidas, com as respostas, em ordem. Só texto e `Answer`, sem tipos de `domain.enigma` (P7) |

## Regras

| ID | Regra |
|---|---|
| A1 | Participantes de uma partida nova. `GRUPO`: o narrador é o iniciador e os investigadores são os informados. `SOLO`: o narrador é o jogador IA e os investigadores são o iniciador seguido dos informados; o domínio rejeita mais de um (H5 estendida da S01, M12) (**Q6**) |
| A2 | Todo ID de jogador informado (iniciador e investigadores) existe no `PlayerRepository`, senão `JOGADOR_INEXISTENTE`. Um jogador do tipo `IA` no lugar de um humano dá `CONFIGURACAO_INVALIDA` |
| A3 | Seleção: candidatos = `EnigmaRepository.publicados()`; já jogados = `GameSessionRepository.enigmasJogadosPor(investigadores)`; elegíveis = `CriterioDeSelecao(categoria, dificuldade).elegiveis(candidatos, jaJogados)`. Lista vazia dá `SEM_ENIGMA_ELEGIVEL`; senão o sorteador escolhe um (**Q5**) |
| A4 | IDs novos vêm do `GeradorDeIds`. O relógio é lido uma vez por execução, e esse instante vai para todos os comandos do agregado naquela execução |
| A5 | Comando sobre partida existente: buscar a partida (`PARTIDA_INEXISTENTE` se não houver), chamar o comando do agregado e salvar. Comando aceito chama `salvar` uma vez; comando rejeitado não chama `salvar` e devolve o `CodigoErro` de mesmo nome da `Violacao` |
| A6 | `ResponderComIA(partida)`: só no `SOLO` (`MODO_INCOMPATIVEL`), partida não encerrada (`PARTIDA_ENCERRADA`) e com pergunta aberta (`SEM_PERGUNTA_ABERTA`). Monta o contexto da narração, chama a porta `Narrador` e aplica a resposta com `responderPergunta` em nome do jogador IA. Se a porta lançar exceção, devolve `IA_INDISPONIVEL` e não salva nada: a pergunta segue aberta (H4) (**Q2**) |
| A7 | `AvaliarComIA(partida)`: só no `SOLO`, partida não encerrada e com tentativa pendente (`SEM_PROPOSTA_PENDENTE`). Chama `VerificadorDeSolucao.avaliar(texto da proposta, solução completa)` e aplica o veredito em nome do jogador IA. Falha da porta: `IA_INDISPONIVEL`, nada salvo, a tentativa segue pendente (**Q1**) |
| A8 | O narrador humano responde com `ResponderPergunta` e julga com `JulgarProposta`, que chama `aplicarVeredito(autor, veredito, agora)` (emenda 1). No solo o narrador é o jogador IA, então um humano recebe `NAO_E_NARRADOR` nos dois (**Q1**) |
| A9 | Visão por papel. O narrador da partida recebe `VisaoNarrador`, com a solução. Um investigador recebe `VisaoInvestigador`, sem a solução enquanto a partida não chega a `RESOLVIDO` ou `ABANDONADO`, e com ela depois (**Q3**). Quem não participa recebe `NAO_PARTICIPA` em `ConsultarPartida` |
| A10 | Fronteira. Entradas e saídas públicas dos casos de uso usam só tipos da biblioteca padrão (`UUID`, `String`) e de `application.dto`: nenhum tipo do domínio aparece nelas. Os enums do DTO espelham os do domínio, nome por nome |
| A11 | `RegistrarJogador(nome)` cria um `Player` `HUMANO` com ID do gerador e o salva. Nome inválido dá `NOME_INVALIDO` (J1 da S02) (**Q7**) |
| A12 | Todo caso de uso de comando devolve, quando aceito, a visão de quem o executou, já com o comando aplicado. `ResponderComIA` e `AvaliarComIA` devolvem a visão do investigador da partida |

### Casos de uso

Todos em `com.quebugesse.application.usecase`. Os IDs entram como `UUID`.

| Caso de uso | Entrada | Comando do agregado | Observação |
|---|---|---|---|
| `IniciarPartida` | modo, iniciador, investigadores, categoria?, dificuldade? | `criar` e `iniciar` | A1 a A4 |
| `FazerPergunta` | partida, autor, texto | `fazerPergunta` | |
| `ResponderPergunta` | partida, autor, pergunta, resposta | `responderPergunta` | narrador humano (A8) |
| `CancelarPergunta` | partida, autor, pergunta | `cancelarPergunta` | M3 |
| `ResponderComIA` | partida | `responderPergunta` | só solo (A6) |
| `ProporSolucao` | partida, autor, texto | `proporSolucao` | |
| `RetirarProposta` | partida, autor | `retirarProposta` | M3 |
| `JulgarProposta` | partida, autor, correta | `aplicarVeredito` | narrador humano (A8, ADR 0001 A) |
| `AvaliarComIA` | partida | `aplicarVeredito` | só solo (A7) |
| `Abandonar` | partida, autor | `abandonar` | H2 da S01 |
| `ConsultarPartida` | partida, solicitante | nenhum | A9 |
| `RegistrarJogador` | nome | nenhum (`Player.criar`) | A11 |

`ResponderComIA` e `AvaliarComIA` não são comandos de jogador: quem os chama é o ouvinte de eventos da S04 (ou o teste). A camada de entrada não deve expô-los como ação de um jogador.

### Portas novas

Em `com.quebugesse.domain.gamesession.port` (M16):

| Porta | Operações | Observação |
|---|---|---|
| `GameSessionRepository` | `buscar(id)`, `salvar(partida)`, `enigmasJogadosPor(jogadores: Set<PlayerRef>): Set<EnigmaId>` | RP1 a RP3 da S02 valem igual. O repositório não guarda eventos pendentes |
| `Narrador` | `responder(pergunta: String, contexto: ContextoDaNarracao): Answer` | Só a IA implementa (ADR 0002 A). Falha = exceção |
| `VerificadorDeSolucao` | `avaliar(proposta: String, solucaoCompleta: String): Veredito` | Só usada no solo (ADR 0001 A). Falha = exceção |

Implementação em memória: `GameSessionRepositoryEmMemoria`, em `infrastructure.persistence.memoria`, com a mesma suíte de contrato que a S09 vai rodar no banco. `Narrador` e `VerificadorDeSolucao` ganham fakes programáveis, inclusive para falhar (P6); os adaptadores reais vêm na S07 e na S08.

### Visões

| Campo | `VisaoNarrador` | `VisaoInvestigador` |
|---|---|---|
| partida, modo, status, fase | sim | sim |
| título, enunciado, categoria, dificuldade | sim | sim |
| narrador e investigadores (ID e nome) | sim | sim |
| perguntas (ID, autor, texto, resposta, cancelada) | sim | sim |
| tentativas (ID, autor, texto, situação) | sim | sim |
| solução completa | sempre | só em `RESOLVIDO` ou `ABANDONADO`; antes, vazia |

### Códigos de erro

`CodigoErro`, em `application.dto`, tem:

- os 13 nomes de `Violacao` da S01, com o mesmo nome: `PARTIDA_ENCERRADA`, `PARTIDA_NAO_INICIADA`, `PARTIDA_JA_INICIADA`, `CONFIGURACAO_INVALIDA`, `NAO_E_INVESTIGADOR`, `NAO_E_NARRADOR`, `NAO_AUTORIZADO`, `TEXTO_INVALIDO`, `PERGUNTA_INEXISTENTE`, `PERGUNTA_JA_FECHADA`, `PERGUNTA_EM_ABERTO`, `PROPOSTA_PENDENTE`, `SEM_PROPOSTA_PENDENTE`;
- `NOME_INVALIDO`, da S02;
- os códigos da aplicação: `PARTIDA_INEXISTENTE`, `JOGADOR_INEXISTENTE`, `SEM_ENIGMA_ELEGIVEL`, `MODO_INCOMPATIVEL`, `SEM_PERGUNTA_ABERTA`, `IA_INDISPONIVEL`, `NAO_PARTICIPA`.

A S04 acrescenta `FALHA_AO_SALVAR`. Dado inconsistente no repositório (uma partida que aponta para um enigma que não existe) não vira código: é erro interno.

### Ordem de verificação

- `IniciarPartida`: `JOGADOR_INEXISTENTE`, depois jogador IA no lugar de humano (`CONFIGURACAO_INVALIDA`), depois `SEM_ENIGMA_ELEGIVEL`, e por fim as violações de `iniciar` (H5).
- `ResponderComIA` e `AvaliarComIA`: `PARTIDA_INEXISTENTE`, `MODO_INCOMPATIVEL`, `PARTIDA_ENCERRADA`, `SEM_PERGUNTA_ABERTA` ou `SEM_PROPOSTA_PENDENTE`, e só então a chamada à porta. Nenhuma rejeição chama a porta.
- Demais casos de uso: `PARTIDA_INEXISTENTE` e depois a ordem da S01.

### Emendas à S01

Entram como tarefas da S03 (ou da S01, se ela ainda não tiver sido implementada) e mudam o modelo da S01:

1. **`aplicarVeredito(autor: PlayerRef, veredito: Veredito, agora: Instant)`.** O autor tem de ser o narrador, senão `NAO_E_NARRADOR`, verificado no passo 3 da ordem da S01 (antes de `SEM_PROPOSTA_PENDENTE`). Novo CA-33 da S01: "tentativa pendente; Ana aplica o veredito; `NAO_E_NARRADOR`". Resolve a Q3 da S01 pela opção A da ADR 0001.
2. **`GameSession.reconstituir(...)`**, que a S01 deixava para a S09. Recebe todos os campos (ID, enigma, modo, participantes, `criadaEm`, status, `resolvidaEm`, perguntas e tentativas), não gera eventos e recusa dados que violem I1 a I8. O repositório em memória precisa dela para guardar cópias (RP3).

## Critérios de aceitação

Participantes de exemplo: narrador N e investigadores Ana e Beto, humanos; Zé, humano que não participa; IA, o jogador IA. Enigmas e1 a e5 do seed da S02. Relógio fixo em t1; gerador de IDs e sorteador determinísticos.

| CA | Regra | Dado | Quando | Então |
|---|---|---|---|---|
| 01 | A1, A4 | N, Ana e Beto registrados | N inicia `GRUPO` com Ana e Beto | partida salva em `EM_ANDAMENTO`, com o ID do gerador, narrador N, investigadores [Ana, Beto], `criadaEm = t1`; devolve `VisaoNarrador` com a solução |
| 02 | A1, A9 | Ana registrada | Ana inicia `SOLO` sem mais ninguém | narrador IA, investigadores [Ana]; devolve `VisaoInvestigador` sem a solução |
| 03 | A1 | Ana e Beto registrados | Ana inicia `SOLO` informando Beto | `CONFIGURACAO_INVALIDA`; nada salvo |
| 04 | A2 | Zé não registrado | N inicia com Zé; Zé inicia | `JOGADOR_INEXISTENTE` nos dois; nada salvo; o sorteador não é chamado |
| 05 | A2 | IA registrado | IA inicia `GRUPO`; N inicia `GRUPO` com IA como investigador | `CONFIGURACAO_INVALIDA` nos dois |
| 06 | A1 | N e Ana registrados | N inicia `GRUPO` com [Ana, N]; com [Ana, Ana] | `CONFIGURACAO_INVALIDA` (H5 da S01); nada salvo |
| 07 | A3 | os cinco enigmas publicados | N inicia com Ana e categoria `DEBUG` | o sorteador recebe só os elegíveis (e1) e a partida usa o escolhido |
| 08 | A3 | Ana já investigou e1 numa partida abandonada; Beto investigou e2 numa resolvida; N narrou e3 | N inicia com Ana e Beto | o sorteador recebe e3, e4 e e5, e nenhum outro |
| 09 | A3 | todos os enigmas `DEBUG` já jogados por Ana | Ana pede `DEBUG` | `SEM_ENIGMA_ELEGIVEL`; nada salvo; o sorteador não é chamado |
| 10 | A5, A12 | partida em grupo em andamento, relógio em t2 | Ana faz a pergunta "foi no deploy?" | partida salva com a pergunta 1 aberta, criada em t2; devolve `VisaoInvestigador` com a pergunta |
| 11 | A5 | partida inexistente | cada caso de uso que recebe partida (10) | `PARTIDA_INEXISTENTE` |
| 12 | A5 | pergunta de Ana aberta | Beto faz pergunta | `PERGUNTA_EM_ABERTO`; `salvar` não é chamado; a partida guardada não muda |
| 13 | A5, A12 | partida em grupo com o cenário certo para cada comando | `ResponderPergunta`, `CancelarPergunta`, `ProporSolucao`, `RetirarProposta` e `Abandonar`, cada um por quem pode | estado salvo igual ao do agregado depois do comando; cada um devolve a visão do autor |
| 14 | A8 | tentativa de Ana pendente no grupo | N julga certa; em outra partida, N julga errada | `RESOLVIDO` e a visão de Ana passa a ter a solução; na outra, tentativa `INCORRETA` e fase `AGUARDANDO_PERGUNTA` |
| 15 | A8 | tentativa pendente no grupo; solo com pergunta aberta; solo com tentativa pendente | Ana julga no grupo; Ana responde no primeiro solo; Ana julga no segundo | `NAO_E_NARRADOR` nos três |
| 16 | A6 | solo com a pergunta 2 aberta e a 1 respondida `NAO`; `Narrador` fake devolve `SIM` | `ResponderComIA` | pergunta 2 com `SIM`, salva; o fake recebeu o texto da pergunta 2 e o contexto com enunciado, solução e a pergunta 1 com `NAO` |
| 17 | A6 | partida em grupo com pergunta aberta; solo sem pergunta aberta; solo resolvido | `ResponderComIA` | `MODO_INCOMPATIVEL`; `SEM_PERGUNTA_ABERTA`; `PARTIDA_ENCERRADA`; a porta não é chamada |
| 18 | A6, H4 | solo com pergunta aberta; fake que lança exceção | `ResponderComIA`; depois, com o fake respondendo `NAO`, de novo | `IA_INDISPONIVEL`, a partida guardada não muda e a pergunta segue aberta; na segunda chamada, resposta `NAO` |
| 19 | A7 | solo com tentativa pendente; `VerificadorDeSolucao` fake | `AvaliarComIA` com o fake dizendo correta; em outra partida, incorreta | `RESOLVIDO`, e o fake recebeu o texto da proposta e a solução; na outra, tentativa `INCORRETA` |
| 20 | A7 | grupo com tentativa pendente; solo sem tentativa; solo com tentativa e fake que falha | `AvaliarComIA` | `MODO_INCOMPATIVEL`; `SEM_PROPOSTA_PENDENTE`; `IA_INDISPONIVEL` com a tentativa ainda pendente e nada salvo |
| 21 | A9 | partida em grupo em andamento | N, Ana e Zé consultam | `VisaoNarrador` com a solução; `VisaoInvestigador` sem a solução; `NAO_PARTICIPA` |
| 22 | A9 | uma partida `RESOLVIDO` e outra `ABANDONADO` | Ana consulta | `VisaoInvestigador` com a solução nas duas |
| 23 | A9, P3 | roteiro que passa por todas as fases, no grupo e no solo | cada caso de uso que devolve `VisaoInvestigador` antes de um estado terminal | o `toString()` da visão não contém o texto da solução |
| 24 | A10 | os enums `Violacao`, `Answer`, `GameStatus`, `ModoDeJogo`, `FaseDaInvestigacao`, `SituacaoTentativa`, `Categoria` e `Dificuldade` | mapear cada valor para o DTO | todo valor tem par de mesmo nome; cada `Violacao` lançada pelo agregado vira o `CodigoErro` de mesmo nome |
| 25 | RP1 a RP3, A3 | `GameSessionRepositoryEmMemoria` | suíte de contrato: buscar inexistente, salvar e substituir, cópia na entrada e na saída, `enigmasJogadosPor` com partidas em vários status | nulo; o último salvo; o guardado só muda com `salvar`; os enigmas das partidas em que algum dos jogadores foi investigador, sem contar partidas em que ele só narrou |
| 26 | emenda 2 | partida com perguntas, tentativas e eventos pendentes | `reconstituir` com os campos dela; e com duas perguntas abertas | partida com a mesma fotografia e sem eventos pendentes, que aceita o próximo comando; a segunda chamada é recusada |
| 27 | A11 | — | `RegistrarJogador("  Ana  ")`; `RegistrarJogador("")` | jogador `HUMANO` salvo com o nome `"Ana"` e o ID do gerador; `NOME_INVALIDO` |

A emenda 1 é coberta pelo CA-33 da S01 e, na aplicação, pelo CA-15.

## Requisitos não funcionais

- **RNF1.** `application` não acessa rede, banco, arquivo nem o relógio do sistema; tempo só pelo `Clock` injetado e aleatoriedade só pelo sorteador.
- **RNF2.** Cobertura de linhas de pelo menos 90% em `com.quebugesse.application` e em `com.quebugesse.infrastructure.persistence`.
- **RNF3.** A suíte de `application` roda em menos de 10 segundos numa máquina de desenvolvimento.
- **RNF4.** `GameSessionRepositoryEmMemoria` aceita chamadas de várias threads, como os repositórios da S02.

## Fora de escopo

- Publicação de eventos depois de salvar e os ouvintes que chamam `ResponderComIA` e `AvaliarComIA` (S04).
- REST, WebSocket, sala de espera, entrada em partida e autenticação (S06, ADR 0007).
- Adaptadores reais de LLM, retentativa com espera e defesa contra injeção de prompt (S07, S08, M7).
- Persistência em banco e bloqueio otimista (S09, M8).
- Estatísticas do jogador (S12), geração de enigma por IA (S11), limites e pontuação (M24), prazo para o narrador humano (M3).

## Divergências em relação aos diagramas

Pelo princípio P9, as fontes em [docs/diagramas/](../../docs/diagramas/README.md) recebem estes pontos no PR que implementar esta spec.

| ID | Diagrama hoje | Esta spec | Origem |
|---|---|---|---|
| D1 | Seq. 1, passo 02: `executar(modo, investigadores)` | `IniciarPartida(modo, iniciador, investigadores, categoria?, dificuldade?)` | M12, M18 |
| D2 | Seq. 1, passos 05 e 06: `narrador = NarradorIA` ou `NarradorHumano` | O narrador é um `PlayerRef`: o iniciador no grupo, o jogador IA no solo | M2, M5 |
| D3 | Seq. 1, passo 07: `criar(enigma.id, narrador, investigadores)` | `criar(id, enigmaId, modo, narrador, investigadores, agora)`, com ID do gerador e instante do relógio | M5, P1 |
| D4 | Seq. 1, passo 13: `PartidaDto(enunciado, status)`; nota "solucaoCompleta nunca sai do domínio" | `VisaoNarrador` ou `VisaoInvestigador`, conforme o papel | M1, P3 |
| D5 | Seq. 2, passos 04 a 11: `PerguntaFeita` dispara `ResponderPerguntaUseCase`, que consulta o `Narrador` humano ou IA | No grupo, o narrador chama `ResponderPergunta` direto. No solo, `ResponderComIA` consulta a porta e aplica a resposta | ADR 0002 A, M2 |
| D6 | Seq. 2, passos 15 a 18: toda proposta vai ao `VerificadorDeSolucao` | Só no solo (`AvaliarComIA`); no grupo, `JulgarProposta` pelo narrador | ADR 0001 A |
| D7 | Seq. 2, passo 19, classes e estados: `aplicarVeredito(veredito, agora)`, sem guarda de autor | `aplicarVeredito(autor, veredito, agora)` | emenda 1 |
| D8 | Componentes: casos de uso `IniciarPartida · FazerPergunta · ResponderPergunta · ProporSolucao · GerarEnigmaComIA · JogarSolo` | Os doze da tabela "Casos de uso". `JogarSolo` não vira caso de uso: o solo é `IniciarPartida(SOLO)` mais `ResponderComIA` e `AvaliarComIA`. `GerarEnigmaComIA` fica na S11 | M3, M10 |
| D9 | Componentes e pacotes: portas sem pacote. Componentes: repositório sem consulta de "já jogados" | `GameSessionRepository`, `Narrador` e `VerificadorDeSolucao` em `domain.gamesession.port`; `enigmasJogadosPor` | M16, M18 |
| D10 | Classes: `GameSession` sem fábrica de reconstrução | `GameSession.reconstituir` | emenda 2 |

## Questões abertas

Todas têm decisão provisória para não travar a implementação. Se a equipe decidir diferente, mudam os CAs indicados.

| Q | Pergunta | Decisão provisória | CAs afetados |
|---|---|---|---|
| Q1 | Quem dá o veredito no grupo? ([ADR 0001](../../docs/adr/0001-veredito-no-modo-grupo.md)) | Opção A: o narrador humano, por `JulgarProposta`; IA só no solo, por `AvaliarComIA`. Exige a emenda 1 à S01 | 14, 15, 19, 20 |
| Q2 | A porta `Narrador` é síncrona? ([ADR 0002](../../docs/adr/0002-contrato-da-porta-narrador.md)) | Opção A: porta síncrona só para a IA; o humano responde por `ResponderPergunta` | 16, 17, 18 |
| Q3 | O investigador vê a solução depois de abandonar? | Sim, em qualquer estado terminal. A partida acabou e não há mais o que esconder | 22 |
| Q4 | Os testes de `application` podem usar os repositórios em memória de `infrastructure`? | Sim, com `testImplementation(project(":infrastructure"))` e as `testFixtures` da S02. F2 da S00 vale para o código de produção, e a DP5 do plano da S00 libera os testes | 01 a 27 |
| Q5 | Como calcular "já jogados" (Q3 da S02)? | Enigmas de partidas em qualquer status em que algum investigador da partida nova foi investigador. Quem só narrou não conta, porque o narrador conhece a solução de qualquer forma | 08, 09, 25 |
| Q6 | Quem inicia no solo (M12)? | O próprio investigador; o narrador é o jogador IA configurado | 02, 03 |
| Q7 | Como um jogador passa a existir antes da S06? ([ADR 0007](../../docs/adr/0007-identidade-e-autenticacao.md)) | `RegistrarJogador(nome)` cria um convidado humano, sem senha; nomes repetidos são permitidos (Q4 da S02). A S06 decide token e sala | 27 |

## Rastreabilidade

| CA | Arquivo de teste |
|---|---|
| 01 a 09 | `application/.../usecase/IniciarPartidaTest.kt` |
| 10 a 13 | `application/.../usecase/ComandosDaPartidaTest.kt` |
| 14, 15 | `application/.../usecase/JulgarPropostaTest.kt` |
| 16 a 18 | `application/.../usecase/ResponderComIATest.kt` |
| 19, 20 | `application/.../usecase/AvaliarComIATest.kt` |
| 21 a 23 | `application/.../usecase/ConsultarPartidaTest.kt` |
| 24 | `application/.../dto/MapeamentoTest.kt` |
| 25 | `infrastructure/src/testFixtures/.../contrato/GameSessionRepositoryContrato.kt`, executado por `GameSessionRepositoryEmMemoriaTest.kt` |
| 26 | `domain/.../gamesession/ReconstituirTest.kt` |
| 27 | `application/.../usecase/RegistrarJogadorTest.kt` |
