# Pendências

O que depende da equipe ou de alguém fora do código: decisões, aprovações, questões das specs e configurações manuais. Quem fecha um item atualiza a linha aqui no mesmo PR (passo 4 do [fluxo das ADRs](adr/README.md)).

Situação em 2026-11-24: ADRs 0001 a 0009 aceitas pela opção mais simples, Constituição 1.0.0 aprovada, marco M1 (S00 a S05) concluído em `Que-Bug-E-Esse/back#1` (`feat/marco-m1`, 208 testes, 97,64% Kover) com entregas iniciais em `Que-Bug-E-Esse/ai#1` (70 testes) e `Que-Bug-E-Esse/front#1` (10 testes + build), marcos M2 e M3 concluídos em `Que-Bug-E-Esse/back#2` (`feat/api-ktor-ia-sqlite` — S06, S07/S08 e S09, 246 testes, 97,35% Kover) e `Que-Bug-E-Esse/front#2` (`feat/figma-ui` — S10 telas 01–07 do Figma, 15 testes + verificação visual 1280/375 px), e documentação SDD do marco M4 fechada (specs S11 e S12 aprovadas e diagramas Mermaid sincronizados em [docs/diagramas/](diagramas/README.md)); restam concluir/mesclar as PRs empilhadas do M4 e as tarefas manuais de infraestrutura/repositório.

## Decisões da equipe (ADRs)

Todas as ADRs 0001 a 0009 foram aceitas em 2026-11-24 pela opção mais simples de implementar (participantes: Diogo, decisão simplificada). As issues #1 a #8 no GitHub permanecem listadas para fechamento manual quando a equipe sincronizar o remoto.

| ADR | Pergunta | Status | Decisão | Quem usa / impacta | Issue |
|---|---|---|---|---|---|
| [0001](adr/0001-veredito-no-modo-grupo.md) | Quem dá o veredito no modo grupo? | Aceita | Opção A: narrador humano no grupo (`JulgarProposta`), IA só no solo (`AvaliarComIA`) | S01 Q3 (emenda 1 da S03), S03, S04, S08 | [#1](https://github.com/Dtma20/que-bug-e-esse/issues/1) |
| [0002](adr/0002-contrato-da-porta-narrador.md) | A porta `Narrador` é síncrona? | Aceita | Opção A: porta síncrona só para a IA; humano responde direto pelo caso de uso `ResponderPergunta` | S03, S04, S07 | [#2](https://github.com/Dtma20/que-bug-e-esse/issues/2) |
| [0003](adr/0003-framework-web.md) | Ktor ou Spring Boot? | Aceita | Opção A: Ktor (REST + WebSocket nativo, composição explícita no `bootstrap`) | S06 | [#3](https://github.com/Dtma20/que-bug-e-esse/issues/3) |
| [0004](adr/0004-banco-de-dados.md) | Qual banco? | Aceita | Opção B: SQLite em arquivo com Flyway, JDBC, modo WAL e bloqueio otimista | S04 Q5 (outbox na S09), S09 | [#4](https://github.com/Dtma20/que-bug-e-esse/issues/4) |
| [0005](adr/0005-provedor-de-llm.md) | Qual provedor de LLM e com que orçamento? | Aceita | Opção A: API comercial com saída estruturada (Gemini `gemini-2.5-flash`), teto US$ 10/mês, 10.000 tokens/partida e `LLM_API_KEY` fora do repositório | S07, S08, S11 | [#5](https://github.com/Dtma20/que-bug-e-esse/issues/5) |
| [0006](adr/0006-modo-offline.md) | O que é o modo "offline"? | Aceita | Opção A: um dispositivo, todos juntos, sem rede ("mesmo dispositivo") | S05, UC1 | [#6](https://github.com/Dtma20/que-bug-e-esse/issues/6) |
| [0007](adr/0007-identidade-e-autenticacao.md) | Como identificar e autenticar jogadores? | Aceita | Opção A: convidado com apelido, código de sala e token assinado por sessão | S03 Q7, S06, S12 | [#7](https://github.com/Dtma20/que-bug-e-esse/issues/7) |
| [0008](adr/0008-stack-do-front-end.md) | Qual stack de front-end? | Aceita | Opção A: SPA mínima em TypeScript + Vite + React (`fetch` e `WebSocket` nativos) | S10 | [#8](https://github.com/Dtma20/que-bug-e-esse/issues/8) |
| [0009](adr/0009-divisao-de-repositorios.md) | Como dividir a arquitetura nos três repositórios da organização? | Aceita | Opção A: `back` (Kotlin/JVM núcleo + API + SDD), `ai` (HTTP JSON stateless `/narrar` e `/verificar`, P3/H4), `front` (SPA mínima) | S00, S06, S07, S08, S10, S11 | — |

## Aprovações e andamento

| Documento | Status | Plano e tarefas | Observação |
|---|---|---|---|
| [Constituição](constitution.md) 1.0.0 | Aprovada | — | Mudança depois de aprovada só por ADR |
| [S00 Fundação](../specs/000-fundacao/spec.md) | Concluída | Escritos | Implementada em `Que-Bug-E-Esse/back#1` (`feat/marco-m1`) |
| [S01 Domínio da partida](../specs/001-dominio-partida/spec.md) | Concluída | Escritos | Implementada com emendas 1 e 2 da S03 em `Que-Bug-E-Esse/back#1` (`feat/marco-m1`) |
| [S02 Catálogo e jogador](../specs/002-catalogo-e-jogador/spec.md) | Concluída | Escritos | Implementada com `testFixtures` e seed em `Que-Bug-E-Esse/back#1` (`feat/marco-m1`) |
| [S03 Casos de uso da partida](../specs/003-casos-de-uso-da-partida/spec.md) | Concluída | Escritos | Implementada seguindo ADRs 0001 A, 0002 A e 0007 A em `Que-Bug-E-Esse/back#1` (`feat/marco-m1`) |
| [S04 Eventos depois de persistir](../specs/004-eventos-apos-persistir/spec.md) | Concluída | Escritos | Implementada com emendas 1 e 2 à S03 em `Que-Bug-E-Esse/back#1` (`feat/marco-m1`); outbox SQLite na S09 |
| [S05 CLI local](../specs/005-cli-local/spec.md) | Concluída | Escritos | Fecha o marco M1 (ADR 0006 A) em `Que-Bug-E-Esse/back#1` (`feat/marco-m1`) |
| [S06 API Ktor, salas e identidade](../specs/006-api-ktor-salas/spec.md) | Concluída | — | Implementada em `Que-Bug-E-Esse/back#2` (`feat/api-ktor-ia-sqlite`): REST + WebSocket, códigos de sala (UC12), tokens HMAC (ADR 0007 A), sigilo P3 |
| [S07/S08 Integração com o serviço de IA](../specs/007-008-integracao-ia/spec.md) | Concluída | — | Implementada em `Que-Bug-E-Esse/back#2` (`feat/api-ktor-ia-sqlite`): `NarradorIaHttp` e `VerificadorDeSolucaoHttp` para `Que-Bug-E-Esse/ai#1` (`/narrar`, `/verificar`), H4, P3 e M7 |
| [S09 Persistência em SQLite](../specs/009-persistencia-sqlite/spec.md) | Concluída | — | Implementada em `Que-Bug-E-Esse/back#2` (`feat/api-ktor-ia-sqlite`): SQLite WAL, Flyway, bloqueio otimista `versao` (M8), `eventos_outbox` (H6) e `testFixtures` |
| S10 Front-end SPA (Figma) | Concluída | — | Implementada em `Que-Bug-E-Esse/front#1` e `Que-Bug-E-Esse/front#2` (`feat/figma-ui`, telas 01–07 + 24 ícones SVG, ADR 0008 A, P3) |
| [S11 Geração de enigmas por IA e curadoria](../specs/011-geracao-e-curadoria-ia/spec.md) | Aprovada | — | Marco M4 (UC8/UC9, `POST /gerar`, `GeradorDeEnigmasHttp`, `/api/curadoria/enigmas/*`, M19, P3/H4): implementação em PRs M4 (`ai` `feat/s11-gerar-enigma`, `back` `feat/m4-curadoria-estatisticas`, `front` `feat/m4-curadoria-e2e`) |
| [S12 Histórico e estatísticas do jogador](../specs/012-historico-e-estatisticas/spec.md) | Aprovada | — | Marco M4 (UC7, M17 `AoEncerrarPartidaAtualizarEstatisticas` em `PartidaResolvida`/`PartidaAbandonada`, SQLite + `/api/perfil` e `/api/historico`): implementação em `back` `feat/m4-curadoria-estatisticas` |

Implementação (cada spec marca a sua no PR que a conclui):

- [x] S00 implementada (`Que-Bug-E-Esse/back#1`, branch `feat/marco-m1`)
- [x] S01 implementada (`Que-Bug-E-Esse/back#1`, branch `feat/marco-m1`)
- [x] S02 implementada (`Que-Bug-E-Esse/back#1`, branch `feat/marco-m1`)
- [x] S03 implementada (`Que-Bug-E-Esse/back#1`, branch `feat/marco-m1`)
- [x] S04 implementada (`Que-Bug-E-Esse/back#1`, branch `feat/marco-m1`)
- [x] S05 implementada (marco M1: jogo inteiro no terminal — `Que-Bug-E-Esse/back#1`, branch `feat/marco-m1`; serviço de IA e SPA inicial também abertos em `Que-Bug-E-Esse/ai#1` e `Que-Bug-E-Esse/front#1`)
- [x] S06 implementada (`Que-Bug-E-Esse/back#2`, branch `feat/api-ktor-ia-sqlite` — API Ktor REST + WebSocket, salas e tokens HMAC)
- [x] S07/S08 implementada (`Que-Bug-E-Esse/back#2`, branch `feat/api-ktor-ia-sqlite` — adaptadores HTTP `NarradorIaHttp` e `VerificadorDeSolucaoHttp` contra `Que-Bug-E-Esse/ai#1`)
- [x] S09 implementada (`Que-Bug-E-Esse/back#2`, branch `feat/api-ktor-ia-sqlite` — persistência SQLite + WAL + Flyway + bloqueio otimista + outbox)
- [x] S10 implementada (`Que-Bug-E-Esse/front#1` e `Que-Bug-E-Esse/front#2`, branch `feat/figma-ui` — SPA mínima + adaptação Figma telas 01–07)
- [ ] S11 mesclada (spec aprovada; em validação/PR no marco M4 em `ai` `feat/s11-gerar-enigma`, `back` `feat/m4-curadoria-estatisticas` e `front` `feat/m4-curadoria-e2e`)
- [ ] S12 mesclada (spec aprovada; em validação/PR no marco M4 em `back` `feat/m4-curadoria-estatisticas`)

## Questões das specs (resolvidas)

Todas as questões Q das specs S00 a S05 foram resolvidas em 2026-11-24 confirmando a opção mais simples (compatível com as ADRs 0001–0009 e o ambiente local).

### [S00](../specs/000-fundacao/spec.md)

| Q | Pergunta | Status | Decisão |
|---|---|---|---|
| Q1 | `interfaces` pode importar tipos do domínio? | Resolvida | Não; a aplicação expõe DTOs e códigos de erro próprios |
| Q2 | Konsist ou ArchUnit? | Resolvida | Konsist |
| Q3 | Onde roda a CI? | Resolvida | GitHub Actions |
| Q4 | Formatador ou lint? | Resolvida | Nenhum na S00; entra por ADR se a equipe quiser |
| Q5 | Qual JDK? | Resolvida | Temurin 25 LTS (instalado localmente com Gradle 9.8.2) |

### [S01](../specs/001-dominio-partida/spec.md)

| Q | Pergunta | Status | Decisão |
|---|---|---|---|
| Q1 | Quem pode abandonar (H2)? | Resolvida | Narrador sempre; investigador só no `SOLO` |
| Q2 | Quem cancela pergunta ou retira proposta? | Resolvida | O autor do item ou o narrador |
| Q3 | Quem aplica o veredito no grupo? | Resolvida | Só o narrador no grupo e o jogador IA no solo (ADR 0001 A; emenda 1 da S03, CA-33) |
| Q4 | Limite de texto | Resolvida | 1 a 500 code points depois de aparar |
| Q5 | Regras extras de H5 | Resolvida | Narrador fora dos investigadores, sem repetidos, `SOLO` com exatamente um |

### [S02](../specs/002-catalogo-e-jogador/spec.md)

| Q | Pergunta | Status | Decisão |
|---|---|---|---|
| Q1 | Limites de texto do enigma | Resolvida | Título de 1 a 100, enunciado e solução de 1 a 2000 code points |
| Q2 | O solo pode jogar enigma não revisado (M19)? | Resolvida | Não; só `PUBLICADO` é elegível |
| Q3 | O que é "já jogado pelo mesmo grupo" (M18)? | Resolvida | O domínio recebe o conjunto pronto; a S03 calcula (S03 Q5) |
| Q4 | Limite do nome do jogador | Resolvida | 1 a 40 code points; nomes repetidos permitidos |
| Q5 | Forma das tags | Resolvida | Aparadas, minúsculas, sem vazias nem repetidas |
| Q6 | Conteúdo do seed | Resolvida | Os cinco enigmas e o jogador IA de [seed.md](enigmas/seed.md) confirmados como seed inicial do M1 |

### [S03](../specs/003-casos-de-uso-da-partida/spec.md)

| Q | Pergunta | Status | Decisão |
|---|---|---|---|
| Q1 | Quem dá o veredito no grupo? | Resolvida | ADR 0001 A: `JulgarProposta` pelo narrador; `AvaliarComIA` só no solo |
| Q2 | A porta `Narrador` é síncrona? | Resolvida | ADR 0002 A: síncrona só para a IA |
| Q3 | O investigador vê a solução depois de abandonar? | Resolvida | Sim, em qualquer estado terminal |
| Q4 | Os testes de `application` podem usar os repositórios em memória? | Resolvida | Sim, como dependência de teste |
| Q5 | Como calcular "já jogados"? | Resolvida | Enigmas de partidas, em qualquer status, em que algum investigador da partida nova foi investigador |
| Q6 | Quem inicia no solo (M12)? | Resolvida | O próprio investigador |
| Q7 | Como um jogador passa a existir antes da S06? | Resolvida | `RegistrarJogador(nome)` cria um convidado humano, sem senha (ADR 0007 A) |

### [S04](../specs/004-eventos-apos-persistir/spec.md)

| Q | Pergunta | Status | Decisão |
|---|---|---|---|
| Q1 | O despacho é síncrono ou assíncrono? | Resolvida | Síncrono, uma fila por thread |
| Q2 | Um ouvinte que falha é chamado de novo? | Resolvida | Não; a falha é registrada e a entrega segue |
| Q3 | A visão de retorno é a de antes ou a de depois dos ouvintes? | Resolvida | A de depois: o caso de uso relê a partida |
| Q4 | Quem reage a `PerguntaFeita` no grupo? | Resolvida | ADR 0002 A: ninguém nesta spec |
| Q5 | E se o processo cair entre `salvar` e `publicar`? | Resolvida | O evento se perde em memória; o outbox em tabela SQLite entra na S09 (ADR 0004 B) |

### [S05](../specs/005-cli-local/spec.md)

| Q | Pergunta | Status | Decisão |
|---|---|---|---|
| Q1 | O que é o modo "offline"? | Resolvida | ADR 0006 A: o grupo num só dispositivo |
| Q2 | A CLI joga o solo? | Resolvida | Não até a S07 |
| Q3 | Como o narrador consulta a solução numa tela compartilhada? | Resolvida | Só por `solucao`, com aviso e confirmação |
| Q4 | Como se decide quem está digitando? | Resolvida | Vez explícita, com `vez <nome>` |
| Q5 | O mesmo nome em duas partidas é o mesmo jogador? | Resolvida | Sim, na sessão do programa, pelo nome normalizado |
| Q6 | Sair no meio da partida a registra como abandonada? | Resolvida | Não |

## Melhorias decididas (M10–M24)

Melhorias do [planejamento](../planejamento-sdd.md) (seção 7) que aguardavam decisão, fechadas em 2026-11-24 pela opção mais simples:

| M | Assunto | Onde está | Status | Decisão |
|---|---|---|---|---|
| M10 | Caso de uso "Abandonar partida" e quem pode abandonar | S01 Q1, S03 (`Abandonar`) | Resolvida | Confirmada H2 (narrador sempre; investigador só no `SOLO`); UC11 aplicado em [casos-de-uso.md](diagramas/casos-de-uso.md) |
| M12 | Quem inicia no solo | S03 Q6 | Resolvida | Confirmado: o próprio investigador inicia no solo; associação Investigador–UC3 aplicada em [casos-de-uso.md](diagramas/casos-de-uso.md) |
| M13 | `«extend»` entre UC7 e UC6 | Diagrama de casos de uso | Resolvida | Relação `«extend»` entre UC7 e UC6 removida em [casos-de-uso.md](diagramas/casos-de-uso.md) |
| M15 | Acesso SSH sem adaptador | S05 D1 e D2 | Resolvida | Confirmado: não haverá acesso SSH em produção; apenas CLI local (`interfaces.cli`) |
| M18 | Critério de seleção e "já jogados" | S02 Q3, S03 Q5, S05 Q5 | Resolvida | Confirmada a regra provisória (filtro por categoria/dificuldade, excluindo enigmas já jogados por algum investigador da partida nova) |
| M22 | Veredito só booleano | S01, S08, ADR 0009 | Resolvida | Manter veredito estritamente booleano (`CORRETA` / `INCORRETA`), sem retorno "quase", mantendo domínio e `POST /verificar` simples |
| M23 | Componentes e implantação divergem no acesso do front-end | Diagramas, ADR 0009 | Resolvida | Padronizar: `front` acessa exclusivamente `back` (via proxy HTTPS/WSS em produção ou direto ao Ktor em desenvolvimento), nunca o serviço `ai` |
| M24 | Sem limites nem pontuação | Fora de escopo da S03/S12 | Resolvida | Não adicionar limites de perguntas/tempo nem sistema de pontuação no MVP; S12 mantém apenas contadores simples de partidas e vitórias |

## Tarefas manuais

- [ ] **Mesclar as PRs empilhadas** dos marcos M1 a M4 na ordem de base em `Que-Bug-E-Esse/back` (`#1` → `#2` → M4), `Que-Bug-E-Esse/ai` (`#1` → M4), `Que-Bug-E-Esse/front` (`#1` → `#2` → M4) e `CLP` (sem push direto em `main`).
- [ ] **Proteger a `main`** no GitHub nos repositórios da organização (`Que-Bug-E-Esse/back`, `front`, `ai`), exigindo o check da CI após o merge das PRs.
- [ ] **Escolher a licença.** Os repositórios públicos precisam de um arquivo `LICENSE` para definir permissões de reutilização.
- [ ] **Fechar as issues #1 a #8** no GitHub de `CLP` referenciando o commit de aceitação das ADRs 0001–0009.
- [ ] **Gerar os PDFs** a partir das fontes Mermaid de [docs/diagramas/](diagramas/README.md), quando a equipe quiser atualizar os originais de 14/10/2026 com as decisões das ADRs 0001–0009 e M10–M24.
- [ ] **Convidar os colaboradores** para os repositórios da organização `Que-Bug-E-Esse`.
- [ ] **Configurar `LLM_API_KEY` real para produção** (ADR 0005 A / ADR 0009): criar a chave da Google Gemini API e guardá-la fora do repositório, apenas no ambiente de execução do serviço `Que-Bug-E-Esse/ai` (`LLM_PROVIDER=gemini`).

## Histórico

| Data | Mudança |
|---|---|
| 26/10/2026 | Primeira versão: ADRs 0001 a 0008, questões da S00 à S05, melhorias sem dono e tarefas manuais |
| 26/10/2026 | Issues #1 a #8 abertas, uma por ADR |
| 2026-11-24 | ADRs 0001 a 0008 aceitas, ADR 0009 criada e aceita, Constituição 1.0.0 e specs S00 a S05 aprovadas, questões Q e melhorias M10–M24 resolvidas pela opção mais simples |
| 2026-11-24 | Planos (`plan.md`) e tarefas (`tasks.md`) das specs S03, S04 e S05 escritos |
| 2026-11-24 | Marco M1 (S00–S05) concluído em `Que-Bug-E-Esse/back#1` (`feat/marco-m1`), entregas iniciais em `Que-Bug-E-Esse/ai#1` e `Que-Bug-E-Esse/front#1`, diagramas Mermaid sincronizados e specs S06, S07/S08 e S09 aprovadas |
| 2026-11-24 | Marcos M2 e M3 concluídos (`Que-Bug-E-Esse/back#2` S06–S09 e `Que-Bug-E-Esse/front#2` S10 Figma); fechamento documental do marco M4 com specs S11 e S12 aprovadas e diagramas Mermaid atualizados (P9) |
