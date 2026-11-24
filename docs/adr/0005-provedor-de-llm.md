# ADR 0005: Provedor de LLM, custo e saída estruturada

| Campo | Valor |
|---|---|
| Status | Aceita |
| Proposta em | 26/10/2026 |
| Decidida em | 2026-11-24 |
| Participantes | Diogo (decisão simplificada) |
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

**Opção A aceita:** API comercial com saída estruturada (JSON Schema), encapsulada pelo serviço stateless `Que-Bug-E-Esse/ai` (ADR 0009).

- **Provedor e modelo padrão:** Google Gemini API (`gemini-2.5-flash`) com `response_schema` nativo, configurável por variável de ambiente `LLM_MODEL` (substituível caso a bateria de referência da S07 aponte modelo com melhor relação acerto/custo).
- **Orçamento mensal:** teto de US$ 10/mês (ou uso dentro da cota gratuita/estudantil).
- **Teto de tokens por partida:** máximo de 10.000 tokens totais por partida (com `max_output_tokens` limitado a 256 por chamada), imposto no serviço `ai`.
- **Guarda da chave:** variável de ambiente `LLM_API_KEY` apenas no servidor do serviço `ai` (fora do repositório; segredo no GitHub Actions somente se algum job manual rodar a bateria real). O fake determinístico (P6) permanece o padrão na CI e em todos os testes de domínio e aplicação.

## Impacto

| Documento | Se a escolha for A | Se for B |
|---|---|---|
| S07, S08, S11 | Adaptadores HTTP para o provedor | Adaptadores para o servidor local |
| [Implantação](../diagramas/implantacao.md) | Nomear o provedor | Trocar "API de LLM (serviço externo)" por um nó local |
| [Pendências](../pendencias.md) | Criar a conta e guardar a chave | Provisionar a máquina |
