# Spec 006: API Ktor, salas e identidade de convidado

| Campo | Valor |
|---|---|
| ID | S06 |
| Status | Aprovada |
| Depende de | S03 (casos de uso, DTOs, `CodigoErro`, `VisaoInvestigador`/`VisaoNarrador`), S04 (`PublicadorDeEventos`, H6), S02 (`EnigmaRepository`, `PlayerRepository`, seed) |
| Casos de uso | UC1 Escolher modo (solo / grupo online), UC2 Selecionar enigma, UC3 Iniciar partida, UC4 Fazer pergunta, UC5 Responder pergunta, UC6 Propor solução, UC7 Consultar histórico (parcial: listagem do jogador), UC11 Abandonar partida, UC12 Entrar em partida (novo, M11) |
| Diagramas de origem | [componentes.md](../../docs/diagramas/componentes.md), [implantacao.md](../../docs/diagramas/implantacao.md), [pacotes.md](../../docs/diagramas/pacotes.md), [casos-de-uso.md](../../docs/diagramas/casos-de-uso.md) |
| Decisões que assume | [ADR 0003](../../docs/adr/0003-framework-web.md) opção A (Ktor REST + WebSocket), [ADR 0007](../../docs/adr/0007-identidade-e-autenticacao.md) opção A (convidado com apelido, código de sala e token assinado por sessão), [ADR 0008](../../docs/adr/0008-stack-do-front-end.md) opção A e [ADR 0009](../../docs/adr/0009-divisao-de-repositorios.md) opção A (`Que-Bug-E-Esse/front` consome exclusivamente `Que-Bug-E-Esse/back`), todas `Aceita` |
| Melhorias incorporadas | M1/P3 (sigilo JSON/WS por papel), M6 (identidade), M11 (sala de espera com código de convite), M23 (acesso único via back) do [planejamento](../../planejamento-sdd.md) |
| Plano | Plano e tarefas: ainda não escritos |

## Contexto

O marco M1 (S00–S05) validou todo o núcleo do jogo localmente via terminal. Para jogar em rede no navegador (`Que-Bug-E-Esse/front`) e suportar partidas em grupo online com múltiplos dispositivos, falta expor a camada de entrada HTTP/WebSocket, a identidade leve de convidado e o fluxo de sala de espera com código de convite.

Esta spec define o contrato da API REST e WebSocket em Ktor (`interfaces.http` / `interfaces.ws`), a emissão e verificação de tokens de sessão assinados com HMAC-SHA256, o gerenciamento de salas em `AGUARDANDO_INICIO` (UC12), os endpoints de catálogo, histórico e perfil, e a verificação automatizada de sigilo JSON/WS (P3). A persistência em SQLite fica na S09 e os adaptadores HTTP de IA ficam na S07/S08.

## Histórias

- **US1.** Como jogador convidado, quero informar apenas meu apelido e receber um token de sessão, para jogar em segundos sem cadastro nem senha (ADR 0007 A, M6).
- **US2.** Como narrador humano no modo grupo online, quero criar uma sala de espera com código de convite, ver os investigadores entrarem e iniciar a partida quando o grupo estiver pronto (UC1, UC3, UC12, M11).
- **US3.** Como investigador, quero entrar numa sala usando o código de convite e jogar por REST e WebSocket recebendo atualizações em tempo real (UC12, UC4, UC6).
- **US4.** Como jogador, quero navegar pelo catálogo de enigmas publicados, consultar meu histórico de partidas e ver meu perfil com contadores (UC2, UC7).
- **US5.** Como equipe, quero garantia automatizada de que nenhum payload JSON (REST ou WebSocket) destinado a investigador contenha `solucaoCompleta` antes de estado terminal (P3, M1).

## Glossário

| Termo | Significado |
|---|---|
| Token de convidado | Token assinado com HMAC-SHA256 emitido em `POST /api/sessoes` contendo `playerId` e timestamp de emissão; enviado em `Authorization: Bearer <token>` ou parâmetro `token` no handshake WebSocket (novo, ADR 0007 A) |
| Código de sala (`codigoSala`) | Identificador curto alfanumérico (6 caracteres maiúsculos sem ambiguidade) associado a uma partida em `AGUARDANDO_INICIO` no modo `GRUPO` para entrada via UC12 (novo, M11) |
| Sala de espera | Partida em `GRUPO` criada em `AGUARDANDO_INICIO` aguardando entrada de investigadores antes do comando `iniciar` (novo, M11) |
| Difusão por papel | Entrega de atualizações WebSocket filtrando a visão conforme o papel do cliente conectado (`VisaoNarrador` para o narrador, `VisaoInvestigador` para cada investigador) (novo, P3) |

## Regras

| ID | Regra |
|---|---|
| API1 | Fronteira de camadas (P7): `interfaces.http` e `interfaces.ws` importam exclusivamente `:application` (casos de uso, DTOs e `CodigoErro`). Nenhum tipo de `:domain` ou `:infrastructure` aparece em `interfaces` |
| API2 | Identidade por token HMAC-SHA256 (ADR 0007 A): `POST /api/sessoes` valida o apelido (`RegistrarJogador`), emite token assinado com segredo configurável por ambiente (`SESSION_TOKEN_SECRET`, nunca fixo em produção nem versionado) e devolve `{ playerId, nome, token }`. Toda rota sob `/api/partidas`, `/api/historico`, `/api/perfil` e `/ws/partidas/{id}` exige token válido; ausência ou assinatura inválida devolve HTTP `401` (`TOKEN_INVALIDO`) |
| API3 | Autoria derivada do token: o servidor nunca aceita `autor` ou `solicitante` arbitrário do corpo HTTP para comandos autenticados; o `playerId` vem sempre do token verificado |
| API4 | Catálogo público (`GET /api/enigmas`): lista apenas enigmas em `PUBLICADO` (filtráveis por `categoria` e `dificuldade`) contendo `id`, `titulo`, `enunciado`, `categoria`, `dificuldade` e `tags`. Nunca inclui `solucaoCompleta` (P3) |
| API5 | Criação e salas de espera (M11, UC12): `POST /api/partidas` aceita `modo` (`SOLO` ou `GRUPO`) e filtro opcional (`enigmaId`, `categoria`, `dificuldade`). No `SOLO`, cria e inicia imediatamente (`EM_ANDAMENTO`), devolvendo `VisaoInvestigador`. No `GRUPO`, cria sala em `AGUARDANDO_INICIO` com `codigoSala` único e devolve `VisaoNarrador` + `codigoSala`. `POST /api/partidas/entrar` adiciona o investigador autenticado à sala aberta em `AGUARDANDO_INICIO` (rejeitando código inexistente, sala já iniciada/encerrada, narrador tentando entrar como investigador ou investigador duplicado). `POST /api/partidas/{id}/iniciar` inicia a sala (apenas o narrador, validando H5) |
| API6 | Sigilo por papel em REST e WebSocket (P3): toda resposta JSON e todo frame WebSocket entregue a um investigador enquanto `status` for `AGUARDANDO_INICIO` ou `EM_ANDAMENTO` omite `solucaoCompleta` (ou retorna `null`, jamais o texto da solução). Em `RESOLVIDO` ou `ABANDONADO`, `solucaoCompleta` passa a ser enviada. `VisaoNarrador` é enviada somente ao narrador autenticado da partida |
| API7 | Mapeamento uniforme de erros: todo `CodigoErro` devolve JSON `{ "codigo": "<NOME>", "mensagem": "<texto em português>" }` com status HTTP semântico: `400` (`TEXTO_INVALIDO`, `NOME_INVALIDO`, `CONFIGURACAO_INVALIDA`, `MODO_INCOMPATIVEL`), `401` (`TOKEN_INVALIDO`), `403` (`NAO_E_INVESTIGADOR`, `NAO_E_NARRADOR`, `NAO_AUTORIZADO`, `NAO_PARTICIPA`), `404` (`PARTIDA_INEXISTENTE`, `JOGADOR_INEXISTENTE`, `PERGUNTA_INEXISTENTE`, `SALA_INEXISTENTE`), `409` (`PARTIDA_ENCERRADA`, `PARTIDA_NAO_INICIADA`, `PARTIDA_JA_INICIADA`, `PERGUNTA_EM_ABERTO`, `PERGUNTA_JA_FECHADA`, `PROPOSTA_PENDENTE`, `SEM_PROPOSTA_PENDENTE`, `SEM_PERGUNTA_ABERTA`, `SEM_ENIGMA_ELEGIVEL`, `CONFLITO_CONCORRENCIA`), `500` (`FALHA_AO_SALVAR`), `503` (`IA_INDISPONIVEL`) |
| API8 | WebSocket em tempo real (`WS /ws/partidas/{id}`): aceita apenas participantes autenticados da partida (`401`/`403` caso contrário). Envia o estado inicial e cada atualização pós-persistência (H6, S04) no formato `{ "tipo": "ESTADO_ATUALIZADO", "visao": VisaoPartida }`, filtrando `solucaoCompleta` pelo papel daquele cliente (`null` para investigador antes de estado terminal) |
| API9 | Histórico e perfil: `GET /api/historico` retorna a lista resumida de partidas das quais o jogador autenticado participa ou participou (ordenada da mais recente para a mais antiga, sem `solucaoCompleta`); `GET /api/perfil` retorna `playerId`, `nome`, `jogadas`, `resolvidas`, `abandonadas` e `emAndamento` |
| API10 | CORS e saúde: `GET /health` responde `200 { "status": "ok" }` sem autenticação; cabeçalhos CORS permitem origens configuráveis (`CORS_ALLOWED_ORIGINS`, padrão `http://localhost:5173` em desenvolvimento) |

### Contrato REST e WebSocket

| Método e rota | Autenticação | Ação / Caso de uso |
|---|---|---|
| `GET /health` | Pública | Verificação de saúde (`{ "status": "ok" }`) |
| `POST /api/sessoes` | Pública | Registra convidado (`RegistrarJogador`) e devolve `{ playerId, nome, token }` |
| `GET /api/enigmas` | Opcional / Pública | Lista enigmas publicados (sem `solucaoCompleta`) |
| `POST /api/partidas` | Bearer token | Cria partida `SOLO` (já iniciada) ou sala `GRUPO` (`AGUARDANDO_INICIO` + `codigoSala`) |
| `POST /api/partidas/entrar` | Bearer token | UC12: entra em sala `GRUPO` via `{ "codigoSala": "..." }` |
| `POST /api/partidas/{id}/iniciar` | Bearer token | Narrador inicia sala em `AGUARDANDO_INICIO` |
| `GET /api/partidas/{id}` | Bearer token | `ConsultarPartida` (visão filtrada por papel) |
| `POST /api/partidas/{id}/perguntas` | Bearer token | `FazerPergunta` |
| `POST /api/partidas/{id}/perguntas/{pid}/responder` | Bearer token | `ResponderPergunta` (narrador humano) |
| `DELETE /api/partidas/{id}/perguntas/{pid}` | Bearer token | `CancelarPergunta` |
| `POST /api/partidas/{id}/propostas` | Bearer token | `ProporSolucao` |
| `POST /api/partidas/{id}/propostas/{propId}/julgar` | Bearer token | `JulgarProposta` (narrador humano) |
| `DELETE /api/partidas/{id}/propostas/{propId}` | Bearer token | `RetirarProposta` |
| `POST /api/partidas/{id}/abandonar` | Bearer token | `Abandonar` |
| `POST /api/partidas/{id}/ia/responder` | Bearer token | Retentativa H4 de `ResponderComIA` (somente investigador no modo `SOLO`) |
| `POST /api/partidas/{id}/ia/avaliar` | Bearer token | Retentativa H4 de `AvaliarComIA` (somente investigador no modo `SOLO`) |
| `GET /api/historico` | Bearer token | Lista partidas do jogador autenticado |
| `GET /api/perfil` | Bearer token | Dados e estatísticas do jogador autenticado |
| `WS /ws/partidas/{id}` | Token (header ou query) | Stream de atualizações da partida filtradas por papel |

## Critérios de aceitação

Testes executados com Ktor `testApplication` (sem abrir porta externa), usando repositórios em memória e fakes de IA (P6). Participantes de exemplo: narrador N, investigadoras Ana e Bia, e Zé (não participante).

| CA | Regra | Dado | Quando | Então |
|---|---|---|---|---|
| 01 | API2 | servidor configurado com segredo HMAC | `POST /api/sessoes` com `{ "nome": "Ana" }`; depois `{ "nome": "" }` | `201` com `playerId`, `nome: "Ana"` e `token` assinado; na segunda chamada, `400` com `codigo: "NOME_INVALIDO"` |
| 02 | API2 | rota protegida (`GET /api/perfil` ou `POST /api/partidas`) | requisição sem header, com token adulterado ou assinatura inválida | `401` com `codigo: "TOKEN_INVALIDO"` |
| 03 | API4, P3 | seed carregado + um enigma em `RASCUNHO` | `GET /api/enigmas` | `200` com apenas os 5 enigmas publicados; o corpo JSON inteiro não contém o texto de nenhuma `solucaoCompleta` |
| 04 | API5 | N e Ana autenticados | N chama `POST /api/partidas` (`GRUPO`); Ana chama `POST /api/partidas/entrar` com `codigoSala`; N chama `POST /api/partidas/{id}/iniciar` | sala nasce `AGUARDANDO_INICIO` com `codigoSala`; Ana entra e aparece em `investigadores`; `iniciar` devolve `EM_ANDAMENTO` |
| 05 | API5 | sala em `AGUARDANDO_INICIO` | N tenta entrar na própria sala; Zé envia código inexistente; Ana tenta entrar após `iniciar` | rejeições com status `400`/`404`/`409` sem corromper a lista de participantes |
| 06 | API5, API3 | Ana autenticada | `POST /api/partidas` (`SOLO`, `categoria: "DEBUG"`) | `201` com partida já em `EM_ANDAMENTO`, narrador IA, `VisaoInvestigador` sem `solucaoCompleta` |
| 07 | API6, P3 | partida em grupo e partida solo passando por todas as fases | serializar TODA resposta JSON REST de Ana (criação, consulta, pergunta, proposta, recusa, histórico) antes de estado terminal | o texto literal de `solucaoCompleta` nunca aparece em nenhum corpo JSON; após `RESOLVIDO` ou `ABANDONADO`, aparece |
| 08 | API6, P3 | partida em grupo em andamento | N chama `GET /api/partidas/{id}`; Zé chama `GET /api/partidas/{id}` | N recebe `200` com `solucaoCompleta`; Zé recebe `403` (`NAO_PARTICIPA`) |
| 09 | API7 | partida em andamento com pergunta aberta de Ana | Bia faz pergunta (`409 PERGUNTA_EM_ABERTO`), Ana responde (`403 NAO_E_NARRADOR`), Ana envia texto vazio (`400 TEXTO_INVALIDO`) | cada resposta traz o status HTTP da tabela API7 e JSON `{ "codigo", "mensagem" }` |
| 10 | API8, P3 | N e Ana conectados em `WS /ws/partidas/{id}` | Ana envia pergunta via REST e N responde via REST | ambos recebem frame atualizado; o frame de N contém `solucaoCompleta` e o frame de Ana NÃO contém `solucaoCompleta` até o fim da partida |
| 11 | API8 | Zé autenticado (não participante) e cliente sem token | conectam em `WS /ws/partidas/{id}` | conexões rejeitadas (`403` / `401`) sem enviar nenhum dado da partida |
| 12 | API9 | Ana concluiu uma partida e tem outra em andamento | `GET /api/historico` e `GET /api/perfil` | histórico retorna as duas partidas (solução oculta na ativa, visível na resolvida); perfil retorna `playerId`, `nome` e contadores |
| 13 | API10 | servidor ativo | `GET /health` | `200` com `{ "status": "ok" }` |
| 14 | API1, P7 | código de produção de `interfaces` | teste de arquitetura Konsist (`ArquiteturaTest`) | zero infrações; nenhum arquivo em `interfaces` importa `com.quebugesse.domain` nem `com.quebugesse.infrastructure` |

## Requisitos não funcionais

- **RNF1.** Conformidade integral com P1, P3 e P7 (verificado pelo Konsist em todo build).
- **RNF2.** Cobertura de linhas de pelo menos 90% nos novos pacotes de `interfaces` e `application` (Kover).
- **RNF3.** Nenhum segredo (`SESSION_TOKEN_SECRET`) fixo no repositório para produção; testes usam segredo efêmero em memória.
- **RNF4.** Suíte de testes HTTP/WebSocket com Ktor `testApplication` roda sem rede externa em menos de 15 segundos.

## Fora de escopo

- Contas com e-mail/senha, OAuth ou recuperação de credenciais (ADR 0007 A).
- Adaptadores reais de IA (S07/S08) e persistência SQLite (S09).
- Telas do navegador (`Que-Bug-E-Esse/front`, S10).
- Atualização de estatísticas por evento (S12, M17) e geração/curadoria de enigmas (S11).
- Rate limiting distribuído em múltiplas instâncias e deploy Nginx.

## Divergências em relação aos diagramas

Pelo princípio P9, as fontes em [docs/diagramas/](../../docs/diagramas/README.md) já foram atualizadas com Ktor, UC12 e tokens HMAC; resta apenas o detalhamento abaixo quando a S06 for implementada:

| ID | Diagrama hoje | Esta spec | Origem |
|---|---|---|---|
| D1 | [sequencia-iniciar-partida.md](../../docs/diagramas/sequencia-iniciar-partida.md) mostra criação e `iniciar` juntos (com nota sobre M11) | No modo `GRUPO` online, `CriarPartida` (sala em `AGUARDANDO_INICIO` + `codigoSala`) e `EntrarNaPartida` (UC12) precedem `IniciarPartida` | M11, ADR 0007 A |

## Questões abertas

Nenhuma questão bloqueante em aberto: todas as decisões estruturais foram fechadas pelas ADRs 0003 A, 0007 A, 0008 A e 0009 A.

## Rastreabilidade

| CA | Arquivo de teste previsto |
|---|---|
| 01, 02, 13 | `interfaces/.../http/SessaoEHealthRoutesTest.kt` |
| 03, 04, 05, 06, 08, 09, 12 | `interfaces/.../http/PartidasRoutesTest.kt` |
| 07 | `interfaces/.../http/SigiloPayloadJsonTest.kt` |
| 10, 11 | `interfaces/.../http/PartidasWebSocketTest.kt` |
| 14 | `bootstrap/src/test/kotlin/com/quebugesse/arquitetura/ArquiteturaTest.kt` |
