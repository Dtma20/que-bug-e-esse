# Diagramas

Fontes Mermaid dos oito diagramas UML do projeto. Cada arquivo `.md` tem um bloco `mermaid`, que o GitHub desenha na própria página.

A regra vem do princípio P9 da [constituição](../constitution.md): "As fontes dos diagramas ficam em `docs/diagramas/` em Mermaid ou PlantUML; o PDF é gerado a partir delas. PR que muda modelo, estados, portas ou pacotes atualiza o diagrama no mesmo PR. Enquanto isso não acontece, a divergência fica listada na seção "Divergências" da spec."

**Estado atual:** transcrição fiel dos PDFs de 14/10/2026. Nenhuma mudança das specs entrou ainda; cada uma entra no PR que implementar a spec de origem (tabela "Pendências por diagrama").

## Fontes e PDFs

| Diagrama | Fonte | PDF de 14/10/2026 |
|---|---|---|
| Casos de uso | [casos-de-uso.md](casos-de-uso.md) | [diagrama_casos.pdf](../../diagramas/diagrama_casos.pdf) |
| Classes do domínio | [classes-dominio.md](classes-dominio.md) | [diagrama_classes.pdf](../../diagramas/diagrama_classes.pdf) |
| Componentes | [componentes.md](componentes.md) | [Diagrama_comp.pdf](../../diagramas/Diagrama_comp.pdf) |
| Implantação | [implantacao.md](implantacao.md) | [Diagrama_imp.pdf](../../diagramas/Diagrama_imp.pdf) |
| Pacotes | [pacotes.md](pacotes.md) | [diagrama_pacotes.pdf](../../diagramas/diagrama_pacotes.pdf) |
| Estados da partida | [estados-partida.md](estados-partida.md) | [diagrama_estados.pdf](../../diagramas/diagrama_estados.pdf) |
| Sequência 1/2: iniciar partida | [sequencia-iniciar-partida.md](sequencia-iniciar-partida.md) | [diagrama_seq_iniciar.pdf](../../diagramas/diagrama_seq_iniciar.pdf) |
| Sequência 2/2: investigar e resolver | [sequencia-investigar-e-resolver.md](sequencia-investigar-e-resolver.md) | [diagrama_seq_jogar.pdf](../../diagramas/diagrama_seq_jogar.pdf) |

Os PDFs em `diagramas/` são os originais da equipe e ficam como referência. Gerar PDFs novos a partir destas fontes é uma tarefa manual listada em [docs/pendencias.md](../pendencias.md).

## Adaptações de notação

O Mermaid não tem todos os símbolos da UML que os PDFs usam. As trocas abaixo não mudam o conteúdo.

| No PDF | Na fonte | Motivo |
|---|---|---|
| Restrições entre chaves: `{readOnly}`, `{não vazia}`, `{= resposta == null}` | Entre colchetes: `[readOnly]` | Chave fecha o corpo da classe no Mermaid |
| `List<PlayerRef>`, `Set<String>` | `List~PlayerRef~`, `Set~String~` | Sintaxe de genéricos do Mermaid |
| `fazerPergunta(...): Question` | `fazerPergunta(...) Question` | O Mermaid escreve o tipo de retorno depois dos parênteses e desenha os dois-pontos |
| Pasta de pacote | `namespace` nas classes; `subgraph` nos demais | |
| Pacote `domain.gamesession` (e os outros três do domínio) | `namespace domain_gamesession` | Ponto no nome do `namespace` dá erro de sintaxe no Mermaid 10 e no 11.0.0 (testado em 26/10/2026); o GitHub pode usar uma versão dessas |
| Ator (boneco) | Retângulo com «ator» | O `flowchart` não tem boneco |
| Elipse de caso de uso; fronteira do sistema | Nó `([...])`; `subgraph` | |
| Interface fornecida ou requerida (○) | Nó circular; linha cheia para quem fornece, seta tracejada para quem requer | |
| Nó «device» (cubo); nuvem | `subgraph` com «device» no título; elipse | |
| Notas, quadros de regras e legenda | Texto abaixo do diagrama, em "Notas do PDF" | Notas dentro do desenho cruzavam as setas |
| Cores do PDF | Só o laranja do fluxo principal (casos de uso) e o fundo escuro dos estados terminais | |
| Números das mensagens (01, 02...) | No texto da mensagem, sem `autonumber` | Mantém a numeração do PDF, que as specs citam |
| Criação de `GameSession` («create») | `create participant` | |

## Pendências por diagrama

Mudanças já registradas nas specs (seção "Divergências", IDs D) e nas ADRs, à espera do PR que as aplica. As specs seguem as recomendações das ADRs, todas em `Proposta`; se a equipe decidir diferente, a seção "Impacto" da ADR diz o que muda aqui. Os M são as melhorias do [planejamento](../../planejamento-sdd.md).

### Casos de uso

| Item | Mudança | Spec ou ADR |
|---|---|---|
| S05 D3 | UC1: "offline" vira "mesmo dispositivo" | [ADR 0006](../adr/0006-modo-offline.md) A, M14 |
| M10 | Acrescentar UC11 Abandonar partida | S01 Q1, S03 |
| M12 | O investigador também inicia a partida, no solo | S03 Q6 |
| M13 | Remover o «extend» entre UC7 e UC6 | decisão da equipe |
| M11 | Acrescentar UC12 Entrar em partida | S06 |
| ADR 0002 | A nota "UC5 no solo executado pelo NarradorIA" passa a citar o caso de uso `ResponderComIA`, não a porta | [ADR 0002](../adr/0002-contrato-da-porta-narrador.md) A |

### Classes do domínio

| Item | Mudança | Spec ou ADR |
|---|---|---|
| S01 D1 a D6, D8 a D10 | `agora: Instant` em todo comando, `modo`, `Tentativa`, cancelar e retirar, nove eventos sob `EventoDePartida : DomainEvent`, H5 estendida, `aberta` com cancelamento, `fase`, `QuestionId` e `TentativaId` numerados | [S01](../../specs/001-dominio-partida/spec.md) |
| S02 D1 a D3, D6 a D8 | `StatusEnigma` e `OrigemEnigma`, `ConteudoEnigma`, `solucaoCompleta` pública com `toString` sem ela, `Player` sem comando, `CriterioDeSelecao`, restrição das estatísticas | [S02](../../specs/002-catalogo-e-jogador/spec.md) |
| S03 D7, D10 | `aplicarVeredito(autor, veredito, agora)`; `GameSession.reconstituir` | [S03](../../specs/003-casos-de-uso-da-partida/spec.md), ADR 0001 A |

### Componentes

| Item | Mudança | Spec ou ADR |
|---|---|---|
| S03 D8, D9 | Os doze casos de uso da S03; portas em `domain.gamesession.port` e `enigmasJogadosPor` | [S03](../../specs/003-casos-de-uso-da-partida/spec.md) |
| S04 D1 | "difusão de DomainEvent" vira `PublicadorDeEventos` em `domain.shared.port` | [S04](../../specs/004-eventos-apos-persistir/spec.md) |
| S05 D4 | `main` no `bootstrap`, que monta os adaptadores e a CLI | [S05](../../specs/005-cli-local/spec.md), M16 |
| ADR 0001 | Tirar "(fornecedor a decidir)" de `VerificadorDeSolucao` e anotar "só no solo" | [ADR 0001](../adr/0001-veredito-no-modo-grupo.md) A |
| ADR 0002 | Remover o adaptador `NarradorHumano` | [ADR 0002](../adr/0002-contrato-da-porta-narrador.md) A, M2 |
| M1 | Trocar a nota "solucaoCompleta não sai do domínio" pela visão por papel | P3 da constituição |
| M23 | Padronizar o acesso do front-end com a implantação (pelo proxy) | decisão da equipe |
| ADR 0007 | Emissão e verificação de token | [ADR 0007](../adr/0007-identidade-e-autenticacao.md), S06 |

### Implantação

| Item | Mudança | Spec ou ADR |
|---|---|---|
| S05 D1 | Tirar o SSH: a CLI roda local, no mesmo processo | [S05](../../specs/005-cli-local/spec.md), M15 |
| S04 D4 | `messaging` tem só o publicador em memória até a S06 | [S04](../../specs/004-eventos-apos-persistir/spec.md) |
| ADR 0002 | Remover o `NarradorHumano` | [ADR 0002](../adr/0002-contrato-da-porta-narrador.md) A |
| ADR 0004 | Nomear o banco | [ADR 0004](../adr/0004-banco-de-dados.md) |
| ADR 0005 | Nomear o provedor de LLM | [ADR 0005](../adr/0005-provedor-de-llm.md) |
| ADR 0007 | Emissão e verificação de token | [ADR 0007](../adr/0007-identidade-e-autenticacao.md) |
| ADR 0008 | Nomear o artefato front-end | [ADR 0008](../adr/0008-stack-do-front-end.md) |
| M23 | Mesmo caminho de acesso do diagrama de componentes | decisão da equipe |

### Pacotes

| Item | Mudança | Spec ou ADR |
|---|---|---|
| S00 D1, D2 | Módulo `bootstrap` com seta para as quatro camadas; regra de imports do domínio | [S00](../../specs/000-fundacao/spec.md), M16 |
| S02 D4, D9 | Portas em `domain.<agregado>.port`; `persistence.memoria` e `persistence.seed` | [S02](../../specs/002-catalogo-e-jogador/spec.md) |
| S03 D9 | Portas da partida em `domain.gamesession.port` | [S03](../../specs/003-casos-de-uso-da-partida/spec.md) |
| S04 D1, D6 | `domain.shared.port`; `application.ouvinte` | [S04](../../specs/004-eventos-apos-persistir/spec.md) |
| S05 D2, D4 | `interfaces.cli`; `main` no `bootstrap` | [S05](../../specs/005-cli-local/spec.md) |

### Estados da partida

| Item | Mudança | Spec ou ADR |
|---|---|---|
| S01 D1, D4, D6, D7 | `agora` nos comandos; `cancelarPergunta` e `retirarProposta` voltam a `AguardandoPergunta`; notas com H5 estendida, H2 definida e R3 | [S01](../../specs/001-dominio-partida/spec.md), M3 |
| S03 D7 | Guarda `[autor = narrador]` em `aplicarVeredito` | [S03](../../specs/003-casos-de-uso-da-partida/spec.md), ADR 0001 A |
| S04 D3 | H6 no quadro "Regras da partida" | [S04](../../specs/004-eventos-apos-persistir/spec.md), M9 |

### Sequência 1/2: iniciar partida

| Item | Mudança | Spec ou ADR |
|---|---|---|
| S02 D5 | Passo 03: `publicados()`, filtro do `CriterioDeSelecao` e sorteio na aplicação | [S02](../../specs/002-catalogo-e-jogador/spec.md), M18 |
| S03 D1 a D4 | Entrada de `IniciarPartida`; narrador como `PlayerRef`; `criar` com ID e instante; visão por papel | [S03](../../specs/003-casos-de-uso-da-partida/spec.md) |
| S04 D7 | Publicar `PartidaIniciada` depois de salvar e reler antes de responder | [S04](../../specs/004-eventos-apos-persistir/spec.md) |
| M11 | Separar criar (sala de espera) de iniciar | S06 |

### Sequência 2/2: investigar e resolver

| Item | Mudança | Spec ou ADR |
|---|---|---|
| S03 D5, D6, D7 | Narrador humano responde direto; verificador só no solo; `aplicarVeredito` com autor | [S03](../../specs/003-casos-de-uso-da-partida/spec.md), ADR 0001 A, ADR 0002 A |
| S04 D2, D5 | `PerguntaFeita` publicado depois de salvar, com o ouvinte do solo; "salva, publica e relê" na nota | [S04](../../specs/004-eventos-apos-persistir/spec.md) |
| S01 D4 | `cancelarPergunta` e `retirarProposta` | [S01](../../specs/001-dominio-partida/spec.md), M3 |
