# ADR 0002: Contrato da porta `Narrador`

| Campo | Valor |
|---|---|
| Status | Proposta |
| Proposta em | 26/10/2026 |
| Decidida em | — |
| Participantes | — |
| Substitui / substituída por | — |

## Contexto

Na sequência 2, `05 responder(pergunta, contexto)` devolve `06 Answer` na mesma chamada. A própria nota do diagrama diz "A validar com a equipe". Um narrador humano responde minutos depois, por WebSocket, e o diagrama de implantação reconhece isso ("O NarradorHumano roda no servidor: só repassa a resposta que o narrador digita aqui"), mas o contrato da porta não (melhoria M2). Além disso, o adaptador `NarradorHumano` mora na camada `interfaces`, mas implementa uma porta do domínio, o que inverte a direção das camadas (P7).

## Opções

### A. Porta só para a IA

A porta `Narrador` passa a ter só o adaptador de IA, com chamada síncrona. No grupo, a mensagem do narrador humano chama `ResponderPerguntaUseCase` direto. No solo, depois de `PerguntaFeita`, um caso de uso `ResponderComIA` consulta a porta e chama o mesmo `ResponderPergunta`. O adaptador `NarradorHumano` deixa de existir.

- A favor: contrato honesto para cada lado; some a anomalia de camadas; H4 fica contida no `ResponderComIA`.
- Contra: dois caminhos para a resposta (humano e IA), que convergem no mesmo caso de uso.

### B. Porta assíncrona para os dois

`solicitarResposta(pergunta)` sem retorno. Cada adaptador entrega a resposta depois, chamando `ResponderPergunta`: a IA quando o LLM responde, o humano quando digita.

- A favor: um único contrato.
- Contra: mais maquinaria (correlação, timeouts) até no caso da IA, que poderia ser síncrono; o adaptador humano continua em `interfaces`.

### C. Manter síncrona, como no diagrama

`NarradorHumano.responder` bloqueia até o narrador digitar, com timeout.

- A favor: segue o diagrama sem mudança.
- Contra: uma thread presa por pergunta; reinício do servidor perde a pergunta em andamento; timeout vira regra de jogo escondida.

## Recomendação

A. Mantém a IA simples e trata o humano como o que ele é: um usuário que envia um comando.

## Decisão da equipe

_Pendente._

## Impacto

| Documento | Se a escolha for A | Se for B | Se for C |
|---|---|---|---|
| [S03 spec](../../specs/003-casos-de-uso-da-partida/spec.md) | Mantém `ResponderComIA` | `ResponderComIA` vira adaptador que chama `ResponderPergunta` | Remove `ResponderComIA`; `FazerPergunta` espera a resposta |
| [S04 spec](../../specs/004-eventos-apos-persistir/spec.md) | Handler de `PerguntaFeita` no solo chama `ResponderComIA` | Handler chama `solicitarResposta` nos dois modos | Sem handler de resposta |
| S07 | Adaptador `NarradorIA` síncrono | Adaptador assíncrono | Adaptador síncrono |
| [Componentes](../diagramas/componentes.md) e [implantação](../diagramas/implantacao.md) | Remover `NarradorHumano` | Mover `NarradorHumano` para `infrastructure.messaging` | Sem mudança |
| [Sequência 2](../diagramas/sequencia-investigar-e-resolver.md) | Passos 04 a 11 só no solo; humano entra por `ResponderPergunta` | Passos 05 e 06 viram mensagem assíncrona | Sem mudança |
| [Casos de uso](../diagramas/casos-de-uso.md) | Nota "UC5 no solo executado pelo NarradorIA" passa a citar o caso de uso, não a porta | Sem mudança | Sem mudança |
