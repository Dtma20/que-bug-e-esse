# Spec 007/008: Integração com o serviço de IA (`NarradorIaHttp` e `VerificadorDeSolucaoHttp`)

| Campo | Valor |
|---|---|
| ID | S07/S08 |
| Status | Aprovada |
| Depende de | S03 (portas `Narrador` e `VerificadorDeSolucao`, `ResponderComIA`, `AvaliarComIA`, `IA_INDISPONIVEL`), S04 (ouvintes `AoFazerPerguntaNoSolo` e `AoProporSolucaoNoSolo`, H4/H6), serviço `Que-Bug-E-Esse/ai` (`POST /narrar`, `POST /verificar`, `GET /health`) |
| Casos de uso | UC5 Responder pergunta (modo solo), UC6 Propor solução (modo solo), UC10 Jogar sozinho |
| Diagramas de origem | [componentes.md](../../docs/diagramas/componentes.md), [implantacao.md](../../docs/diagramas/implantacao.md), [sequencia-investigar-e-resolver.md](../../docs/diagramas/sequencia-investigar-e-resolver.md) |
| Decisões que assume | [ADR 0001](../../docs/adr/0001-veredito-no-modo-grupo.md) opção A (IA julga só no solo), [ADR 0002](../../docs/adr/0002-contrato-da-porta-narrador.md) opção A (porta `Narrador` síncrona só para IA), [ADR 0005](../../docs/adr/0005-provedor-de-llm.md) opção A (Gemini `gemini-2.5-flash` com saída estruturada, US$ 10/mês, 10.000 tokens/partida, `LLM_API_KEY` só no serviço `ai`), [ADR 0009](../../docs/adr/0009-divisao-de-repositorios.md) opção A (`Que-Bug-E-Esse/ai` HTTP stateless), todas `Aceita` |
| Melhorias incorporadas | M2 (porta `Narrador` síncrona só para IA), M3 (retentativa com backoff e falha H4 sem travar), M7 (enum fechado e resistência a injeção de prompt), M22 (veredito estritamente booleano) do [planejamento](../../planejamento-sdd.md) |
| Plano | Plano e tarefas: ainda não escritos |

## Contexto

Na S03 e na S04, o ciclo completo do modo solo já funciona com fakes em memória (`FakeNarrador` e `FakeVerificador`) disparados pelos ouvintes `AoFazerPerguntaNoSolo` e `AoProporSolucaoNoSolo`. Em paralelo, o serviço stateless `Que-Bug-E-Esse/ai` (PR #1) encapsula o Google Gemini (`gemini-2.5-flash`), os prompts defensivos (M7) e o orçamento de tokens (ADR 0005 A) atrás de dois endpoints HTTP JSON mínimos: `POST /narrar` e `POST /verificar`.

Esta spec une S07 e S08 no `Que-Bug-E-Esse/back`: especifica os dois adaptadores HTTP em `com.quebugesse.infrastructure.ai` (`NarradorIaHttp` e `VerificadorDeSolucaoHttp`), o mapeamento exato de contratos JSON, a política de timeout/retentativa preservando H4, as garantias de sigilo (P3) e a bateria de verificação sem depender da rede externa na CI (P6).

## Histórias

- **US1.** Como jogador solo, quero que minhas perguntas sejam respondidas por `SIM`, `NAO` ou `IRRELEVANTE` pelo serviço de IA, nunca com texto livre que revele a solução (UC5, UC10, P3, M7).
- **US2.** Como jogador solo, quero que minha proposta de solução seja julgada como `correta` ou `incorreta` pelo serviço de IA (UC6, UC10, ADR 0001 A, M22).
- **US3.** Como jogador solo, quero que falhas transitórias de rede ou HTTP 5xx sejam retentadas com backoff curto e, se a IA continuar indisponível, minha partida não mude de estado para que eu possa tentar novamente ou cancelar/retirar (H4, M3).
- **US4.** Como desenvolvedor, quero testar ambos os adaptadores na CI contra um servidor HTTP local simulado, mantendo a bateria contra o Gemini real opt-in fora da CI padrão (P6, ADR 0005 A).

## Glossário

| Termo | Significado |
|---|---|
| `NarradorIaHttp` | Adaptador em `infrastructure.ai` que implementa `domain.gamesession.port.Narrador` chamando `POST {AI_SERVICE_URL}/narrar` (novo) |
| `VerificadorDeSolucaoHttp` | Adaptador em `infrastructure.ai` que implementa `domain.gamesession.port.VerificadorDeSolucao` chamando `POST {AI_SERVICE_URL}/verificar` (novo) |
| `IaIndisponivelException` | Exceção de infraestrutura lançada pelos adaptadores quando o serviço `ai` falha após retentativas ou devolve payload inválido; capturada por `ResponderComIA` e `AvaliarComIA` como `IA_INDISPONIVEL` (novo, H4) |
| Falha transitória | Erro de conexão, timeout ou status HTTP `429`, `500`, `502`, `503` ou `504` elegível a retentativa limitada (novo, M3) |

## Regras

| ID | Regra |
|---|---|
| IA1 | Contrato exato com `Que-Bug-E-Esse/ai` (`schemas.py`, `extra="forbid"`): `NarradorIaHttp` envia `POST {AI_SERVICE_URL}/narrar` com `{ "enunciado", "solucaoCompleta", "pergunta", "historico": [{ "pergunta", "resposta" }] }` e aceita exclusivamente `{ "resposta": "SIM" | "NAO" | "IRRELEVANTE" }`. `VerificadorDeSolucaoHttp` envia `POST {AI_SERVICE_URL}/verificar` com `{ "enunciado", "solucaoCompleta", "proposta" }` e aceita exclusivamente `{ "correta": true | false }`. Qualquer campo desconhecido, tipo inválido ou valor fora do conjunto fechado gera `IaIndisponivelException` |
| IA2 | Configuração externa: a URL base vem de `AI_SERVICE_URL` (padrão local `http://localhost:8000`), timeout configurável (padrão 10 s por tentativa) e número máximo de tentativas configurável (padrão 3 tentativas com backoff exponencial curto, ex.: 100 ms, 300 ms). O `back` nunca lê nem armazena `LLM_API_KEY` (ADR 0005 A, ADR 0009 A) |
| IA3 | Política de retentativa e H4 (M3): falhas transitórias (erro de rede, timeout, HTTP `429` ou `5xx`) são retentadas até o limite configurado. Erros `4xx` (exceto `429`) e JSON inválido/malformado falham imediatamente sem retentativa. Ao esgotar tentativas ou falhar, o adaptador lança `IaIndisponivelException`; os casos de uso `ResponderComIA` e `AvaliarComIA` devolvem `IA_INDISPONIVEL` sem salvar nem alterar o estado da partida |
| IA4 | Sigilo nos adaptadores e logs (P3): `NarradorIaHttp`, `VerificadorDeSolucaoHttp` e suas exceções nunca registram `solucaoCompleta` (nem o corpo bruto de requisição) em logs, mensagens de erro ou `toString()` |
| IA5 | Proteção contra injeção de prompt (M7): o texto da pergunta ou da proposta vai estritamente no campo de dados correspondente (`pergunta` / `proposta`), sem concatenação de prompt no `back`. O adaptador rejeita qualquer resposta que não corresponda exatamente ao enum `Answer` ou booleano `Veredito` |
| IA6 | Isolamento na CI (P6): testes unitários e de integração em `Que-Bug-E-Esse/back` usam servidor HTTP local simulado (ou motor mock) sem acesso à internet. A bateria contra o serviço `ai` + Gemini real (perguntas de referência e adversariais de M7) fica segregada fora da CI padrão |
| IA7 | Uso exclusivo no modo solo (ADR 0001 A, ADR 0002 A): nenhuma rota ou caso de uso do modo `GRUPO` instancia ou chama `NarradorIaHttp` ou `VerificadorDeSolucaoHttp` |

### Emenda à S03 (`VerificadorDeSolucao`)

O contrato HTTP de `Que-Bug-E-Esse/ai` (`POST /verificar`, `VerificarRequest`) exige o campo não vazio `enunciado` além de `solucaoCompleta` e `proposta`. Para manter compatibilidade entre `back` e `ai`:

- `VerificadorDeSolucao.avaliar` recebe `enunciado: String` (por exemplo `avaliar(proposta: String, solucaoCompleta: String, enunciado: String = "")` ou sobrecarga compatível com os testes existentes da S03) e `AvaliarComIA` passa `enigma.conteudo.enunciado`.

## Critérios de aceitação

Testes em `infrastructure` executados contra servidor HTTP local embarcado (sem rede externa), com backoff configurável (0 ms nos testes) e captura de logs.

| CA | Regra | Dado | Quando | Então |
|---|---|---|---|---|
| 01 | IA1 | servidor mock respondendo `200 { "resposta": "SIM" }`, `"NAO"` e `"IRRELEVANTE"` | `NarradorIaHttp.responder(pergunta, contexto)` | envia JSON exato (`enunciado`, `solucaoCompleta`, `pergunta`, `historico` em ordem) para `POST /narrar` e devolve `Answer.SIM`, `NAO` e `IRRELEVANTE` |
| 02 | IA1 | servidor mock respondendo `200 { "correta": true }` e `{ "correta": false }` | `VerificadorDeSolucaoHttp.avaliar(proposta, solucao, enunciado)` | envia JSON exato (`enunciado`, `solucaoCompleta`, `proposta`) para `POST /verificar` e devolve `Veredito(true)` e `Veredito(false)` |
| 03 | IA1, IA5 | servidor mock devolvendo `200` com `{ "resposta": "TALVEZ" }`, `{ "resposta": "SIM", "extra": 1 }`, `{ "correta": "true" }` ou JSON malformado | chamar `NarradorIaHttp` e `VerificadorDeSolucaoHttp` | lançam `IaIndisponivelException` sem retentativa |
| 04 | IA3, H4 | servidor mock falha duas vezes com HTTP `503` e responde `200 { "resposta": "NAO" }` na 3ª tentativa | `NarradorIaHttp.responder(...)` com máximo de 3 tentativas | executa 3 chamadas e devolve `Answer.NAO` |
| 05 | IA3, H4 | servidor mock retorna `503` em todas as tentativas, ou estoura timeout, ou recusa conexão | `NarradorIaHttp` e `VerificadorDeSolucaoHttp` | após esgotar tentativas, lançam `IaIndisponivelException`; integrados a `ResponderComIA`/`AvaliarComIA`, retornam `IA_INDISPONIVEL` e a partida salva permanece inalterada |
| 06 | IA3 | servidor mock retorna HTTP `400` ou `422` (`ErrorResponse`) | `NarradorIaHttp` e `VerificadorDeSolucaoHttp` | falham imediatamente na 1ª tentativa (sem retry) lançando `IaIndisponivelException` |
| 07 | IA4, P3 | `solucaoCompleta = "SEGREDO_CRITICO_42"` em cenário de sucesso e em cenário de falha (`500`, `422`, timeout) | executar ambos os adaptadores e inspecionar logs e `exception.message` | a string `"SEGREDO_CRITICO_42"` nunca aparece em logs nem em mensagens de exceção |
| 08 | IA2, P7 | módulos `domain`, `application` e `infrastructure` | teste de arquitetura Konsist (`ArquiteturaTest`) | `domain` permanece puro (P1); adaptadores HTTP vivem apenas em `infrastructure.ai`; `bootstrap` escolhe entre adaptadores HTTP e fakes por configuração |

## Requisitos não funcionais

- **RNF1.** Domínio puro (P1) e fronteira de camadas preservada (P7).
- **RNF2.** Cobertura de linhas de pelo menos 90% em `com.quebugesse.infrastructure.ai` (Kover).
- **RNF3.** Suíte de `infrastructure.ai` com servidor local simulado roda em menos de 10 segundos e sem internet (P6).
- **RNF4.** `LLM_API_KEY` nunca existe nem é referenciada em `Que-Bug-E-Esse/back`.

## Fora de escopo

- Prompts defensivos, chamada à Google Gemini API e controle de orçamento/tokens (já implementados em `Que-Bug-E-Esse/ai#1`).
- Geração de enigmas por IA (`POST /gerar-enigma`, S11).
- Veredito parcial ("quase") ou texto explicativo livre (M22 resolvida: veredito estritamente booleano).

## Divergências em relação aos diagramas

Pelo princípio P9, [componentes.md](../../docs/diagramas/componentes.md), [implantacao.md](../../docs/diagramas/implantacao.md) e [sequencia-investigar-e-resolver.md](../../docs/diagramas/sequencia-investigar-e-resolver.md) já refletem `NarradorIaHttp`, `VerificadorDeSolucaoHttp` e `Que-Bug-E-Esse/ai`:

| ID | Diagrama hoje | Esta spec | Origem |
|---|---|---|---|
| D1 | [sequencia-investigar-e-resolver.md](../../docs/diagramas/sequencia-investigar-e-resolver.md) já lista `avaliar(proposta, solucaoCompleta, enunciado)` | Emenda à S03 acrescentando `enunciado` à porta `VerificadorDeSolucao` para casar com `VerificarRequest` de `Que-Bug-E-Esse/ai` | ADR 0009 A |

## Questões abertas

Nenhuma questão bloqueante em aberto: todas as escolhas de provedor, saída estruturada, divisão de repositórios e veredito booleano foram fechadas pelas ADRs 0001 A, 0002 A, 0005 A e 0009 A e pela melhoria M22.

## Rastreabilidade

| CA | Arquivo de teste previsto |
|---|---|
| 01, 03 (narrador), 04, 05 (narrador), 06 (narrador), 07 (narrador) | `infrastructure/.../ai/NarradorIaHttpTest.kt` |
| 02, 03 (verificador), 05 (verificador), 06 (verificador), 07 (verificador) | `infrastructure/.../ai/VerificadorDeSolucaoHttpTest.kt` |
| 08 | `bootstrap/src/test/kotlin/com/quebugesse/arquitetura/ArquiteturaTest.kt` |
