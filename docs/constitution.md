# Constituição do projeto Que Bug é Esse?

Versão 1.0.0 · Rascunho para aprovação da equipe · 25/10/2026

Estas regras valem para toda spec, plano, tarefa e PR do projeto. Quando uma spec contradiz a constituição, vale a constituição até que uma ADR a emende (ver "Emendas").

Origem: seção 2 de [planejamento-sdd.md](../planejamento-sdd.md), com base nos diagramas UML em [diagramas/](../diagramas/).

---

## Princípios

### P1. Domínio puro

O módulo `domain` (`com.quebugesse.domain`) depende só da biblioteca padrão do Kotlin, de `java.time` e de `java.util.UUID` como tipo. Não importa framework, IO, log, serialização nem anotação de persistência. Não lê o relógio do sistema nem gera aleatoriedade: `Instant.now()`, `Clock`, `UUID.randomUUID()` e `Random` são proibidos ali. Tempo entra como parâmetro `agora: Instant` em todo comando que registra um instante; IDs de agregado chegam prontos de fora.

*Por quê:* testes determinísticos e adaptadores trocáveis "sem tocar no domínio", como dizem os diagramas de componentes e de implantação.
*Como se verifica:* o `build.gradle.kts` do módulo não declara dependências além do Kotlin; um teste de arquitetura (S00) falha se aparecer import fora da lista ou uma das chamadas proibidas.

### P2. Agregados protegem suas invariantes

O estado de um agregado só muda pelos métodos da raiz. Propriedades têm `private set`; coleções são expostas como `List` somente leitura. Outro agregado é referenciado só por ID. Um comando rejeitado lança exceção de domínio sem mutação parcial e sem evento.

*Como se verifica:* todo teste de rejeição compara uma fotografia do agregado antes e depois do comando.

### P3. Sigilo da solução por papel

`solucaoCompleta` nunca vai em payload destinado a investigador enquanto a partida não estiver em estado terminal. Pode ir ao narrador humano autenticado da própria partida e aos adaptadores de IA (`NarradorIA`, `VerificadorDeSolucao`), que nunca a repassam ao jogador.

*Por quê:* substitui a nota "solucaoCompleta não sai do domínio", que não se sustenta no modo grupo (o narrador humano precisa da solução para responder) nem no passo 17 da sequência 2 (melhoria M1).
*Como se verifica:* teste que serializa todo DTO de investigador e procura o texto da solução (S03/S06); bateria adversarial do `NarradorIA` (S07).

### P4. Regras do jogo são testes executáveis

Cada regra (H1 a H8 e as que vierem) e cada guarda do diagrama de estados tem ao menos um critério de aceitação (CA) na spec e um teste cujo nome começa pelo ID do CA. Regra sem teste não conta como implementada.

### P5. Teste primeiro

Ciclo vermelho, verde, refatorar. Nenhum código de produção entra sem um teste que falhava antes dele. A descrição do PR lista os CAs cobertos.

### P6. Toda porta tem um fake

Para cada porta do domínio existe implementação em memória ou fake programável (inclusive para falhar). Testes de domínio e de aplicação não acessam rede, banco nem LLM. Suítes que chamam LLM real ficam fora do CI padrão.

### P7. Dependências entre camadas verificadas

Direção permitida, conforme o diagrama de pacotes: `interfaces` → `application` → `domain` ← `infrastructure`. O módulo `bootstrap` (raiz de composição) é o único que conhece todos e liga adaptadores às portas. Dentro de `domain`, os pacotes `gamesession`, `enigma` e `player` dependem só de `shared`.

*Como se verifica:* teste de arquitetura com Konsist ou ArchUnit (S00), que roda em todo build.

### P8. Decisões em ADR

Toda decisão que outra spec precise assumir (framework, banco, contrato de porta, regra de jogo ambígua) vira ADR em `docs/adr/NNNN-titulo.md`, com contexto, opções, decisão e consequências. Uma spec pode seguir com uma questão em aberto se registrar uma **decisão provisória** e listar os CAs que mudam caso a equipe decida diferente.

### P9. Diagramas acompanham o código

As fontes dos diagramas ficam em `docs/diagramas/` em Mermaid ou PlantUML; o PDF é gerado a partir delas. PR que muda modelo, estados, portas ou pacotes atualiza o diagrama no mesmo PR. Enquanto isso não acontece, a divergência fica listada na seção "Divergências" da spec.

### P10. Linguagem ubíqua

Termos do jogo em português, como nos diagramas: Partida, Enigma, Narrador, Investigador, Veredito, Tentativa. Nomes que os diagramas já fixaram em inglês (`GameSession`, `Question`, `Answer`, `Player`, `GameStatus`) ficam como estão, para manter a rastreabilidade. Termo novo entra primeiro no glossário da spec.

---

## Fluxo de trabalho

1. **`spec.md`**: o quê e por quê. Histórias, regras, CAs, divergências, questões abertas. Sem escolha de tecnologia.
2. **`plan.md`**: como. Modelo, decisões de projeto, estrutura de arquivos, estratégia de testes. Passa pelo portão da constituição (uma linha por princípio).
3. **`tasks.md`**: tarefas pequenas, cada uma com o teste escrito primeiro e o comando que prova que passou.
4. **Implementação** tarefa a tarefa, em PRs pequenos.
5. **Fechamento** pela definição de pronto abaixo.

Status de uma spec: `Rascunho` → `Aprovada` (equipe revisou) → `Em implementação` → `Concluída`.

## Definição de pronto de uma spec

- Todos os CAs com teste verde, nomeado pelo ID.
- Teste de arquitetura verde.
- Cobertura de linhas de pelo menos 90% nos pacotes tocados (Kover).
- Diagramas atualizados, ou divergência registrada na spec.
- Questões abertas resolvidas ou com decisão provisória registrada.
- Nenhuma exceção a P1–P10 sem ADR.

## Convenções

- Kotlin na JVM, build Gradle com Kotlin DSL e catálogo de versões em `gradle/libs.versions.toml`.
- Módulos Gradle: `domain`, `application`, `infrastructure`, `interfaces`, `bootstrap`. Pacote raiz `com.quebugesse`.
- Testes com Kotest. O nome do teste começa pelo ID do CA: `"CA-04 rejeita segunda pergunta com uma em aberto"`.
- Specs em `specs/NNN-nome/`; ADRs em `docs/adr/NNNN-titulo.md`.
- Mensagem de commit curta, no imperativo, citando a spec: `S01: aplica veredito correto`.

## Emendas

Mudar esta constituição exige uma ADR e a aprovação da equipe. A versão segue SemVer: MAJOR quando um princípio é removido ou redefinido, MINOR quando um princípio é adicionado, PATCH para ajuste de texto.

## Histórico

| Versão | Data | Mudança |
|---|---|---|
| 1.0.0 | 25/10/2026 | Versão inicial, a partir de `planejamento-sdd.md` |
