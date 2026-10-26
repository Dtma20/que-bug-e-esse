# Spec 004: Eventos depois de persistir

| Campo | Valor |
|---|---|
| ID | S04 |
| Status | Rascunho |
| Depende de | S03 (casos de uso, `GameSessionRepository`, `ResponderComIA`, `AvaliarComIA`, `CodigoErro`), S01 (`puxarEventos()` e os nove eventos de `EventoDePartida`) |
| Casos de uso | Nenhum novo. Fecha o UC10 Jogar sozinho: a resposta e o veredito da IA passam a ser disparados pelos eventos `PerguntaFeita` e `SolucaoProposta` |
| Diagramas de origem | `Diagrama_comp.pdf` (porta "difusão de DomainEvent" e adaptador `messaging`), `diagrama_seq_jogar.pdf` (passo 04 e nota "salva ao final"), `Diagrama_imp.pdf` (`WebSocket / SSE (messaging)`) |
| Decisões que assume | [ADR 0002](../../docs/adr/0002-contrato-da-porta-narrador.md) e [ADR 0001](../../docs/adr/0001-veredito-no-modo-grupo.md), opção A das duas, como na S03; [ADR 0004](../../docs/adr/0004-banco-de-dados.md) só para registrar que o outbox fica na S09. Todas em `Proposta` |
| Melhorias incorporadas | M9 (recuperar H6), M2 (lado do solo) e o ponto de extensão que a M17 vai usar na S12, do [planejamento](../../planejamento-sdd.md) |
| Plano | Plano e tarefas: ainda não escritos |

## Contexto

A S01 acumula os eventos na partida e a S03 salva a partida, mas ninguém lê os eventos: eles somem com o objeto. Os rascunhos de setembro tinham a regra H6, "eventos distribuídos após persistência", que caiu dos PDFs (M9). Sem ela, um evento pode avisar uma tela de uma mudança que não foi gravada.

Esta spec cria a porta de publicação, um adaptador em memória síncrono, a regra "salva, depois publica" em todos os casos de uso e os dois ouvintes que fecham o ciclo do solo: pergunta feita, IA responde; proposta feita, IA avalia. Levar os eventos às telas pela rede fica para a S06; o outbox, para a S09.

## Histórias

- **US1.** Como participante, quero ser avisado só de mudanças que foram gravadas, para a tela nunca mostrar algo que se perdeu (H6, M9).
- **US2.** Como jogador solo, quero que a IA responda à minha pergunta e avalie a minha proposta logo depois que eu as envio, sem pedir de novo (UC10).
- **US3.** Como jogador solo, quero que uma falha da IA deixe a pergunta aberta e não estrague a jogada que eu já fiz (H4).
- **US4.** Como desenvolvedor, quero acrescentar ouvintes (as telas na S06, as estatísticas na S12) sem mudar os casos de uso.
- **US5.** Como desenvolvedor, quero que um ouvinte com defeito não impeça os outros nem desfaça o comando do jogador.

## Glossário

| Termo | Significado |
|---|---|
| Publicar | Entregar uma lista de eventos aos ouvintes registrados |
| Publicador (`PublicadorDeEventos`) | Porta do domínio que recebe os eventos a publicar (novo) |
| Ouvinte | Função chamada com cada evento publicado. Os ouvintes do jogo ficam em `application.ouvinte` (novo) |
| Fila de despacho | Eventos publicados e ainda não entregues, na ordem de chegada. Cada thread tem a sua (novo) |
| Chamada externa | `publicar` feito quando a fila da thread está vazia e nenhuma entrega está em curso, como o de um caso de uso chamado pela camada de entrada. As chamadas feitas por um ouvinte durante uma entrega são internas |
| Registro de falhas | Função que o publicador chama com o evento, o ouvinte e a exceção quando um ouvinte falha. No `bootstrap` escreve uma linha de log; no teste, acrescenta a uma lista (novo) |

## Regras

| ID | Regra |
|---|---|
| H6 | Eventos só saem depois de persistir. Depois de um comando aceito, o caso de uso chama `salvar`; só quando ele retorna, puxa os eventos da partida (`puxarEventos()`) e chama `publicar` uma vez, com todos, na ordem em que ocorreram. Se `salvar` lançar exceção, nada é publicado e o caso de uso devolve `FALHA_AO_SALVAR`. Comando rejeitado não publica (R3 da S01) (**Q5**) |
| EV1 | Isolamento. A exceção de um ouvinte vai para o registro de falhas e não impede a entrega aos outros ouvintes nem dos eventos seguintes. O resultado do caso de uso que publicou não muda (**Q2**) |
| EV2 | Ordem. Cada evento é entregue a todos os ouvintes, na ordem de registro, antes de o próximo evento ser entregue. Os eventos de uma chamada são entregues na ordem da lista |
| EV3 | Fila FIFO sem recursão. Um `publicar` interno põe os eventos no fim da fila e retorna sem entregá-los. A chamada externa entrega a fila até ela ficar vazia e só então retorna |
| EV4 | Despacho síncrono, na thread de quem fez a chamada externa. Publicações de threads diferentes não se misturam (**Q1**) |
| EV5 | Visão de retorno. O caso de uso monta a visão depois de publicar, relendo a partida do repositório. Assim, no solo, quem perguntou já recebe a resposta da IA (**Q3**) |
| OS1 | Ouvinte de `PerguntaFeita`: carrega a partida; se o modo é `SOLO`, chama `ResponderComIA(partida)`; no `GRUPO`, não faz nada (**Q4**) |
| OS2 | Ouvinte de `SolucaoProposta`: carrega a partida; se o modo é `SOLO`, chama `AvaliarComIA(partida)`; no `GRUPO`, não faz nada |
| OS3 | Se `ResponderComIA` ou `AvaliarComIA` devolver um código de erro, como `IA_INDISPONIVEL`, o ouvinte lança uma exceção com o código, e o publicador a registra (EV1). Nada é salvo: a pergunta segue aberta, ou a tentativa segue pendente (H4). Não há retentativa (**Q2**) |

### Porta nova

Em `com.quebugesse.domain.shared.port` (M16), porque os eventos de qualquer agregado passam por ela:

| Porta | Operação | Observação |
|---|---|---|
| `PublicadorDeEventos` | `publicar(eventos: List<DomainEvent>)` | Nunca lança por causa de um ouvinte (EV1) |

Implementação: `PublicadorDeEventosEmMemoria`, em `infrastructure.messaging`. Recebe na construção o registro de falhas e aceita ouvintes como funções de `DomainEvent`, registrados a qualquer momento. Registrar depois de criar desfaz o ciclo de montagem: o `bootstrap` cria o publicador, entrega-o aos casos de uso, cria os ouvintes com esses casos de uso e só então os registra. Para os testes de `application`, um publicador espião que só guarda as chamadas (P6).

### Ouvintes

Em `com.quebugesse.application.ouvinte` (pacote novo). Cada um recebe todos os eventos e ignora os que não são do seu tipo.

| Ouvinte | Evento | Chama | Regras |
|---|---|---|---|
| `AoFazerPerguntaNoSolo` | `PerguntaFeita` | `ResponderComIA` | OS1, OS3 |
| `AoProporSolucaoNoSolo` | `SolucaoProposta` | `AvaliarComIA` | OS2, OS3 |

O `bootstrap` os registra nessa ordem. Ouvintes de tela (S06) e de estatísticas (S12) entram do mesmo jeito, sem mudar os casos de uso.

### Códigos de erro

`CodigoErro` ganha `FALHA_AO_SALVAR` e passa a ter 22 valores. Ele vale para todo caso de uso que chama `salvar`, inclusive `RegistrarJogador`, que não gera eventos. Só aparece depois de todas as verificações da S03, porque só um comando aceito chega a salvar.

### Emendas à S03

Entram como tarefas desta spec (ou da S03, se ela ainda não tiver sido implementada):

1. **A12 com releitura.** A visão devolvida por um caso de uso de comando é montada depois de publicar, a partir da partida relida do repositório (EV5), e não do objeto em memória.
2. **Publicação em todos os casos de uso de comando.** Os dez casos de uso que executam comando sobre a partida (todos menos `ConsultarPartida` e `RegistrarJogador`) recebem o `PublicadorDeEventos` e seguem H6. Os testes da S03 passam a usar o publicador espião; os CAs dela não mudam.

## Critérios de aceitação

Participantes de exemplo: narrador N e investigadores Ana e Beto, humanos; IA, o jogador IA. Enigmas do seed da S02, relógio fixo, gerador de IDs e sorteador determinísticos, como na S03. "Linha do tempo" é uma lista única em que o repositório espião, o publicador e os ouvintes anotam cada chamada que recebem.

| CA | Regra | Dado | Quando | Então |
|---|---|---|---|---|
| 01 | H6 | partida em grupo em andamento; repositório e ouvinte escrevendo na linha do tempo | Ana faz a pergunta "foi no deploy?" | linha do tempo: `salvar`, depois a entrega de `PerguntaFeita`; o ouvinte, ao receber o evento, busca a partida e encontra a pergunta 1 aberta |
| 02 | H6 | o cenário certo para cada comando | `IniciarPartida`, `FazerPergunta`, `ResponderPergunta`, `CancelarPergunta`, `ProporSolucao`, `RetirarProposta`, `JulgarProposta` (certa e errada) e `Abandonar` | uma chamada a `publicar` por comando, com, respectivamente: `[PartidaIniciada]`, `[PerguntaFeita]`, `[PerguntaRespondida]`, `[PerguntaCancelada]`, `[SolucaoProposta]`, `[PropostaRetirada]`, `[PartidaResolvida]` e `[SolucaoRecusada]`, `[PartidaAbandonada]`. `ConsultarPartida` e `RegistrarJogador` não publicam |
| 03 | H6 | repositório de partidas e de jogadores que lançam exceção em `salvar` | N inicia uma partida; Ana faz pergunta; `RegistrarJogador("Zé")` | `FALHA_AO_SALVAR` nos três; `publicar` não é chamado |
| 04 | H6, R3 | pergunta de Ana aberta | Beto faz pergunta; N abandona uma partida inexistente; Ana julga a proposta | `PERGUNTA_EM_ABERTO`, `PARTIDA_INEXISTENTE`, `NAO_E_NARRADOR`; `publicar` não é chamado em nenhum |
| 05 | EV1 | ouvintes O1, O2 e O3, nessa ordem; O2 lança exceção | Ana faz pergunta | O1 e O3 recebem `PerguntaFeita`; o registro de falhas tem uma entrada com o evento, O2 e a exceção; o caso de uso devolve a visão de Ana com a pergunta aberta |
| 06 | EV2 | ouvintes A, B e C, nessa ordem | `publicar([e1, e2])` | entregas, na ordem: A(e1), B(e1), C(e1), A(e2), B(e2), C(e2) |
| 07 | EV3 | ouvintes A e B; A, ao receber e1, publica `[e3]` e anota o retorno | `publicar([e1, e2])` | entregas: A(e1), B(e1), A(e2), B(e2), A(e3), B(e3); o `publicar` interno retorna antes de qualquer entrega de e3; a chamada externa só retorna depois de B(e3). Um ouvinte que republica em cadeia 10 000 vezes não estoura a pilha |
| 08 | EV4 | ouvinte que anota a thread em que roda | duas threads publicam ao mesmo tempo, cada uma 100 eventos próprios | cada evento é entregue na thread que o publicou; cada chamada externa retorna depois de entregar os seus 100 e nenhum da outra thread |
| 09 | OS1, EV5 | solo de Ana em andamento; ouvintes do solo registrados; `Narrador` fake responde `SIM` | Ana faz a pergunta "foi no deploy?" | publicados, na ordem: `[PerguntaFeita]` e depois `[PerguntaRespondida]`; a partida salva e a visão devolvida a Ana têm a pergunta 1 respondida `SIM` |
| 10 | OS1, OS2 | partida em grupo; ouvintes do solo registrados; `Narrador` e `VerificadorDeSolucao` fakes espiões | Ana pergunta; N responde; Ana propõe | os fakes não são chamados; a pergunta fica aberta até N responder e a tentativa fica pendente até N julgar |
| 11 | OS3, H4 | solo de Ana; `Narrador` fake que lança exceção | Ana faz pergunta | a visão devolvida tem a pergunta 1 aberta; publicado só `[PerguntaFeita]`; o registro de falhas tem uma entrada com `IA_INDISPONIVEL`; a partida salva tem a pergunta aberta |
| 12 | OS2, EV5 | solo de Ana sem pergunta aberta; `VerificadorDeSolucao` fake diz correta; em outra partida, incorreta | Ana propõe uma solução | publicados `[SolucaoProposta]` e `[PartidaResolvida]`, e a visão de Ana vem `RESOLVIDO`, com a solução; na outra, `[SolucaoProposta]` e `[SolucaoRecusada]`, tentativa `INCORRETA` e fase `AGUARDANDO_PERGUNTA` |

## Requisitos não funcionais

- **RNF1.** O publicador em memória não cria threads, não usa temporizador e não guarda eventos depois de entregá-los.
- **RNF2.** Cobertura de linhas de pelo menos 90% em `com.quebugesse.application.ouvinte` e em `com.quebugesse.infrastructure.messaging`.
- **RNF3.** O publicador aceita publicações e registros de ouvinte de várias threads (a S06 vai usá-lo com várias conexões).
- **RNF4.** As suítes de `application` e `infrastructure` continuam abaixo de 10 segundos cada.

## Fora de escopo

- Levar os eventos às telas por WebSocket ou SSE, e os DTOs de evento (S06).
- Outbox, entrega garantida e reenvio depois de uma queda (S09, ADR 0004).
- Retentativa com espera no adaptador de IA (S07, M3).
- Estatísticas atualizadas por evento (S12, M17).
- Despacho assíncrono, em outra thread ou num broker externo.
- Eventos de enigma e de jogador, que a S02 não gera.

## Divergências em relação aos diagramas

Pelo princípio P9, as fontes em [docs/diagramas/](../../docs/diagramas/README.md) recebem estes pontos no PR que implementar esta spec.

| ID | Diagrama hoje | Esta spec | Origem |
|---|---|---|---|
| D1 | Componentes: interface requerida "difusão de DomainEvent", sem nome nem pacote. Pacotes: `domain.shared` sem `port` | Porta `PublicadorDeEventos` em `domain.shared.port` | M16 |
| D2 | Seq. 2, passo 04: `PerguntaFeita` como mensagem assíncrona de `FazerPerguntaUseCase` para `ResponderPerguntaUseCase` | Publicação síncrona depois de `salvar`; no solo, o ouvinte `AoFazerPerguntaNoSolo` chama `ResponderComIA` | H6, ADR 0002 A |
| D3 | Estados: quadro "Regras da partida" com H1 a H5 | Acrescentar H6 ao quadro | M9 |
| D4 | Implantação: `WebSocket / SSE (messaging)` | Nesta spec, `infrastructure.messaging` tem só o publicador em memória. A entrega às telas por WebSocket fica na S06 | — |
| D5 | Seq. 2, nota: "Cada caso de uso carrega a partida do GameSessionRepository e a salva ao final" | Salva, publica os eventos e só então monta a resposta, relendo a partida | H6, EV5 |
| D6 | Pacotes: `application` só com `usecase` e `dto` | Pacote novo `application.ouvinte` | — |
| D7 | Seq. 1, passos 12 e 13: `salvar(partida)` e logo depois o DTO | `salvar`, `publicar([PartidaIniciada])`, releitura e só então a visão | H6, EV5 |

## Questões abertas

Todas têm decisão provisória para não travar a implementação. Se a equipe decidir diferente, mudam os CAs indicados.

| Q | Pergunta | Decisão provisória | CAs afetados |
|---|---|---|---|
| Q1 | O despacho é síncrono ou assíncrono? | Síncrono, na thread de quem publicou, com uma fila por thread. É determinístico nos testes e basta para um processo só | 07, 08, 09, 12 |
| Q2 | Um ouvinte que falha é chamado de novo? | Não: a falha é registrada e a entrega segue. A retentativa com espera é do adaptador de IA (S07). Se a IA falhar mesmo assim, o investigador cancela a pergunta (M3) e pergunta de novo, o que gera outro `PerguntaFeita` | 05, 11 |
| Q3 | A visão de retorno é a de antes ou a de depois dos ouvintes? | A de depois: o caso de uso relê a partida. Custa uma leitura a mais por comando e evita que o jogador solo precise consultar de novo para ver a resposta da IA | 09, 12 |
| Q4 | Quem reage a `PerguntaFeita` no grupo? ([ADR 0002](../../docs/adr/0002-contrato-da-porta-narrador.md)) | Opção A: ninguém nesta spec. O narrador humano responde por `ResponderPergunta`; a tela dele é avisada na S06 | 09, 10, 11 |
| Q5 | E se o processo cair entre `salvar` e `publicar`? | O evento se perde. Aceito enquanto tudo fica em memória, porque a queda perde a partida também. Na S09, com banco, entra o outbox ([ADR 0004](../../docs/adr/0004-banco-de-dados.md)) | 01 |

## Rastreabilidade

| CA | Arquivo de teste |
|---|---|
| 01 a 04 | `application/.../usecase/PublicacaoAposSalvarTest.kt` |
| 05 a 08 | `infrastructure/.../messaging/PublicadorDeEventosEmMemoriaTest.kt` |
| 09 a 12 | `application/.../ouvinte/OuvintesDoSoloTest.kt`, com o publicador em memória (Q4 da S03) |
