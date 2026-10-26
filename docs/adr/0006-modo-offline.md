# ADR 0006: O que é o modo "offline"

| Campo | Valor |
|---|---|
| Status | Proposta |
| Proposta em | 26/10/2026 |
| Decidida em | — |
| Participantes | — |
| Substitui / substituída por | — |

## Contexto

O UC1 "Escolher modo de jogo" cita "solo / grupo online / offline", e nenhum outro diagrama volta ao offline (melhoria M14). A spec S01 já trata o offline como um `GRUPO` jogado num só dispositivo, sem mudança no domínio.

## Opções

### A. Um dispositivo, todos juntos

Narrador e investigadores se revezam no mesmo terminal ou na mesma tela, sem rede. A CLI da S05 já entrega isso.

- A favor: sai de graça com o marco M1.
- Contra: o narrador precisa esconder a solução dos demais na mesma tela.

### B. Sem internet, inclusive no solo

O jogo roda instalado, sem servidor, com o LLM local.

- A favor: joga em qualquer lugar.
- Contra: depende da opção B da ADR 0005 e de empacotar um app; muito escopo.

### C. Cortar o offline

Tirar "offline" do UC1.

- A favor: menos escopo.
- Contra: perde um modo que a CLI daria quase sem custo.

## Recomendação

A.

## Decisão da equipe

_Pendente._

## Impacto

| Documento | Se a escolha for A | Se for B | Se for C |
|---|---|---|---|
| [Casos de uso](../diagramas/casos-de-uso.md) (UC1) | Trocar "offline" por "mesmo dispositivo" | Manter | Remover "offline" |
| [S05 spec](../../specs/005-cli-local/spec.md) | Sem mudança | Acrescentar solo com LLM local | A CLI vira só ferramenta de desenvolvimento |
| S10 | Considerar uma tela "mesmo dispositivo" | Considerar app instalável | Nada |
