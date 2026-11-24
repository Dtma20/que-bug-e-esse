# Spec 009: Persistência em SQLite, bloqueio otimista e outbox

| Campo | Valor |
|---|---|
| ID | S09 |
| Status | Aprovada |
| Depende de | S01 (`GameSession.reconstituir`), S02 (`EnigmaRepository`, `PlayerRepository`, `SeedInicial`, `testFixtures` de contrato), S03 (`GameSessionRepository`, `enigmasJogadosPor`, `testFixtures` de contrato), S04 (`PublicadorDeEventos`, H6) |
| Casos de uso | Todos os casos de uso de partida, catálogo e jogador (persistência durável transparente ao domínio e à aplicação) |
| Diagramas de origem | [implantacao.md](../../docs/diagramas/implantacao.md), [componentes.md](../../docs/diagramas/componentes.md), [pacotes.md](../../docs/diagramas/pacotes.md) |
| Decisões que assume | [ADR 0004](../../docs/adr/0004-banco-de-dados.md) opção B (SQLite em arquivo com Flyway, JDBC, modo WAL `busy_timeout=5000`, bloqueio otimista por `versao` e tabela de outbox na mesma transação), [ADR 0009](../../docs/adr/0009-divisao-de-repositorios.md) opção A, ambas `Aceita` |
| Melhorias incorporadas | M8 (concorrência entre investigadores via bloqueio otimista `versao`), M9 / H6 (outbox transacional `eventos_outbox`) do [planejamento](../../planejamento-sdd.md) |
| Plano | Plano e tarefas: ainda não escritos |

## Contexto

Até o marco M1 (S00–S05), todos os repositórios residem em memória (`infrastructure.persistence.memoria`), o que perde partidas ao reiniciar o processo e não garante atomicidade entre gravação e eventos (S04 Q5) nem proteção contra escritas concorrentes entre dois investigadores no grupo online (M8).

Esta spec define os adaptadores SQLite em `com.quebugesse.infrastructure.persistence.sqlite` conforme a ADR 0004 B: migrações versionadas com Flyway, configuração WAL com `busy_timeout=5000`, bloqueio otimista por coluna `versao` em `game_sessions`, gravação atômica de eventos na tabela `eventos_outbox` (H6) e execução obrigatória das mesmas suítes de contrato (`testFixtures`) criadas na S02 e S03 — garantindo paridade comportamental sem tocar no domínio (P1, P6).

## Histórias

- **US1.** Como equipe, quero que partidas, enigmas e jogadores sobrevivam a reinícios do servidor em um arquivo SQLite único com esquema migrado pelo Flyway (ADR 0004 B).
- **US2.** Como investigador em grupo online, quero que duas perguntas simultâneas nunca sobrescrevam uma à outra nem violem a invariante de no máximo uma pergunta aberta (M8, I1).
- **US3.** Como sistema, quero que eventos de domínio de uma partida sejam gravados na tabela `eventos_outbox` na mesma transação que salva o agregado, garantindo H6 mesmo em caso de queda entre salvar e publicar (H6, M9, S04 Q5).
- **US4.** Como desenvolvedor, quero provar paridade entre os repositórios em memória e SQLite rodando exatamente as mesmas suítes de contrato (`testFixtures`), sem precisar de Docker nem servidor externo (P6, ADR 0004 B).

## Glossário

| Termo | Significado |
|---|---|
| Modo WAL | Write-Ahead Logging do SQLite (`PRAGMA journal_mode=WAL`), permitindo leituras concorrentes com escrita e reduzindo contenção com `PRAGMA busy_timeout=5000` (ADR 0004 B) |
| Bloqueio otimista (`versao`) | Controle de concorrência por contador inteiro monotonicamente crescente em `game_sessions`; atualização exige `WHERE id = ? AND versao = ?` (novo, M8) |
| `ConflitoDeConcorrenciaException` | Exceção lançada quando `UPDATE` afeta 0 linhas por divergência de `versao`; traduzida na aplicação/API sem sobrescrever a escrita concorrente (novo, M8) |
| `eventos_outbox` | Tabela transacional que registra cada evento de domínio junto com o `salvar` da partida e marca `publicado_em` após entrega bem-sucedida (novo, H6) |

## Regras

| ID | Regra |
|---|---|
| DB1 | Configuração SQLite obrigatória ao abrir o banco: `PRAGMA journal_mode=WAL;`, `PRAGMA busy_timeout=5000;` e `PRAGMA foreign_keys=ON;`. Caminho do arquivo configurável via `DB_PATH` (ou URL `jdbc:sqlite:<caminho>`), criando o diretório pai se necessário |
| DB2 | Esquema gerenciado exclusivamente por migrações Flyway em `infrastructure/src/main/resources/db/migration/` (começando por `V1__inicial.sql`), contendo as tabelas `players`, `enigmas`, `game_sessions` (com `versao INTEGER NOT NULL`) e `eventos_outbox` (`id`, `partida_id`, `tipo`, `payload_json`, `ocorrido_em`, `publicado_em`). Nenhum DDL manual solto no código dos repositórios |
| DB3 | Paridade de contrato (`testFixtures`): `EnigmaRepositorySqlite`, `PlayerRepositorySqlite` e `GameSessionRepositorySqlite` implementam as mesmas portas (`domain.*.port`) e passam nas mesmas suítes de contrato (`EnigmaRepositoryContrato`, `PlayerRepositoryContrato`, `GameSessionRepositoryContrato`) que os repositórios em memória, incluindo `buscar`, `salvar`, isolamento de cópias (RP1–RP3), `publicados()` (RP4) e `enigmasJogadosPor` |
| DB4 | Reconstrução fiel (P1, P2): `Enigma.reconstituir`, `Player.reconstituir` e `GameSession.reconstituir` são usados ao ler do SQLite, preservando ordem de `perguntas` e `tentativas`, `tags`, `status`, `modo`, `criadaEm`, `resolvidaEm`, sem gerar eventos pendentes na leitura |
| DB5 | Bloqueio otimista em `game_sessions` (M8): inserção grava `versao = 1`; toda atualização rastreia a versão carregada na leitura e executa `UPDATE game_sessions SET ..., versao = versao + 1 WHERE id = ? AND versao = ?`. Se `rowsUpdated == 0`, lança `ConflitoDeConcorrenciaException` e faz rollback, preservando o estado gravado pela transação vencedora |
| DB6 | Outbox transacional (H6, M9): ao persistir uma partida acompanhada de seus eventos de domínio, o `INSERT`/`UPDATE` em `game_sessions` e os `INSERT`s em `eventos_outbox` (com `publicado_em = NULL`) executam na mesma transação SQLite. Em falha/rollback, nenhuma linha entra em `eventos_outbox`. Após despacho, os registros são marcados com `publicado_em`; eventos pendentes (`publicado_em IS NULL`) podem ser listados em ordem FIFO para reenvio após reinício |
| DB7 | Seed idempotente (SD1–SD3): `SeedInicial.carregar` sobre os repositórios SQLite é idempotente entre reinícios (não duplica jogador IA nem enigmas e não sobrescreve um enigma arquivado manualmente) |
| DB8 | Domínio puro preservado (P1, P7): nenhuma anotação SQL/JDBC/Flyway nem campo `versao` vaza para as regras do domínio; o rastreamento de versão para bloqueio otimista fica encapsulado no adaptador SQLite (ou em metadado de infraestrutura) |

### Esquema inicial (`V1__inicial.sql`)

| Tabela | Colunas principais | Restrições |
|---|---|---|
| `players` | `id`, `nome`, `tipo`, `partidas_jogadas`, `partidas_resolvidas` | `PRIMARY KEY (id)` |
| `enigmas` | `id`, `titulo`, `categoria`, `dificuldade`, `enunciado`, `solucao_completa`, `tags_json`, `origem`, `status` | `PRIMARY KEY (id)` |
| `game_sessions` | `id`, `enigma_id`, `modo`, `narrador_id`, `investigadores_json`, `status`, `criada_em`, `resolvida_em`, `perguntas_json`, `tentativas_json`, `versao` | `PRIMARY KEY (id)`, `versao INTEGER NOT NULL DEFAULT 1` |
| `eventos_outbox` | `id` (`INTEGER PRIMARY KEY AUTOINCREMENT`), `partida_id`, `tipo`, `payload_json`, `ocorrido_em`, `publicado_em` | Índice em `(publicado_em, id)` para leitura ordenada de pendentes |

## Critérios de aceitação

Testes executados em `infrastructure` sobre arquivos SQLite temporários (`@TempDir` / `Files.createTempDirectory`), sem Docker nem servidor externo.

| CA | Regra | Dado | Quando | Então |
|---|---|---|---|---|
| 01 | DB1, DB2 | caminho de arquivo SQLite novo | inicializar conexão + rodar migrações Flyway | `PRAGMA journal_mode` retorna `wal`, `PRAGMA busy_timeout` retorna `5000`, `PRAGMA foreign_keys` retorna `1` e as 4 tabelas existem |
| 02 | DB3, DB4 | `EnigmaRepositorySqlite`, `PlayerRepositorySqlite` e `GameSessionRepositorySqlite` | executar as suítes `testFixtures` da S02 e S03 (`EnigmaRepositoryContrato`, `PlayerRepositoryContrato`, `GameSessionRepositoryContrato`) | todos os testes de contrato passam integralmente |
| 03 | DB4, DB7 | banco SQLite com seed, jogadores e partida com perguntas e tentativas salvos | fechar conexões, reabrir o mesmo arquivo SQLite, rodar `SeedInicial.carregar` novamente e consultar | dados sobrevivem intactos com mesma ordem de perguntas/tentativas e sem duplicar o seed; enigma arquivado continua `ARQUIVADO` |
| 04 | DB5, M8 | partida salva em `versao = 1`; duas leituras independentes `A` e `B` carregam a mesma partida | `A` faz pergunta 1 e salva (vai para `versao = 2`); em seguida `B` tenta salvar outra pergunta baseada na leitura antiga | `A` sucede (`versao = 2`); `B` falha com `ConflitoDeConcorrenciaException`; o banco preserva apenas a pergunta de `A` (invariante I1 preservada) |
| 05 | DB5, M8 | duas threads competindo para gravar comandos simultâneos na mesma partida | ambas executam `salvar` concorrentemente | exatamente uma vence e incrementa `versao`; a outra recebe `ConflitoDeConcorrenciaException` sem corromper o banco nem travar com `SQLITE_BUSY` |
| 06 | DB6, H6 | partida em andamento com evento `PerguntaFeita` | salvar partida com outbox na mesma transação | `game_sessions` atualizada e linha em `eventos_outbox` gravada com `publicado_em = NULL`; ao marcar publicada, `publicado_em` é preenchido e não retorna mais em pendentes |
| 07 | DB6, H6 | transação de salvamento forçada a falhar (ex.: conflito de `versao` ou erro de escrita) | tentar salvar partida com eventos | rollback completo: nem `game_sessions` nem `eventos_outbox` recebem alteração |
| 08 | DB8, P1, P7 | código de produção de todos os módulos | teste de arquitetura Konsist (`ArquiteturaTest`) | `domain` segue sem dependências externas; código SQLite/JDBC/Flyway restrito a `infrastructure.persistence.sqlite` e ligação no `bootstrap` |

## Requisitos não funcionais

- **RNF1.** Domínio puro (P1) e fronteira de camadas preservada (P7).
- **RNF2.** Cobertura de linhas de pelo menos 90% em `com.quebugesse.infrastructure.persistence.sqlite` (Kover).
- **RNF3.** Suíte de persistência SQLite roda em menos de 15 segundos numa máquina de desenvolvimento, sem Docker nem serviço externo (ADR 0004 B).
- **RNF4.** Os repositórios em memória continuam como padrão nos testes de domínio e aplicação (P6).

## Fora de escopo

- Migração para PostgreSQL ou replicação multi-instância (ADR 0004 B permite trocar apenas `infrastructure.persistence` no futuro).
- Broker de mensageria externo (RabbitMQ/Kafka) ou worker distribuído para consumo do outbox (ADR 0009 A).
- Backup automático em nuvem e criptografia em disco.

## Divergências em relação aos diagramas

Pelo princípio P9, [implantacao.md](../../docs/diagramas/implantacao.md), [componentes.md](../../docs/diagramas/componentes.md) e [pacotes.md](../../docs/diagramas/pacotes.md) já foram atualizados para SQLite + WAL + Flyway + outbox:

| ID | Diagrama hoje | Esta spec | Origem |
|---|---|---|---|
| D1 | [implantacao.md](../../docs/diagramas/implantacao.md) mostra o nó SQLite WAL + Flyway + outbox em alto nível | Detalha as quatro tabelas (`players`, `enigmas`, `game_sessions` com `versao` e `eventos_outbox`) e o rastreamento de versão encapsulado na infraestrutura | ADR 0004 B, M8, H6 |

## Questões abertas

Nenhuma questão bloqueante em aberto: a ADR 0004 B fechou SQLite em arquivo com Flyway, JDBC, WAL (`busy_timeout=5000`), bloqueio otimista `versao` e tabela de outbox.

## Rastreabilidade

| CA | Arquivo de teste previsto |
|---|---|
| 01, 03 | `infrastructure/.../persistence/sqlite/SqliteConfiguracaoEMigracaoTest.kt` |
| 02 | `infrastructure/.../persistence/sqlite/{Enigma,Player,GameSession}RepositorySqliteTest.kt` (executando `testFixtures` da S02/S03) |
| 04, 05 | `infrastructure/.../persistence/sqlite/BloqueioOtimistaSqliteTest.kt` |
| 06, 07 | `infrastructure/.../persistence/sqlite/EventosOutboxSqliteTest.kt` |
| 08 | `bootstrap/src/test/kotlin/com/quebugesse/arquitetura/ArquiteturaTest.kt` |
