# Pendências

O que depende da equipe ou de alguém fora do código: decisões, aprovações, questões com decisão provisória e configurações manuais. Quem fecha um item atualiza a linha aqui no mesmo PR (passo 4 do [fluxo das ADRs](adr/README.md)).

Situação em 26/10/2026: nenhuma ADR decidida, nenhuma spec aprovada, nenhum código.

## Decisões da equipe (ADRs)

Todas em `Proposta`. Enquanto não forem decididas, as specs seguem a recomendação (P8 da [constituição](constitution.md)); a seção "Impacto" de cada ADR diz o que muda se a escolha for outra.

| ADR | Pergunta | Recomendação | Quem segue a recomendação hoje | Bloqueia |
|---|---|---|---|---|
| [0001](adr/0001-veredito-no-modo-grupo.md) | Quem dá o veredito no modo grupo? | A: narrador humano no grupo, IA só no solo | S01 Q3 (emenda 1 da S03), S03 Q1, S04 | S03, S08 |
| [0002](adr/0002-contrato-da-porta-narrador.md) | A porta `Narrador` é síncrona? | A: porta só para a IA; o humano responde pelo caso de uso | S03 Q2, S04 Q4 | S03, S04, S07 |
| [0003](adr/0003-framework-web.md) | Ktor ou Spring Boot? | A: Ktor, salvo se a maioria da equipe já dominar Spring | ninguém ainda | S06 |
| [0004](adr/0004-banco-de-dados.md) | Qual banco? | A: PostgreSQL com Flyway | S04 Q5 (outbox na S09) | S09 |
| [0005](adr/0005-provedor-de-llm.md) | Qual provedor de LLM e com que orçamento? | A: API comercial com saída estruturada, provedor escolhido pela bateria da S07 e teto de tokens por partida | ninguém ainda | S07, S08, S11 |
| [0006](adr/0006-modo-offline.md) | O que é o modo "offline"? | A: um dispositivo, todos juntos | S05 Q1 | S05, UC1 |
| [0007](adr/0007-identidade-e-autenticacao.md) | Como identificar e autenticar jogadores? | A: convidado com apelido; C se a equipe quiser estatísticas persistentes | S03 Q7 | S06, S12 |
| [0008](adr/0008-stack-do-front-end.md) | Qual stack de front-end? | Decidir no começo da S10 | ninguém ainda | S10 |

As que mais travam: 0001, 0002 e 0006, porque as specs do marco M1 (S00 a S05) já dependem delas. A 0007 trava a S06.

## Aprovações e andamento

| Documento | Status | Plano e tarefas | Observação |
|---|---|---|---|
| [Constituição](constitution.md) 1.0.0 | Rascunho para aprovação | — | Mudança depois de aprovada só por ADR |
| [S00 Fundação](../specs/000-fundacao/spec.md) | Rascunho | Escritos | Primeira a implementar |
| [S01 Domínio da partida](../specs/001-dominio-partida/spec.md) | Rascunho | Escritos | Recebe as emendas 1 e 2 da S03 (`aplicarVeredito` com autor, `reconstituir`) |
| [S02 Catálogo e jogador](../specs/002-catalogo-e-jogador/spec.md) | Rascunho | Escritos | Pode andar junto com a S01 |
| [S03 Casos de uso da partida](../specs/003-casos-de-uso-da-partida/spec.md) | Rascunho | A escrever | |
| [S04 Eventos depois de persistir](../specs/004-eventos-apos-persistir/spec.md) | Rascunho | A escrever | Emendas 1 e 2 à S03 |
| [S05 CLI local](../specs/005-cli-local/spec.md) | Rascunho | A escrever | Fecha o marco M1 |

Implementação (cada spec marca a sua no PR que a conclui):

- [ ] S00 implementada
- [ ] S01 implementada
- [ ] S02 implementada
- [ ] S03 implementada
- [ ] S04 implementada
- [ ] S05 implementada (marco M1: jogo inteiro no terminal)

## Questões abertas das specs

Cada uma tem decisão provisória para não travar a implementação. Se a equipe decidir diferente, a spec diz quais CAs mudam.

### [S00](../specs/000-fundacao/spec.md)

| Q | Pergunta | Decisão provisória |
|---|---|---|
| Q1 | `interfaces` pode importar tipos do domínio? | Não; a aplicação expõe DTOs e códigos de erro próprios |
| Q2 | Konsist ou ArchUnit? | Konsist |
| Q3 | Onde roda a CI? | GitHub Actions |
| Q4 | Formatador ou lint? | Nenhum na S00; entra por ADR se a equipe quiser |
| Q5 | Qual JDK? | Temurin 25 |

### [S01](../specs/001-dominio-partida/spec.md)

| Q | Pergunta | Decisão provisória |
|---|---|---|
| Q1 | Quem pode abandonar (H2)? | Narrador sempre; investigador só no `SOLO` |
| Q2 | Quem cancela pergunta ou retira proposta? | O autor do item ou o narrador |
| Q3 | Quem aplica o veredito no grupo? | Na S01, o domínio não verifica o autor. A S03 adota a opção A da ADR 0001: só o narrador (emenda 1, CA-33) |
| Q4 | Limite de texto | 1 a 500 code points depois de aparar |
| Q5 | Regras extras de H5 | Narrador fora dos investigadores, sem repetidos, `SOLO` com exatamente um |

### [S02](../specs/002-catalogo-e-jogador/spec.md)

| Q | Pergunta | Decisão provisória |
|---|---|---|
| Q1 | Limites de texto do enigma | Título de 1 a 100, enunciado e solução de 1 a 2000 code points |
| Q2 | O solo pode jogar enigma não revisado (M19)? | Não; só `PUBLICADO` é elegível |
| Q3 | O que é "já jogado pelo mesmo grupo" (M18)? | O domínio recebe o conjunto pronto; a S03 calcula (S03 Q5) |
| Q4 | Limite do nome do jogador | 1 a 40 code points; nomes repetidos permitidos |
| Q5 | Forma das tags | Aparadas, minúsculas, sem vazias nem repetidas |
| Q6 | Conteúdo do seed | Os cinco enigmas e o jogador IA de [seed.md](enigmas/seed.md), à espera de revisão |

### [S03](../specs/003-casos-de-uso-da-partida/spec.md)

| Q | Pergunta | Decisão provisória |
|---|---|---|
| Q1 | Quem dá o veredito no grupo? | ADR 0001 A: `JulgarProposta` pelo narrador; `AvaliarComIA` só no solo |
| Q2 | A porta `Narrador` é síncrona? | ADR 0002 A: síncrona só para a IA |
| Q3 | O investigador vê a solução depois de abandonar? | Sim, em qualquer estado terminal |
| Q4 | Os testes de `application` podem usar os repositórios em memória? | Sim, como dependência de teste |
| Q5 | Como calcular "já jogados"? | Enigmas de partidas, em qualquer status, em que algum investigador da partida nova foi investigador |
| Q6 | Quem inicia no solo (M12)? | O próprio investigador |
| Q7 | Como um jogador passa a existir antes da S06? | `RegistrarJogador(nome)` cria um convidado humano, sem senha |

### [S04](../specs/004-eventos-apos-persistir/spec.md)

| Q | Pergunta | Decisão provisória |
|---|---|---|
| Q1 | O despacho é síncrono ou assíncrono? | Síncrono, uma fila por thread |
| Q2 | Um ouvinte que falha é chamado de novo? | Não; a falha é registrada e a entrega segue |
| Q3 | A visão de retorno é a de antes ou a de depois dos ouvintes? | A de depois: o caso de uso relê a partida |
| Q4 | Quem reage a `PerguntaFeita` no grupo? | ADR 0002 A: ninguém nesta spec |
| Q5 | E se o processo cair entre `salvar` e `publicar`? | O evento se perde; o outbox entra na S09 |

### [S05](../specs/005-cli-local/spec.md)

| Q | Pergunta | Decisão provisória |
|---|---|---|
| Q1 | O que é o modo "offline"? | ADR 0006 A: o grupo num só dispositivo |
| Q2 | A CLI joga o solo? | Não até a S07 |
| Q3 | Como o narrador consulta a solução numa tela compartilhada? | Só por `solucao`, com aviso e confirmação |
| Q4 | Como se decide quem está digitando? | Vez explícita, com `vez <nome>` |
| Q5 | O mesmo nome em duas partidas é o mesmo jogador? | Sim, na sessão do programa, pelo nome normalizado |
| Q6 | Sair no meio da partida a registra como abandonada? | Não |

## Melhorias à espera da equipe

Melhorias do [planejamento](../planejamento-sdd.md) (seção 7) que ainda pedem uma decisão ou não têm spec dona. As demais já entraram numa spec ou têm spec no backlog.

| M | Assunto | Onde está | O que falta |
|---|---|---|---|
| M10 | Caso de uso "Abandonar partida" e quem pode abandonar | S01 Q1 (H2 provisória), S03 (`Abandonar`) | Confirmar H2 e acrescentar o UC11 ao diagrama de casos de uso |
| M12 | Quem inicia no solo | S03 Q6 | Confirmar e ligar o Investigador ao UC3 no diagrama de casos de uso |
| M13 | `«extend»` entre UC7 e UC6 | Só no diagrama de casos de uso | Decidir se a relação sai |
| M15 | Acesso SSH sem adaptador | S05 D1 e D2 (CLI local, sem SSH) | Confirmar que não haverá SSH em produção |
| M18 | Critério de seleção e "já jogados" | S02 Q3, S03 Q5, S05 Q5 | Confirmar a regra provisória |
| M22 | Veredito só booleano | Nenhuma spec | Decidir se entra um retorno como "quase"; se entrar, muda `Veredito` (S01) e a S08 |
| M23 | Componentes e implantação divergem no acesso do front-end | Nenhuma spec | Padronizar (pelo proxy, como na implantação) |
| M24 | Sem limites nem pontuação | Fora de escopo da S03 | Decidir se entra e em qual spec |

## Tarefas manuais

- [ ] **Proteger a `main`** no GitHub, exigindo o check `build` da CI, depois que a tarefa 4 da [S00](../specs/000-fundacao/tasks.md) criar o workflow.
- [ ] **Escolher a licença.** O repositório é público; sem um arquivo `LICENSE`, ninguém tem permissão de reutilizar o código.
- [ ] **Revisar o seed** em [docs/enigmas/seed.md](enigmas/seed.md): enunciados, soluções e o jogador IA (S02 Q6).
- [ ] **Gerar os PDFs** a partir das fontes Mermaid de [docs/diagramas/](diagramas/README.md), quando a equipe quiser trocar os originais de 14/10/2026.
- [ ] **Convidar os colaboradores** para o repositório.
- [ ] **Depois da ADR 0005:** criar a conta no provedor e guardar a chave fora do repositório (opção A) ou provisionar a máquina do modelo local (opção B).

## Histórico

| Data | Mudança |
|---|---|
| 26/10/2026 | Primeira versão: ADRs 0001 a 0008, questões da S00 à S05, melhorias sem dono e tarefas manuais |
