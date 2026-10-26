# Spec 002: Catálogo de enigmas e jogador

| Campo | Valor |
|---|---|
| ID | S02 |
| Status | Rascunho |
| Depende de | S00 (módulos `domain` e `infrastructure` com Kotest). Usa `EnigmaId`, `PlayerId` e `PlayerRef` de `domain.shared`, criados na tarefa 1 da S01; se a S02 vier antes, ela cria esses tipos com as assinaturas do plano da S01 |
| Casos de uso | UC2 Selecionar enigma (parte de domínio), UC9 Cadastrar/revisar enigma (só o ciclo de vida; telas e caso de uso na S11) |
| Diagramas de origem | `diagrama_classes.pdf` (pacotes `domain.enigma`, `domain.player` e `domain.shared`), `diagrama_seq_iniciar.pdf` (passos 03 e 04), `diagrama_pacotes.pdf` (`infrastructure.persistence`) |
| Melhorias incorporadas | M16 (lugar das portas), M17 (só registra que as estatísticas mudam na S12), M18, M19, M21 do [planejamento](../../planejamento-sdd.md) |
| Plano | [plan.md](plan.md) · Tarefas: [tasks.md](tasks.md) |

## Contexto

A partida (S01) referencia um enigma e jogadores só por ID. Esta spec cria esses dois agregados, as portas de repositório, a versão em memória delas e um catálogo inicial (seed) para o terminal da S05 ter o que jogar. O enigma ganha um ciclo de vida de curadoria (M19), e a escolha do enigma ganha um critério (M18). O sorteio e o caso de uso de iniciar partida ficam na S03.

## Histórias

- **US1.** Como curador, quero cadastrar um enigma como rascunho e revisá-lo antes de publicar, para que só enigma revisado chegue aos jogadores (UC9, M19).
- **US2.** Como curador, quero arquivar um enigma, para tirá-lo de circulação sem apagá-lo das partidas antigas.
- **US3.** Como quem inicia a partida, quero filtrar por categoria e dificuldade e não receber enigma que o grupo já jogou (UC2, M18).
- **US4.** Como jogador, quero ter nome, tipo (humano ou IA) e estatísticas que começam zeradas.
- **US5.** Como desenvolvedor, quero repositórios em memória que passem na mesma suíte de contrato que o banco vai rodar na S09, para testar casos de uso sem banco (P6).
- **US6.** Como equipe, quero um catálogo inicial com um enigma por categoria e o jogador IA, para jogar no terminal (S05) sem cadastrar nada.

## Glossário

| Termo | Significado |
|---|---|
| Enigma (`Enigma`) | Agregado raiz do catálogo |
| Conteúdo (`ConteudoEnigma`) | Valor com título, enunciado, solução completa, categoria, dificuldade e tags. Revisar um enigma troca o conteúdo inteiro (novo) |
| Origem (`OrigemEnigma`) | `MANUAL` (cadastrado por pessoa) ou `IA` (gerado pela S11) (novo, M19) |
| Status do enigma (`StatusEnigma`) | `RASCUNHO`, `PUBLICADO` ou `ARQUIVADO` (novo, M19) |
| Elegível | Enigma que pode ser sorteado para uma partida nova |
| Critério de seleção (`CriterioDeSelecao`) | Categoria e dificuldade desejadas; vazio quer dizer "qualquer" (novo, M18) |
| Já jogados | Conjunto de `EnigmaId` que a aplicação calcula para o grupo e passa à seleção |
| Jogador (`Player`) | Agregado raiz do jogador, humano ou IA |
| Estatísticas (`EstatisticasJogador`) | Partidas jogadas e resolvidas |
| Suíte de contrato | Testes escritos uma vez contra a porta e executados em cada implementação (memória agora, banco na S09) |
| Seed | Carga inicial do catálogo e do jogador IA |

## Regras

| ID | Regra |
|---|---|
| E1 | Título, enunciado e solução: espaços nas pontas são removidos; o resultado tem de 1 a 100 (título), 1 a 2000 (enunciado) e 1 a 2000 (solução) code points (**Q1**) |
| E2 | Tags: espaços nas pontas removidos, letras minúsculas, vazias descartadas, sem repetição (**Q5**) |
| E3 | Enigma nasce em `RASCUNHO` |
| E4 | O conteúdo só muda em `RASCUNHO` |
| E5 | Publicar só a partir de `RASCUNHO` |
| E6 | Arquivar a partir de `RASCUNHO` ou `PUBLICADO`; `ARQUIVADO` é terminal |
| E7 | Comando rejeitado não altera o enigma (como R3 da S01) |
| E8 | O texto de `toString` do conteúdo e do enigma não contém a solução (P3) |
| EL1 | Elegível = `PUBLICADO`, fora de "já jogados" e com a categoria e a dificuldade do critério, quando informadas. Vale também no modo solo (**Q2**) |
| EL2 | A seleção devolve os elegíveis na ordem em que os recebeu e não sorteia; o sorteio fica na aplicação (P1) |
| J1 | Nome do jogador: espaços nas pontas removidos; de 1 a 40 code points (**Q4**) |
| J2 | Estatísticas: `0 ≤ partidasResolvidas ≤ partidasJogadas`. Jogador novo começa com zero nas duas |
| J3 | `Player` não tem comando nesta spec. As estatísticas só mudam na S12 (M17) |
| RP1 | `buscar` de um ID que não existe devolve nulo |
| RP2 | `salvar` insere ou, se o ID já existe, substitui |
| RP3 | O repositório guarda uma cópia: mudar um objeto depois de salvá-lo, ou mudar o objeto que `buscar` devolveu, não muda o que está guardado até o próximo `salvar` |
| RP4 | `publicados()` devolve só enigmas `PUBLICADO`, sem ordem garantida |
| SD1 | O seed grava o jogador IA, com ID fixo `00000000-0000-0000-0000-0000000000a1`, e cinco enigmas `PUBLICADO` e `MANUAL`, um por categoria (**Q6**) |
| SD2 | O seed é idempotente: só grava o que ainda não existe. Rodar de novo não duplica nem desfaz mudanças feitas depois (um enigma do seed arquivado continua arquivado) |
| SD3 | Os textos do seed são os de [docs/enigmas/seed.md](../../docs/enigmas/seed.md), que a equipe revisa |

## Comportamento esperado

Status do enigma × comando. "→" indica o novo status; um nome em maiúsculas sozinho indica a violação que rejeita o comando.

| | `revisar` | `publicar` | `arquivar` |
|---|---|---|---|
| `RASCUNHO` | → `RASCUNHO`, com o conteúdo novo | → `PUBLICADO` | → `ARQUIVADO` |
| `PUBLICADO` | `ENIGMA_NAO_EDITAVEL` | `TRANSICAO_INVALIDA` | → `ARQUIVADO` |
| `ARQUIVADO` | `ENIGMA_NAO_EDITAVEL` | `TRANSICAO_INVALIDA` | `TRANSICAO_INVALIDA` |

Violações do conteúdo, na ordem em que são verificadas: `TITULO_INVALIDO`, `ENUNCIADO_INVALIDO`, `SOLUCAO_INVALIDA`. Violações do jogador: `NOME_INVALIDO`, `ESTATISTICAS_INVALIDAS`.

Enigma e jogador não geram eventos de domínio nesta spec.

## Critérios de aceitação

Em todo CA de rejeição do enigma vale E7, verificado pela comparação do enigma antes e depois (CA-15). Exemplos: enigmas E1 a E3, jogadores Ana (humana) e o jogador IA.

| CA | Regra | Dado | Quando | Então |
|---|---|---|---|---|
| 01 | E1, E2 | título `"  Férias fatais "`, enunciado e solução com espaços nas pontas, tags `[" Rede ", "", "rede", "PRODUCAO"]` | `ConteudoEnigma.de(...)` | textos aparados; tags `{"rede", "producao"}` |
| 02 | E1 | — | título `""`, `"   "` ou 101 caracteres | `TITULO_INVALIDO`; 100 caracteres, ou 100 emojis, são aceitos |
| 03 | E1 | — | enunciado `""`, `"   "` ou 2001 caracteres | `ENUNCIADO_INVALIDO`; 2000 são aceitos |
| 04 | E1 | — | solução `""`, `"   "` ou 2001 caracteres | `SOLUCAO_INVALIDA`; 2000 são aceitos |
| 05 | E2 | lista mutável de tags | `de(...)` e depois a lista ganha `"nova"` | o conteúdo não muda |
| 06 | E8 | conteúdo com solução `"o notebook hospedava o endpoint"` | `toString()` do conteúdo e do enigma | nenhum dos dois contém o texto da solução |
| 07 | E1 | título e enunciado inválidos ao mesmo tempo | `de(...)` | `TITULO_INVALIDO` |
| 08 | E3 | — | `Enigma.criar(E1, conteudo, IA)` | status `RASCUNHO`, origem `IA`, o mesmo conteúdo |
| 09 | E4 | enigma em `RASCUNHO` | `revisar(novo)` | conteúdo novo; segue `RASCUNHO` |
| 10 | E4 | enigma `PUBLICADO` e enigma `ARQUIVADO` | `revisar(novo)` | `ENIGMA_NAO_EDITAVEL` |
| 11 | E5 | enigma em `RASCUNHO` | `publicar()` | `PUBLICADO` |
| 12 | E5, E6 | enigma `PUBLICADO` e enigma `ARQUIVADO` | `publicar()` | `TRANSICAO_INVALIDA` |
| 13 | E6 | enigma em `RASCUNHO` e enigma `PUBLICADO` | `arquivar()` | `ARQUIVADO` |
| 14 | E6 | enigma `ARQUIVADO` | `arquivar()` | `TRANSICAO_INVALIDA` |
| 15 | E7 | qualquer CA de rejeição do enigma | — | enigma idêntico antes e depois |
| 16 | — | — | `Enigma.reconstituir(E1, conteudo, MANUAL, PUBLICADO)` | mesmos ID, conteúdo, origem e status; `arquivar()` é aceito e `publicar()` dá `TRANSICAO_INVALIDA` |
| 17 | EL1 | candidatos em `RASCUNHO`, `PUBLICADO` e `ARQUIVADO` | `CriterioDeSelecao().elegiveis(candidatos, emptySet())` | só o `PUBLICADO` |
| 18 | EL1 | publicados de várias categorias e dificuldades | critério só com `DEBUG`; só com `FACIL`; com os dois; sem nenhum | só `DEBUG`; só `FACIL`; os dois ao mesmo tempo; todos |
| 19 | EL1 | E1 e E2 publicados | `elegiveis(..., jaJogados = {E1})` | só E2 |
| 20 | EL2 | E3, E1 e E2 publicados, nessa ordem | critério vazio; critério sem nenhum candidato | `[E3, E1, E2]`; lista vazia |
| 21 | J1, J2 | — | `Player.criar(id, "  Ana  ", HUMANO)` | nome `"Ana"`, tipo `HUMANO`, estatísticas `(0, 0)` |
| 22 | J1 | — | nome `""`, `"   "` ou 41 caracteres | `NOME_INVALIDO`; 40 caracteres, ou 40 emojis, são aceitos |
| 23 | J2 | — | `EstatisticasJogador(-1, 0)`, `(0, -1)` e `(1, 2)` | `ESTATISTICAS_INVALIDAS`; `(2, 1)` e `(0, 0)` são aceitas |
| 24 | J1, J2 | — | `Player.reconstituir(id, "Ana", HUMANO, EstatisticasJogador(5, 2))`; e com nome `""` | mesmos campos; `NOME_INVALIDO` |
| 25 | RP1 | repositório vazio | `buscar` de enigma e de jogador | nulo |
| 26 | RP2 | — | salvar E1, buscar; salvar E1 revisado, buscar | os campos do salvo; depois, o conteúdo revisado |
| 27 | RP3 | E1 salvo em `RASCUNHO` | publicar o objeto salvo sem salvar de novo; publicar o objeto devolvido por `buscar` sem salvar | nos dois casos, `buscar` devolve E1 em `RASCUNHO` |
| 28 | RP4 | E1 em `RASCUNHO`, E2 `PUBLICADO` e E3 `ARQUIVADO` salvos | `publicados()` | só E2 |
| 29 | RP1, RP2 | — | salvar Ana, buscar; salvar outro jogador com o mesmo ID, buscar | Ana; depois, o outro |
| 30 | SD1 | repositórios vazios | `SeedInicial.carregar` | jogador IA com o ID fixo, tipo `IA`; cinco enigmas `PUBLICADO` e `MANUAL`, um de cada categoria, com as dificuldades de [seed.md](../../docs/enigmas/seed.md) |
| 31 | SD2 | seed carregado e um enigma dele arquivado | `carregar` de novo | um jogador IA; cinco enigmas; o arquivado segue `ARQUIVADO` |
| 32 | SD3 | o seed e o arquivo `docs/enigmas/seed.md` | comparar | título, enunciado e solução de cada enigma aparecem literalmente no arquivo |

## Requisitos não funcionais

- **RNF1.** `domain.enigma` e `domain.player` puros conforme P1 da [constituição](../../docs/constitution.md).
- **RNF2.** Cobertura de linhas de pelo menos 90% em `com.quebugesse.domain.enigma`, `com.quebugesse.domain.player` e `com.quebugesse.infrastructure.persistence`.
- **RNF3.** As suítes de `domain` e `infrastructure` rodam, cada uma, em menos de 10 segundos numa máquina de desenvolvimento.
- **RNF4.** Os repositórios em memória aceitam chamadas de várias threads (a S06 vai usá-los com várias conexões antes da S09).

## Fora de escopo

- Casos de uso e telas de cadastro e revisão de enigma (S11).
- Geração de enigma por IA (S11, porta `GeradorDeEnigmas`).
- Sorteio do enigma, cálculo de "já jogados" e o caso de uso de iniciar partida (S03).
- Atualização das estatísticas do jogador (S12, M17).
- Persistência em banco (S09) e bloqueio otimista (M8).
- Identidade e autenticação do jogador (S06, [ADR 0007](../../docs/adr/0007-identidade-e-autenticacao.md)).
- Eventos de domínio de enigma e de jogador; busca por texto ou por tag.

## Divergências em relação aos diagramas

Pelo princípio P9, `diagrama_classes`, `diagrama_seq_iniciar` e `diagrama_pacotes` precisam ser atualizados com estes pontos (tarefa 7 de [tasks.md](tasks.md)).

| ID | Diagrama hoje | Esta spec | Origem |
|---|---|---|---|
| D1 | `Enigma` sem status nem origem | `status: StatusEnigma` e `origem: OrigemEnigma` | M19 |
| D2 | Título, enunciado, solução, categoria, dificuldade e tags soltos em `Enigma` | Agrupados no valor `ConteudoEnigma`, trocado inteiro por `revisar` | — |
| D3 | `~ solucaoCompleta` (interno ao pacote) | `val` pública em `ConteudoEnigma`. Kotlin não tem visibilidade de pacote; o sigilo vem de P3 na aplicação e de `toString` sem a solução | M21, P3 |
| D4 | Portas sem pacote definido | `domain.enigma.port` e `domain.player.port` | M16 |
| D5 | Seq. 1, passo 03: "selecionar enigma" no `EnigmaRepository` | O repositório só devolve `publicados()`; o filtro é `CriterioDeSelecao`, no domínio, e o sorteio fica na aplicação | M18, P1 |
| D6 | `Player.stats` sem método que a altere | `Player` sem comando nesta spec; mudanças só na S12 | M17 |
| D7 | Seleção sem critério | `CriterioDeSelecao(categoria?, dificuldade?)` e o conjunto `jaJogados` | M18 |
| D8 | `EstatisticasJogador` sem restrição | `0 ≤ partidasResolvidas ≤ partidasJogadas` | — |
| D9 | `infrastructure.persistence` sem subpacotes | `persistence.memoria` (repositórios em memória) e `persistence.seed` | — |

## Questões abertas

Todas têm decisão provisória para não travar a implementação. Se a equipe decidir diferente, mudam os CAs indicados.

| Q | Pergunta | Decisão provisória | CAs afetados |
|---|---|---|---|
| Q1 | Limites de texto do enigma | Título de 1 a 100, enunciado de 1 a 2000 e solução de 1 a 2000 code points, depois de aparar | 02, 03, 04 |
| Q2 | O solo pode jogar enigma não revisado (M19)? | Não. Só `PUBLICADO` é elegível, em qualquer modo; enigma gerado por IA passa pela curadoria da S11 | 17, 28 |
| Q3 | O que é "já jogado pelo mesmo grupo" (M18)? | O domínio só recebe o conjunto pronto. A S03 decide como calculá-lo | 19 |
| Q4 | Limite do nome do jogador | De 1 a 40 code points depois de aparar; nomes repetidos são permitidos, porque a identidade é o ID | 21, 22 |
| Q5 | Forma das tags | Aparadas, minúsculas, vazias descartadas; sem limite de quantidade nem de tamanho | 01, 05 |
| Q6 | Conteúdo do seed | Os cinco enigmas e o jogador IA "Narrador IA" de [seed.md](../../docs/enigmas/seed.md), à espera de revisão da equipe | 30, 31, 32 |

## Rastreabilidade

| CA | Arquivo de teste |
|---|---|
| 01 a 05, 06 (conteúdo), 07 | `domain/.../enigma/ConteudoEnigmaTest.kt` |
| 06 (enigma), 08 a 14, 16 | `domain/.../enigma/CicloDeVidaEnigmaTest.kt` |
| 17 a 20 | `domain/.../enigma/CriterioDeSelecaoTest.kt` |
| 21 a 24 | `domain/.../player/PlayerTest.kt` |
| 25 a 28 | `infrastructure/src/testFixtures/.../contrato/EnigmaRepositoryContrato.kt`, executado por `EnigmaRepositoryEmMemoriaTest.kt` |
| 25, 29 | `infrastructure/src/testFixtures/.../contrato/PlayerRepositoryContrato.kt`, executado por `PlayerRepositoryEmMemoriaTest.kt` |
| 30 a 32 | `infrastructure/.../seed/SeedInicialTest.kt` |
| 15 | todos os testes de rejeição do enigma (via `deveRejeitar`) |
