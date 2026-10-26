# ADR 0007: Identidade e autenticação

| Campo | Valor |
|---|---|
| Status | Proposta |
| Proposta em | 26/10/2026 |
| Decidida em | — |
| Participantes | — |
| Substitui / substituída por | — |

## Contexto

Todas as guardas da partida dependem de saber quem é o autor (`autor ∈ investigadores`, `autor = narrador`), mas nenhum diagrama mostra cadastro, login, sessão ou entrada em sala (melhoria M6). No terminal (S05) todos estão no mesmo dispositivo e isso não importa; no grupo online (S06) importa.

## Opções

### A. Convidado com apelido

O jogador digita um apelido; o servidor cria um `Player` e devolve um token assinado ligado a ele. Quem cria a partida recebe um código de sala para os outros entrarem (UC12, M11).

- A favor: sem senha e sem dados pessoais; entra-se em segundos.
- Contra: estatísticas (S12) ficam presas ao token; trocar de dispositivo é começar de novo.

### B. Conta com login

E-mail e senha, ou login social (Google, GitHub).

- A favor: estatísticas acompanham a pessoa em qualquer dispositivo.
- Contra: dados pessoais (LGPD), recuperação de senha, mais telas e mais escopo.

### C. Convidado agora, conta opcional depois

A na S06; B entra numa spec futura, que liga o convidado a uma conta.

- A favor: começa simples sem fechar a porta.
- Contra: a migração de convidado para conta precisa ser pensada desde já.

## Recomendação

A, ou C se a equipe quiser estatísticas persistentes no futuro.

## Decisão da equipe

_Pendente._

## Impacto

| Documento | Se a escolha for A ou C | Se for B |
|---|---|---|
| S06 | Rota de convidado, token, código de sala | Cadastro, login, sessão |
| S12 | Estatísticas por `Player` convidado | Estatísticas por conta |
| [S02 spec](../../specs/002-catalogo-e-jogador/spec.md) | `Player.nome` é o apelido | `Player` ganha referência à conta |
| [Componentes](../diagramas/componentes.md) e [implantação](../diagramas/implantacao.md) | Acrescentar emissão e verificação de token | Acrescentar autenticação e armazenamento de credenciais |
