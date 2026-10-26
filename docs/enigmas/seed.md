# Catálogo inicial (seed)

Rascunho dos cinco enigmas e do jogador IA que a S02 grava ao subir o jogo (regras SD1 a SD3 da [spec S02](../../specs/002-catalogo-e-jogador/spec.md)). **Status: à espera de revisão da equipe** (questão Q6 da S02, listada em [docs/pendencias.md](../pendencias.md)).

Como revisar: edite o texto aqui, num PR. O teste CA-32 da S02 confere que título, enunciado e solução do código Kotlin aparecem literalmente neste arquivo, então quem implementar copia daqui, sem mudar uma letra. Por isso cada campo fica numa linha só, sem formatação Markdown dentro do texto.

## Jogador IA

| Campo | Valor |
|---|---|
| ID | `00000000-0000-0000-0000-0000000000a1` |
| Nome | Narrador IA |
| Tipo | `IA` |

## Enigmas

Todos com origem `MANUAL` e status `PUBLICADO`.

| ID | Categoria | Dificuldade | Título |
|---|---|---|---|
| `00000000-0000-0000-0000-0000000000e1` | `DEBUG` | `MEDIA` | Férias fatais |
| `00000000-0000-0000-0000-0000000000e2` | `SINTAXE` | `DIFICIL` | O comentário indispensável |
| `00000000-0000-0000-0000-0000000000e3` | `GESTAO_PROJETO` | `FACIL` | Velocidade em dobro |
| `00000000-0000-0000-0000-0000000000e4` | `DESIGN` | `MEDIA` | A madrugada maldita |
| `00000000-0000-0000-0000-0000000000e5` | `DISCOVERY` | `FACIL` | Exportou e sumiu |

### e1 · DEBUG · MEDIA

Título:

Férias fatais

Enunciado:

Toda vez que o Rafael sai de férias, a produção começa a falhar em algumas requisições. Quando ele volta, tudo se normaliza, sem nenhum deploy no meio. Ninguém mexe no código dele enquanto ele está fora. O que está acontecendo?

Solução:

O notebook do Rafael hospedava um endpoint que a produção chamava: num teste antigo, alguém apontou a configuração de produção para a máquina dele e esqueceu de desfazer. Nas férias o notebook fica desligado e as chamadas falham; quando ele volta e liga o notebook, tudo funciona de novo.

Tags: configuracao, producao, rede

### e2 · SINTAXE · DIFICIL

Título:

O comentário indispensável

Enunciado:

Numa limpeza de código Java, a Bia apagou uma linha que era só um comentário. O projeto parou de compilar, com erro numa linha bem mais abaixo. Ela restaurou o comentário e tudo voltou a compilar. Como apagar um comentário quebrou o build?

Solução:

O comentário continha \u000a seguido de uma declaração. O compilador Java traduz escapes Unicode antes de analisar o código, então \u000a virou uma quebra de linha, encerrou o comentário, e a declaração depois dele era código de verdade. Apagar o comentário apagou a declaração de uma variável usada mais abaixo.

Tags: java, unicode, compilador

### e3 · GESTAO_PROJETO · FACIL

Título:

Velocidade em dobro

Enunciado:

Depois de uma retrospectiva, a velocidade do time dobrou em duas sprints e o gráfico ficou lindo. Na reunião de resultados, o cliente disse que não percebeu nenhuma diferença no produto. Ninguém mentiu. O que mudou?

Solução:

O time passou a quebrar as mesmas histórias em pedaços menores e a estimar cada pedaço. A soma dos pontos dobrou, mas o trabalho entregue por sprint continuou o mesmo.

Tags: scrum, metricas, estimativa

### e4 · DESIGN · MEDIA

Título:

A madrugada maldita

Enunciado:

Um sistema de agendamento usado em Portugal funciona sem problema o ano inteiro. Uma vez por ano, sempre na mesma madrugada de domingo, alguns agendamentos são recusados com erro de registro duplicado. Nenhum dado foi digitado duas vezes. Por quê?

Solução:

A chave única da tabela usava a data e a hora local. Quando o horário de verão termina, os relógios voltam uma hora e o mesmo intervalo de horário local acontece duas vezes; o segundo agendamento naquele horário colide com o primeiro. Guardar o instante em UTC, ou com o deslocamento do fuso, resolve.

Tags: fuso-horario, banco-de-dados, chave-unica

### e5 · DISCOVERY · FACIL

Título:

Exportou e sumiu

Enunciado:

Os usuários pediram com insistência um botão de exportar para Excel. O time entregou o botão, muito bem feito. Um mês depois, o uso do sistema tinha caído pela metade, e ninguém reclamou de nada. O que aconteceu?

Solução:

Os usuários exportavam os dados para montar no Excel um relatório que o sistema não tinha. Com o botão, passaram a trabalhar só na planilha. O pedido real era o relatório, e ninguém perguntou para que serviria a exportação.

Tags: requisitos, entrevista, relatorio
