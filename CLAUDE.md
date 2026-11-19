# CheckFit — contexto do projeto (para continuar em outra conversa)

Atualizado em 19/11/2026, versão 55 do app.

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
- **Som do treino:** `audioAmbiente()` usa `navigator.audioSession.type = 'ambient'` (iOS 16.4+) para o bipe do descanso misturar com a música, sem pausar o app de música. Com o som desligado (`S.prof.somDesc === false`), o app não toca nenhum áudio.
- **Personalizar Início:** `inicioModo()` lê `S.prof.iniModo` = 'simples' (padrão), 'completo' ou 'meu' (compatível com o antigo `S.prof.simples === false` → completo). `modoSimples()` é verdadeiro fora do Completo. Módulos em `INI_MODS` (resumo, agora, treino, agenda, seq, agua, remedios, bem, desafio, grupos, peso, calorias, jornada); Simples = `INI_SIMPLES`; no 'meu', ordem em `S.prof.iniOrd` e ligados em `S.prof.iniOn` (`iniOrdem`, `iniLigados`, `iniCard`; ações `inimodo`, `initog`, `iniup`, `inidn`; pelo menos um ligado). Avisos (cardio em andamento, instalar, notificações, retorno) sempre aparecem. Com Refeição da vez desligada, tocar numa refeição em Ainda hoje (`cabre`) abre o cartão dela.
- **Início enxuto:** `viewDay` chama `casaView`, que desenha os módulos ligados: `casaResumo` (anel, dias seguidos, remédios, refeições, kcal, água com +200/+500/+1 L), `casaAgora` (refeição da vez ou a escolhida em `S.mOpen[date]`, desenhada por `mealCompacto`: uma linha por item, "trocar ›"/"qtd ›" abre o painel em `S.cTroca`, extra em `S.cExtra`), `casaTreino` e `casaAgenda` (botões das outras refeições, `mtog`, e das doses de remédio, `remtoma`). No modo Completo, volta o Início completo antigo.
- **Situação da refeição:** `mealEstado(m, r)` dá 'nada' (neutro), 'plano' (verde, status ok de `mealCalc`; trocas entre opções do plano contam como plano), 'outro' (âmbar: parte, quantidade alterada ou extra, inclusive pulada com extra) e 'pulou' (cinza). Usado nos botões de Ainda hoje (mostram as kcal da refeição) e no cartão compacto (borda `mc-<st>` e selo). O resumo mostra "Sobram X kcal hoje" ou "Passou X kcal da meta" (`.cs-saldo`).
- **Aba Mais:** substitui Perfil na barra (`S.tab = 'mais'`, `maisView`). Seções internas em `S.maisSec` (`MAIS_SEC`: bem, desafio, peso, ativ, resumo, bio, conq, perfil); ações `maisx`, `maisback`. `setTab('perfil')` abre Mais › Perfil.
- **Topo:** `topoProfs()` escreve no `#nutri` a nutricionista (🍽️) e o personal (🏋️, de `S.treino.prof`).
- **Lista de compras:** `S.mode = 'compras'` (`comprasView`, `lcGerar`, `lcFmt`, `comprasAct`). Multiplica os itens do plano pelos dias, usa a opção e a escolha mais marcadas nas últimas 3 semanas (`lcPreferidos`), agrupa em Hortifrúti/Carnes e ovos/Laticínios/Mercearia por palavras-chave (`LC_KW`). Estado só no aparelho: localStorage `cf-compras-{uid}` (dias, edit, tenho, ok, opc). Entradas: ＋, Planos e Mais.
- **Aviso de retorno:** `retornoCard` aparece no Início quando a data do plano (`PLANINFO.data`, dd/mm/aaaa) completa `S.prof.retDias` (30/45/60, escolhido no Perfil por `retornoPerfil`). "Lembrar em 7 dias" grava `S.prof.retorno = {plano, ate}`; "Gerar relatório" abre o relatório personalizado desde a data do plano.
- **Textos do topo:** data em `dataTopo()` ("Quinta, 19 de nov."); o botão Hoje some quando já está no período atual.
- **Refeições:** ficam encolhidas (`mealCard`, `mealHead`, `mealDaVez`, `mealAct`). A refeição da hora abre sozinha. "Marcar tudo" ou "Pulei" fecham e passam para a próxima.

## Dados no Firestore
- `users/{uid}`:
  - `dias/{AAAA-MM-DD}`: meals, agua, peso, treinos, musc, cardio, tomas, med, bio, bem, `_t`;
  - `config/plano`, `config/treino` (origem: prof, proprio ou ia);
  - `config/perfil`, `config/historico`, `config/alimentos`, `config/ajustes`;
  - `diario/*`.
- `assinaturas/{uid}`: `ativoAte` ou `vitalicio`. A regra `ativo(uid)` libera a gravação.
- `lembretes/{uid}`: tokens de notificação, preferências, refeições, remédios, fuso, descanso.
- `iaUso/{uid}`: limite de 5 leituras por dia.
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
- `avisarDesafio`: avisa os membros do grupo sobre um desafio novo.
- **Como o app chama o servidor:** por `fetch` em `https://us-central1-dieta-alexandre.cloudfunctions.net/<nome>` com o token do usuário. Não usar o SDK de functions, porque ele quebra o service worker das notificações.

## O que já está pronto
Login e assinatura manual, plano da nutricionista (por texto ou leitura com IA), modo "monte o seu", modo hábitos, água, peso, medidas, bioimpedância, musculação com ficha (semanas e periodização, descanso, recordes, cartão de story), cardio, suplementos e remédios com estoque, notificações push, Minha Jornada com cartas para o futuro, grupos e desafios fase 1 (prontos, personalizado e perda de peso com ranking em %), refeições encolhidas, cópia de segurança do dia, relatório semanal, mensal e personalizado em imagem e PDF, Início personalizável (Simples, Completo ou Do meu jeito) com aba Mais, lista de compras e aviso de retorno à nutricionista.

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
