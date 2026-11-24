# ADR 0009: Divisão de repositórios da organização `Que-Bug-E-Esse`

| Campo | Valor |
|---|---|
| Status | Aceita |
| Proposta em | 2026-11-24 |
| Decidida em | 2026-11-24 |
| Participantes | Diogo (decisão simplificada) |
| Substitui / substituída por | — |

## Contexto

A organização GitHub `Que-Bug-E-Esse` foi criada com três repositórios independentes: `Que-Bug-E-Esse/back`, `Que-Bug-E-Esse/front` e `Que-Bug-E-Esse/ai`. O planejamento inicial (`CLP`, S00 e diagramas de implantação) descrevia apenas um projeto Kotlin/JVM falando direto com uma API de LLM e um artefato genérico de front-end.

Sem definir com clareza o que vive em cada repositório, há risco de duplicar regras de jogo fora do domínio (P1), vazar `solucaoCompleta` ou chaves de API para o cliente (P3), ou acoplar testes de domínio e aplicação a rede/LLM (P6).

## Opções

### A. Divisão enxuta por responsabilidade (`back` núcleo + API + SDD, `ai` serviço HTTP stateless mínimo, `front` SPA mínima)

Cada repositório tem uma única responsabilidade clara e nenhum duplica regras do jogo:

1. **`Que-Bug-E-Esse/back` (Kotlin/JVM, JDK 25 LTS, Gradle 9.8.2 multi-módulo)**
   - Contém os cinco módulos da S00 (`domain`, `application`, `infrastructure`, `interfaces`, `bootstrap`), a documentação SDD (`docs/`, `specs/`), a CLI local (S05), a API REST + WebSocket em Ktor (ADR 0003, S06), a autenticação de convidados por apelido + token (ADR 0007) e a persistência SQLite + Flyway (ADR 0004, S09).
   - É a única fonte de verdade das regras do jogo, da máquina de estados e do sigilo por papel (P1, P2, P3, P7).
   - Os adaptadores `NarradorIA` (S07) e `VerificadorDeSolucao` (S08) em `infrastructure` chamam o serviço `ai` via HTTP JSON; nos testes e na CI, continuam usando fakes em memória (P6).

2. **`Que-Bug-E-Esse/ai` (Python 3.13 + `uv` + FastAPI, serviço HTTP JSON stateless)**
   - Serviço mínimo sem banco de dados que encapsula o provedor de LLM (ADR 0005), os prompts defensivos contra injeção (M7), o controle de teto de tokens e a validação estrita de saída estruturada.
   - Expõe apenas dois endpoints de domínio (além de `GET /health` opcional):
     - `POST /narrar`: recebe `{ "enunciado", "solucaoCompleta", "historico", "pergunta" }` e devolve exclusivamente `{ "resposta": "SIM" | "NAO" | "IRRELEVANTE" }` — nunca texto livre, preservando P3.
     - `POST /verificar`: recebe `{ "enunciado", "solucaoCompleta", "proposta" }` e devolve exclusivamente `{ "correta": true | false }` — nunca devolve `solucaoCompleta` ao chamador, preservando P3.
   - Acessível apenas pela rede interna do `back` (nunca pelo `front`), mantendo `solucaoCompleta` e `LLM_API_KEY` fora do cliente.
   - Falhas de rede, timeout, HTTP 4xx/5xx ou JSON inválido são tratadas pelos adaptadores no `back` sem mutar o estado da partida, preservando H4 (a pergunta ou proposta permanece pendente para retentativa ou cancelamento).
   - (Futuro, S11: `GeradorDeEnigmas` poderá acrescentar `POST /gerar-enigma` sem alterar os dois endpoints do MVP.)

3. **`Que-Bug-E-Esse/front` (SPA mínima em TypeScript + Vite + React, Node 24 / pnpm, ADR 0008)**
   - Cliente web leve para investigador e narrador, consumindo apenas a API REST e WebSocket do `back`.
   - Não possui lógica de regras do jogo, não fala diretamente com `ai` nem com o banco, e nunca recebe `solucaoCompleta` no papel de investigador antes de estado terminal (P3).

- A favor: alinha exatamente aos três repositórios criados; cada stack (Kotlin/JVM, Python, TypeScript) tem build e CI próprios e simples; `back` preserva intactas todas as garantias de P1, P3, P6, P7 e H4; `ai` tem contrato minúsculo de dois endpoints fácil de testar e trocar de provedor.
- Contra: exige manter sincronizado o contrato JSON de dois endpoints (`back` ↔ `ai`) e da API REST/WebSocket (`back` ↔ `front`), mitigado por contratos simples e versionados nas specs.

### B. Monorepo único (ignorar `front` e `ai`)

Colocar tudo dentro de `back` e deixar os outros repositórios vazios.

- A favor: um único repositório e um único PR por mudança.
- Contra: contraria a organização já criada para trabalho paralelo em três stacks distintas e mistura builds Gradle, Python e Node na mesma CI.

### C. Microsserviços distribuídos com mensageria externa

Adicionar broker (RabbitMQ/Kafka), banco próprio no serviço `ai` e orquestração distribuída entre serviços.

- A favor: desacoplamento assíncrono em larga escala.
- Contra: complexidade operacional desproporcional ao escopo do projeto.

## Recomendação

A. É a divisão mais simples que respeita os três repositórios da organização sem duplicar domínio nem enfraquecer P3 e H4.

## Decisão da equipe

**Opção A aceita:** `back` concentra domínio, aplicação, persistência, CLI, API Ktor e docs SDD; `ai` é um serviço HTTP JSON stateless com dois endpoints (`/narrar` e `/verificar`) consumido exclusivamente pelo `back` preservando P3 e H4; `front` é a SPA mínima da ADR 0008 consumindo exclusivamente o `back`.

## Impacto

| Documento | Atualização após a decisão |
|---|---|
| [S00](../../specs/000-fundacao/spec.md) | Código da S00 vive em `Que-Bug-E-Esse/back` com os cinco módulos Gradle e CI própria |
| S06 | Documenta o contrato REST + WebSocket consumido por `Que-Bug-E-Esse/front` |
| S07 e S08 | Adaptadores `NarradorIA` e `VerificadorDeSolucao` em `back/infrastructure` chamam `POST /narrar` e `POST /verificar` em `Que-Bug-E-Esse/ai` |
| S10 | Implementada em `Que-Bug-E-Esse/front` conforme ADR 0008 |
| S11 | Pode acrescentar `POST /gerar-enigma` em `Que-Bug-E-Esse/ai` quando for especificada |
| [Componentes](../diagramas/componentes.md) e [implantação](../diagramas/implantacao.md) | Refletir o nó `ai` (`/narrar`, `/verificar`) entre `back` e a API de LLM, e mapear cada nó ao seu repositório |
