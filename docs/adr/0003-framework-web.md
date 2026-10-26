# ADR 0003: Framework web (Ktor ou Spring Boot)

| Campo | Valor |
|---|---|
| Status | Proposta |
| Proposta em | 26/10/2026 |
| Decidida em | — |
| Participantes | — |
| Substitui / substituída por | — |

## Contexto

Os diagramas de componentes e de implantação têm REST e WebSocket atrás de um proxy Nginx, rodando numa JVM, mas não dizem com qual framework. A escolha afeta os módulos `interfaces` e `bootstrap`; o domínio não muda (P1). Não bloqueia o marco M1, que roda no terminal.

## Opções

### A. Ktor

- A favor: leve, feito para Kotlin, corrotinas e WebSocket nativos, sobe rápido; a composição de dependências fica explícita no `bootstrap`.
- Contra: menos recursos prontos (injeção de dependências, validação e métricas vêm de bibliotecas à parte); menos material de estudo.

### B. Spring Boot com Kotlin

- A favor: ecossistema grande (injeção, validação, segurança, métricas); muita documentação.
- Contra: mais pesado e com mais configuração implícita; anotações tentam se espalhar e exigem atenção ao P1 e ao teste de arquitetura.

## Recomendação

A, salvo se a maioria da equipe já dominar Spring: o que a equipe conhece pesa mais que a diferença técnica entre os dois.

## Decisão da equipe

_Pendente._

## Impacto

| Documento | Se a escolha for A | Se for B |
|---|---|---|
| S06 | Rotas e WebSocket do Ktor | Controllers e WebSocket do Spring |
| [S00](../../specs/000-fundacao/spec.md) | `bootstrap` ganha o plugin do Ktor quando a S06 começar | `bootstrap` ganha o plugin do Spring Boot quando a S06 começar |
| [Constituição](../constitution.md), seção "Convenções" | Citar o framework escolhido (exige ADR de emenda só se mudar princípio; citar é ajuste de texto, versão PATCH) | Idem |
