# Diagrama de classes do domínio

| Campo | Valor |
|---|---|
| Fonte original | [`diagramas/diagrama_classes.pdf`](../../diagramas/diagrama_classes.pdf), versão de 14/10/2026 |
| Estado | Transcrição fiel do PDF. As mudanças já decididas nas specs e ADRs estão em [Pendências por diagrama](README.md#pendências-por-diagrama) |

```mermaid
classDiagram
    direction LR
    namespace domain_gamesession {
        class GameSession {
            <<AggregateRoot>>
            +id: GameSessionId [readOnly]
            +enigmaId: EnigmaId [readOnly]
            +narrador: PlayerRef [readOnly]
            +investigadores: List~PlayerRef~ [readOnly, 1..*]
            +status: GameStatus [readOnly]
            +criadaEm: Instant [readOnly]
            +resolvidaEm: Instant? [readOnly]
            +iniciar()
            +fazerPergunta(autor: PlayerRef, texto: String) Question
            +responderPergunta(autor: PlayerRef, id: QuestionId, resposta: Answer)
            +proporSolucao(autor: PlayerRef, texto: String)
            +aplicarVeredito(veredito: Veredito, instante: Instant)
            +abandonar(autor: PlayerRef)
        }
        class Question {
            <<Entity>>
            +id: QuestionId [readOnly]
            +autor: PlayerRef [readOnly]
            +texto: String [readOnly]
            +resposta: Answer?
            +criadaEm: Instant [readOnly]
            +/aberta: Boolean [= resposta == null]
        }
        class Veredito {
            <<ValueObject>>
            +correta: Boolean
        }
        class Answer {
            <<enumeration>>
            SIM
            NAO
            IRRELEVANTE
        }
        class GameStatus {
            <<enumeration>>
            AGUARDANDO_INICIO
            EM_ANDAMENTO
            RESOLVIDO
            ABANDONADO
        }
    }
    namespace domain_enigma {
        class Enigma {
            <<AggregateRoot>>
            +id: EnigmaId [readOnly]
            +titulo: String
            +categoria: Categoria
            +dificuldade: Dificuldade
            +enunciado: String
            ~solucaoCompleta: String [não vazia]
            +tags: Set~String~
        }
        class Categoria {
            <<enumeration>>
            SINTAXE
            DEBUG
            GESTAO_PROJETO
            DESIGN
            DISCOVERY
        }
        class Dificuldade {
            <<enumeration>>
            FACIL
            MEDIA
            DIFICIL
        }
    }
    namespace domain_player {
        class Player {
            <<AggregateRoot>>
            +id: PlayerId [readOnly]
            +nome: String
            +tipo: PlayerType
            +stats: EstatisticasJogador
        }
        class EstatisticasJogador {
            <<ValueObject>>
            +partidasJogadas: Int
            +partidasResolvidas: Int
        }
        class PlayerType {
            <<enumeration>>
            HUMANO
            IA
        }
    }
    namespace domain_shared {
        class PlayerRef {
            <<ValueObject>>
            +id: PlayerId
        }
        class IDsTipados["IDs tipados"] {
            <<value class>>
            GameSessionId
            EnigmaId
            PlayerId
            QuestionId
        }
    }

    GameSession "1" *-- "0..*" Question : perguntas [ordered, ≤ 1 aberta]
    GameSession ..> Enigma : «por id» enigmaId
    GameSession ..> Player : «por id» narrador, investigadores
```

## Notas do PDF

- "Toda mutação passa pela raiz: status e perguntas só mudam pelos métodos (private set / lista privada)."
- "~ solucaoCompleta: visível só no pacote domain — nunca vai para a tela durante a investigação."
- "Tipos de valor (VO, enum, IDs) aparecem só como tipo de atributo: as linhas ficam reservadas para composição e referência entre agregados."

Notação do PDF:

| Notação | Leitura |
|---|---|
| `+` / `-` / `~` | público / privado / interno ao pacote |
| `{readOnly}` (aqui `[readOnly]`) | Kotlin `val` (ou `var` com `private set`) |
| `Tipo?` · `/nome` | anulável (Kotlin) · atributo derivado |
| ◆—— | composição: a parte só existe dentro da raiz |
| `- - -> «por id»` | referência a outro agregado apenas por ID |

Legenda do PDF: cabeçalho escuro = «AggregateRoot» · bege = «Entity» · claro = «ValueObject» / «enumeration» · pasta laranja = pacote · ◆ composição · tracejada = referência por ID.
