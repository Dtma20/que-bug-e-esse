# Spec 011: Geração de enigmas por IA e curadoria

| Campo | Valor |
|---|---|
| ID | S11 |
| Status | Aprovada |
| Depende de | S02 (`Enigma`, `ConteudoEnigma`, `OrigemEnigma`, `StatusEnigma`, `EnigmaRepository`, regras E1–E8 e EL1), S06 (API Ktor, autenticação Bearer HMAC, `CodigoErro`), S07/S08 (cliente HTTP para `Que-Bug-E-Esse/ai`, H4, M7, P3) |
| Casos de uso | UC8 Gerar enigma via IA, UC9 Cadastrar / revisar enigma manualmente (ciclo completo de curadoria: criar, revisar, publicar e arquivar) |
| Diagramas de origem | [casos-de-uso.md](../../docs/diagramas/casos-de-uso.md), [classes-dominio.md](../../docs/diagramas/classes-dominio.md), [componentes.md](../../docs/diagramas/componentes.md), [implantacao.md](../../docs/diagramas/implantacao.md) |
| Decisões que assume | [ADR 0003](../../docs/adr/0003-framework-web.md) opção A (Ktor), [ADR 0005](../../docs/adr/0005-provedor-de-llm.md) opção A (Gemini `gemini-2.5-flash` com saída estruturada e `LLM_API_KEY` fora do repositório), [ADR 0007](../../docs/adr/0007-identidade-e-autenticacao.md) opção A (token HMAC de convidado), [ADR 0008](../../docs/adr/0008-stack-do-front-end.md) opção A (SPA TypeScript + Vite + React), [ADR 0009](../../docs/adr/0009-divisao-de-repositorios.md) opção A (`Que-Bug-E-Esse/ai` HTTP stateless; `front` consome só `back`), todas `Aceita` |
| Melhorias incorporadas | M7 (isolamento de entrada contra injeção de prompt), M16 (porta em `domain.enigma.port`), M19 (fluxo `RASCUNHO -> PUBLICADO / ARQUIVADO` com rascunhos não jogáveis), P3 (sigilo no catálogo público), H4 (falha da IA sem mutar o catálogo) do [planejamento](../../planejamento-sdd.md) |
| Plano | Plano e tarefas: ainda não escritos (implementação do marco M4 em `Que-Bug-E-Esse/ai`, `back` e `front`) |

## Contexto

A S02 já entrega o agregado `Enigma` com `ConteudoEnigma`, `OrigemEnigma` (`MANUAL` / `IA`), `StatusEnigma` (`RASCUNHO` / `PUBLICADO` / `ARQUIVADO`) e as transições `revisar`, `publicar` e `arquivar`, mas nenhum caso de uso, rota REST ou endpoint de IA expunha ainda a criação, geração e curadoria de enigmas.

Esta spec fecha UC8 e UC9 nos três repositórios da organização: acrescenta `POST /gerar` em `Que-Bug-E-Esse/ai`, cria a porta `GeradorDeEnigmas` (`domain.enigma.port`), o adaptador `GeradorDeEnigmasHttp` (`infrastructure.ai`), os casos de uso e as rotas `/api/curadoria/enigmas/*` em `Que-Bug-E-Esse/back`, e a interface de curadoria em `Que-Bug-E-Esse/front`. Pela melhoria M19 e pelo princípio P3, todo enigma gerado ou cadastrado nasce em `RASCUNHO` e permanece invisível e não jogável em `GET /api/enigmas` até ser explicitamente publicado.

## Histórias

- **US1.** Como curador, quero gerar um rascunho de enigma pela IA informando categoria, dificuldade e um tema opcional, para ampliar o catálogo rapidamente (UC8, M19).
- **US2.** Como curador, quero cadastrar manualmente um enigma em rascunho, revisar seus campos enquanto estiver em `RASCUNHO`, publicá-lo quando estiver pronto ou arquivá-lo para tirá-lo de circulação (UC9, M19).
- **US3.** Como investigador, quero que apenas enigmas `PUBLICADO` apareçam no catálogo público e sejam sorteados nas partidas, e que `GET /api/enigmas` nunca exponha `solucaoCompleta` (M19, P3).
- **US4.** Como curador, quero que uma falha do serviço de IA retorne erro claro (`IA_INDISPONIVEL`) sem gravar rascunho corrompido nem alterar o catálogo existente (H4).
- **US5.** Como equipe, quero que a entrada opcional `tema` seja isolada contra injeção de prompt e que a CI rode sem chamadas à internet nem ao Gemini real (M7, P6).

## Glossário

| Termo | Significado |
|---|---|
| `GeradorDeEnigmas` | Porta do domínio em `domain.enigma.port` que solicita um rascunho de enigma para `(categoria, dificuldade, tema?)` e devolve `EnigmaGerado` (novo, M16) |
| `EnigmaGerado` | Value object em `domain.enigma.port` contendo `titulo`, `enunciado`, `solucaoCompleta` e `tags` retornados pela IA antes da criação do agregado `Enigma` (novo) |
| `GeradorDeEnigmasHttp` | Adaptador em `infrastructure.ai` que implementa `GeradorDeEnigmas` chamando `POST {AI_SERVICE_URL}/gerar` com timeout e retentativa limitada (novo) |
| `EnigmaCuradoriaDto` | DTO da camada de aplicação/HTTP para curadoria: inclui `solucaoCompleta`, `status`, `origem` e `criadoEm`, restrito às rotas autenticadas `/api/curadoria/enigmas/*` (novo) |

## Regras

| ID | Regra |
|---|---|
| CUR1 | Fronteira e portas (P1, P6, P7, M16): `GeradorDeEnigmas` e `EnigmaGerado` vivem em `com.quebugesse.domain.enigma.port` sem dependências externas. Existe `FakeGeradorDeEnigmas` programável (inclusive para falhar) para testes de aplicação; `GeradorDeEnigmasHttp` vive em `com.quebugesse.infrastructure.ai` e é ligado em `bootstrap/ServidorHttp.kt`. `interfaces.http` importa exclusivamente `:application` |
| CUR2 | Contrato `POST /gerar` em `Que-Bug-E-Esse/ai` (`schemas.py`, `extra="forbid"`): recebe `{ "categoria": "SINTAXE"\|"DEBUG"\|"GESTAO_PROJETO"\|"DESIGN"\|"DISCOVERY", "dificuldade": "FACIL"\|"MEDIA"\|"DIFICIL", "tema": string \| null }` (`tema` opcional, padrão `null`, aparado, branco normalizado para `null`, máximo de 200 caracteres). Devolve HTTP `200` com `{ "titulo" (1..100), "enunciado" (1..2000), "solucaoCompleta" (1..2000), "tags": string[] }` (tags aparadas, minúsculas, únicas, não vazias e com pelo menos 1 item). Erros devolvem o mesmo `ErrorResponse` sanitizado de `/narrar` e `/verificar`: `422` (`REQUISICAO_INVALIDA` / `LIMITE_DE_TOKENS_EXCEDIDO`), `502` (`RESPOSTA_INVALIDA_DO_PROVEDOR`), `503` (`PROVEDOR_INDISPONIVEL` / `PROVEDOR_NAO_CONFIGURADO`) e `500` (`ERRO_INTERNO`) |
| CUR3 | Isolamento de prompt e provedores em `ai` (M7, ADR 0005 A): `LLMProvider` expõe `gerar_enigma(...)`; `FakeLLMProvider` devolve enigmas determinísticos coerentes com `categoria`/`dificuldade` e aceita falha programada; `GeminiLLMProvider` usa saída estruturada (`responseSchema`), `maxOutputTokens=1024` e isola o campo opcional `tema` entre delimitadores XML no prompt |
| CUR4 | Retentativa e H4 em `GeradorDeEnigmasHttp`: falhas transitórias (erro de rede, timeout, HTTP `429` ou `5xx`) são retentadas até o limite configurado; erros `4xx` (exceto `429`) e JSON inválido falham imediatamente sem retry. Ao falhar, lança `ServicoIaIndisponivelException`; `GerarEnigmaComIA` devolve `IA_INDISPONIVEL` (HTTP `503`) sem persistir nenhum enigma |
| CUR5 | Ciclo de vida e validação (S02 E1–E7, M19): `GerarEnigmaComIA` cria `Enigma` com `status = RASCUNHO` e `origem = IA`; `CadastrarEnigmaManual` cria com `status = RASCUNHO` e `origem = MANUAL`. `RevisarEnigma` só altera enigmas em `RASCUNHO` (`409 ENIGMA_NAO_EDITAVEL` em `PUBLICADO` ou `ARQUIVADO`). `PublicarEnigma` transita `RASCUNHO -> PUBLICADO` (`409 TRANSICAO_INVALIDA` caso contrário). `ArquivarEnigma` transita `RASCUNHO` ou `PUBLICADO -> ARQUIVADO` (`409 TRANSICAO_INVALIDA` em `ARQUIVADO`). Limites de `ConteudoEnigma` seguem E1–E2 (`TITULO_INVALIDO`, `ENUNCIADO_INVALIDO`, `SOLUCAO_INVALIDA`) |
| CUR6 | Rascunhos não jogáveis e sigilo no catálogo público (M19, P3): `GET /api/enigmas`, `IniciarPartida` e `CriarSalaGrupo` continuam operando exclusivamente sobre `EnigmaRepository.publicados()`. Nenhum enigma em `RASCUNHO` ou `ARQUIVADO` aparece em `GET /api/enigmas` nem pode ser sorteado ou selecionado em partidas `SOLO` ou `GRUPO`; `GET /api/enigmas` jamais retorna `solucaoCompleta` |
| CUR7 | Autenticação e CORS das rotas de curadoria: todas as rotas `/api/curadoria/enigmas/*` exigem `Authorization: Bearer <token>` válido (ADR 0007 A), devolvendo HTTP `401` `{ "codigo": "NAO_AUTORIZADO", "mensagem": "..." }` quando ausente ou inválido. A configuração CORS do servidor inclui o método `PUT` além de `GET`, `POST` e `DELETE` |
| CUR8 | Sigilo em logs e front-end (P3): `GeradorDeEnigmasHttp`, `Que-Bug-E-Esse/ai` e exceções nunca registram `solucaoCompleta` nem `LLM_API_KEY` em logs. No `Que-Bug-E-Esse/front`, `solucaoCompleta` é exibida apenas na interface de curadoria e jamais vaza para as telas de investigador (`01 · Enigmas`, `03 · Partida` antes de estado terminal, `07 · Histórico`) |

### Contrato REST de curadoria (`Que-Bug-E-Esse/back`)

Tipo `EnigmaCuradoriaDto`:

```json
{
  "id": "UUID",
  "titulo": "string (1..100)",
  "enunciado": "string (1..2000)",
  "solucaoCompleta": "string (1..2000)",
  "categoria": "SINTAXE | DEBUG | GESTAO_PROJETO | DESIGN | DISCOVERY",
  "dificuldade": "FACIL | MEDIA | DIFICIL",
  "tags": ["string"],
  "status": "RASCUNHO | PUBLICADO | ARQUIVADO",
  "origem": "MANUAL | IA",
  "criadoEm": "ISO-8601 UTC"
}
```

| Método e rota | Caso de uso | Requisição | Sucesso | Erros principais |
|---|---|---|---|---|
| `POST /api/curadoria/enigmas/gerar` | `GerarEnigmaComIA` | `{ categoria, dificuldade, tema?: string \| null }` | `201 EnigmaCuradoriaDto` (`RASCUNHO`, `IA`) | `400 CONFIGURACAO_INVALIDA`, `401 NAO_AUTORIZADO`, `503 IA_INDISPONIVEL` |
| `POST /api/curadoria/enigmas` | `CadastrarEnigmaManual` | `{ titulo, enunciado, solucaoCompleta, categoria, dificuldade, tags?: string[] }` | `201 EnigmaCuradoriaDto` (`RASCUNHO`, `MANUAL`) | `400 TITULO_INVALIDO \| ENUNCIADO_INVALIDO \| SOLUCAO_INVALIDA \| CONFIGURACAO_INVALIDA`, `401 NAO_AUTORIZADO` |
| `GET /api/curadoria/enigmas` | `ListarEnigmasCuradoria` | Query opcional `?status=RASCUNHO\|PUBLICADO\|ARQUIVADO` | `200 EnigmaCuradoriaDto[]` | `400 CONFIGURACAO_INVALIDA`, `401 NAO_AUTORIZADO` |
| `PUT /api/curadoria/enigmas/{id}` | `RevisarEnigma` | `{ titulo, enunciado, solucaoCompleta, categoria, dificuldade, tags?: string[] }` | `200 EnigmaCuradoriaDto` (`RASCUNHO`) | `400 ...`, `401 NAO_AUTORIZADO`, `404 ENIGMA_INEXISTENTE`, `409 ENIGMA_NAO_EDITAVEL` |
| `POST /api/curadoria/enigmas/{id}/publicar` | `PublicarEnigma` | — | `200 EnigmaCuradoriaDto` (`PUBLICADO`) | `401 NAO_AUTORIZADO`, `404 ENIGMA_INEXISTENTE`, `409 TRANSICAO_INVALIDA` |
| `POST /api/curadoria/enigmas/{id}/arquivar` | `ArquivarEnigma` | — | `200 EnigmaCuradoriaDto` (`ARQUIVADO`) | `401 NAO_AUTORIZADO`, `404 ENIGMA_INEXISTENTE`, `409 TRANSICAO_INVALIDA` |

## Critérios de aceitação

Testes em `ai` executados com `pytest` offline (`FakeLLMProvider` e mocks de Gemini); testes em `back` executados com Ktor `testApplication`, `FakeGeradorDeEnigmas` e servidor HTTP local simulado (P6); testes em `front` executados com Vitest.

| CA | Regra | Dado | Quando | Então |
|---|---|---|---|---|
| 01 | CUR2, CUR3 | `Que-Bug-E-Esse/ai` com `FakeLLMProvider` | `POST /gerar` com `{ "categoria": "DEBUG", "dificuldade": "MEDIA", "tema": "  cache  " }` | `200` com `titulo` (1..100), `enunciado` (1..2000), `solucaoCompleta` (1..2000) e `tags` normalizadas (minúsculas, únicas, `>= 1`) |
| 02 | CUR2, CUR3 | `Que-Bug-E-Esse/ai` | `POST /gerar` com `tema` > 200 chars, campo extra desconhecido, ou falha de provedor/schema | `422 REQUISICAO_INVALIDA` para payload inválido; `502 RESPOSTA_INVALIDA_DO_PROVEDOR` ou `503 PROVEDOR_INDISPONIVEL` em falha do provedor; `tema` adversarial vai isolado em delimitadores XML no Gemini |
| 03 | CUR4, H4 | `GeradorDeEnigmasHttp` contra servidor local simulado | servidor falha 2× com `503` e responde `200` na 3ª; depois retorna `503` em todas ou `422` | na 1ª sequência, retorna `EnigmaGerado` na 3ª tentativa; em falha persistente ou `422` (sem retry), lança `ServicoIaIndisponivelException` e `GerarEnigmaComIA` devolve `503 IA_INDISPONIVEL` sem salvar rascunho |
| 04 | CUR5, CUR6, M19 | seed com 5 enigmas publicados e curador autenticado | `POST /api/curadoria/enigmas/gerar` (`DEBUG`, `MEDIA`) | `201` com `status: "RASCUNHO"` e `origem: "IA"`; `GET /api/enigmas` continua devolvendo exatamente 5 enigmas e nenhuma `solucaoCompleta` |
| 05 | CUR5, CUR6, M19 | curador autenticado | `POST /api/curadoria/enigmas` com conteúdo válido; depois `PUT /api/curadoria/enigmas/{id}` com título revisado | nasce `201` em `RASCUNHO` (`origem: "MANUAL"`); revisão devolve `200` com conteúdo atualizado ainda em `RASCUNHO` e invisível em `GET /api/enigmas` |
| 06 | CUR5, CUR6 | enigma em `RASCUNHO` | `POST /api/curadoria/enigmas/{id}/publicar` | `200` com `status: "PUBLICADO"`; `GET /api/enigmas` passa a retornar 6 enigmas (ainda sem `solucaoCompleta`) e o enigma torna-se jogável |
| 07 | CUR5 | enigma `PUBLICADO` | `PUT /api/curadoria/enigmas/{id}`; depois `POST /api/curadoria/enigmas/{id}/publicar` | `409 ENIGMA_NAO_EDITAVEL` na edição e `409 TRANSICAO_INVALIDA` na republicação |
| 08 | CUR5, CUR6 | enigma `PUBLICADO` (ou `RASCUNHO`) | `POST /api/curadoria/enigmas/{id}/arquivar`; depois novo `arquivar` ou `publicar` | `200` com `status: "ARQUIVADO"` e desaparece de `GET /api/enigmas`; novas transições devolvem `409 TRANSICAO_INVALIDA` |
| 09 | CUR7 | rotas `/api/curadoria/enigmas/*` | chamada sem header `Authorization` ou com token adulterado; chamada com `id` inexistente | `401 NAO_AUTORIZADO` sem autenticação válida; `404 ENIGMA_INEXISTENTE` para ID desconhecido |
| 10 | CUR7 | curadoria com enigmas nos três status | `GET /api/curadoria/enigmas?status=RASCUNHO` e `GET /api/curadoria/enigmas?status=INVALIDO` | lista apenas os `RASCUNHO` no primeiro caso; `400 CONFIGURACAO_INVALIDA` no segundo |
| 11 | CUR8, P3 | `solucaoCompleta = "SEGREDO_CURADORIA_99"` em sucesso e falha de `GeradorDeEnigmasHttp` e no catálogo público | inspecionar logs/exceções e respostas de `GET /api/enigmas` e telas de investigador no `front` | `"SEGREDO_CURADORIA_99"` não aparece em logs, exceções, `GET /api/enigmas` nem em telas de investigador |
| 12 | CUR1, P1, P7 | módulos do `back` | teste de arquitetura Konsist (`ArquiteturaTest`) | zero infrações; `domain` puro, `interfaces` sem imports de `domain` ou `infrastructure` |

## Requisitos não funcionais

- **RNF1.** Domínio puro (P1) e fronteira de camadas preservada (P7), validados pelo Konsist em todo build.
- **RNF2.** Cobertura de linhas de pelo menos 90% nos pacotes tocados em `Que-Bug-E-Esse/back` (Kover).
- **RNF3.** Todas as suítes da CI (`pytest` em `ai`, Gradle em `back`, Vitest em `front`) rodam offline sem internet nem chamadas reais ao Gemini (P6).
- **RNF4.** `LLM_API_KEY` existe apenas no ambiente de execução de `Que-Bug-E-Esse/ai` e nunca é referenciada em `back` ou `front` (ADR 0005 A, ADR 0009 A).

## Fora de escopo

- Controle de papéis administrativo (RBAC / contas separadas de curador além do token de convidado da ADR 0007 A).
- Histórico/versionamento de múltiplas revisões de um mesmo rascunho.
- Geração em lote ou agendamento automático de novos enigmas.

## Divergências em relação aos diagramas

Pelo princípio P9, [casos-de-uso.md](../../docs/diagramas/casos-de-uso.md), [componentes.md](../../docs/diagramas/componentes.md), [implantacao.md](../../docs/diagramas/implantacao.md) e [README.md](../../docs/diagramas/README.md) são atualizados nesta mesma entrega com `POST /gerar`, `GeradorDeEnigmasHttp` e o fluxo de curadoria.

| ID | Diagrama / doc anterior | Esta spec | Origem |
|---|---|---|---|
| D1 | ADR 0009, S07/S08 e `docs/diagramas/README.md` citavam o nome provisório `POST /gerar-enigma` | Endpoint consolidado como `POST /gerar` em `Que-Bug-E-Esse/ai`, mantendo simetria com `POST /narrar` e `POST /verificar` | Contrato M4 (`ai` + `back`) |
| D2 | `casos-de-uso.md` herdava `UC8 -.-> «extend» UC10` do PDF original | Pela M19, rascunhos gerados em UC8 passam obrigatoriamente pela curadoria (UC9: `RASCUNHO -> PUBLICADO`) antes de ficarem jogáveis em UC10 ou no modo grupo | M19, S02 Q2 |

## Questões abertas

Nenhuma questão bloqueante em aberto: o ciclo de vida (`RASCUNHO -> PUBLICADO / ARQUIVADO`), a não jogabilidade de rascunhos (M19, S02 Q2), o provedor de IA (ADR 0005 A, ADR 0009 A) e a autenticação por token convidado (ADR 0007 A) estão fechados.

## Rastreabilidade

| CA | Arquivo de teste |
|---|---|
| 01, 02 | `Que-Bug-E-Esse/ai`: `tests/test_gerar.py` |
| 03, 11 (adaptador) | `Que-Bug-E-Esse/back`: `infrastructure/.../ai/GeradorDeEnigmasHttpTest.kt` |
| 04 a 10 | `Que-Bug-E-Esse/back`: `application/.../usecase/CuradoriaEnigmaTest.kt` e `interfaces/.../http/CuradoriaRoutesTest.kt` (ou `ApiHttpIntegracaoTest.kt`) |
| 11 (front) | `Que-Bug-E-Esse/front`: `src/test/App.test.tsx` |
| 12 | `Que-Bug-E-Esse/back`: `bootstrap/src/test/kotlin/com/quebugesse/arquitetura/ArquiteturaTest.kt` |
