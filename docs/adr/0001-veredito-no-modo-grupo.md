# ADR 0001: Quem dá o veredito no modo grupo

| Campo | Valor |
|---|---|
| Status | Proposta |
| Proposta em | 26/10/2026 |
| Decidida em | — |
| Participantes | — |
| Substitui / substituída por | — |

## Contexto

O diagrama de componentes traz a porta `VerificadorDeSolucao` com a nota "(fornecedor a decidir)". A sequência 2 (passos 15 a 18) manda toda proposta ao verificador, com `avaliar(texto, enigma.solucaoCompleta)`, sem distinguir o modo de jogo. Só que, no modo grupo, o narrador humano conhece a solução e é quem melhor julga uma proposta (melhoria M1).

Hoje a spec S01 segue o diagrama: `aplicarVeredito` não pergunta quem julgou (decisão provisória Q3). Esta ADR decide se isso continua assim.

## Opções

### A. Narrador humano no grupo, IA só no solo

No grupo, o narrador envia o veredito por um caso de uso próprio (`JulgarProposta`). A porta `VerificadorDeSolucao` só é usada no solo, com um adaptador de IA.

- A favor: quem julga conhece a solução; não gasta LLM no modo grupo; funciona como a resposta às perguntas, que o humano também dá.
- Contra: a proposta fica parada se o narrador sumir. A spec S01 já mitiga isso com `retirarProposta` (M3).

### B. IA julga nos dois modos

Toda proposta vai para o `VerificadorDeSolucao` de IA, como no diagrama.

- A favor: veredito uniforme e imediato; o narrador não precisa decidir nada.
- Contra: custo de LLM também no grupo; o narrador humano, que sabe a resposta, não pode corrigir um erro da IA; o grupo passa a depender da ADR 0005.

### C. IA sugere, narrador confirma

No grupo, a IA dá um parecer e o narrador aceita ou troca.

- A favor: ajuda narradores inseguros.
- Contra: duas etapas e um estado a mais ("sugestão pendente"), além do custo de LLM.

## Recomendação

A. É o arranjo mais simples, reaproveita o mesmo padrão das respostas (humano no grupo, IA no solo) e só depende de LLM onde não há humano.

## Decisão da equipe

_Pendente._

## Impacto

| Documento | Se a escolha for A | Se for B ou C |
|---|---|---|
| [S01 spec](../../specs/001-dominio-partida/spec.md) (Q3, CAs 10 a 12) | `aplicarVeredito` ganha `autor: PlayerRef` e a guarda `NAO_E_NARRADOR`; no solo o narrador é o jogador IA. Um CA novo cobre a rejeição | Nada muda no domínio |
| [S03 spec](../../specs/003-casos-de-uso-da-partida/spec.md) | Mantém `JulgarProposta` (grupo) e `AvaliarComIA` (solo) | Remove `JulgarProposta`; `AvaliarComIA` vale nos dois modos |
| S08 | Dois caminhos: humano no grupo, IA no solo | Só o caminho da IA |
| [Diagrama de componentes](../diagramas/componentes.md) | Tirar "(fornecedor a decidir)"; anotar "só no solo" | Tirar "(fornecedor a decidir)" |
| [Sequência 2](../diagramas/sequencia-investigar-e-resolver.md) | Passos 15 a 18 dentro de um `alt [modo solo]`, com o veredito do narrador no `alt [modo grupo]` | Sem mudança |
