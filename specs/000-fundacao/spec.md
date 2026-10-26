# Spec 000: Fundação do projeto

| Campo | Valor |
|---|---|
| ID | S00 |
| Status | Rascunho |
| Depende de | — |
| Casos de uso | — |
| Diagramas de origem | `diagrama_pacotes.pdf` (camadas e pacotes), `Diagrama_comp.pdf` (camadas) |
| Princípios | P1, P5 e P7 da [constituição](../../docs/constitution.md), mais a definição de pronto |
| Melhorias incorporadas | M16 (raiz de composição `bootstrap`) do [planejamento](../../planejamento-sdd.md) |
| Plano | [plan.md](plan.md) · Tarefas: [tasks.md](tasks.md) |

## Contexto

Toda spec seguinte precisa de um build que compile, teste e meça cobertura, e de uma CI que barre código que quebre as camadas. Esta spec cria esse esqueleto, ainda vazio: cinco módulos Gradle, Kotest, Kover, teste de arquitetura e CI no GitHub. Não há regra de jogo aqui.

## Histórias

- **US1.** Como desenvolvedor, quero clonar o repositório e rodar o build tendo só o JDK instalado, para começar a trabalhar no mesmo dia.
- **US2.** Como equipe, quero que todo PR rode o build na CI, para não juntar código quebrado na `main`.
- **US3.** Como equipe, quero que o build falhe quando alguém contrariar as camadas do diagrama de pacotes, para a arquitetura não se degradar em silêncio (P7).
- **US4.** Como equipe, quero medir cobertura por pacote, para aplicar a definição de pronto.

## Regras

| ID | Regra |
|---|---|
| F1 | Módulos Gradle `domain`, `application`, `infrastructure`, `interfaces` e `bootstrap`; o código de cada um fica em `com.quebugesse.<módulo>` |
| F2 | Dependências permitidas entre módulos: `application` → `domain`; `infrastructure` → `domain`; `interfaces` → `application`; `bootstrap` → todos. Nenhuma outra (**Q1**) |
| F3 | `domain` não declara dependência de produção além da biblioteca padrão do Kotlin (P1) |
| F4 | Versões de plugins e bibliotecas só no catálogo `gradle/libs.versions.toml`; a versão do JDK vem da toolchain (**Q5**) |
| F5 | O teste de arquitetura lê o código de produção de todos os módulos e falha quando um import contraria F2, P1 ou a regra entre agregados de P7 |
| F6 | Cada regra do teste de arquitetura é provada contra um exemplo de violação: com o exemplo, ela acusa a infração |
| F7 | A CI roda o build completo (compilação, testes, arquitetura e cobertura) em todo push na `main` e em todo PR (**Q3**) |

Os pacotes que o diagrama mostra dentro de cada camada (`usecase`, `dto`, `rest`, `ws`, `persistence`, `ai`, `messaging`) nascem com as specs que os usam, não aqui.

## Critérios de aceitação

| CA | Regra | Dado | Quando | Então |
|---|---|---|---|---|
| 01 | F4 | clone limpo, só com o JDK da toolchain instalado | `./gradlew build` | `BUILD SUCCESSFUL`, usando o Gradle baixado pelo wrapper |
| 02 | F1 | o projeto | `./gradlew projects` | lista exatamente `domain`, `application`, `infrastructure`, `interfaces` e `bootstrap` |
| 03 | F2, F5 | arquivo em `com.quebugesse.domain` importando `com.quebugesse.application`, e arquivo em `com.quebugesse.interfaces` importando `com.quebugesse.domain` | teste de arquitetura | uma infração para cada um; um import de `application` para `domain` não gera infração |
| 04 | P7, F5 | arquivo em `domain.gamesession` importando `domain.enigma` | teste de arquitetura | uma infração; import de `domain.shared` não gera infração |
| 05 | P1, F5 | arquivo do domínio com `Instant.now()` e outro importando uma biblioteca fora da lista | teste de arquitetura | uma infração por arquivo; imports de `kotlin.*`, `java.time.*`, `java.util.UUID` e do próprio domínio passam |
| 06 | F5 | código de produção do repositório | teste de arquitetura | nenhuma infração |
| 07 | F7 | PR com um teste falhando | CI | check vermelho; corrigido o teste, check verde |
| 08 | US4 | os cinco módulos | `./gradlew koverHtmlReport` na raiz | relatório único com as classes de todos os módulos |
| 09 | F6 | cada regra (CAs 03, 04 e 05) | teste de arquitetura contra o exemplo de violação | a regra acusa exatamente as infrações plantadas |

## Requisitos não funcionais

- **RNF1.** A CI termina em menos de 5 minutos enquanto o projeto tiver só o domínio e a aplicação.
- **RNF2.** Nenhum segredo no repositório (ele é público). A S00 não precisa de nenhum.

## Fora de escopo

- Framework web (ADR 0003), banco (ADR 0004), Docker, Nginx e deploy.
- Front-end (ADR 0008).
- Formatador e lint (**Q4**).
- Proteção da branch `main` no GitHub: configuração manual, listada em [docs/pendencias.md](../../docs/pendencias.md).

## Divergências em relação aos diagramas

| ID | Diagrama hoje | Esta spec | Origem |
|---|---|---|---|
| D1 | Quatro camadas, sem raiz de composição | Quinto módulo `bootstrap`, único que conhece todos e liga adaptadores às portas | M16, P7 |
| D2 | Setas «use» entre camadas, sem regra para imports externos | P1 vira regra verificável: o domínio só importa `kotlin.*`, `java.time.*`, `java.util.UUID` e a si mesmo | P1 |

## Questões abertas

| Q | Pergunta | Decisão provisória | CAs afetados |
|---|---|---|---|
| Q1 | `interfaces` pode importar tipos do domínio? | Não. Segue P7 ao pé da letra: a aplicação expõe DTOs e códigos de erro próprios. Isso também ajuda o sigilo da solução (P3), porque nenhum agregado chega à camada de entrada | 03 |
| Q2 | Konsist ou ArchUnit (P7 aceita os dois)? | Konsist: lê o código Kotlin, e os testes ficam em Kotest como o resto | 03 a 06, 09 |
| Q3 | Onde roda a CI? | GitHub Actions, porque o repositório está no GitHub | 07 |
| Q4 | Formatador ou lint (ktlint, detekt)? | Nenhum na S00. Entra por ADR se a equipe quiser | — |
| Q5 | Qual JDK? | Temurin 25, a LTS atual, já instalada na máquina de desenvolvimento | 01 |

## Rastreabilidade

| CA | Verificação |
|---|---|
| 01, 02 | comando no terminal (tarefa 1) |
| 03, 04, 05, 06, 09 | `bootstrap/src/test/kotlin/com/quebugesse/arquitetura/ArquiteturaTest.kt` |
| 07 | PR de teste no GitHub (tarefa 4) |
| 08 | comando no terminal (tarefa 2) |
