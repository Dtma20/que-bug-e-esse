# Diagrama de sequência 1/2: iniciar partida

| Campo | Valor |
|---|---|
| Fonte original | [`diagramas/diagrama_seq_iniciar.pdf`](../../diagramas/diagrama_seq_iniciar.pdf), versão de 14/10/2026 |
| Estado | Transcrição fiel do PDF. As mudanças já decididas nas specs e ADRs estão em [Pendências por diagrama](README.md#pendências-por-diagrama) |

```mermaid
sequenceDiagram
    actor Q as Quem inicia
    participant T as «interfaces»<br/>Tela de partida
    participant UC as «useCase»<br/>IniciarPartidaUseCase
    participant ER as «port»<br/>EnigmaRepository

    Q->>T: 01 escolhe modo e inicia
    T->>UC: 02 executar(modo, investigadores)
    UC->>ER: 03 selecionar enigma
    ER-->>UC: 04 enigma
    alt modo solo
        UC->>UC: 05 narrador = NarradorIA
        Note right of UC: No solo, o enigma também pode vir do<br/>GeradorDeEnigmas (UC8 «extend» UC10).
    else modo em grupo
        UC->>UC: 06 narrador = NarradorHumano
    end
    create participant GS as «AggregateRoot»<br/>GameSession
    UC-->>GS: 07 «create» criar(enigma.id, narrador, investigadores)
    GS-->>UC: 08 AGUARDANDO_INICIO
    UC->>GS: 09 iniciar()
    GS->>GS: 10 valida configuração (H5)
    GS-->>UC: 11 EM_ANDAMENTO / PartidaIniciada
    participant GSR as «port»<br/>GameSessionRepository
    UC->>GSR: 12 salvar(partida)
    UC-->>T: 13 PartidaDto(enunciado, status)
    T-->>Q: 14 exibe enunciado e partida iniciada
```

## Notas do PDF

- "A tela recebe só enunciado e estado público: solucaoCompleta nunca sai do domínio durante a investigação."
- "«Quem inicia» = narrador humano (modo em grupo) ou o próprio investigador (modo solo)."

Legenda do PDF: cabeçalho escuro = «AggregateRoot» · bege = «useCase» · laranja-claro = «port» · seta cheia = chamada · tracejada = retorno / «create» · quadro laranja = fragmento (alt).
