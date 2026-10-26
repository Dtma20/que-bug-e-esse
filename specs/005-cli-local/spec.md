# Spec 005: CLI local

| Campo | Valor |
|---|---|
| ID | S05 |
| Status | Rascunho |
| Depende de | S03 (casos de uso, DTOs e `CodigoErro`), S04 (`FALHA_AO_SALVAR` e o publicador que o `bootstrap` liga), S02 (repositórios em memória e seed) |
| Casos de uso | UC1 Escolher modo de jogo (só o grupo no mesmo dispositivo), UC2 Selecionar enigma, UC3 Iniciar partida, UC4 Fazer pergunta, UC5 Responder pergunta, UC6 Propor solução, UC11 Abandonar partida |
| Diagramas de origem | `Diagrama_imp.pdf` ("Terminal do Usuário / CLI" e a nota "Cliente SSH: jogar via terminal"), `diagrama_pacotes.pdf` (`interfaces`), `diagrama_casos.pdf` (UC1 "offline") |
| Decisões que assume | [ADR 0006](../../docs/adr/0006-modo-offline.md), opção A, em `Proposta`: "offline" é o grupo reunido num só dispositivo |
| Melhorias incorporadas | M14 (offline), M15 (`interfaces.cli` local, sem SSH) e M16 (`main` no `bootstrap`) do [planejamento](../../planejamento-sdd.md); fecha o marco M1 |
| Plano | Plano e tarefas: ainda não escritos |

## Contexto

O marco M1 é jogar uma partida inteira no terminal, sem rede nem IA, para validar o domínio e os casos de uso antes da S06. O diagrama de implantação já previa um terminal "útil para validar o fluxo manualmente durante o TDD", mas por SSH; aqui ele roda no mesmo processo, sem rede (M15).

Narrador e investigadores se revezam no mesmo teclado (ADR 0006 A). A CLI é uma camada de entrada como as outras: traduz linhas de texto em chamadas aos casos de uso e traduz visões e códigos de erro em texto. Ela não conhece o domínio.

## Histórias

- **US1.** Como grupo reunido num computador, quero abrir uma partida digitando os nomes e escolhendo categoria e dificuldade, para começar sem cadastro (UC1, UC2, UC3).
- **US2.** Como investigador, quero perguntar, propor, cancelar a minha pergunta e retirar a minha proposta com comandos curtos (UC4, UC6).
- **US3.** Como narrador, quero responder, julgar e consultar a solução, com um aviso antes de ela aparecer na tela que todos veem (UC5, P3).
- **US4.** Como jogador, quero mensagens em português que digam o que deu errado.
- **US5.** Como grupo, quero jogar outra partida sem digitar os nomes de novo e sem repetir enigma (M18).
- **US6.** Como desenvolvedor, quero testar a CLI com roteiros de entrada, sem um terminal de verdade.

## Glossário

| Termo | Significado |
|---|---|
| Abertura | Sequência de perguntas que monta uma partida nova: narrador, investigadores, categoria e dificuldade |
| Abertura curta | Abertura de "Jogar outra?": só categoria e dificuldade, com os mesmos participantes |
| Vez | Participante em nome de quem os comandos são executados. Aparece no prompt e só muda com `vez <nome>` (novo) |
| Normalizar | Aparar, passar para minúsculas e tirar acentos; espaços e hífens viram `_` (novo) |
| Sessão do programa | Do início do processo até `sair` ou o fim da entrada |
| Roteiro | Texto com uma linha por digitação, usado como entrada nos testes |
| Código da CLI | Erro que só a CLI conhece: `COMANDO_DESCONHECIDO`, `JOGADOR_DESCONHECIDO`, `ARGUMENTO_INVALIDO` (novo) |

## Regras

| ID | Regra |
|---|---|
| CL1 | `interfaces.cli` usa só casos de uso, DTOs e `CodigoErro` (F2 e Q1 da S00). O `main` fica no `bootstrap`, que monta repositórios em memória, seed (S02), publicador (S04), relógio do sistema, sorteador aleatório e gerador de IDs com UUID aleatório |
| CL2 | A abertura pergunta, nesta ordem: `Narrador: `, `Investigadores (separados por vírgula): `, `Categoria (Enter = qualquer): ` e `Dificuldade (Enter = qualquer): `. Na lista de investigadores, pedaços vazios depois de aparar são ignorados |
| CL3 | Categoria, dificuldade, nomes de comando e argumentos fixos (`sim`, `certa`, `s`) são comparados depois de normalizar. Categoria e dificuldade aceitam o nome do enum ou o nome de exibição (tabela abaixo). Valor fora da lista dá `ARGUMENTO_INVALIDO` |
| CL4 | Cada nome digitado, aparado, vira jogador por `RegistrarJogador`. Na sessão do programa, o mesmo nome normalizado é o mesmo jogador: "Ana" e "ana" têm o mesmo ID (**Q5**) |
| CL5 | A CLI faz todas as perguntas da abertura e só então valida, nesta ordem: nomes (narrador e depois investigadores), categoria, dificuldade e `IniciarPartida`. O primeiro erro mostra a mensagem e recomeça a abertura do primeiro prompt; na abertura curta, recomeça a abertura curta |
| CL6 | A tela inicial mostra título, categoria, dificuldade, enunciado, narrador e investigadores. Nunca mostra a solução, embora a visão do narrador a traga (P3) |
| CL7 | O prompt é `[Nome]> `, com o nome como foi digitado na primeira vez. A vez começa com o primeiro investigador e só muda com `vez <nome>` (**Q4**) |
| CL8 | Os comandos são os da tabela abaixo, reconhecidos pela primeira palavra normalizada. O resto da linha é o argumento; texto livre vai ao caso de uso sem normalizar |
| CL9 | Linha vazia ou só com espaços é ignorada e o prompt volta. O fim da entrada (EOF) encerra o programa com código 0, em qualquer ponto. Toda a saída vai para stdout, em UTF-8, e a entrada é lida em UTF-8 |
| CL10 | Todo `CodigoErro` (22) e todo código da CLI (3) tem uma mensagem em português, conforme a tabela "Mensagens de erro" |
| CL11 | Quando a partida chega a `RESOLVIDO` ou `ABANDONADO`, a CLI mostra o desfecho e a solução e pergunta `Jogar outra? (s/n) `. `s` ou `sim` abre a abertura curta; `n` ou `nao` encerra com código 0; outra resposta repete a pergunta |
| CL12 | Só o modo `GRUPO`. A CLI nunca chama `ResponderComIA` nem `AvaliarComIA` (**Q2**) |

### Comandos

| Comando | Caso de uso | Observação |
|---|---|---|
| `vez <nome>` | nenhum | Passa a vez ao participante com esse nome normalizado. Sem nome: `ARGUMENTO_INVALIDO`; nome que não participa: `JOGADOR_DESCONHECIDO` |
| `perguntar <texto>` | `FazerPergunta` | |
| `responder sim\|nao\|irrelevante` | `ResponderPergunta` | Responde à pergunta aberta, que a CLI tira da visão mais recente. Sem pergunta aberta: `SEM_PERGUNTA_ABERTA`, sem chamar o caso de uso |
| `cancelar` | `CancelarPergunta` | Cancela a pergunta aberta; sem ela, `SEM_PERGUNTA_ABERTA` |
| `propor <texto>` | `ProporSolucao` | |
| `retirar` | `RetirarProposta` | |
| `julgar certa\|errada` | `JulgarProposta` | |
| `solucao` | `ConsultarPartida` | Só o narrador (senão `NAO_E_NARRADOR`). Avisa que a solução vai aparecer na tela e pede `Mostrar? (s/n) `; só `s` ou `sim` mostra (**Q3**) |
| `ver` | `ConsultarPartida` | Título, enunciado, fase, perguntas (número, autor, texto e resposta, "aberta" ou "cancelada") e tentativas (número, autor, texto e situação). Nunca mostra a solução antes do fim, nem para o narrador |
| `abandonar` | `Abandonar` | Pede `Abandonar a partida? (s/n) `; só `s` ou `sim` chama o caso de uso |
| `ajuda` | nenhum | Lista os doze comandos, um por linha, com uma frase cada |
| `sair` | nenhum | Encerra com código 0, sem confirmação. A partida em memória se perde (**Q6**) |

Os argumentos `sim`, `nao`, `irrelevante`, `certa` e `errada` passam pela normalização: `Não` vale `nao`. Outro valor dá `ARGUMENTO_INVALIDO`, e essa verificação vem antes da procura pela pergunta aberta.

### Saída dos comandos aceitos

| Comando | Linha mostrada |
|---|---|
| `perguntar` | `Pergunta <n> de <autor>: <texto>` |
| `responder` | `Resposta à pergunta <n>: Sim`, `Não` ou `Irrelevante` |
| `cancelar` | `Pergunta <n> cancelada.` |
| `propor` | `Proposta <n> de <autor>: <texto>. O narrador vai julgar.` |
| `retirar` | `Proposta <n> retirada.` |
| `julgar errada` | `Proposta <n> está errada. A investigação continua.` |
| `julgar certa` | `Resolvido! Proposta <n> de <autor> está certa.`, seguida do desfecho (CL11) |
| `abandonar` confirmado | `Partida abandonada por <nome>.`, seguida do desfecho (CL11) |

O desfecho é `Solução: <texto>` numa linha e, na seguinte, `Jogar outra? (s/n) `.

### Nomes de exibição

| Enum | Exibição |
|---|---|
| `SINTAXE`, `DEBUG`, `GESTAO_PROJETO`, `DESIGN`, `DISCOVERY` | Sintaxe, Debug, Gestão de projeto, Design, Discovery |
| `FACIL`, `MEDIA`, `DIFICIL` | Fácil, Média, Difícil |

### Mensagens de erro

| Código | Mensagem |
|---|---|
| `PARTIDA_ENCERRADA` | A partida já terminou. |
| `PARTIDA_NAO_INICIADA` | A partida ainda não começou. |
| `PARTIDA_JA_INICIADA` | A partida já começou. |
| `CONFIGURACAO_INVALIDA` | Participantes inválidos: é preciso ao menos um investigador, o narrador não pode investigar e ninguém entra duas vezes. |
| `NAO_E_INVESTIGADOR` | Só os investigadores podem fazer isso. |
| `NAO_E_NARRADOR` | Só o narrador pode fazer isso. |
| `NAO_AUTORIZADO` | Você não pode fazer isso: só o autor ou o narrador cancela e retira, e no grupo só o narrador abandona. |
| `TEXTO_INVALIDO` | O texto precisa ter de 1 a 500 caracteres. |
| `PERGUNTA_INEXISTENTE` | Essa pergunta não existe. |
| `PERGUNTA_JA_FECHADA` | Essa pergunta já foi respondida ou cancelada. |
| `PERGUNTA_EM_ABERTO` | Há uma pergunta esperando a resposta do narrador. |
| `PROPOSTA_PENDENTE` | Há uma proposta esperando o julgamento do narrador. |
| `SEM_PROPOSTA_PENDENTE` | Não há proposta esperando julgamento. |
| `NOME_INVALIDO` | O nome precisa ter de 1 a 40 caracteres. |
| `PARTIDA_INEXISTENTE` | Partida não encontrada. |
| `JOGADOR_INEXISTENTE` | Jogador não encontrado. |
| `SEM_ENIGMA_ELEGIVEL` | Não há enigma dessa categoria e dificuldade que este grupo ainda não tenha jogado. |
| `MODO_INCOMPATIVEL` | Isso não vale neste modo de jogo. |
| `SEM_PERGUNTA_ABERTA` | Não há pergunta esperando resposta. |
| `IA_INDISPONIVEL` | A IA não respondeu. Tente de novo. |
| `NAO_PARTICIPA` | Você não participa desta partida. |
| `FALHA_AO_SALVAR` | Não foi possível salvar a jogada. Nada mudou; tente de novo. |
| `COMANDO_DESCONHECIDO` | Comando desconhecido. Digite ajuda para ver os comandos. |
| `JOGADOR_DESCONHECIDO` | Não há participante com esse nome nesta partida. |
| `ARGUMENTO_INVALIDO` | Valor inválido. Opções: `<lista do campo>`. |

## Critérios de aceitação

Cada CA roda um roteiro na CLI montada com os repositórios em memória, o seed da S02, relógio fixo, IDs sequenciais e um sorteador que escolhe o primeiro elegível, e compara a saída. "Partida aberta" é a abertura com narrador `N`, investigadores `Ana, Beto` e categoria `debug`, que escolhe e1.

| CA | Regra | Dado | Quando | Então |
|---|---|---|---|---|
| 01 | CL2, CL6, CL7 | seed carregado | roteiro `N`, `Ana, Beto`, Enter, Enter | os quatro prompts, na ordem; título, categoria, dificuldade, enunciado e participantes do enigma sorteado; prompt `[Ana]> `; o texto da solução não aparece |
| 02 | CL3, CL9 | seed carregado | categoria `debug`, dificuldade `MÉDIA` | partida com e1; a saída tem "Férias fatais" e "Média" com acento |
| 03 | CL3 | seed carregado | categoria ` Gestão de projeto `, dificuldade `facil`; depois, noutro roteiro, `gestao-projeto` e `FÁCIL` | e3 nos dois |
| 04 | CL3, CL5 | seed carregado | categoria `redes`, depois uma abertura válida | mensagem de `ARGUMENTO_INVALIDO` com as cinco categorias; a abertura recomeça em `Narrador: ` e a segunda tentativa abre a partida |
| 05 | CL4, CL5 | seed carregado | narrador `Ana`, investigadores `ana, Beto`; depois narrador Enter | mensagem de `CONFIGURACAO_INVALIDA`; depois a de `NOME_INVALIDO`; a abertura recomeça nas duas vezes |
| 06 | CL2, CL4 | seed carregado | investigadores `Ana, , Beto,` | investigadores Ana e Beto, nessa ordem |
| 07 | CL7, CL8 | partida aberta | `perguntar foi no deploy?`, `vez n`, `responder nao`, `vez BETO`, `perguntar foi no fim de semana?` | `Pergunta 1 de Ana: foi no deploy?`, `Resposta à pergunta 1: Não`, `Pergunta 2 de Beto: ...`; prompts `[Ana]> `, `[N]> ` e `[Beto]> `, com o nome como foi digitado na abertura |
| 08 | CL8, CL11 | partida aberta | `propor foi o cache`, `vez N`, `julgar errada`, `vez Ana`, `propor o notebook do Rafael era um servidor`, `vez N`, `julgar certa` | a primeira proposta errada e a investigação segue; depois `Resolvido!`, a linha `Solução:` com a solução de e1 e `Jogar outra? (s/n) ` |
| 09 | CL8 | partida aberta | `cancelar`; `perguntar foi no deploy?`, `cancelar`; `retirar` | `SEM_PERGUNTA_ABERTA`; `Pergunta 1 cancelada.`; mensagem de `SEM_PROPOSTA_PENDENTE` |
| 10 | CL8, P3 | partida aberta, vez de Ana | `solucao`; `vez N`, `solucao`, `n`; `ver`; `solucao`, `s` | `NAO_E_NARRADOR`; aviso, sem a solução; `ver` sem a solução; aviso e a solução. Em nenhum `ver` antes do fim, de qualquer participante, aparece o texto da solução |
| 11 | CL8, H2 | partida aberta, vez de Ana | `abandonar`, `s`; `vez N`, `abandonar`, `n`; `abandonar`, `s` | `NAO_AUTORIZADO`; nada muda; `Partida abandonada por N.`, a solução e `Jogar outra? (s/n) ` |
| 12 | CL11, CL5 | partida aberta e resolvida como no CA-08 | `talvez`; `s`, categoria `debug`, Enter; depois Enter, Enter | `Jogar outra?` repetida; mensagem de `SEM_ENIGMA_ELEGIVEL` e a abertura curta recomeça em `Categoria`; por fim, partida com outro enigma, os mesmos participantes e prompt `[Ana]> ` |
| 13 | CL9 | — | linha vazia e linha com espaços durante a partida; EOF na abertura; EOF na partida; EOF em `Jogar outra?`; `sair` | as linhas vazias só repetem o prompt; o programa termina com código 0 nos quatro casos e não escreve em stderr |
| 14 | CL10 | os 22 valores de `CodigoErro` e os 3 códigos da CLI | buscar a mensagem de cada um | todos têm mensagem não vazia e diferente das outras; um valor novo sem mensagem faz o teste falhar |
| 15 | CL3, CL8 | partida aberta | `xyz`, `vez Zé`, `vez`, `responder talvez`, `PERGUNTAR é rede?`, `ajuda` | `COMANDO_DESCONHECIDO`; `JOGADOR_DESCONHECIDO`; `ARGUMENTO_INVALIDO`; `ARGUMENTO_INVALIDO` com as três respostas; `Pergunta 1 de Ana: é rede?`; os doze comandos |

## Requisitos não funcionais

- **RNF1.** `interfaces.cli` não importa `com.quebugesse.domain` nem `com.quebugesse.infrastructure`; o teste de arquitetura da S00 cobre.
- **RNF2.** Cobertura de linhas de pelo menos 90% em `com.quebugesse.interfaces.cli`.
- **RNF3.** Acentos corretos no Windows Terminal, no PowerShell e num terminal Linux, sem configurar nada além do JDK.
- **RNF4.** A CLI recebe entrada, saída e casos de uso pelo construtor; os testes não abrem terminal nem processo.

## Fora de escopo

- Modo solo na CLI (depois da S07).
- SSH ou qualquer acesso remoto ao terminal (M15).
- Guardar partidas entre execuções (S09).
- Sala, código de convite e autenticação (S06, ADR 0007).
- Cores, histórico de comandos, autocompletar e cronômetro.

## Divergências em relação aos diagramas

Pelo princípio P9, as fontes em [docs/diagramas/](../../docs/diagramas/README.md) recebem estes pontos no PR que implementar esta spec.

| ID | Diagrama hoje | Esta spec | Origem |
|---|---|---|---|
| D1 | Implantação: "Terminal do Usuário / CLI" ligado por SSH ao "Servidor Web / Proxy"; nota "Cliente SSH: jogar via terminal" | CLI local, no mesmo processo dos casos de uso, sem rede nem SSH | M15 |
| D2 | Pacotes: `interfaces` só com `rest` e `ws` | Pacote `interfaces.cli` | M15 |
| D3 | Casos de uso: UC1 "(solo / grupo online / offline)" | "offline" passa a "mesmo dispositivo" | ADR 0006 A, M14 |
| D4 | Pacotes e componentes sem ponto de entrada | `main` no `bootstrap`, que monta os adaptadores e a CLI | M16 |

## Questões abertas

Todas têm decisão provisória para não travar a implementação. Se a equipe decidir diferente, mudam os CAs indicados.

| Q | Pergunta | Decisão provisória | CAs afetados |
|---|---|---|---|
| Q1 | O que é o modo "offline"? ([ADR 0006](../../docs/adr/0006-modo-offline.md)) | Opção A: o grupo num só dispositivo, que é esta CLI | todos |
| Q2 | A CLI joga o solo? | Não até a S07: sem adaptador de IA real, o fake não serve para jogar | — |
| Q3 | Como o narrador consulta a solução numa tela compartilhada? | Só pelo comando `solucao`, com aviso e confirmação; `ver` nunca a mostra antes do fim | 10 |
| Q4 | Como se decide quem está digitando? | Vez explícita, com `vez <nome>`. Rodízio automático erraria sempre que o narrador precisa responder | 07 |
| Q5 | O mesmo nome em duas partidas é o mesmo jogador? | Sim, dentro da sessão do programa, comparando o nome normalizado. Assim "Jogar outra?" respeita os já jogados (M18) | 05, 12 |
| Q6 | Sair no meio da partida a registra como abandonada? | Não. Nada é guardado depois que o processo termina, então não há o que registrar | 13 |

## Rastreabilidade

| CA | Arquivo de teste |
|---|---|
| 01 a 13, 15 | `bootstrap/src/test/kotlin/com/quebugesse/cli/RoteirosTest.kt`: o `bootstrap` é o único módulo que pode montar a CLI com os adaptadores em memória |
| 14 | `interfaces/.../cli/MensagensTest.kt` |
