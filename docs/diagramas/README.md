# Diagramas

Fontes Mermaid dos oito diagramas UML do projeto. Cada arquivo `.md` tem um bloco `mermaid`, que o GitHub desenha na própria página.

A regra vem do princípio P9 da [constituição](../constitution.md): "As fontes dos diagramas ficam em `docs/diagramas/` em Mermaid ou PlantUML; o PDF é gerado a partir delas. PR que muda modelo, estados, portas ou pacotes atualiza o diagrama no mesmo PR. Enquanto isso não acontece, a divergência fica listada na seção "Divergências" da spec."

**Estado atual (2026-11-24):** todos os oito diagramas Mermaid foram atualizados para refletir os marcos M1 a M3 concluídos (`Que-Bug-E-Esse/back#1`, `back#2`, `ai#1`, `front#1`, `front#2`), as ADRs 0001 a 0009 aceitas, as melhorias M1–M24 decididas e o marco M4 (specs [S11](../../specs/011-geracao-e-curadoria-ia/spec.md) e [S12](../../specs/012-historico-e-estatisticas/spec.md)). Os PDFs de 14/10/2026 em `diagramas/` permanecem como referência histórica.

## Fontes e PDFs

| Diagrama | Fonte | PDF de 14/10/2026 |
|---|---|---|
| Casos de uso | [casos-de-uso.md](casos-de-uso.md) | [diagrama_casos.pdf](../../diagramas/diagrama_casos.pdf) |
| Classes do domínio | [classes-dominio.md](classes-dominio.md) | [diagrama_classes.pdf](../../diagramas/diagrama_classes.pdf) |
| Componentes | [componentes.md](componentes.md) | [Diagrama_comp.pdf](../../diagramas/Diagrama_comp.pdf) |
| Implantação | [implantacao.md](implantacao.md) | [Diagrama_imp.pdf](../../diagramas/Diagrama_imp.pdf) |
| Pacotes | [pacotes.md](pacotes.md) | [diagrama_pacotes.pdf](../../diagramas/diagrama_pacotes.pdf) |
| Estados da partida | [estados-partida.md](estados-partida.md) | [diagrama_estados.pdf](../../diagramas/diagrama_estados.pdf) |
| Sequência 1/2: iniciar partida | [sequencia-iniciar-partida.md](sequencia-iniciar-partida.md) | [diagrama_seq_iniciar.pdf](../../diagramas/diagrama_seq_iniciar.pdf) |
| Sequência 2/2: investigar e resolver | [sequencia-investigar-e-resolver.md](sequencia-investigar-e-resolver.md) | [diagrama_seq_jogar.pdf](../../diagramas/diagrama_seq_jogar.pdf) |

Os PDFs em `diagramas/` são os originais da equipe e ficam como referência. Gerar PDFs novos a partir destas fontes é uma tarefa manual listada em [docs/pendencias.md](../pendencias.md).

## Adaptações de notação

O Mermaid não tem todos os símbolos da UML que os PDFs usam. As trocas abaixo não mudam o conteúdo.

| No PDF | Na fonte | Motivo |
|---|---|---|
| Restrições entre chaves: `{readOnly}`, `{não vazia}`, `{= resposta == null}` | Entre colchetes: `[readOnly]` | Chave fecha o corpo da classe no Mermaid |
| `List<PlayerRef>`, `Set<String>` | `List~PlayerRef~`, `Set~String~` | Sintaxe de genéricos do Mermaid |
| `fazerPergunta(...): Question` | `fazerPergunta(...) Question` | O Mermaid escreve o tipo de retorno depois dos parênteses e desenha os dois-pontos |
| Pasta de pacote | `namespace` nas classes; `subgraph` nos demais | |
| Pacote `domain.gamesession` (e os outros três do domínio) | `namespace domain_gamesession` | Ponto no nome do `namespace` dá erro de sintaxe no Mermaid 10 e no 11.0.0 (testado em 26/10/2026); o GitHub pode usar uma versão dessas |
| Ator (boneco) | Retângulo com «ator» | O `flowchart` não tem boneco |
| Elipse de caso de uso; fronteira do sistema | Nó `([...])`; `subgraph` | |
| Interface fornecida ou requerida (○) | Nó circular; linha cheia para quem fornece, seta tracejada para quem requer | |
| Nó «device» (cubo); nuvem | `subgraph` com «device» no título; elipse | |
| Notas, quadros de regras e legenda | Texto abaixo do diagrama, em "Notas atualizadas (P9)" | Notas dentro do desenho cruzavam as setas |
| Cores do PDF | Só o laranja do fluxo principal (casos de uso) e o fundo escuro dos estados terminais | |
| Números das mensagens (01, 02...) | No texto da mensagem, sem `autonumber` | Mantém a numeração legível citada nas specs |
| Criação de `GameSession` («create») | `create participant` | |

## Mudanças aplicadas em 2026-11-24 (S00–S12 e ADRs 0001–0009)

Todas as divergências registradas nas specs S00 a S12 e nas ADRs 0001 a 0009 foram incorporadas às fontes Mermaid acima:

- **Casos de uso:** UC1 atualizado para "mesmo dispositivo" (ADR 0006 A), UC11 Abandonar partida (M10), UC12 Entrar em partida (M11), Investigador ligado a UC3 no solo (M12), removido `«extend»` entre UC7 e UC6 (M13) e UC7 ampliado com estatísticas (S12), removido `UC8 -.-> «extend» UC10` em favor do fluxo de curadoria `UC8 -> UC9` (`RASCUNHO -> PUBLICADO`, M19, S11, `POST /gerar`), nota atualizada para `ResponderComIA`/`AvaliarComIA` e serviço `Que-Bug-E-Esse/ai` + Gemini.
- **Classes do domínio:** `agora: Instant` em todos os comandos (P1), `modo`, `fase`, `Tentativa` + `SituacaoTentativa`, `cancelarPergunta` / `retirarProposta` (M3), `aplicarVeredito(autor, veredito, agora)` (ADR 0001 A), `reconstituir`, `EventoDePartida : DomainEvent` (9 eventos, M20), `ConteudoEnigma` + `StatusEnigma` + `OrigemEnigma` + `CriterioDeSelecao` (S02), métodos puros `registrarPartidaResolvida()` e `registrarPartidaAbandonada()` em `Player` e `EstatisticasJogador` (S12, M17), `QuestionId`/`TentativaId` numerados.
- **Componentes:** módulo `bootstrap` (`Main` / `ServidorHttp` / `AplicacaoCli`, M16), remoção de `NarradorHumano` (ADR 0002 A), `VerificadorDeSolucao` só no solo (ADR 0001 A), portas em `domain.<agregado>.port` (`GeradorDeEnigmas` na S11) e `domain.shared.port` (`PublicadorDeEventos`), casos de uso de partida, salas/histórico/perfil (S06/S12) e curadoria (S11), ouvintes `AoFazerPerguntaNoSolo`, `AoProporSolucaoNoSolo` e `AoEncerrarPartidaAtualizarEstatisticas` (S12, M17), `GeradorDeEnigmasHttp` (S11), `Que-Bug-E-Esse/ai` (`/narrar`, `/verificar`, `/gerar`, `/health`) + Gemini, `Que-Bug-E-Esse/front` via proxy (M23), SQLite WAL e tokens HMAC (S06/S09).
- **Implantação:** remoção de SSH (M15), CLI local no mesmo processo (ADR 0006 A), `Que-Bug-E-Esse/front` (SPA TypeScript + Vite + React, S10/S11, ADR 0008 A), Ktor REST/WS (S06/S11/S12, ADR 0003 A), SQLite em arquivo WAL + Flyway + outbox + tabela `players` atualizada por evento (S09/S12, ADR 0004 B), `Que-Bug-E-Esse/ai` (Python + FastAPI, `/narrar`, `/verificar`, `/gerar`, `/health`, ADR 0009 A) → Google Gemini API `gemini-2.5-flash` (ADR 0005 A), tokens HMAC (ADR 0007 A).
- **Pacotes:** módulo `bootstrap`, `domain.<agregado>.port` e `domain.shared.port`, `application.ouvinte`, `interfaces.cli` / `http` / `ws`, `infrastructure.persistence.{memoria,seed,sqlite}`, `infrastructure.ai`, regras Konsist (P1/P7).
- **Estados da partida:** `agora` nos comandos, `cancelarPergunta` e `retirarProposta` (M3), guarda `[autor = narrador]` em `aplicarVeredito` (ADR 0001 A), falha H4 em `SolucaoEmAvaliacao`, H2 definida, H5 estendida e H6 recuperada (M9).
- **Sequências 1 e 2:** `publicados()` + `CriterioDeSelecao` + sorteio na aplicação (M18), narrador como `PlayerRef`, `salvar` → `publicar` → releitura (H6, EV5), visão por papel (`VisaoInvestigador`/`VisaoNarrador`, P3), caminho separado solo (ouvintes + `ResponderComIA`/`AvaliarComIA`) vs grupo (narrador humano direto em `ResponderPergunta`/`JulgarProposta`), `cancelarPergunta`/`retirarProposta` (M3).

## Pendências menores por diagrama

As pendências de S11 (`POST /gerar`, `GeradorDeEnigmasHttp`, fluxo de curadoria) e S12 (`Player.registrarPartidaResolvida`/`Abandonada` e `AoEncerrarPartidaAtualizarEstatisticas`) foram incorporadas em 2026-11-24. Permanecem apenas refinamentos opcionais de diagramas de sequência/modelo físico:

| Diagrama | Item | Refinamento opcional | Spec de origem |
|---|---|---|---|
| Sequência 1 | S06 D1 | Separar visualmente `CriarSalaGrupo` → `codigoSala` → UC12 `EntrarNaSala` → `IniciarSalaGrupo` no diagrama de sequência 1 | [S06](../../specs/006-api-ktor-salas/spec.md), M11 |
| Implantação | S09 D1 | Detalhar o modelo físico das tabelas (`players`, `enigmas`, `game_sessions` com `versao` e `eventos_outbox`) em diagrama dedicado, se a equipe quiser | [S09](../../specs/009-persistencia-sqlite/spec.md), ADR 0004 B |
