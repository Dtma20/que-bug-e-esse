# Plano técnico: S00 Fundação do projeto

Spec: [spec.md](spec.md) · Tarefas: [tasks.md](tasks.md) · Constituição 1.0.0: [docs/constitution.md](../../docs/constitution.md)

## Resumo

Um build Gradle com cinco módulos que compartilham um plugin de convenção (Kotlin JVM, toolchain do JDK, Kotest e Kover). As regras de arquitetura são funções que recebem um escopo do Konsist e devolvem a lista de infrações; os testes no módulo `bootstrap` aplicam cada função ao código de produção (lista vazia) e a um exemplo de violação (infrações esperadas). A CI do GitHub Actions roda `./gradlew build`.

## Contexto técnico

| Item | Escolha |
|---|---|
| JDK | Temurin 25 (LTS), via toolchain |
| Build | Gradle (wrapper versionado), Kotlin DSL, catálogo `gradle/libs.versions.toml`, plugin de convenção em `build-logic` |
| Testes | Kotest (runner JUnit 5, `assertions-core`) |
| Arquitetura | Konsist, só em `testImplementation` do `bootstrap` |
| Cobertura | Kover, com relatório agregado na raiz |
| CI | GitHub Actions |

## Portão da constituição

| Princípio | Como a S00 cumpre |
|---|---|
| P1 Domínio puro | Regra `infracoesDoDominioPuro`; `domain` sem dependências de produção |
| P2 Invariantes | Não se aplica: não há agregado |
| P3 Sigilo | Não se aplica diretamente; Q1 impede que agregados cheguem a `interfaces` |
| P4 Regras como testes | Cada regra de arquitetura tem CA e teste nomeado |
| P5 Teste primeiro | As regras nascem dos testes contra os exemplos de violação |
| P6 Fakes | Não se aplica: não há porta |
| P7 Camadas | Regras `infracoesDeCamada` e `infracoesEntreAgregados` |
| P8 ADR | Q1 a Q5 com decisão provisória na spec |
| P9 Diagramas | Divergências D1 e D2 registradas; a fonte do diagrama de pacotes ganha o `bootstrap` |
| P10 Linguagem ubíqua | Nomes de módulo e pacote iguais aos da constituição |

## Decisões de projeto

**DP1. Plugin de convenção em vez de `subprojects {}`.** `build-logic` é um build incluído com o plugin pré-compilado `quebugesse.kotlin-module`. Configuração entre projetos (`subprojects`, `allprojects`) é desencorajada no Gradle 9 e atrapalha o isolamento de projetos.

**DP2. Toolchain fixa o JDK.** `kotlin { jvmToolchain(25) }` no plugin de convenção. A CI instala o mesmo JDK.

**DP3. Regras como funções sobre um escopo.** `infracoesDeCamada(escopo)`, `infracoesDoDominioPuro(escopo)` e `infracoesEntreAgregados(escopo)` devolvem `List<Infracao>`. O teste chama cada uma com `Konsist.scopeFromProduction()` e espera lista vazia, e com o escopo do diretório de exemplo e espera as infrações plantadas. Assim cada regra prova que morde (F6) sem precisar de código errado no build.

**DP4. Exemplos de violação fora da compilação.** Ficam em `bootstrap/src/test/resources/arquitetura/<regra>/`, como arquivos `.kt` que o Gradle não compila (são recursos) e o Konsist lê. Como `scopeFromProduction()` só olha `src/main`, eles não contaminam a verificação real.

**DP5. Regras só sobre produção.** Testes podem importar fakes e utilitários de qualquer camada; as dependências do Gradle já limitam o resto.

**DP6. Cobertura agregada, regras por spec.** O plugin do Kover vai em todos os módulos; a raiz agrega com `kover(project(":<módulo>"))`. Cada spec acrescenta a sua regra de verificação no módulo que toca (a S01 põe 90% de linhas em `domain.gamesession`).

**DP7. Wrapper com checksum.** `distributionSha256Sum` preenchido no `gradle-wrapper.properties`, e a CI valida o `gradle-wrapper.jar`. `.gitattributes` fixa `eol=lf` no `gradlew` e `eol=crlf` no `gradlew.bat`, porque o repositório é editado no Windows.

**DP8. Módulos vazios.** Nenhum módulo ganha código de exemplo. O primeiro código de produção chega com a S01 (domínio) e a S02.

## Estrutura de arquivos

```
.gitattributes
settings.gradle.kts                 # pluginManagement { includeBuild("build-logic") }, include dos 5 módulos
build.gradle.kts                    # raiz: Kover agregado
gradle/libs.versions.toml
gradle/wrapper/gradle-wrapper.jar
gradle/wrapper/gradle-wrapper.properties
gradlew, gradlew.bat
build-logic/
  settings.gradle.kts               # lê o catálogo da raiz
  build.gradle.kts                  # `kotlin-dsl`, dependências dos plugins Kotlin e Kover
  src/main/kotlin/quebugesse.kotlin-module.gradle.kts
domain/build.gradle.kts
application/build.gradle.kts        # implementation(project(":domain"))
infrastructure/build.gradle.kts     # implementation(project(":domain"))
interfaces/build.gradle.kts         # implementation(project(":application"))
bootstrap/build.gradle.kts          # implementation dos quatro; testImplementation(konsist)
bootstrap/src/test/kotlin/com/quebugesse/arquitetura/
  Regras.kt                         # Infracao e as três funções
  ArquiteturaTest.kt
bootstrap/src/test/resources/arquitetura/
  camadas/*.kt
  dominio-puro/*.kt
  agregados/*.kt
.github/workflows/build.yml
```

## Estratégia de testes

- **Arquitetura.** Um teste por regra contra o exemplo (infrações exatas) e um teste por regra contra a produção (vazio).
- **Build.** CA-01, 02 e 08 são verificados por comando no terminal, registrados nas tarefas.
- **CI.** CA-07 é verificado uma vez, com um PR de teste.

## Riscos

| Risco | Mitigação |
|---|---|
| Versões de Kotlin, Kover e Konsist incompatíveis com o Gradle 9 | Escolher as versões juntas na tarefa 1 e rodar `./gradlew build` antes de seguir |
| API do Konsist mudar entre versões | As regras usam só arquivo, pacote, imports e texto, a parte mais estável da API |
| `scopeFromDirectory` resolver caminho a partir de outra pasta | A tarefa 3 confere o caminho no primeiro teste que falha |
