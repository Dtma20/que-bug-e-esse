# ADR 0004: Banco de dados

| Campo | Valor |
|---|---|
| Status | Aceita |
| Proposta em | 26/10/2026 |
| Decidida em | 2026-11-24 |
| Participantes | Diogo (decisão simplificada) |
| Substitui / substituída por | — |

## Contexto

O diagrama de implantação tem um "Banco de Dados" genérico, acessado por JDBC/TCP, que "persiste partidas, enigmas e jogadores". Até a S09, os repositórios ficam em memória (S02, S03). A escolha precisa suportar várias partidas ao mesmo tempo com bloqueio otimista (M8) e a publicação de eventos depois de salvar (H6, S04).

## Opções

### A. PostgreSQL com migrações Flyway

- A favor: robusto para escrita concorrente; JSONB guarda listas do agregado sem tabelas extras; Testcontainers roda o mesmo banco nos testes de contrato.
- Contra: precisa de um servidor (Docker no desenvolvimento).

### B. SQLite em arquivo com Flyway

- A favor: zero servidor; um arquivo só.
- Contra: escrita concorrente limitada, justamente o caso do grupo online (M8).

### C. MongoDB

- A favor: um agregado vira um documento, sem mapeamento objeto-relacional.
- Contra: transação e outbox exigem mais cuidado; a equipe talvez não conheça.

A forma de acesso (JDBC puro, Exposed ou jOOQ) fica para o plano da S09.

## Recomendação

A.

## Decisão da equipe

**Opção B aceita:** **SQLite em arquivo com migrações Flyway**, acessado via JDBC com modo WAL (`PRAGMA journal_mode=WAL; PRAGMA busy_timeout=5000`), bloqueio otimista por coluna `versao` (M8) e tabela de outbox na mesma transação (H6/S09). É a opção mais simples de implementar, testar e operar (zero servidor externo, sem exigência de Docker/Testcontainers nos testes) e mantém o mesmo contrato JDBC/Flyway da opção A, permitindo migrar para PostgreSQL apenas dentro de `infrastructure.persistence` se o projeto vier a escalar para múltiplas instâncias. Até a S09, seguem os repositórios em memória.

## Impacto

| Documento | Se a escolha for A ou B | Se for C |
|---|---|---|
| S09 | Migrações Flyway, repositórios JDBC, suíte de contrato da S02 rodando no banco | Repositórios com o driver do MongoDB |
| [S04 spec](../../specs/004-eventos-apos-persistir/spec.md) | Outbox numa tabela | Outbox numa coleção |
| [Implantação](../diagramas/implantacao.md) | Nomear o banco | Nomear o banco; trocar "JDBC / TCP" pelo protocolo do driver |
