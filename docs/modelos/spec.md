# Spec NNN: <nome>

| Campo | Valor |
|---|---|
| ID | SNN |
| Status | Rascunho |
| Depende de | Specs anteriores e o que cada uma entrega a esta |
| Casos de uso | UCs do diagrama de casos de uso que esta spec cobre, inteiros ou em parte |
| Diagramas de origem | PDFs ou fontes em `docs/diagramas/` de onde saem as regras |
| Decisões que assume | ADRs, com a opção seguida e o status de cada uma |
| Melhorias incorporadas | Itens M do [planejamento](../../planejamento-sdd.md) |
| Plano | Plano e tarefas: ainda não escritos (depois, links para `plan.md` e `tasks.md`) |

<!-- Copie este arquivo para specs/NNN-nome/spec.md; os links relativos já valem lá. Sem escolha de tecnologia: isso vai no plan.md. -->

## Contexto

O que as specs anteriores já entregam, o que falta e o que esta spec cria. Em seguida, o que fica para outras specs.

## Histórias

- **US1.** Como <ator>, quero <ação>, para <valor> (UC, M ou regra de origem).

## Glossário

Termos novos ou com sentido próprio nesta spec (P10). Marque "(novo)" no que ainda não existe no código.

| Termo | Significado |
|---|---|
| | |

## Regras

Uma linha por regra, com um prefixo próprio da spec. Cite a questão aberta que a regra assume: (**Q1**).

| ID | Regra |
|---|---|
| X1 | |

<!-- Subseções opcionais, conforme a spec: portas novas, casos de uso, códigos de erro, ordem de verificação, emendas a outra spec. -->

## Critérios de aceitação

Participantes e dados de exemplo usados nos CAs.

| CA | Regra | Dado | Quando | Então |
|---|---|---|---|---|
| 01 | X1 | | | |

## Requisitos não funcionais

- **RNF1.**

## Fora de escopo

- O que não entra, com a spec ou ADR que vai tratar.

## Divergências em relação aos diagramas

Pelo princípio P9, as fontes em [docs/diagramas/](../../docs/diagramas/README.md) recebem estes pontos no PR que implementar esta spec. Copie cada linha também para a tabela "Pendências por diagrama" do README dos diagramas.

| ID | Diagrama hoje | Esta spec | Origem |
|---|---|---|---|
| D1 | | | |

## Questões abertas

Todas têm decisão provisória para não travar a implementação. Se a equipe decidir diferente, mudam os CAs indicados. Copie cada uma também para [docs/pendencias.md](../../docs/pendencias.md).

| Q | Pergunta | Decisão provisória | CAs afetados |
|---|---|---|---|
| Q1 | | | |

## Rastreabilidade

| CA | Arquivo de teste |
|---|---|
| 01 | |
