# ADR 0005: Provedor de LLM, custo e saída estruturada

| Campo | Valor |
|---|---|
| Status | Proposta |
| Proposta em | 26/10/2026 |
| Decidida em | — |
| Participantes | — |
| Substitui / substituída por | — |

## Contexto

Três portas dependem de um LLM: `Narrador` no solo (S07), `VerificadorDeSolucao` no solo (S08) e `GeradorDeEnigmas` (S11). O diagrama de componentes só diz "API de LLM". Requisitos que já vêm dos diagramas e das melhorias:

- resposta restrita a `SIM`, `NAO` ou `IRRELEVANTE`, nunca texto livre para o jogador (M7);
- pergunta do jogador tratada como dado no prompt, para resistir a injeção (M7);
- custo previsível por partida;
- a chave de API fica fora do repositório (público).

## Opções

### A. API comercial com saída estruturada

Um provedor que ofereça JSON Schema ou tool calling. A escolha entre provedores sai da bateria de perguntas de referência e adversariais da S07, medindo acerto, custo e latência.

- A favor: melhor qualidade; saída estruturada garantida pela API.
- Contra: custo por chamada; depende de internet e de conta com cartão (verificar cotas para estudantes).

### B. Modelo local (Ollama ou similar)

- A favor: sem custo por chamada; funciona sem internet (ver ADR 0006).
- Contra: qualidade menor em pensamento lateral; exige máquina com GPU ou bastante memória no servidor.

## Recomendação

A, com o provedor escolhido pelos números da bateria da S07, e um teto de tokens por partida definido junto. O fake determinístico (P6) continua sendo o padrão nos testes e na CI.

## Decisão da equipe

_Pendente._ Além da opção, registrar: provedor e modelo, orçamento mensal, teto de tokens por partida e onde a chave fica guardada (variável de ambiente do servidor; segredo do GitHub só se algum job precisar).

## Impacto

| Documento | Se a escolha for A | Se for B |
|---|---|---|
| S07, S08, S11 | Adaptadores HTTP para o provedor | Adaptadores para o servidor local |
| [Implantação](../diagramas/implantacao.md) | Nomear o provedor | Trocar "API de LLM (serviço externo)" por um nó local |
| [Pendências](../pendencias.md) | Criar a conta e guardar a chave | Provisionar a máquina |
