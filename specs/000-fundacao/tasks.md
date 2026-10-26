# Tarefas: S00 Fundação do projeto

> **Para agentes:** SUB-SKILL OBRIGATÓRIA: `superpowers:subagent-driven-development` ou `superpowers:executing-plans`, tarefa a tarefa. Os passos usam caixas de seleção (`- [ ]`) para acompanhamento.

**Objetivo:** criar o build com cinco módulos, testes, cobertura agregada, teste de arquitetura e CI, sem código de produção.

**Arquitetura:** plugin de convenção em `build-logic` aplicado aos cinco módulos; regras de arquitetura como funções sobre escopos do Konsist, testadas contra exemplos de violação e contra a produção (DP3).

**Stack:** Gradle (wrapper), Kotlin DSL, Kotest, Kover, Konsist, GitHub Actions, JDK Temurin 25.

**Spec:** [spec.md](spec.md) · Plano técnico: [plan.md](plan.md) · Constituição: [docs/constitution.md](../../docs/constitution.md)

## Restrições globais

- Nenhum código de produção nos módulos (DP8).
- Versões só em `gradle/libs.versions.toml`: na tarefa 1, escolher a versão estável atual de Kotlin, Kotest, Kover e Konsist, compatíveis entre si e com o Gradle do wrapper.
- `domain` sem dependência de produção (F3).
- Dependências entre módulos exatamente as de F2.
- Commits: `git add` só com os caminhos da tarefa; mensagem `S00: <verbo no presente>`.
- No PowerShell, `.\gradlew.bat` no lugar de `./gradlew`.

## Foco de revisão

1. Fim de linha: `gradlew` com CRLF quebra na CI (Linux). Conferido na tarefa 1 pelo `.gitattributes` e pela CI na tarefa 4.
2. Exemplo de violação contaminando a verificação real: os arquivos de `src/test/resources` não podem entrar em `scopeFromProduction()`. Teste CA-06 na tarefa 3.
3. Regra que nunca falha: cada regra tem teste contra o exemplo (CA-09) na tarefa 3.
4. Import com curinga (`import com.quebugesse.application.*`) também conta como infração. Exemplo na tarefa 3.
5. Wrapper adulterado: a CI valida o `gradle-wrapper.jar`. Tarefa 4.

---

### Tarefa 1: wrapper, catálogo, plugin de convenção e módulos

**Files:**
- Create: `.gitattributes`, `settings.gradle.kts`, `build.gradle.kts`, `gradle/libs.versions.toml`, `gradle/wrapper/*`, `gradlew`, `gradlew.bat`
- Create: `build-logic/settings.gradle.kts`, `build-logic/build.gradle.kts`, `build-logic/src/main/kotlin/quebugesse.kotlin-module.gradle.kts`
- Create: `domain/build.gradle.kts`, `application/build.gradle.kts`, `infrastructure/build.gradle.kts`, `interfaces/build.gradle.kts`, `bootstrap/build.gradle.kts`

**Interfaces:**
- Produz: plugin `id("quebugesse.kotlin-module")`, que aplica Kotlin JVM, `jvmToolchain(25)`, Kover, Kotest (`kotest-runner-junit5` e `kotest-assertions-core` em `testImplementation`) e `useJUnitPlatform()`; aliases do catálogo `libs.konsist`, `libs.kotest.runner.junit5`, `libs.kotest.assertions.core`, `libs.kotest.property`.

- [ ] **Passo 1: gerar o wrapper**

Com o Gradle instalado: `gradle wrapper --gradle-version <versão estável atual> --distribution-type bin --gradle-distribution-sha256-sum <sha256 publicado em services.gradle.org>`.

`.gitattributes`:

```
gradlew text eol=lf
*.bat text eol=crlf
*.jar binary
```

- [ ] **Passo 2: catálogo e plugin de convenção**

- `gradle/libs.versions.toml`: versões `kotlin`, `kotest`, `kover`, `konsist`; bibliotecas do Kotest, do Konsist e os artefatos de plugin que o `build-logic` usa (`kotlin-gradle-plugin`, `kover-gradle-plugin`).
- `build-logic/settings.gradle.kts`: `dependencyResolutionManagement` lendo `../gradle/libs.versions.toml` como `libs`.
- `build-logic/build.gradle.kts`: plugin `kotlin-dsl`; os dois artefatos de plugin em `implementation`.
- `quebugesse.kotlin-module.gradle.kts`: o que está em "Produz" acima.

- [ ] **Passo 3: módulos e raiz**

- `settings.gradle.kts`: `pluginManagement { includeBuild("build-logic") }`, `rootProject.name = "que-bug-e-esse"`, `include("domain", "application", "infrastructure", "interfaces", "bootstrap")`.
- Cada `<módulo>/build.gradle.kts` aplica `id("quebugesse.kotlin-module")` e declara só as dependências de F2: `application` e `infrastructure` → `project(":domain")`; `interfaces` → `project(":application")`; `bootstrap` → os quatro.
- `build.gradle.kts` da raiz: plugin do Kover (`apply false` não basta: a raiz precisa dele para agregar), sem outras configurações por enquanto.

- [ ] **Passo 4: verificar**

Run: `./gradlew projects`
Expected: lista `:application`, `:bootstrap`, `:domain`, `:infrastructure` e `:interfaces`, e nada mais (CA-02).

Run: `./gradlew build`
Expected: `BUILD SUCCESSFUL` (CA-01).

Run (Git Bash): `git ls-files --eol gradlew gradlew.bat`
Expected: `gradlew` com `w/lf`, `gradlew.bat` com `w/crlf` (depois do `git add`).

- [ ] **Passo 5: commit**

```bash
git add .gitattributes settings.gradle.kts build.gradle.kts gradle gradlew gradlew.bat build-logic domain/build.gradle.kts application/build.gradle.kts infrastructure/build.gradle.kts interfaces/build.gradle.kts bootstrap/build.gradle.kts
git commit -m "S00: cria build com cinco módulos e plugin de convenção"
```

---

### Tarefa 2: cobertura agregada

**Files:**
- Modify: `build.gradle.kts`

**Interfaces:**
- Consome: tarefa 1.
- Produz: `./gradlew koverHtmlReport` e `./gradlew koverVerify` na raiz, cobrindo os cinco módulos.

- [ ] **Passo 1: agregar**

Na raiz, `dependencies { kover(project(":domain")); kover(project(":application")); kover(project(":infrastructure")); kover(project(":interfaces")); kover(project(":bootstrap")) }`. Sem regra de verificação: cada spec põe a sua (DP6).

- [ ] **Passo 2: verificar**

Run: `./gradlew koverHtmlReport`
Expected: `BUILD SUCCESSFUL` e `build/reports/kover/html/index.html` criado (CA-08; com os módulos vazios o relatório não tem classes, e a tarefa 3 acrescenta as do teste de arquitetura, que não contam por serem de teste).

- [ ] **Passo 3: commit**

```bash
git add build.gradle.kts
git commit -m "S00: agrega cobertura dos módulos"
```

---

### Tarefa 3: regras de arquitetura

**Files:**
- Modify: `bootstrap/build.gradle.kts` (`testImplementation(libs.konsist)`)
- Create: `bootstrap/src/test/kotlin/com/quebugesse/arquitetura/Regras.kt`
- Test: `bootstrap/src/test/kotlin/com/quebugesse/arquitetura/ArquiteturaTest.kt`
- Create: `bootstrap/src/test/resources/arquitetura/camadas/*.kt`, `.../dominio-puro/*.kt`, `.../agregados/*.kt`

**Interfaces:**
- Consome: tarefa 1.
- Produz: `data class Infracao(val arquivo: String, val detalhe: String)`, `fun infracoesDeCamada(escopo: KoScope): List<Infracao>`, `fun infracoesDoDominioPuro(escopo: KoScope): List<Infracao>`, `fun infracoesEntreAgregados(escopo: KoScope): List<Infracao>`. `detalhe` tem o formato `"<pacote do arquivo> -> <import ou trecho proibido>"`.

- [ ] **Passo 1: escrever os exemplos de violação**

Cada arquivo tem só `package`, imports e, quando for o caso, uma linha de código. O nome do arquivo diz o que ele viola.

| Arquivo | `package` | Conteúdo relevante | Esperado |
|---|---|---|---|
| `camadas/DominioImportaAplicacao.kt` | `com.quebugesse.domain.exemplo` | `import com.quebugesse.application.Qualquer` | infração |
| `camadas/InterfacesImportaDominio.kt` | `com.quebugesse.interfaces.exemplo` | `import com.quebugesse.domain.Qualquer` | infração |
| `camadas/InfraImportaAplicacaoComCuringa.kt` | `com.quebugesse.infrastructure.exemplo` | `import com.quebugesse.application.*` | infração |
| `camadas/AplicacaoImportaDominio.kt` | `com.quebugesse.application.exemplo` | `import com.quebugesse.domain.Qualquer` | nada |
| `dominio-puro/UsaRelogio.kt` | `com.quebugesse.domain.exemplo` | `import java.time.Instant` e `val agora = Instant.now()` | infração (`Instant.now(`) |
| `dominio-puro/ImportaBibliotecaExterna.kt` | `com.quebugesse.domain.exemplo` | `import com.fasterxml.jackson.databind.ObjectMapper` | infração |
| `dominio-puro/ImportsPermitidos.kt` | `com.quebugesse.domain.exemplo` | `import kotlin.math.max`, `import java.time.Instant`, `import java.util.UUID`, `import com.quebugesse.domain.shared.PlayerRef` | nada |
| `agregados/PartidaImportaEnigma.kt` | `com.quebugesse.domain.gamesession` | `import com.quebugesse.domain.enigma.Enigma` | infração |
| `agregados/PartidaImportaShared.kt` | `com.quebugesse.domain.gamesession` | `import com.quebugesse.domain.shared.PlayerRef` | nada |

- [ ] **Passo 2: escrever os testes**

```kotlin
class ArquiteturaTest : FunSpec({
    val producao = Konsist.scopeFromProduction()
    fun exemplo(regra: String) = Konsist.scopeFromDirectory("bootstrap/src/test/resources/arquitetura/$regra")

    test("CA-03 regra de camadas acusa os imports proibidos do exemplo") {
        infracoesDeCamada(exemplo("camadas")).map { it.detalhe } shouldContainExactlyInAnyOrder listOf(
            "com.quebugesse.domain.exemplo -> com.quebugesse.application.Qualquer",
            "com.quebugesse.interfaces.exemplo -> com.quebugesse.domain.Qualquer",
            "com.quebugesse.infrastructure.exemplo -> com.quebugesse.application.*",
        )
    }
    test("CA-04 agregados do domínio só dependem de shared") {
        infracoesEntreAgregados(exemplo("agregados")).map { it.detalhe } shouldContainExactly listOf(
            "com.quebugesse.domain.gamesession -> com.quebugesse.domain.enigma.Enigma",
        )
    }
    test("CA-05 domínio puro: só imports permitidos e sem relógio nem aleatoriedade") {
        infracoesDoDominioPuro(exemplo("dominio-puro")).map { it.detalhe } shouldContainExactlyInAnyOrder listOf(
            "com.quebugesse.domain.exemplo -> Instant.now(",
            "com.quebugesse.domain.exemplo -> com.fasterxml.jackson.databind.ObjectMapper",
        )
    }
    test("CA-06 código de produção não tem infração") {
        (infracoesDeCamada(producao) + infracoesEntreAgregados(producao) + infracoesDoDominioPuro(producao))
            .shouldBeEmpty()
    }
})
```

Os três primeiros testes cobrem também o CA-09.

- [ ] **Passo 3: rodar e ver falhar**

Run: `./gradlew :bootstrap:test`
Expected: FAIL na compilação, `Unresolved reference: infracoesDeCamada`.

- [ ] **Passo 4: implementar `Regras.kt`**

- Camada de um arquivo: o segmento depois de `com.quebugesse.` no pacote. Dependências permitidas (F2): `domain` → {`domain`}; `application` → {`application`, `domain`}; `infrastructure` → {`infrastructure`, `domain`}; `interfaces` → {`interfaces`, `application`}; `bootstrap` sem restrição.
- Domínio puro: import aceito se começar com `kotlin.`, `java.time.`, `com.quebugesse.domain.` ou for `java.util.UUID`; texto do arquivo sem `Instant.now(`, `Clock`, `UUID.randomUUID(` e `Random`.
- Entre agregados: em `com.quebugesse.domain.<x>`, com x em {`gamesession`, `enigma`, `player`}, import de `com.quebugesse.domain.<y>` só se y for `x` ou `shared`.
- Usar só arquivo, pacote, imports e texto da API do Konsist (ver Riscos do plan.md).

- [ ] **Passo 5: rodar e ver passar**

Run: `./gradlew :bootstrap:test`
Expected: PASS, 4 testes. Se `scopeFromDirectory` não achar os arquivos (lista vazia e falha nos três primeiros testes), ajustar o caminho conforme a documentação da versão do Konsist e registrar no plan.md.

- [ ] **Passo 6: commit**

```bash
git add bootstrap/build.gradle.kts bootstrap/src/test
git commit -m "S00: verifica camadas, agregados e pureza do domínio"
```

---

### Tarefa 4: CI no GitHub Actions

**Files:**
- Create: `.github/workflows/build.yml`

**Interfaces:**
- Consome: tarefas 1 a 3.
- Produz: check `build` em todo PR e em todo push na `main`.

- [ ] **Passo 1: escrever o workflow**

```yaml
name: build
on:
  push:
    branches: [main]
  pull_request:
concurrency:
  group: build-${{ github.ref }}
  cancel-in-progress: true
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: gradle/actions/wrapper-validation@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '25'
      - uses: gradle/actions/setup-gradle@v4
      - run: ./gradlew build
```

Antes do commit, conferir a versão principal atual de cada action e atualizar o `@v4` se houver uma mais nova.

- [ ] **Passo 2: provar o vermelho e o verde (CA-07)**

Numa branch `s00-ci`: commit do workflow, push e PR. Expected: check `build` verde.
Na mesma branch, commit com um teste que falha (por exemplo `test("falha de propósito") { 1 shouldBe 2 }` em `ArquiteturaTest`). Expected: check vermelho. Reverter o commit. Expected: verde de novo.

- [ ] **Passo 3: commit e merge**

```bash
git add .github/workflows/build.yml
git commit -m "S00: roda o build na CI"
```

Merge do PR depois do verde. A proteção da `main` (exigir o check `build`) é manual, na lista de [docs/pendencias.md](../../docs/pendencias.md).

---

### Tarefa 5: documentação

**Files:**
- Modify: `README.md` (seção "Como rodar"), `docs/pendencias.md`, `docs/diagramas/pacotes.md`, `specs/000-fundacao/spec.md`

- [ ] **Passo 1:** em `README.md`, trocar "Ainda não há código" pelos comandos `./gradlew build`, `./gradlew koverHtmlReport` e onde abrir o relatório.
- [ ] **Passo 2:** em `docs/diagramas/pacotes.md`, acrescentar o pacote `bootstrap` com a seta para as quatro camadas (D1) e marcar D1 e D2 como aplicadas na spec.
- [ ] **Passo 3:** em `docs/pendencias.md`, marcar "S00 implementada" e manter o item manual da proteção da `main`.
- [ ] **Passo 4: commit**

```bash
git add README.md docs/pendencias.md docs/diagramas/pacotes.md specs/000-fundacao/spec.md
git commit -m "S00: atualiza documentação do build"
```

---

## Cobertura dos CAs

| Tarefa | CAs |
|---|---|
| 1 | 01, 02 |
| 2 | 08 |
| 3 | 03, 04, 05, 06, 09 |
| 4 | 07 |
