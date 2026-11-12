<sub>[← Voltar ao índice](../README.md)</sub>

# FinFlow

**Finanças e agenda da família pelo WhatsApp, em produção. Um agente de IA com 55 ferramentas em que o modelo interpreta a mensagem e o código valida, grava e confere o que foi respondido.**

SaaS brasileiro em produção, com clientes pagantes: [finflows.com.br](https://www.finflows.com.br/). A pessoa escreve, manda áudio ou foto de comprovante no WhatsApp e o agente registra despesas, contas a pagar, cartões, metas e compromissos. Um painel web (PWA) mostra o mesmo estado para revisar, organizar e planejar.

![Painel do FinFlow com saldo, valor pendente, insight de gastos e últimas transações](assets/painel.png)

<sub>Todas as telas usam uma família de demonstração com dados fictícios. O código-fonte é privado. Os trechos abaixo são simplificados.</sub>

---

## Resumo

| | |
|---|---|
| **Problema** | Controle financeiro morre porque registrar dá trabalho. Planilha e app exigem disciplina, mas mandar mensagem no WhatsApp a pessoa já faz |
| **Meu papel** | Fundador. Produto, arquitetura, desenvolvimento full stack, operação, suporte e cobrança, conduzidos com agentes de IA |
| **Stack** | TypeScript · Node.js · Fastify · Genkit + Gemini · Firestore · Cloud Run · Cloud Functions · Cloud Tasks · React + Vite (PWA) · Zustand · Dexie · Asaas · Google Calendar |
| **Dimensão técnica** | 589 commits em 10 meses · 55 ferramentas do agente · 97 arquivos de teste · ~140 mil linhas de TypeScript |
| **Estágio** | Em produção desde janeiro de 2026, com assinaturas pagas via Pix, boleto e cartão |

**Destaques**

- **O código confere o modelo:** um classificador manda ao modelo só as ferramentas do assunto, e uma conferência compara a resposta com o que foi executado, desfaz o que a tentativa gravou e tenta de novo. [→ Como o agente funciona](#como-o-agente-funciona)
- **Erros que só a conversa real mostra:** pergunta tratada como comando, patrimônio como receita, consulta como lançamento. Corrigidos na ferramenta, não no prompt. [→ Conversas reais](#o-que-as-conversas-reais-ensinaram)
- **Custo unitário medido em produção:** cerca de R$ 0,046 de IA por mensagem, ~8% do ticket numa família típica, e as decisões de infraestrutura que mantêm esse número baixo. [→ Decisão de base](#decisão-de-base-infraestrutura-proporcional-ao-estágio)
- **Dinheiro contado uma vez só:** prévia e confirmação sobre o mesmo conjunto, meta como reserva de saldo, idempotência que cobre todos os efeitos da cobrança. [→ Decisões técnicas](#decisões-técnicas)
- **Testes para um sistema não determinístico:** corpus de regressão do agente, guardas testadas com texto real de produção e validação pelo estado gravado. [→ Testes](#como-testo-um-agente-que-não-é-determinístico)
- **Operação real:** o postmortem das 22 horas em que o bot ficou fora do ar sem ninguém perceber. [→ Operação](#operação-o-dia-em-que-o-bot-ficou-mudo-por-22-horas)

## Contexto

Fazer um modelo de linguagem entender "gastei 50 no mercado" é a parte fácil. O difícil é que **o modelo interpretou, a ferramenta executou e o usuário leu** são três coisas diferentes, e num produto financeiro elas precisam coincidir. Um bot que responde "registrado!" sem gravar, que altera 20 linhas depois de mostrar 18 ou que conta duas vezes o mesmo dinheiro perde a confiança do usuário na primeira vez.

Por isso a regra de negócio fica nas ferramentas, não no prompt. O modelo escolhe a intenção e preenche os parâmetros. Quem valida, calcula, aplica permissões e decide o que pode ser gravado é o código, que também confere a resposta do modelo antes de ela chegar ao usuário.

## Arquitetura

```mermaid
flowchart LR
    WA["WhatsApp<br/>texto · áudio · imagem"] --> EV["Evolution API"]
    EV -- "webhook + X-API-KEY" --> API["Backend Fastify<br/>Cloud Run"]
    API --> AG["Agente Genkit<br/>Gemini + 55 ferramentas"]
    AG --> TL["Ferramentas<br/>validação · permissões · confirmação"]
    TL --> DB[("Firestore<br/>isolado por família")]
    PWA["Painel React (PWA)<br/>cache IndexedDB + sync"] <--> DB
    DB --> FN["Cloud Functions<br/>triggers · crons"]
    FN --> CT["Cloud Tasks<br/>avisos no horário de cada pessoa"]
    API <--> EXT["Asaas · Google Calendar"]
```

Três fronteiras orientam o resto do sistema:

- **Multi-tenant por família.** Todo dado vive em `families/{familyId}/…`. O painel lê o Firestore direto, e nesse caminho as Security Rules restringem os dados à família e ao papel do usuário, com suíte própria que roda contra o emulador. O backend usa o Admin SDK, que não passa pelas Rules, então ali o escopo por família é aplicado pelo código em toda consulta.
- **O agente propõe, a ferramenta decide.** Cada ferramenta valida a entrada e devolve erros estruturados (`DUPLICATE_SOFT`, `LIKELY_QUESTION_NOT_COMMAND`, …) que dizem ao modelo o que perguntar. A resposta final é conferida contra o que as ferramentas executaram.
- **Painel local-first.** O PWA guarda transações, eventos e cadastros em IndexedDB e sincroniza só o que mudou desde a última visita, com fila para operações feitas offline.

## Como o agente funciona

Cada mensagem passa pelo mesmo caminho. O modelo participa em um ponto só: escolher ferramentas e redigir a resposta. Antes dele, o código decide o que ele pode usar. Depois dele, o código confere o que ele disse.

```mermaid
flowchart TD
    M["Mensagem do WhatsApp"] --> L["Trava por número<br/>uma mensagem por vez"]
    L --> C["Contexto<br/>dados da família · histórico curto · listas recentes"]
    C --> K["Classificador por regex<br/>escolhe os grupos de ferramentas"]
    K --> G["Gemini, temperatura 0<br/>só com as ferramentas escolhidas"]
    G -- "pediu ferramenta fora da lista" --> G2["refaz uma vez<br/>com as 55"]
    G2 --> T
    G --> T["Ferramentas<br/>validam · gravam ou devolvem prévia"]
    T --> V{"A resposta bate com<br/>o que foi executado?"}
    V -- "sim" --> R["Resposta ao usuário<br/>+ telemetria"]
    V -- "não" --> RB["desfaz o que a<br/>tentativa criou"]
    RB --> N{"Há outro modelo?"}
    N -- "sim" --> G
    N -- "não" --> E["erro genérico,<br/>sem confirmar nada"]
```

### Só as ferramentas do assunto

**Problema.** As 55 ferramentas somam cerca de 20 mil tokens só de definições, e cada rodada de chamada de ferramenta paga isso de novo. Além do custo, o modelo leve escolhe pior quando tem dezenas de opções parecidas (`cancelLastTransaction`, `cancelTransactions`, `cancelLastCreditCardExpense`…).

**Decisão.** Antes de chamar o modelo, um classificador por regex escolhe os grupos de ferramentas da mensagem, sem chamada extra de IA e sem latência. São 7 grupos: transações, metas, recorrências, agenda, cartão, relatórios e ajuda. O caso difícil são as mensagens curtas, porque "sim" ou "o segundo" não dizem o assunto. Por isso a escolha soma quatro sinais:

1. Padrões da própria mensagem, tolerantes a erro de digitação ("evnetos") e a frases sem palavra-chave ("guardei 200 pro carro" traz as ferramentas de meta).
2. O grupo da ferramenta chamada no turno anterior, para que o "sim" de uma confirmação alcance a ferramenta que fez a pergunta.
3. Listas mostradas ao usuário nos últimos 180 segundos, para que "1 e 3" alcance a ferramenta da lista.
4. A última resposta do bot, quando a mensagem não traz pista nenhuma.

Transações e ajuda entram sempre, porque cobrem a maior parte das mensagens. Uma chamada típica leva entre 16 e 30 das 55 ferramentas.

**A rede de segurança.** Regex tem buracos. "Torneio de vôlei" não parecia evento, e o modelo pedia `createEvent`, que não estava na lista. O Genkit acusa ferramenta não encontrada, e o fluxo refaz a chamada uma vez com as 55. O buraco custa uma segunda chamada, em vez de uma mensagem perdida.

**Como evoluiu.** Nos primeiros dias da telemetria, 4,3% das mensagens caíam nessa rede. No acumulado desde 19/04 são 1,4%. Cada ocorrência fica registrada com a ferramenta que faltou e o começo da mensagem, e vira padrão novo. Em outubro, "todo dia 25 pago 9,99" rodou sem as ferramentas de recorrência. O caso mostrou que o buraco nem sempre é inofensivo: sem a ferramenta certa à mão, o modelo usou a mais parecida e gravou uma regra de categoria que ninguém pediu. Desde então, frases com dia fixo trazem as recorrências na primeira tentativa.

**Por que não um roteador com IA.** Um classificador feito por modelo acertaria mais frases, mas somaria uma chamada e sua latência a todas as mensagens. A regex paga uma segunda chamada em 1,4% delas, e o preço dela é manutenção: cada buraco precisa ser visto na telemetria e virar padrão.

### A resposta é conferida contra o que foi executado

**Problema.** Às vezes o modelo escreve a confirmação sem ter feito a ação. Ele copia o formato de uma confirmação anterior, responde "pronto" em cima de uma prévia que ainda espera o "sim", ou diz "registrei 5" depois de 3 chamadas. Num app financeiro, o usuário acredita no que lê e não confere.

**Decisão.** Depois que o modelo responde, e antes de a mensagem sair, o código compara o texto com os resultados das ferramentas daquele turno. Cada caso abaixo nasceu de uma falha vista em produção ou em teste:

| A resposta diz | Mas no turno |
|---|---|
| "registrada", "agendei", "encontrei 2 eventos" | nenhuma ferramenta foi chamada |
| "saldo ajustado" | o ajuste não foi confirmado com sucesso |
| "registrei 5 transações" | menos de 5 chamadas deram certo, e o número não veio do resultado de uma ferramenta de lote |
| "registrei tudo" | nenhuma ferramenta foi chamada |
| "R$ 664,00 registrado no cartão" | todas as ferramentas falharam (faltava escolher o cartão) |
| "✅ Pronto, atualizadas!" | toda ferramenta devolveu só uma prévia, e prévia nunca grava |

Quando a conferência reprova a resposta:

- **Desfaz o que a tentativa criou.** As ferramentas gravam durante a geração, antes da conferência. Sem desfazer, a nova tentativa duplicaria o lançamento. Os registros criados naquela tentativa são desativados pelos IDs que as próprias ferramentas devolveram. Alterações e cancelamentos em lote não precisam disso, porque só gravam depois do "sim" do usuário.
- **Tenta de novo no modelo seguinte**, com um timeout que cancela a chamada de verdade, para que uma tentativa abandonada não continue gravando enquanto a próxima roda.
- **Se as tentativas acabam, falha de forma visível.** O usuário recebe um erro genérico e pode reenviar. É melhor dizer que não deu do que confirmar algo que não existe.

**O preço: alarme falso.** Uma regex sobre texto livre erra para os dois lados, e aqui o erro caro é o alarme falso. Uma resposta honesta reprovada vira chamada extra ou, no pior caso, o erro genérico para um cliente que não fez nada de errado. Numa revisão em 30/09, 5 das 6 detecções do caso genérico nos 30 dias anteriores eram alarme falso. As correções foram de precisão, não de remoção:

- Negações ("nada foi registrado") e perguntas sem número saem do texto conferido.
- Frases no futuro ("será registrada uma receita de ajuste") descrevem a prévia e não contam como afirmação.
- Os padrões de "gravou" são estreitos de propósito. Antes de ligar o caso das prévias, conferi que nenhuma das 13 mensagens de prévia do código dispara o padrão.
- Cada caso de produção, verdadeiro ou falso, virou teste com o texto original.

### Em produção

| Desde 19/04/2026, sem a família de testes | |
|---|---|
| Mensagens processadas | 4.470 |
| Tentativas refeitas com as 55 ferramentas (o classificador errou) | 62 (1,4% das mensagens) |
| Tentativas refeitas pela conferência da resposta, alarmes falsos incluídos | 132 (~3%) |
| Mensagens atendidas pelo modelo de fallback | 140 (3,1%) |
| Duplicatas exatas bloqueadas pela ferramenta | 28 |

<sub>Contadores gravados pelo fluxo a cada mensagem, por família e no total, lidos em 12/11/2026.</sub>

## O que as conversas reais ensinaram

Os casos mais caros não apareceram em teste. Apareceram lendo conversas reais: em maio, uma auditoria das conversas de 12 famílias pagantes gerou 24 correções numa única rodada. O padrão se repete. O modelo entende a frase, mas confunde o **tipo** de pedido. A correção vai para a ferramenta, que recusa com um erro estruturado e devolve ao modelo o que perguntar.

| Mensagem | O que acontecia | Hoje |
|---|---|---|
| "Meu saldo é de 570?" | Virava ajuste de saldo e criava uma transação para zerar a diferença | A ferramenta de ajuste reconhece a pergunta (`LIKELY_QUESTION_NOT_COMMAND`) e pergunta se é consulta ou ajuste |
| "Tenho R$ 2.000 em título de capitalização" | Virava receita e inflava a conta corrente | Declaração de patrimônio não é movimento (`LIKELY_ASSET_STATEMENT`) |
| "Puxe o histórico das minhas despesas" | Criou três transações a partir das palavras da frase | Pedido de consulta sem verbo de criação é recusado pelas ferramentas que gravam (`LIKELY_HISTORY_REQUEST`) |
| "farmácia R$ 92" logo depois de "saída de R$ 92" | Duas despesas para a mesma compra | Mesmo valor, conta e tipo em 3 minutos: `DUPLICATE_SOFT` pergunta se é engano, correção ou outra compra |
| Duas mensagens seguidas do mesmo número | Processadas em paralelo, o texto de uma foi parar na descrição da outra | Uma mensagem por vez por número |

O ciclo é sempre o mesmo: ler a conversa e o estado gravado, classificar a falha, pôr a regra no código, escrever o teste com a frase real e acompanhar o erro estruturado na telemetria. Uma instrução no prompt é a última linha de defesa, não a primeira.

---

## Decisões técnicas

### 1. A família ensina a categoria, sem mudar o produto para os outros

**Problema.** Categoria errada era a principal queixa de quem cancelava. Num pequeno comércio que usa o FinFlow com funcionários, um deles pediu: *"toda vez que eu falar paguei, coloca em Contas"*. O bot concordou e esqueceu, porque a memória de conversa é curta e não existia operação para guardar uma preferência. Na semana seguinte, a família corrigiu à mão o que o modelo classificou por significado.

**Decisão.** Uma regra global resolveria aquele cliente e estragaria a organização de todos os outros. As regras passaram a ser **por família e gravadas na própria categoria**. O documento já é lido em toda mensagem, então a regra não custa leitura extra, não pede Security Rule nova e some junto com a categoria.

A aplicação segue uma escada, do mais específico ao mais geral:

1. Categoria dita na frase vence qualquer regra ("paguei o aluguel, põe em Moradia").
2. Texto ensinado encontrado **literalmente na mensagem original**. A checagem roda no código, porque "paguei" é um verbo que nunca chega à descrição que o modelo extrai.
3. Regra por membro ("tudo que o João mandar vai para Contas").
4. Regra semântica ("tudo que for fruta vai para Feira"): uma chamada auxiliar curta só pergunta se o item pertence ao grupo. Ela tem timeout de 8 s e, em caso de erro ou dúvida, grava como antes.

Na bateria de desenvolvimento, só o modelo principal acertou 3 de 6 frutas. Com a chamada auxiliar foram 8 de 8, e itens fora do grupo (parafuso, sabão, ração) continuaram sem categoria.

**O efeito colateral.** A frase de ensinar tem o mesmo formato de uma correção ("coloca isso em Contas"), e o agente reescrevia a última transação. A instrução no prompt perdeu para o modelo mais barato, então a ferramenta de correção passou a recusar com `LIKELY_TEACHING` quando a mensagem traz um marcador de permanência ("toda vez", "sempre que").

<table>
<tr>
<td width="42%"><img src="assets/conversa-regra-de-categoria.png" alt="Conversa no WhatsApp: criação da categoria Contas, regra 'toda vez que eu falar paguei' e despesa de internet registrada em Contas"></td>
<td width="58%"><img src="assets/lancamentos.png" alt="Tela de transações com internet em Contas, ônibus e Uber recategorizados para Lazer e a conta de luz agendada"></td>
</tr>
<tr>
<td><sub>A regra é ensinada na conversa e vale para todo o grupo</sub></td>
<td><sub>O painel mostra o resultado: internet em Contas, a luz ainda agendada</sub></td>
</tr>
</table>

### 2. A confirmação altera exatamente o que a prévia mostrou

**Problema.** Operações em lote ("muda a categoria das despesas de transporte para Lazer") mostram uma prévia e esperam um "sim". Em teste, a prévia listou 18 despesas e a confirmação gravou nas 20 linhas da busca, carimbando categoria de despesa em duas receitas. Os dois caminhos filtravam de formas diferentes, e outro corte fazia a prévia mostrar 10 itens enquanto o "sim" alterava 20.

**Decisão.** Prévia e confirmação passaram a usar a mesma função de estreitamento (`narrowToFilters`). Tipo, valor e data são filtros rígidos, e um índice dito pelo usuário vence qualquer filtro inferido pelo modelo. Quando a busca atinge o limite de varredura, a resposta diz "ao menos N" em vez de prometer um total que o sistema não conhece.

**Como validei.** O teste de regressão não verifica a mensagem de sucesso. Ele compara as linhas gravadas com as que deveriam ficar intactas, e falha sem a correção.

<table>
<tr>
<td width="42%"><img src="assets/conversa-edicao-em-lote.png" alt="Bot lista as duas despesas de transporte, pede 'sim' e só então altera a categoria"></td>
<td width="58%">

**A lista que o usuário viu é a verdade.** A causa raiz recorrente desse tipo de bug era refazer a busca na hora de confirmar. Hoje a confirmação parte do conjunto exibido, aplica os mesmos filtros e só então grava.

O salário e as demais despesas da tela de transações acima continuaram com as categorias originais.

</td>
</tr>
</table>

### 3. Guardar dinheiro na meta não é receber dinheiro

**Problema.** "Guardei 400 na meta da viagem" virava uma receita vinculada à meta. O dinheiro já estava na conta e passava a ser contado duas vezes.

**Decisão.** A meta passou a ser uma **parte nomeada do saldo**. Reservar e liberar geram movimentos próprios (`goals/{id}/movements`) e atualizam o valor da meta na mesma transação do Firestore, sem criar transação nem alterar os totais. Receitas e despesas realmente vinculadas a uma meta continuam movendo saldo e meta. A consulta de saldo mostra quanto dele está reservado.

O mesmo princípio vale para o cartão de crédito. A compra é uma obrigação que entra na fatura, e só o pagamento da fatura é saída de caixa. Tratar as duas como despesas independentes dobraria os gastos do mês.

### 4. Lançar sem ver o caixa

**Problema.** Um cliente com comércio queria que funcionários registrassem vendas e gastos sem enxergar saldo, relatórios nem lançamentos dos outros.

**Decisão.** Um quinto papel, **Operador**, com `canViewReports: false`. O desafio não é esconder um card no painel, e sim garantir que nenhum caminho revele valores: resposta do agente, notificação automática, relatório exportado. Duas regras sustentam isso:

- **Um único helper (`canSeeMoney`) com fail-open para campo ausente.** Só nega quando a permissão é explicitamente `false`. Assim, documentos antigos e objetos de permissão parciais não viram regressão para quem já usava o produto.
- **Leitura escopada por autoria nas Security Rules.** O Operador só lê transações com `userId` igual ao dele. Como o Firestore avalia a query, e não cada documento, toda consulta do painel desse papel precisa filtrar por `userId`.

### 5. Cobrança: evento repetido e evento atrasado

A cobrança da assinatura roda por webhooks do Asaas, e cada um deles produziu um bug que custava dinheiro ou confiança:

- **Evento atrasado.** No cartão, o Asaas envia `PAYMENT_CONFIRMED` na cobrança e `PAYMENT_RECEIVED` na liquidação, cerca de 32 dias depois. O evento de liquidação chegava após a cobrança do ciclo seguinte e recalculava o vencimento a partir do ciclo antigo, **expirando clientes que estavam pagando**. A correção é uma guarda monotônica: nenhum evento pode mover o vencimento para trás.
- **Evento repetido.** A idempotência (registro do evento dentro de uma transação do Firestore) já impedia reprocessar a assinatura, mas o aviso de cobrança no WhatsApp e o log de auditoria rodavam depois dela, sem condição. Um cliente recebeu o mesmo aviso duas vezes, com 24 s de intervalo.

```ts
let firstDelivery = false

await db.runTransaction(async (tx) => {
  firstDelivery = false                       // a transação pode ser reexecutada
  if ((await tx.get(eventRef)).exists) return // entrega repetida
  tx.create(eventRef, { eventId, processedAt, expiresAt: ttl90days })
  firstDelivery = true
  // ... atualiza a assinatura, nunca recuando expiresAt
})

if (firstDelivery && event === 'PAYMENT_OVERDUE') notifyOverdue(family)
```

Idempotência precisa cobrir **todos os efeitos** do fluxo, não só a escrita principal. Em desenvolvimento, o mesmo evento enviado duas vezes passou a gerar 1 mensagem e 1 auditoria (antes eram 2 e 2).

---

## Decisão de base: infraestrutura proporcional ao estágio

No início, a pergunta principal era de negócio, não técnica: alguém usa isso todo dia e paga por isso? A infraestrutura precisava permitir validar com custo próximo de zero, sem servidor para operar, e sem virar um teto se a resposta fosse sim.

O preço reforça essa restrição. O FinFlow custa **R$ 19,90 por mês, ou R$ 29,90 para uma família de até 5 pessoas**, e o custo de cada mensagem decide se a margem existe. Custo unitário é requisito de produto, não otimização para depois.

Medido nos logs de produção: cada mensagem custa em média **US$ 0,0084 (R$ 0,046) em IA**. Na família mediana, com 33 mensagens no mês, isso dá **R$ 1,52, ou ~8% do ticket**. A família mais ativa da amostra, com 109 mensagens, chega a R$ 5,02, ~25%. A IA é o maior custo variável do produto, e é por isso que as decisões abaixo existem.

<sub>Método: mensagens recebidas pelo chat entre 13/10 e 11/11/2026, sem a família de testes. Cada mensagem inclui todas as rodadas de chamada de ferramenta, e as rodadas anteriores à última foram contadas como se não tivessem cache, o que superestima o custo. Preço do Gemini 3.1 Flash-Lite (US$ 0,25 por milhão de tokens de entrada, US$ 1,50 por milhão de saída) e câmbio de R$ 5,50. Cobre só o modelo do chat: transcrição de áudio, leitura de comprovante e infraestrutura ficam de fora.</sub>

| Escolha | Por quê | O preço que aceitei | Como compensei |
|---|---|---|---|
| **Firestore (NoSQL)** | O free tier cobre a fase de validação. Auth, Security Rules, persistência offline e triggers vêm prontos | Dado financeiro é relacional. Somar o mês, agrupar por categoria ou cruzar conta com período seria um `SUM … GROUP BY` em SQL | Documentos de resumo mantidos por trigger, e cache local no painel com sincronização incremental, para não reler o que não mudou |
| **Cloud Run com zero instâncias mínimas** | Sem tráfego, a computação do serviço não custa nada | Cold start de alguns segundos na primeira requisição depois de um período ocioso, no WhatsApp e no painel | Backend em Fastify, escolhido pela inicialização rápida, e painel que abre pelo cache local |
| **Escala gerenciada** | Cloud Run, Cloud Functions e Cloud Tasks escalam sozinhos | Menos controle fino do que numa VM própria | Teto configurado em 10 instâncias × 80 requisições simultâneas, ainda sem teste de carga. Subir o teto é mudar uma flag, mas a capacidade real também depende da cota do Gemini, do Firestore e da integração com o WhatsApp |
| **Modelo leve por padrão** | Gemini Flash-Lite atende a maior parte das mensagens e dos comprovantes pelo menor custo por token | Modelos leves erram mais em instruções longas | Regras críticas ficam no código das ferramentas, e a resposta é conferida antes de sair. O modelo maior entra só como fallback, em ~3% das mensagens |
| **WhatsApp pela Evolution API, em VM gratuita** | A API oficial cobra pelas mensagens que a empresa inicia e exige template aprovado para cada uma. Lembretes e resumos diários pesariam muito num ticket de R$ 19,90 | Integração não oficial: o número pode ser desconectado ou bloqueado, e uma desconexão já tirou o bot do ar. Também é um servidor a mais, sem redundância | Isolada atrás de webhook com chave e trocável por variável de ambiente, sem mudar código. Migrar para a API oficial é o plano quando a receita cobrir o custo por mensagem |

<details>
<summary><b>Detalhe: o custo do NoSQL e o cold start</b></summary>

**O custo do NoSQL, pago no design.** O Firestore cobra por documento lido. Abrir o painel e somar todas as transações da família faria cada visita custar proporcionalmente ao histórico. Por isso o trigger `onTransactionWrite` mantém `stats/current` e `stats/monthly-{YYYY-MM}` a cada escrita. Ele calcula a diferença entre o antes e o depois do documento e separa saldo disponível (concluído) de pendente (agendado ou vencido). O painel lê um documento em vez de centenas.

Num banco relacional isso seria uma consulta com índice. Aqui virou código a manter: edição, exclusão, mudança de status e troca de conta precisam ajustar o resumo corretamente, e o resumo é atualizado de forma assíncrona. Foi uma troca consciente: complexidade de engenharia em vez de custo fixo, numa fase em que o custo fixo pesava mais.

**Cold start nos dois canais.** Com zero instâncias, a primeira requisição depois de um período ocioso espera alguns segundos enquanto o Cloud Run sobe o contêiner. No WhatsApp o atraso pesa menos, porque a pessoa manda a mensagem e segue o que estava fazendo. No painel pesa mais, e a mitigação é o cache local: quem volta vê na hora o último estado salvo, enquanto a sincronização espera o backend acordar. A primeira visita, sem cache, ainda sente o atraso. A saída é conhecida e tem preço: uma instância sempre ligada evita o cold start depois de ociosidade, em troca de custo fixo mensal, que passa a valer quando a base pagante cobrir esse custo. Instâncias novas, abertas numa expansão ou num deploy, continuam subindo a frio.

</details>

**O custo que mais pesa é o da IA.** Cada mensagem envia ao modelo cerca de 23 mil tokens de entrada na mediana (instruções, dados da família e definições das ferramentas), e cada rodada de chamada de ferramenta paga esse prefixo de novo. Perto disso, as leituras do Firestore são troco. Três mecanismos atacam esse custo:

- **Só as ferramentas do assunto.** O [classificador](#só-as-ferramentas-do-assunto) manda tipicamente de 16 a 30 das 55 ferramentas, não todas.
- **Consumo medido por família.** Toda mensagem registra tokens de entrada, de saída e servidos do cache, por modelo. O trial tem cota diária de tokens.
- **Cache de prompt calculado, não presumido.** O cache implícito do Gemini quase não era aproveitado entre mensagens, por causa da ordem do prompt. O explícito ainda não se paga com a frequência de uso atual. Reordenar o prompt é a próxima otimização, com estimativa de ~40% a menos no custo de entrada.

<details>
<summary><b>Detalhe: as contas do cache de prompt</b></summary>

**Implícito, que não custa nada para manter.** A documentação interna afirmava que ele já reduzia a fatura. Quando passei a registrar os tokens servidos do cache, a medição mostrou outra coisa: o desconto vinha quase todo de dentro da mesma mensagem, e entre mensagens o aproveitamento ficava entre 0% e 23%. A causa provável é a ordem do prompt, porque os dados da família ficam no meio dele e quebram o prefixo comum. Movê-los para o fim deixaria cerca de 11 mil tokens idênticos para todas as famílias, cobrados a 10% do preço. Isso a confirmar com mais amostras.

**Explícito, que só vale quando se paga.** O cache explícito garante o desconto em troca de uma taxa por hora para mantê-lo vivo. Com o 3.1 Flash-Lite, manter o cache custa cerca de US$ 1 por milhão de tokens por hora, e cada leitura economiza 90% do preço de entrada. Ele só compensa a partir de **~4,4 leituras por hora**, sustentadas enquanto o cache estiver vivo. Quem decide isso é a frequência de uso dos próprios usuários, não o total do mês: as pessoas lançam em rajadas, depois de uma compra ou ao conferir o saldo de manhã. Nos últimos 30 dias, só **22% das horas tiveram alguma mensagem**, e só **8% chegaram ao ponto de equilíbrio**. Nas outras horas, o cache seria pago para ficar ocioso.

Há também uma restrição estrutural: o cache congela o conjunto de ferramentas, e como o classificador gera combinações diferentes, cada combinação precisaria de um cache próprio. Por isso a ordem é aproveitar primeiro o implícito e ligar o explícito quando o uso passar do ponto de equilíbrio.

</details>

**Quando eu mudaria.** Os gatilhos para levar o núcleo financeiro a um banco relacional são dois: o custo de leitura do Firestore passar o de uma instância PostgreSQL gerenciada, ou relatórios com recortes livres (categoria × conta × período escolhido pelo usuário) exigirem mais documentos de resumo do que vale manter. A estrutura já ajuda essa migração: tudo vive sob `families/{familyId}`, o que se traduz direto numa coluna `family_id`.

---

## Outras entradas: foto e agenda

Comprovantes passam por um fluxo de visão, que extrai valor, tipo, destinatário e data e **mostra uma prévia antes de gravar**. Compromissos viram eventos com lembrete, sincronizados com o Google Calendar e avisados pelo WhatsApp no fuso da família.

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuário
    participant B as Backend (Cloud Run)
    participant G as Gemini Vision
    participant S as Sessão da conversa
    participant F as Dados da família
    U->>B: foto do comprovante (webhook)
    B-->>U: "📸 Deixa eu dar uma olhada..."
    B->>G: extrair valor, tipo, destinatário e data
    G-->>B: dados extraídos
    B->>S: guarda o comprovante pendente
    B-->>U: prévia + "sim, editar ou cancelar"
    U->>B: "sim" (webhook)
    B->>S: há comprovante pendente?
    Note over B: parser determinístico reconhece a resposta, sem chamar o modelo
    B->>F: grava a transação
    B->>S: remove o pendente
    B-->>U: "Despesa de R$ 42,50 registrada"
```

Duas decisões aparecem no diagrama. A prévia fica na sessão da conversa, não nos dados da família, então nada entra no saldo antes da confirmação. E a resposta à prévia é reconhecida por um parser, não pelo modelo: "sim", "editar" e "cancelar" não custam tokens, respondem mais rápido e não correm o risco de uma confirmação alucinada.

<table>
<tr>
<td width="40%"><img src="assets/conversa-comprovante.png" alt="Comprovante Pix fictício enviado no WhatsApp, prévia com valor, categoria e conta, e registro após o 'sim'"></td>
<td width="40%"><img src="assets/conversa-agenda.png" alt="Mensagem 'Dentista sexta às 15h' e confirmação do evento com horário e lembrete"></td>
<td width="20%"><img src="assets/painel-celular.png" alt="Painel do FinFlow no celular com saldo, agenda e resumo semanal"></td>
</tr>
<tr>
<td><sub>Comprovante (fictício): prévia e confirmação</sub></td>
<td><sub>"Dentista sexta às 15h" vira evento com lembrete</sub></td>
<td><sub>O mesmo estado no painel móvel</sub></td>
</tr>
</table>

## Como testo um agente que não é determinístico

A mesma mensagem pode gerar respostas diferentes. Por isso o que precisa ser exato fica no código, onde se testa como qualquer função, e o comportamento do modelo é medido contra casos fixos.

- **Corpus de regressão do agente.** São 85 intenções, cada uma com a mensagem do usuário (às vezes em vários turnos, com dados preparados antes), a ferramenta esperada e os parâmetros que precisam sair dela: "gastei 50 no mercado" tem de chamar `createExpense` com `amount: 50`. O corpus nasceu como linha de base antes de reescrever o prompt e ganha casos novos a cada correção de comportamento.
- **Guardas testadas com texto real.** A conferência da resposta e as demais guardas do chat têm 75 testes, escritos com as frases de produção que elas pegaram ou reprovaram por engano. Mudar um padrão sem quebrar um desses casos é a condição para subir.
- **Lógica de domínio com testes unitários.** Cerca de 60 arquivos cobrem ferramentas e bibliotecas: resolução de categoria, projeção de recorrências, âncora de datas, filtros de lote.
- **Security Rules testadas contra o emulador**, incluindo a leitura escopada do Operador. A cópia das regras de permissão que roda nas Cloud Functions tem teste de paridade com a original.
- **Ponta a ponta com WhatsApp real, isolado.** Um número de desenvolvimento e um proxy de captura permitem exercitar o fluxo inteiro sem tocar clientes. O critério de aprovação é o estado gravado no banco, não a frase do bot.

## Operação: o dia em que o bot ficou mudo por 22 horas

| | |
|---|---|
| **Impacto** | Em 08 e 09/06/2026, todas as mensagens receberam "😕 Desculpe" durante 22 horas |
| **Causa** | O saldo pré-pago da API do Gemini zerou |
| **Por que o fallback não salvou** | A troca para o modelo maior usava a mesma chave e o mesmo saldo, então falhava junto |
| **Por que ninguém viu** | O erro estava nos logs, mas nada lia os logs. Não havia alerta |
| **Correção** | Alerta no Cloud Monitoring disparado pelo próprio log da falha, com no máximo um aviso a cada 30 minutos, e um script que confere se o canal de notificação está verificado. Canal não verificado não envia nada, e o alerta vira decoração |

Ficaram duas lições. **Um fallback só protege se não compartilha o domínio de falha do principal.** E **um alerta só existe se alguém confere que ele dispara.** O mesmo princípio vale para o deploy: uma vez, o Cloud Run subiu uma imagem antiga em silêncio, porque o pipe no comando de build escondia o código de saída. Desde então, cada deploy é conferido pelo comportamento da revisão nova, não pela saída do comando.

## Segurança e LGPD

- **Isolamento por família em cada caminho.** No painel, que lê o Firestore direto, as Security Rules restringem os dados à família e ao papel. No backend, que usa o Admin SDK e não passa pelas Rules, o escopo por família é aplicado pelo código em toda consulta.
- **Permissões no token.** Família e papel viajam como custom claims no JWT. As Rules decidem sem ler documento extra, o que é mais rápido e mais barato, e só consultam o banco como fallback.
- **Entradas autenticadas.** Webhooks do WhatsApp exigem chave no cabeçalho, e os do Asaas exigem token, com idempotência transacional. A API usa `helmet` e rate limit de 200 requisições por minuto por IP.
- **Direitos do titular.** A exclusão de conta acontece só pelo painel e fica bloqueada enquanto houver assinatura ativa, para não deixar cobrança órfã. O usuário exporta seus dados em CSV, PDF e Excel, e as ações sensíveis ficam em log de auditoria com retenção de 90 dias.
- **Privacidade até no painel administrativo.** O admin vê contagens e agregados, nunca valores de transação. Telefones são mascarados nos logs, e toda ação do admin é auditada.

## Como trabalho com agentes de IA

Desenvolvo com agentes de IA. Eu defino as regras de negócio, avalio as propostas técnicas e valido as entregas. Trato o que o agente produz como um pull request a revisar, não como resposta pronta:

- **Telemetria do agente em produção.** Contadores de chamadas de ferramenta, erros por código, novas tentativas, fallback de modelo e duplicatas bloqueadas, por família e no total, exibidos num painel administrativo.
- **Causa antes de correção.** Ao comparar o relatório de outra IA com os dados de produção, o sintoma estava certo e a causa não: uma falha atribuída às tarefas assíncronas era um índice ausente no Firestore.
- **O teste precisa falhar sem a correção.** A massa de teste reproduz a forma dos dados de produção. Um teste que passa com e sem o conserto não prova nada.
- **Cada ciclo vira comunicação para o cliente.** As correções saem com notas de atualização em linguagem de produto. Entre abril e maio foram oito, uma a cada poucos dias.

## O que eu faria diferente

- **Medir desde o primeiro dia.** A telemetria do agente veio em abril. Atividade por família, custo por interação e taxa de operações corrigidas deveriam ter nascido junto com o produto.
- **Um portão de qualidade automático antes do deploy.** Hoje há hook de pre-commit, o deploy é manual e o corpus de regressão roda fora de um pipeline. Eu colocaria typecheck, testes e o corpus como avaliação do agente num CI que bloqueia o deploy quando algo regride.
- **Ordem por número fora da memória da instância.** A trava que processa uma mensagem por vez por número vive na memória de cada instância do Cloud Run. Com tráfego baixo isso basta, mas duas mensagens roteadas para instâncias diferentes escapam dela. Com mais escala, a fila por número precisa morar fora do processo.
- **Uma única fonte de verdade para cada cálculo.** Durante um tempo, o saldo por conta era calculado pela ferramenta e reexplicado no prompt, e as duas versões divergiram. Hoje o número vem só da ferramenta e o modelo apenas o apresenta.
- **Modelar o domínio financeiro antes das telas.** Cartão como livro separado, metas como reserva e recorrências projetadas versus materializadas chegaram por correção. Definidos no início, teriam evitado migrações.

## O que aprendi

- **Controlar um agente é conferir o que ele executou, não confiar no que ele respondeu.** As garantias moram nas ferramentas e na conferência da resposta, com erros estruturados que o fluxo sabe tratar.
- **A guarda também erra, e o erro dela tem custo.** Alarme falso trava o usuário honesto. Toda guarda precisa de telemetria própria e de testes com os casos que ela não deve pegar.
- **Personalização é regra de produto, não memória de conversa.** O que o usuário ensina precisa ser persistido, ter escopo e ter precedência definida.
- **Toda garantia tem um efeito colateral.** A regra de categoria criou o risco de reescrever a última transação, e a idempotência da assinatura deixou de fora a notificação. Cada correção gerou um teste para a próxima.

---

<sub>[← Voltar ao índice](../README.md) · [finflows.com.br](https://www.finflows.com.br/) · Repositório privado. Este estudo descreve decisões e mecanismos sem expor código ou dados de clientes.</sub>
