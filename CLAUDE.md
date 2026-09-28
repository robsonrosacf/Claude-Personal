# Log de Treino: contexto do app

Documento de passagem para o chat que vai cuidar do app. O conteúdo dos treinos (prescrições, cargas e resultados) é responsabilidade de outro chat, o do treinador. Este chat cuida do código, do visual e das funcionalidades.

## Para que serve

App pessoal do Robson, que é atleta de CrossFit. Ele mostra:

- a programação diária de treino, montada por um treinador (Claude) em ciclos mensais;
- os resultados que o Robson reporta;
- ferramentas de apoio: escala de RPE, técnica de corrida e um gerador de posts para Instagram Stories (aba Recalibra).

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

- **Chat do treinador:** edita apenas o bloco `MONTH_PROGRAM` (prescrições e resultados) e algumas constantes de conteúdo, como `RC_WEEK_PHASES` e textos das abas RPE e Corrida. Ele atualiza o app sempre que o Robson reporta um resultado.
- **Chat do app:** edita o código, o CSS e as funcionalidades. **Não altere prescrições nem resultados dentro de `MONTH_PROGRAM`.** Se uma mudança de estrutura exigir mexer nos dados, preserve cada valor.
- **Regras para os dois:**
  - rodar `git pull` antes de editar;
  - fazer commits pequenos;
  - nunca sobrescrever o arquivo inteiro a partir de uma cópia antiga, porque isso apaga resultados registrados pelo outro chat.

## Stack

- HTML, CSS e JS puros num único arquivo, sem build e sem framework.
- Dois `<script>` inline:
  1. **mp4-muxer 5.2.2** (licença MIT) embutido inteiro, usado na exportação de vídeo;
  2. o **app**, a partir de `const STORAGE_KEY`.
- Dependência externa: html2canvas 1.4.1, pelo cdnjs.
- Fontes do Google Fonts: Anton (títulos), Work Sans (corpo) e Space Mono (dados e rótulos).
- Persistência em `localStorage`, na chave `workouts`.

## Modelo de dados

```js
it(exercise, prescribed)                     // item: {exercise, prescribed, logged:'', done:false}
blk(title, category, items, rpe, done)       // bloco; `done` = texto do RESULTADO do bloco
wk(date, name, blocks)                       // dia: {id, date:'AAAA-MM-DD', name, notes, blocks}
```

- **Categorias** (em `CATEGORIES`, cada uma com rótulo e cor): `potencia`, `forca`, `metcon`, `gym`, `resistencia`, `recovery`, `rest`.
- **Resultado:** fica no 5º argumento de `blk()`, como texto livre. Exemplos: `'3:32 · RPE 9'`, `'160kg'`, `'6 rounds + 8 reps'`. O campo `logged` dos itens é legado e não é mais usado.
- **Programa:** `MONTH_PROGRAM` é o array com todos os dias do ciclo. Hoje cobre de 31/08 a 31/10/2026. **O programa no código é a fonte da verdade.**
- **Carga na abertura:** `mergeMonthProgram()` sobrescreve no `localStorage` todo dia que não tenha `logged` preenchido. Na prática, uma edição feita pelo botão "Editar treino" pode ser apagada no próximo carregamento, se o programa do código tiver aquele dia. Isso é conhecido e aceito, porque os resultados entram pelo código.

## Abas e funcionamento

Navegação em `nav.tabs`, e cada aba renderiza sob demanda.

1. **Treino (`hoje`)** mostra o treino do dia selecionado.
   - Blocos com título, tag de RPE e tag de categoria, e itens com a prescrição.
   - O resultado aparece em `.result-panel`: rótulo "Resultado" com borda verde à esquerda. Não há checkboxes.
   - Navegação entre dias:
     - swipe horizontal e setas do teclado;
     - botões "‹ Anterior / Próximo ›" e "ir para hoje";
     - faixa da semana, com ponto verde para dia com resultado e cinza para dia planejado;
     - calendário mensal recolhível.
   - Botão "Editar treino" abre um formulário de edição local.
2. **Programação (`historico`)** lista todos os dias, que expandem para mostrar blocos e resultados.
3. **RPE** mostra a escala de 10 a 5 com reps na reserva, as zonas aeróbicas e como usar a carga de referência junto com o alvo de RPE.
4. **Corrida** traz pontos técnicos (cadência, apoio do pé, postura) e onde a técnica melhora.
5. **Recalibra** é o gerador de Stories em 1080×1920 a partir do treino do dia.
   - **Montagem do texto:**
     - puxa os blocos do dia, exceto `recovery` e `rest`;
     - remove itens de instrução usando a lista `RC_META_LABELS`;
     - separa quantidade e carga com `rcSplitPrescribed`: a carga vai para o nome do exercício, e a coluna da direita mostra só o esquema ou o número;
     - destaca por padrão o metcon, com o score limpo por `rcCleanScore` (só tempo ou reps) e o cap extraído do título.
   - **Contador:** mostra "Semana N | Dia X", a partir de `RC_WEEK_PHASES` e de `RC_START_DATE = '2026-08-31'`, que é o dia 1.
   - **Mídia de fundo:** foto ou vídeo, com modo de ajuste (arrastar, pinça e slider de zoom de 100 a 300%) e slider de contraste da máscara.
   - **Exportação:**
     - PNG por composição em canvas: mídia + overlay capturado por html2canvas;
     - vídeo em MP4 padrão, não fragmentado, com H.264 e AAC via WebCodecs + mp4-muxer (`fastStart: 'in-memory'`), para funcionar na galeria e no Instagram;
     - fallback para MediaRecorder;
     - wake lock ligado durante a gravação e limite de 60 s.
   - **Salvamento:** `rcSaveBlob` tenta a capability `downloads` do claude.ai e, fora dele, cai no download normal por `<a download>`. **No GitHub Pages vale o download normal. Isso ainda não foi testado no celular.**

**Código morto:** `renderProgressao`, `renderEvolucao` e `buildCategorySeries` são de abas removidas e dependiam de Chart.js, que não é mais carregado. Podem ser removidas.

## Design (definido pelo Robson, manter)

- **Paleta:**
  - fundo `#0c0e0f`, elevado `#16191a`, card `#14171a`;
  - texto `#f4f3ee`, texto secundário `#9aa09c`, `--steel` `#5f6462`, linhas `#262a29`;
  - acento verde-limão `#c6ff3a`.
- **Tipografia:** Anton em caixa alta para títulos, Work Sans para o corpo e Space Mono para dados.
- **Preferências de layout do Robson:** poucas cores (verde + branco e cinza), texto grande e margens generosas.
- **Recalibra:** zona segura do Stories de 320 px em cima e embaixo e 110 px nas laterais; rótulos de bloco em verde, nomes em branco a 34 px e quantidades a 27 px; bloco de resultado só com borda esquerda, em 4 linhas (Resultado / formato / score grande / cap). **Sem a palavra "RECALIBRA" e sem "Treino do dia".**
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
