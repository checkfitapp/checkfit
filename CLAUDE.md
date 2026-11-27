# CheckFit — contexto do projeto (para continuar em outra conversa)

Atualizado em 27/11/2026, versão 59 do app.

## Quem sou e como trabalho
- **Quem:** Alexandre (pt-BR). Uso iPhone e não sou programador. Respostas em português, diretas, com passo a passo quando eu precisar mexer em algo.
- **Como atualizo o app:** subo os arquivos pelo GitHub (site, pelo celular). Sempre me entregue os arquivos prontos (`index.html`, `teste.html`, `regras-firebase.txt`, `functions/index.js`), sem trechos para eu colar.
- **Segurança:** nunca peça nem aceite a chave da Anthropic ou o JSON da conta de serviço do Firebase. Eu colo esses dados só nos Secrets do GitHub.
- **Prints:** antes de pedir, lembre de cobrir dados pessoais (CPF, endereço).

## O que é
O CheckFit é um app (PWA) de checklist diário de saúde: plano alimentar, água, peso, treinos (musculação com ficha e cardio), suplementos e remédios, medidas e bioimpedância, diário "Minha Jornada", desafios em grupo e notificações.
- **Público:** quem tem nutricionista/personal e quem quer montar o próprio plano. Também tem o modo "hábitos", para quem não tem dieta.
- **Modelo de cobrança:** assinatura mensal paga pelo usuário.
- **Liberação de acesso:** por enquanto é manual (o cliente fala comigo). O pagamento automático foi adiado de propósito.

## Onde está tudo
- **Site:** https://checkfitapp.com.br (publicado pelo GitHub Pages).
- **Código:** repositório GitHub `checkfitapp/checkfit`.
  - `index.html`: o app inteiro em um arquivo só (HTML, CSS e JS, cerca de 5.500 linhas).
  - `teste.html`: cópia idêntica, usada para testes.
  - `sw.js` e `manifest.webmanifest`: PWA e notificações.
  - `functions/index.js` e `functions/package.json`: servidor (Cloud Functions).
  - `.github/workflows/deploy-servidor.yaml`: publica o servidor sozinho a cada alteração em `functions/`. Secrets usados: `FIREBASE_SERVICE_ACCOUNT` e `ANTHROPIC_API_KEY`. A chave fica no Secret Manager.
- **Firebase:** projeto `dieta-alexandre`, plano Blaze.
  - Usa o SDK compat 10.12.2 (app, auth, firestore, messaging).
  - Persistência offline ligada.
  - As regras do Firestore são coladas à mão no console (arquivo `regras-firebase.txt`).
- **Contas e admins:** os e-mails de admin estão definidos nas regras do Firestore (`isAdmin()`).
- **IA para ler planos e fichas:** conta da Anthropic com créditos pré-pagos.

## Como o app é organizado (index.html)
- **Estado global:** `S` (`S.user`, `S.db` = `users/{uid}`, `S.days`, `S.prof`, `S.date`, `S.mode`, `S.tab`). Ação principal: `render()`.
- **Cliques:** passam por `data-act="..."` em um único `document.addEventListener('click')`. Módulos novos usam o padrão `if(/^xxx/.test(a) && xxxAct(a,b)) return;`.
- **Módulos novos:** ficam antes do comentário `/* ---------- Treino de musculação: ficha, modo treinar, descanso, recordes ---------- */`.
- **Salvar o dia:** `save(date)` carimba `_t`, guarda uma cópia no aparelho (`cofreGuardar`, no localStorage `cf-cofre-{uid}`) e envia depois de 450 ms (`cofreEnviar`, que faz `set` do documento inteiro).
  - Ao ler do servidor, `cofreConferir` restaura e reenvia a cópia local se ela for mais nova.
  - Ao sair do app, `cofreDescarregar` envia na hora.
  - O aviso "Enviando… / Sem internet" fica em `#syncpill`.
- **Leitura dos dados:** `diasBind` / `diasOuvir` ficam ouvindo os últimos 70 dias. Os dias antigos viram um resumo em `config/historico`.
- **iPhone instalado (PWA):**
  - `html.pwa` faz o `.wrap` virar o container de rolagem.
  - Depois que o teclado fecha, a tela é reposicionada.
  - A virada do dia é conferida ao voltar para o app (`viraDia`).
- **Relatórios:** `relView` (tela e versão completa para o PDF) usa `relDados()`, que calcula o período pelo `relPeriodo()` (`S.relPer` = 'sem', 'mes' ou 'pers').
  - Personalizado: datas em `S.relIni`/`S.relFim` (`relPersIntervalo`, máximo de 120 dias), atalhos "Desde o início do plano" (`relInicioPlano`: `S.prof.dini` ou o primeiro dia com registro), 30 e 90 dias. Até 10 dias mostra o "Dia a dia"; acima disso, o calendário.
  - Tela com botões Semanal/Mensal/Personalizado e setas para trocar o período (`relPerAct`).
  - Imagem estilo infográfico de 1 página em canvas (`relImgGerar`, `relImgPrep`, `relImgBox`), com o mesmo visual do relatório de bioimpedância (helpers `IC`, `icT`, `icPanel`).
  - Na semana aparece o "Dia a dia"; no mês, um calendário colorido.
  - Destaques e próximos passos são gerados automaticamente (`relDestaques`, `relInsights`).
  - Entradas: cartão "Relatório da semana" na visão semanal (`relsem`) e "Relatório do mês" na visão mensal (`rel`).
- **Quando o dia foi registrado:** `save()` chama `regMarca(date)`, que grava `_rg` no dia: `dia:1` se foi editado no próprio dia (até 10h do dia seguinte vale) e `dep:N` (maior atraso em dias) se foi editado depois. `regResumo()` alimenta a seção "Como os registros foram feitos" do relatório, o destaque na imagem e o ↺ no calendário e no dia a dia. Dias anteriores à v52 não têm `_rg` e aparecem como "sem essa informação".
- **Som do treino:** o bipe toca pelo `<audio>` (`BEEP_EL`, destravado com play mudo no toque) e, se falhar, pelo Web Audio (`beepWA`, destravado com um buffer silencioso). `S.prof.somModo`: 'misto' (padrão, `navigator.audioSession.type = 'ambient'`: toca junto com a música, mas não no silencioso) ou 'sempre' ('playback': toca no silencioso e pausa a música). A escolha aparece ao tocar em Testar som (`tsommodo`). Com o som desligado (`S.prof.somDesc === false`), nada toca. Não voltar a pular o `<audio>` quando existir `audioSession`: foi isso que deixou o bipe mudo na v52 a v55.
- **Personalizar Início:** `inicioModo()` lê `S.prof.iniModo` = 'simples' (padrão), 'completo' ou 'meu' (compatível com o antigo `S.prof.simples === false` → completo). `modoSimples()` é verdadeiro fora do Completo. Módulos em `INI_MODS` (resumo, agora, treino, agenda, seq, agua, remedios, bem, desafio, grupos, peso, calorias, jornada); Simples = `INI_SIMPLES`; no 'meu', ordem em `S.prof.iniOrd` e ligados em `S.prof.iniOn` (`iniOrdem`, `iniLigados`, `iniCard`; ações `inimodo`, `initog`, `iniup`, `inidn`; pelo menos um ligado). Avisos (cardio em andamento, instalar, notificações, retorno) sempre aparecem. Com Refeição da vez desligada, tocar numa refeição em Ainda hoje (`cabre`) abre o cartão dela.
- **Início enxuto:** `viewDay` chama `casaView`, que desenha os módulos ligados: `casaResumo` (anel, dias seguidos, remédios, refeições, kcal, água com +200/+500/+1 L), `casaAgora` (refeição da vez ou a escolhida em `S.mOpen[date]`, desenhada por `mealCompacto`: uma linha por item, "trocar ›"/"qtd ›" abre o painel em `S.cTroca`, extra em `S.cExtra`), `casaTreino` e `casaAgenda` (botões das outras refeições, `mtog`, e das doses de remédio, `remtoma`). No modo Completo, volta o Início completo antigo.
- **Situação da refeição:** `mealEstado(m, r)` dá 'nada' (neutro), 'plano' (verde, status ok de `mealCalc`; trocas entre opções do plano contam como plano), 'outro' (âmbar: parte, quantidade alterada ou extra, inclusive pulada com extra) e 'pulou' (cinza). Usado nos botões de Ainda hoje, que mostram todas as refeições do dia em ordem, inclusive a que está aberta em cima (contorno `.cur`), com as kcal de cada uma e no cartão compacto (borda `mc-<st>` e selo). O resumo mostra "Sobram X kcal hoje" ou "Passou X kcal da meta" (`.cs-saldo`).
- **Aba Mais:** substitui Perfil na barra (`S.tab = 'mais'`, `maisView`). Seções internas em `S.maisSec` (`MAIS_SEC`: bem, desafio, peso, ativ, resumo, bio, conq, perfil); ações `maisx`, `maisback`. `setTab('perfil')` abre Mais › Perfil.
- **Topo:** `topoProfs()` escreve no `#nutri` a nutricionista (🍽️) e o personal (🏋️, de `S.treino.prof`).
- **Lista de compras:** `S.mode = 'compras'` (`comprasView`, `lcGerar`, `lcFmt`, `comprasAct`). Multiplica os itens do plano pelos dias, usa a opção e a escolha mais marcadas nas últimas 3 semanas (`lcPreferidos`), agrupa em Hortifrúti/Carnes e ovos/Laticínios/Mercearia por palavras-chave (`LC_KW`). Estado só no aparelho: localStorage `cf-compras-{uid}` (dias, edit, tenho, ok, opc). Entradas: ＋, Planos e Mais.
- **Aviso de retorno:** `retornoCard` aparece no Início quando a data do plano (`PLANINFO.data`, dd/mm/aaaa) completa `S.prof.retDias` (30/45/60, escolhido no Perfil por `retornoPerfil`). "Lembrar em 7 dias" grava `S.prof.retorno = {plano, ate}`; "Gerar relatório" abre o relatório personalizado desde a data do plano.
- **Textos do topo:** data em `dataTopo()` ("Quinta, 19 de nov."); o botão Hoje some quando já está no período atual.
- **PDF do relatório (v59):** `pdfRender` nunca parte um tópico entre páginas: mede cada seção (`pdfMede`), distribui os tópicos inteiros e espalha a sobra para as páginas ficarem cheias. Página 2 em diante tem um cabeçalho curto (CheckFit · título · nome), o aviso final fica no pé da última página e cada página mostra "página X de Y". `pdfRenderFit` escolhe o maior tamanho de letra (`PG.f`) que cabe em 1 página ou, se não der, em 2 páginas cheias. O "Dia a dia" fica em uma coluna (só vira duas se passar de 92% da página). Há também o **PDF colorido** (`relPdfCor`), no mesmo visual da imagem: `relImgGerar(D, true)` monta páginas A4 (1080 × 1527, escala 1,5), com tópicos inteiros, equilíbrio entre as páginas, faixa curta do CheckFit no topo da página 2 e a faixa final no pé da última. A **imagem do relatório** agora é um story do Instagram (1080 × 1920, `relStoryGerar`, usada em `relImgPrep`): cabeçalho, variação de peso com gráfico, 6 números, calendário (ou os 7 dias na semana), destaques e faixa final; o espaço é distribuído para não sobrar vazio (menos destaques e células menores quando o mês tem 6 semanas). O relatório longo `relImgGerar` continua sendo usado só no PDF colorido.
- **Estimar com IA (v58):** janela `iaeSheet()` (estado `S.iae`, ações `iae*`, módulo antes do comentário de Engajamento). Abre pelo "+ Comi algo fora do plano" das refeições (botão `ia-{mid}`), pelo ＋ e por Mais › Alimentação. Fluxo: descrição + peso + foto (reduzida para 1100 px por `lerReduz(file, max)`) → `pushFn('estimarRefeicao')` → itens editáveis (kcal) → salva cada item como extra da refeição com `ia:1` (aparece com ✨). A tela sempre diz que é estimativa.
- **Figurinhas (v57):** modo "Figurinha" na janela de compartilhar (`postSheet`, `S.post.modo = 'figura'`, modelo em `S.post.fig`, ação `postfig`) para os tipos treino, cardio e dia. Código em `figuraPng(kind, ref, estilo)` com os helpers `fg*` (antes do comentário da musculação). Modelos: `FG_TREINO` (destaque, halteres, faixa, coluna), `FG_CARDIO` (contador, faixa) e `FG_DIA` (lista, grade/"Quadros", faixa). Visual escolhido por mim: caixas pretas com borda verde, iguais às minhas artes, e transparente em volta (`FG.modo = 'preto'`; o contorno é um anel escuro fino em `fgContorno`). Não colocar o branco em volta nem voltar para "só letras".
  - Treino: `fgDadosTreino` (de `treinoPostDados`). Cardio/atividade: `fgDadosCardio` (de `cardioPostDados`, que agora aceita o `ini` ou `data|índice` para registros antigos sem id e calcula as kcal de atividades manuais pelo `trainK`; selo "CARDIO CONCLUÍDO" para aparelhos e "TREINO CONCLUÍDO" para esportes). Resumo do dia: `fgDadosDia` (consumidas, água, kcal de todos os treinos do dia, déficit = `c.saldo`, que precisa da taxa basal do Perfil; sem ela mostra refeições no plano).
  - O 📸 aparece em todas as atividades em Treinos e atividades (exceto musculação com ficha, que já abre no fim do treino). Atividades novas registradas à mão ganham `ini`.
  - "Postar meu dia" também aparece em Mais › Calorias do dia.
- **Refeições:** ficam encolhidas (`mealCard`, `mealHead`, `mealDaVez`, `mealAct`). A refeição da hora abre sozinha. "Marcar tudo" ou "Pulei" fecham e passam para a próxima.

## Dados no Firestore
- `users/{uid}`:
  - `dias/{AAAA-MM-DD}`: meals, agua, peso, treinos, musc, cardio, tomas, med, bio, bem, `_t`;
  - `config/plano`, `config/treino` (origem: prof, proprio ou ia);
  - `config/perfil`, `config/historico`, `config/alimentos`, `config/ajustes`;
  - `diario/*`.
- `assinaturas/{uid}`: `ativoAte` ou `vitalicio`. A regra `ativo(uid)` libera a gravação.
- `lembretes/{uid}`: tokens de notificação, preferências, refeições, remédios, fuso, descanso.
- `iaUso/{uid}`: limite de 5 leituras de plano por dia (`dia`, `n`) e de 15 estimativas de refeição por dia (`estDia`, `estN`, tokens em `estTokE`/`estTokS`).
- Grupos:
  - `codigos/{CODIGO}` {gid, admin, nome};
  - `grupos/{gid}` {nome, codigo, admin, membros[]}, com as subcoleções `membros/{uid}`, `desafios/{did}/part/{uid}` e `mural/{mid}`.
  - Convite por link: `?g=CODIGO`.
- Outras coleções do painel de admin: `clientes`, `log`, `app`.
- **Atenção nas regras:** usar `$(db)` (não `$(database)`).

## Servidor (functions/index.js, Cloud Functions v2)
- `agendar` / `enviarAgendado`: aviso de fim do descanso.
- `testar`: notificação de teste.
- `lembretesTick`: roda a cada 15 min (refeições, água, peso, sequência, cartas, remédios).
- `lerPlano`: lê foto ou PDF do plano alimentar ou da ficha de treino com o Claude (modelo `claude-sonnet-5-5`, tools `salvar_plano`/`salvar_ficha`, max_tokens 32000, timeout de 540 s).
- `estimarRefeicao` (v58): estima calorias de comida fora do plano a partir de descrição, peso/tamanho e foto opcional (modelo `claude-haiku-5-5`, tool `salvar_estimativa`, max_tokens 4000, timeout 90 s, limite de 15 por dia). Devolve `{ok, itens:[{nome, porcao, g, kcal, p, c, gd}], confianca, obs, dica, restantes}`. A foto não é guardada.
- `avisarDesafio`: avisa os membros do grupo sobre um desafio novo.
- **Como o app chama o servidor:** por `fetch` em `https://us-central1-dieta-alexandre.cloudfunctions.net/<nome>` com o token do usuário. Não usar o SDK de functions, porque ele quebra o service worker das notificações.

## O que já está pronto
Login e assinatura manual, plano da nutricionista (por texto ou leitura com IA), modo "monte o seu", modo hábitos, água, peso, medidas, bioimpedância, musculação com ficha (semanas e periodização, descanso, recordes, cartão de story), cardio, suplementos e remédios com estoque, notificações push, Minha Jornada com cartas para o futuro, grupos e desafios fase 1 (prontos, personalizado e perda de peso com ranking em %), refeições encolhidas, cópia de segurança do dia, relatório semanal, mensal e personalizado em imagem e PDF, Início personalizável (Simples, Completo ou Do meu jeito) com aba Mais, lista de compras, aviso de retorno à nutricionista e figurinhas sem fundo para os Stories (treino, aeróbio e resumo do dia).

## Pendências e próximos passos
1. Apagar os 2 grupos órfãos "Desafio teste" (Editar › Encerrar).
2. Próximas melhorias do documento "CheckFit — melhorias para implementar e testar": link do profissional (substitui a ideia da área da nutricionista), teste grátis (depende de uma Cloud Function para liberar o acesso e, de preferência, de pagamento automático), fotos de progresso, bioimpedância por foto, indique e ganhe.
3. Depois: pagamento automático, grupos fase 2, lojas (App Store e Google Play).
4. Ideia em aberto: juntar as alterações quando o app for usado em dois aparelhos ao mesmo tempo (hoje vale a última gravação).
5. Ideia pequena: mostrar "500 ml 💧" em vez de "0 kcal" nas refeições só de água.

## Documentos já feitos (artefatos no Claude)
- Manual do CheckFit: https://claude.ai/artifact/AmcE5kqVTQVz9BkU52cpYD
- CheckFit para nutricionistas, argumento de venda: https://claude.ai/artifact/EeMww8fExA5PG1ijD688Qi

## Como testar antes de entregar
Pegue o `index.html` do repositório e teste com Playwright no Chromium, simulando um iPhone e trocando o Firebase por um stub em memória. Sempre confira se não há erros de JavaScript e mande prints das telas novas antes de eu subir.
