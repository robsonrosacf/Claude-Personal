# Log de Treino: contexto do app

Documento de passagem para o chat que vai cuidar do app. O conteúdo dos treinos (prescrições, cargas e resultados) é responsabilidade de outro chat, o do treinador. Este chat cuida do código, do visual e das funcionalidades.

## Para que serve

App pessoal do Robson, que é atleta de CrossFit. Ele mostra:

- a programação diária de treino, montada por um treinador (Claude) em ciclos mensais;
- os treinos de outros atletas, cada um com programa próprio (abas Batata e Thomals), montados com o mesmo treinador;
- os resultados que o Robson reporta;
- ferramentas de apoio: nota da sessão, escala de RPE e técnica de corrida (aba Guia) e um gerador de posts para Instagram Stories (aba Mídia).

Uso principal: abrir no celular Android, ver o treino do dia, navegar entre os dias e consultar resultados passados.

## Onde está e como é publicado

| Item | Detalhe |
|---|---|
| Repositório | github.com/robsonrosacf/Claude-Personal (público, branch `main`) |
| Endereço | https://robsonrosacf.github.io/Claude-Personal/ (GitHub Pages, publica do `main` automaticamente) |
| Instalação | PWA instalado pelo Chrome no Android ("Instalar"), abre em tela cheia |
| Arquivos | `index.html` (o app inteiro, cerca de 4.760 linhas), `manifest.json`, `sw.js`, `icon.svg`, `icon-v2-192.png`, `icon-v2-512.png` |
| Cópia antiga | Existe uma versão como artifact no claude.ai. **Não usar**: no Android o link abre o app do Claude em vez da página. O GitHub Pages substituiu essa versão. |

**PWA:**
- O service worker usa estratégia **rede primeiro**: sempre busca a versão nova e só usa o cache quando não há internet. O Robson atualiza fechando e abrindo o app.
- O registro do service worker só roda em https fora do claude.ai.
- Ao trocar o ícone, mude o **nome do arquivo** (foi assim de `icon-*` para `icon-v2-*`). Sem isso, o Chrome não detecta a troca no app instalado.

## Divisão de trabalho entre os chats (importante)

Os dois chats fazem push no mesmo repositório e no mesmo `index.html`.

- **Chat do treinador:** edita apenas os blocos `MONTH_PROGRAM`, `BATATA_PROGRAM` e `THOMALS_PROGRAM` (prescrições e resultados) e algumas constantes de conteúdo, como `RC_WEEK_PHASES` e os textos da aba Guia (em `renderRPE`, `SESSION_SCALE`, `RPE_SCALE`, `renderCorrida` e `RUN_CUES`). Ele atualiza o app sempre que o Robson reporta um resultado.
- **Chat do app:** edita o código, o CSS e as funcionalidades. **Não altere prescrições nem resultados dentro de `MONTH_PROGRAM` nem de `BATATA_PROGRAM` e `THOMALS_PROGRAM`.** Se uma mudança de estrutura exigir mexer nos dados, preserve cada valor.
- **Regras para os dois:**
  - rodar `git pull` antes de editar;
  - fazer commits pequenos;
  - nunca sobrescrever o arquivo inteiro a partir de uma cópia antiga, porque isso apaga resultados registrados pelo outro chat.

## Stack

- HTML, CSS e JS puros num único arquivo, sem build e sem framework.
- Três `<script>` inline:
  1. no `<head>`, um script curto que liga o tema de teste (`?tema=azul`, ver Design);
  2. **mp4-muxer 5.2.2** (licença MIT) embutido inteiro, usado na exportação de vídeo;
  3. o **app**, a partir de `const STORAGE_KEY`.
- Dependência externa: html2canvas 1.4.1, pelo cdnjs.
- Fontes do Google Fonts: Anton (títulos), Work Sans (corpo) e Space Mono (dados e rótulos).
- Persistência em `localStorage`: chave `workouts` para os treinos do Robson, `batata` para os da aba Batata, `thomals` para os da aba Thomals e `envio-pendente` para a fila de envios à planilha.
- Envio externo: só a cópia dos resultados dos atletas para o Google Apps Script do treinador (ver aba Batata e Thomals).

## Modelo de dados

```js
it(exercise, prescribed)                     // item: {exercise, prescribed, logged:'', done:false}
blk(title, category, items, rpe, done)       // bloco; `done` = texto do RESULTADO do bloco
wk(date, name, blocks, notes)                // dia: {id, date:'AAAA-MM-DD', name, notes, blocks}; notes = observações do dia (opcional)
```

- **Categorias** (em `CATEGORIES`, cada uma com rótulo e cor): `potencia`, `forca`, `metcon`, `gym`, `resistencia`, `aquecimento`, `recovery`, `rest`. Blocos de aquecimento usam `aquecimento` (cinza `--steel`), não `recovery`.
- **Wall ball em libras:** o box só tem bolas em lb. Prescrever sempre em libras, no nome do exercício: `Wall Balls (20 lb)` para o Robson e `Wall Balls (14 lb)` para a Batata e o Thomals. A exceção é a descrição das regras da prova HYROX, que usa bola oficial de 6 kg.
- **Observações do dia:** 4º argumento do `wk()`, texto livre. Aparecem no fim do dia, depois do último bloco, na seção "Observações" (`.day-notes`), só quando existem, nas abas Treino, Batata e Thomals. No merge, a observação do programa vence a salva no celular (`progW.notes || existing.notes || ''`). Formato usado pelo treinador: um tópico por linha, `• Rótulo: texto`; `formatDayNotes` põe o rótulo em negrito.
- **Itens são exercícios, instrução vai para as observações:** cada `it()` de um bloco pontuável deve ser um exercício com quantidade (`it('Remo (calorias)', '40')`). Instruções ("Por quê", "Ritmo", "Anote", "Cargas", "Estação do round"...) vão no 4º argumento do `wk()`. A aba Mídia monta a arte com os itens: instrução como item vira linha de lixo na arte, e um round descrito em texto corrido não aparece como exercícios.
- **Resultado:** fica no 5º argumento de `blk()`, como texto livre. Exemplos: `'3:32 · RPE 9'`, `'160kg'`, `'6 rounds + 8 reps'`. O campo `logged` dos itens é legado e não é mais usado.
- **Programas de outros atletas:** `BATATA_PROGRAM` (aba Batata) e `THOMALS_PROGRAM` (aba Thomals), logo depois do `MONTH_PROGRAM`, no mesmo formato (`wk`, `blk`, `it`) e com as mesmas categorias. Cada um é de **outra pessoa**, não do Robson: as restrições, RMs e regras de carga do Robson não valem para eles, e um atleta não herda as do outro. O Thomals começou como cópia do Batata e é ajustado à parte. São programas independentes: o merge, as datas e os resultados não se misturam. O site é público, então só entram dados de treino; nada que identifique a pessoa além do apelido.
- **Programa:** `MONTH_PROGRAM` é o array com todos os dias do ciclo. Hoje cobre de 31/08 a 31/10/2026. **O programa no código é a fonte da verdade.**
- **Carga na abertura:** `mergeMonthProgram()` sobrescreve no `localStorage` todo dia que não tenha `logged` preenchido, com duas exceções que sobrevivem: o **resultado de um bloco** salvo no celular, enquanto o bloco de mesmo título no programa estiver sem resultado (quando o treinador registra, o do código vence), e o **comentário** do dia (`comment`). Outras edições feitas pelo "Editar treino" (nome, prescrição, blocos) continuam sendo apagadas se o programa tiver aquele dia.
- **Comentário do dia:** campo `comment` do dia, escrito pelo atleta no formulário. Aparece na seção "Comentário", depois das "Observações" do treinador. Não confundir com `notes`, que é do treinador.

## Abas e funcionamento

Menu (`nav.tabs`) com cinco abas: **TREINO**, **BATATA**, **THOMALS**, **GUIA** e **MÍDIA**. A troca passa por `showTab(tab)`, e cada aba renderiza sob demanda. Identificadores internos: TREINO é `hoje`, BATATA é `batata`, THOMALS é `thomals`, MÍDIA é `recalibra`.

**Programa ativo:** `PROGRAMS` descreve os programas (`treino`, `batata`, `thomals`: chave de storage, fonte, view, aba, textos do estado vazio, rótulo de volta da lista e categoria padrão de bloco novo). Para um atleta novo: um array `NOME_PROGRAM`, uma entrada em `PROGRAMS`, um botão no `nav.tabs` e uma `div.view`; `programForTab` e `loadWorkouts` pegam o resto sozinhos. `setActiveProgram(nome)` troca o array `workouts`, que é o que todas as telas do dia leem (`renderHoje`, `renderHero`, faixa da semana, calendário, formulário de edição, lista completa). `showTab` ativa o programa do atleta só na aba dele (`programForTab`); Treino e Guia usam sempre `treino`, e Mídia usa o programa escolhido no seletor dela (`rcSource`).

1. **TREINO (`hoje`)** mostra o treino do dia selecionado.
   - Blocos com título, tag de RPE e tag de categoria, e itens com a prescrição.
   - O resultado aparece em `.result-panel`: rótulo "Resultado" com borda verde à esquerda. Não há checkboxes.
   - Navegação entre dias:
     - swipe horizontal e setas do teclado;
     - botões "‹ Anterior / Próximo ›" e "ir para hoje";
     - faixa da semana, com ponto verde para dia com resultado e cinza para dia planejado;
     - calendário mensal recolhível.
   - Botão "Editar treino" abre um formulário de edição local: nome, blocos, exercícios com a prescrição, **resultado de cada bloco** (`done`) e **comentário** do dia.
   - Link discreto "ver lista completa" no fim da aba abre a lista de todos os dias (`historico`).
2. **BATATA (`batata`) e THOMALS (`thomals`)** funcionam igual à aba Treino (mesmas telas, textos e navegação), cada uma com o próprio programa.
   - **Registro direto na tela, sem "Editar treino":** nessas abas o formulário de edição não existe (nem no dia vazio).
     - Cada bloco que não seja `aquecimento`, `recovery` ou `rest` mostra um botão discreto **"+ Resultado"** no lugar do painel. Ao salvar, vira o painel "Resultado" (com um "alterar" discreto) e envia à planilha só aquele bloco: `{chave, atleta, data, treino, linhas: [{bloco, resultado}], observacoes: ''}`.
     - No fim do dia, depois das Observações, o card **"Como foi"**: nota do treino de 0 a 10 em botões, comentário livre e **Enviar**. Envia `{chave, atleta, data, treino, linhas: [], observacoes: comentário, nota: N}` (o campo `nota` só vai quando há nota). Guarda no dia `nota`, `comment` e `sentAt`; o merge preserva os três.
   - **Cópia para o treinador:** Google Apps Script em `RESULTS_URL`, chave `RESULTS_KEY`. `fetch` POST sem cabeçalho (texto simples, sem preflight) e com `mode: 'no-cors'`. Sem internet, fica na fila `envio-pendente` do `localStorage` e sai na próxima abertura ou quando a conexão volta. A aba Treino não envia nada. Cada salvamento gera um envio. A chave fica no código público: serve para filtrar envios, não é segredo. Renderizam em `view-batata` e `view-thomals`. A aba Mídia usa esses programas quando escolhidos no seletor dela.
3. **Lista completa (`historico`)**, antiga aba Programação: sem botão no menu, abre pelo link da aba Treino ou de um atleta e mostra o programa da aba de origem, com o rótulo de volta dela no topo ("‹ voltar ao treino", "‹ voltar à Batata", "‹ voltar ao Thomals"). Os dias expandem para mostrar blocos e resultados. Com ela aberta, o menu marca a aba de origem.
4. **GUIA (`guia`)**, junção das antigas abas RPE e Corrida. `renderGuia()` chama `renderRPE(container)` e `renderCorrida(container)`, em três seções nesta ordem:
   - **Nota da sessão:** escala de 0 a 10 do treino inteiro (`SESSION_SCALE`);
   - **RPE de série:** escala de 10 a 5 com reps na reserva (`RPE_SCALE`), zonas aeróbicas e "Como usar" a carga de referência com o alvo de RPE;
   - **Técnica de corrida:** pontos técnicos (`RUN_CUES`) e onde a técnica melhora.
5. **MÍDIA (`recalibra`)**, antiga aba Recalibra, renomeada sem mudança de funcionalidade. Nomes internos (`renderRecalibraTab`, `view-recalibra`, prefixo `rc`) continuam iguais.
   É o gerador de Stories em 1080×1920 a partir do treino do dia.
   - **Tela (de cima para baixo):** seletor "Treino de", barra de data compacta (`rcDayBar`: ‹ seg, 19/10 › e "hoje"), card **Resultados** (um campo por bloco, com o nome do bloco como rótulo; já vem com o resultado do programa limpo por `rcCleanResult`; campo vazio tira a faixa daquele bloco), a arte em prévia, a dica "Toque numa linha da arte pra esconder", os chips das linhas escondidas e um card com **Fundo** ("Fazer upload de mídia"/"Trocar mídia" e "Fundo padrão"; contraste e "Ajustar posição" só com foto ou vídeo própria; zoom só durante o ajuste) e o botão **Exportar mídia**. Exercícios, blocos e prescrições vêm do programa e não são editáveis. Sem destaque único, sem tags, sem faixa da semana e sem calendário nesta aba.
   - **Esconder linhas:** cada item tem `hidden`. Instruções conhecidas (`RC_META_LABELS`) já começam escondidas; tocar numa linha da arte esconde (fora do modo de ajuste); os chips "+ nome" abaixo da arte mostram de volta. Bloco sem linha visível some da arte.
   - **Seletor "Treino de":** no topo da aba, botões Treino, Batata e Thomals (`rcSourceSelector`, `RC_SOURCE_LABELS`). Troca o programa ativo (`rcSource`) e recarrega os blocos; a escolha continua valendo ao sair e voltar à aba. Padrão: Treino.
   - **Montagem do texto:**
     - puxa os blocos do dia, exceto `aquecimento`, `recovery` e `rest`;
     - marca como escondidos os itens de instrução da lista `RC_META_LABELS`;
     - separa quantidade e carga com `rcSplitPrescribed`: a carga vai para o nome do exercício, e a coluna da direita mostra só o esquema ou o número;
     - cada bloco fecha com a própria faixa de resultado (`rcResultMarkup`): número grande e o RPE, se houver, menor e em verde. `rcCleanResult` pega o valor antes do primeiro " · ", tira parênteses e acrescenta o RPE: "24:41 · dentro do cap · RPE 8 · sessão: 6" vira "24:41 · RPE 8".
   - **Contador:** `rcCounterText`. No Treino, "Semana N | Dia X" a partir de `RC_WEEK_PHASES` e de `RC_START_DATE = '2026-08-31'`, que é o dia 1. Nos outros atletas, conta a partir do primeiro dia do programa deles, em semanas de 7 dias; antes do início, o contador não aparece.
   - **Mídia de fundo:** foto ou vídeo, com modo de ajuste (arrastar, pinça e slider de zoom de 100 a 300%) e slider de contraste da máscara.
   - **Fundo padrão:** sem foto ou vídeo, usa `recalibra-fundo-v2.jpg` (arte RBSN com o selo já aplicado), sem máscara e sem modo de ajuste.
   - **Selo:** `selo-v1.png` (escudo RBSN + "A vida é f▭da!") vai na camada de texto em foto e vídeo, em x 766, y 1449, 161 px de largura, a mesma posição do selo da arte padrão. É PNG porque o html2canvas desenha o SVG errado.
   - **Área do texto:** termina em y 1430 para não encostar no selo. Se o dia não couber, a classe `rc-compact` aperta só os espaçamentos.
   - **Nomes longos:** o nome ocupa o espaço que a quantidade não usa; se ainda não couber, `rcNameCandidates` encurta em etapas sem remover a carga (só na arte, nunca no programa).
   - **Exportação:**
     - PNG por composição em canvas: mídia + overlay capturado por html2canvas;
     - vídeo em MP4 padrão, não fragmentado, com H.264 e AAC via WebCodecs + mp4-muxer (`fastStart: 'in-memory'`), para funcionar na galeria e no Instagram;
     - fallback para MediaRecorder;
     - **iPhone (Safari):** um vídeo criado pelo app só carrega depois de um `play()`. `rcWaitVideoReady` toca mudo, espera dados, pausa e volta ao início. O vídeo do MediaRecorder é criado e tocado dentro do toque em "Exportar mídia" (`rcPrimedVideo`), porque só assim o iPhone deixa tocar com som depois. Se o vídeo tem som mas o navegador não tem `AudioEncoder` para AAC, o MP4 devolve `'needs-audio'` e a gravação vai para o MediaRecorder, que mantém o som; só se ele falhar sai MP4 sem áudio, com aviso. O `AudioContext` da gravação também nasce dentro do toque (`rcPrimedAudioCtx`): no iPhone, `resume()` fora do toque nunca termina. Toda espera da exportação passa por `rcWithTimeout`, e cada etapa escreve uma mensagem própria no status ("Abrindo o vídeo para gravar...", "Preparando o áudio...", "Iniciando a gravação..."), para saber onde parou;
     - wake lock ligado durante a gravação e limite de 60 s.
   - **Salvamento:** `rcSaveBlob` tenta a capability `downloads` do claude.ai e, fora dele, cai no download normal por `<a download>`. **No GitHub Pages vale o download normal. Isso ainda não foi testado no celular.**

**Código morto:** `renderProgressao`, `renderEvolucao` e `buildCategorySeries` são de abas removidas e dependiam de Chart.js, que não é mais carregado. Podem ser removidas.

## Design (definido pelo Robson, manter)

- **Paleta:**
  - fundo `#0c0e0f`, elevado `#16191a`, card `#14171a`;
  - texto `#f4f3ee`, texto secundário `#9aa09c`, `--steel` `#5f6462`, linhas `#262a29`;
  - acento verde-limão `#c6ff3a`.
- **Tema azul (em teste):** com `?tema=azul` no endereço, `:root[data-tema="azul"]` troca `--accent` e `--done` por `#3d7bff` (e os tons escuros e o fundo do `.result-panel`). Vale para a interface e para os textos da arte da aba Mídia; selo e fundo padrão continuam verdes. Sem o parâmetro, e no app instalado, tudo fica verde.
- **Tipografia:** Anton em caixa alta para títulos, Work Sans para o corpo e Space Mono para dados.
- **Preferências de layout do Robson:** poucas cores (verde + branco e cinza), texto grande e margens generosas.
- **Recalibra:** zona segura do Stories de 320 px em cima e embaixo e 110 px nas laterais; rótulos de bloco em verde, nomes em branco a 34 px e quantidades a 27 px; cada bloco com resultado termina numa faixa com borda esquerda verde: "RESULTADO" em mono, o valor em Anton 58 px (40 px se longo) e o RPE em mono verde, menor. **Sem a palavra "RECALIBRA" e sem "Treino do dia".**
- **Ícone:** escudo com as letras RBSN, escudo em verde e letras em branco sobre o fundo escuro.

## Preferências do Robson para quem trabalha com ele

- Diretor de criação, com repertório em design. Nota detalhes visuais e dá feedback direto.
- Quer entender o porquê das decisões. Seja objetivo, sem enrolação.
- Nos textos: evitar travessão (—) e tiques de texto de chatbot.
- Não concorde por padrão. Se uma ideia dele tiver um problema, diga, com argumento.
- Usa Android com Chrome.

## Validação antes de publicar

- Extrair os `<script>` e rodar `node --check` em cada um.
- Carregar a página num navegador headless (jsdom ou Playwright), sem erros no console, e conferir alguns dias do `MONTH_PROGRAM` renderizando com o resultado certo.
- Depois do push, o GitHub Pages leva cerca de 1 a 2 minutos para publicar.
