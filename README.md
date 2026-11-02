# Inspire-se v1.16 — Android nativo (Kotlin + Room)

App de apoio a momentos de ansiedade: respiração guiada (SOS), registro de humor,
dicas, **chat híbrido** e **lembrete diário**. Banco local (Room/SQLite), sem login.

## Novidades da v1.1

| Função | Como funciona | Precisa de internet? |
|---|---|---|
| Aba **Conversa** (modo guiado) | Árvore de respostas com botões, sem digitar. Pode registrar o humor no mesmo banco do Histórico | Não |
| Aba **Conversa** (modo IA, opcional) | Caixa de texto + IA via seu proxy. Só liga com internet, com aceite do usuário e com o proxy configurado | Sim |
| **Lembrete diário** (aba Dicas) | Notificação local uma vez por dia, no horário escolhido. Tocar nela abre a respiração guiada | Não |

## Estrutura

```
app/src/main/java/com/example/inspirese/
  MainActivity.kt          menu inferior com as abas e abertura pela notificação
  HomeFragment.kt          saudação + registro de humor (5 estados)
  ChatFragment.kt          conversa: texto livre por padrão, botões guiados opcionais (Ajustes), IA opcional (menu ⋮)
  TipsFragment.kt          dicas, contatos de confiança (Ligar, SMS, WhatsApp) e lembrete
  HistoryFragment.kt       histórico e resumo de 7 dias
  SosActivity.kt           respiração guiada 4-7-8
  WelcomeActivity.kt       tela de Boas-vindas
  EmergencyContactsActivity.kt, TipDetailActivity.kt
  chat/                    CrisisDetector, OfflineResponder, GuidedTree (árvore, pura), AiClient, AiConfig
  data/                    Room: mood_entries, chat_messages, emergency_contacts, activity_entries
  reminder/                ReminderTime (pura, testada), ReminderScheduler, ReminderWorker, Notifier
  sos/BreathCycle.kt       fases da respiração (pura, testada)
  util/                    Prefs, Dialer, Sms, WhatsApp, PhoneValidation
proxy/                     servidor intermediário da IA (Flask)
tests/                     testes da lógica pura e do proxy
```

## Banco de dados

`AppDatabase` está na **versão 4**. Cada migração só cria tabela nova:

- `mood_entries(id, mood, timestamp)` — versão 1. Humor: `otimo`, `bem`, `normal`, `triste`, `muito_mal`.
- `chat_messages(id, role, text, timestamp)` — versão 2 (`MIGRATION_1_2`).
- `emergency_contacts(id, position, name, phone)` — versão 3 (`MIGRATION_2_3`), com índice único em `position` (1 a 3).
- `activity_entries(id, type, label, timestamp, durationSec)` — versão 4 (`MIGRATION_3_4`), alimenta o Histórico.

**O histórico salvo não é apagado nas migrações.** Não use `fallbackToDestructiveMigration`.
O chat grava o humor do dia na mesma tabela `mood_entries` da tela inicial.

## IA do chat (ChatGPT): como funciona e como publicar

**Para quem usa o app: nada a configurar.** Na primeira mensagem, o app pergunta se pode usar a IA (uma vez só).
Com o aceite, a conversa é respondida pelo ChatGPT. Quem não quiser pode desligar em ⋮ › **Desligar IA online**,
e as respostas simples continuam funcionando sem internet. **As respostas simples são um roteiro**: não entendem a
conversa como uma IA.

**Para a equipe (uma vez só):** o app precisa saber onde fica o servidor. Por isso:

1. Publique o proxy num serviço com Python (Render, Railway ou similar), com a pasta `proxy` como raiz, o comando
   de início `gunicorn -w 1 --threads 4 -b 0.0.0.0:$PORT server:app` e a variável `OPENAI_API_KEY` nas configurações
   do serviço. Para um teste rápido, dá para rodar no PC (`flask --app server run --host 0.0.0.0 --port 8000`)
   com um túnel HTTPS.
2. Cole o endereço HTTPS em `app/src/main/java/com/example/inspirese/chat/AiConfig.kt`, na constante `PROXY_URL`,
   sem barra no final.
3. Gere um novo APK. Quem instalar esse APK já conversa com a IA.

Para testar no emulador, com o proxy rodando no PC: `PROXY_URL = "http://10.0.2.2:8000"`. A configuração de rede
do app libera HTTP só para `10.0.2.2` e `localhost`.

Antes de publicar, crie a chave em platform.openai.com. **A API é cobrada à parte da assinatura do ChatGPT Plus.**
Defina um limite de gasto mensal no painel da OpenAI: ele é a proteção principal contra gasto inesperado.
Confira o servidor com:
`curl -s -X POST https://SEU-ENDERECO/chat -H "Content-Type: application/json" -d '{"messages":[{"role":"user","content":"oi, tudo bem?"}]}'`
Deve voltar `{"reply": "..."}` com uma frase em português. Se voltar erro, o terminal (ou o painel do serviço) mostra
o motivo que a OpenAI deu.

No plano gratuito do Render, o servidor dorme sem uso, e a primeira mensagem pode demorar (o app espera até 45 s).
Para a conversa sair na hora, use um plano que fique ligado.

Variáveis do servidor:

- `OPENAI_API_KEY`: chave da API (obrigatória).
- `OPENAI_MODEL`: padrão `gpt-5.6-terra` (US$ 2 por milhão de tokens de entrada e US$ 12 de saída).
  Mais barato: `gpt-5.6-luna` (US$ 0,20 e US$ 1,20). Uma resposta do app custa uns US$ 0,004 com o Terra
  e US$ 0,0004 com o Luna, em estimativa. Os preços vieram de páginas de terceiros; confira no painel da OpenAI.
- `OPENAI_REASONING_EFFORT`: padrão `low`. Se o modelo recusar esse parâmetro (erro 400 no terminal),
  defina a variável como vazia.
- `OPENAI_TEMPERATURE`: opcional. Deixe sem definir: modelos GPT-5 só aceitam o padrão.
- `OPENAI_API_URL`: só para testes com um servidor falso.
- `OPENAI_MAX_PER_IP_HOUR`: limite de chamadas por IP numa hora (padrão 60).
- `OPENAI_MAX_PER_DAY`: limite total de chamadas por dia (padrão 2000). Passou disso, o app usa as respostas
  simples até o dia seguinte.

Dados: com a IA ligada, as últimas mensagens e o humor de hoje saem do aparelho, passam pelo servidor e
vão à OpenAI, que processa fora do Brasil. Pelo que a OpenAI publica (página "your data", consultada em
29/10/2026), dados enviados à API não são usados para treinar modelos por padrão, e os registros de uso
abusivo ficam guardados por até 30 dias. Isso precisa estar no consentimento e ser conferido com o
professor ou responsável (LGPD, transferência internacional).

Por que um proxy e não a chave no app: a chave da OpenAI não pode ficar dentro do APK, porque qualquer pessoa
consegue ler o conteúdo de um APK e usar a chave às suas custas. O endereço do servidor, ao contrário, vai dentro
do APK e qualquer pessoa pode lê-lo. Por isso o servidor tem um freio simples de uso (acima), e o limite de gasto
no painel da OpenAI é a proteção principal. Não há autenticação de cliente.
A arquitetura completa, com o que sai do aparelho, está em `docs/` e no projeto Inspire (ver o resumo da versão).

## Segurança e privacidade (leia antes de usar com pessoas reais)

- Texto livre com sinais de crise **não** é enviado à IA: o app responde com CVV 188 / SAMU 192.
  O detector é uma busca por trechos: pode deixar passar frases que não previmos e pode
  disparar por engano. Ele é uma rede de segurança, não um classificador.
- Com a IA ligada, as últimas mensagens da conversa saem do aparelho. Dados de saúde são
  dados pessoais sensíveis pela LGPD: confirme com o professor/responsável o que é exigido.
- O app não substitui acompanhamento profissional (aviso na tela de Dicas e no aceite da IA).

## Lembrete diário: limites

- Usa WorkManager: funciona offline e sobrevive a reiniciar o celular, mas **não é relógio exato**;
  o Android pode adiantar ou atrasar alguns minutos para poupar bateria.
- Android 13+ pede permissão de notificações; sem ela o lembrete não liga.

## Testes

- Lógica pura (respiração, crise, chat offline, árvore de respostas, telefone, horário do lembrete). Cada arquivo
  em `tests/logica/` tem o seu `main`, então rode um por vez, compilando junto com os fontes
  que ele usa. Exemplo com o detector de crise e o lembrete:
  ```
  S=app/src/main/java/com/example/inspirese
  kotlinc $S/chat/CrisisDetector.kt $S/reminder/ReminderTime.kt tests/logica/CrisisAndReminderTest.kt -include-runtime -d t.jar && java -jar t.jar
  ```
  `BreathCycleTest` usa `sos/BreathCycle.kt`; `OfflineResponderTest` usa `chat/OfflineResponder.kt` e
  `chat/CrisisDetector.kt`; `SuggestedQuestionsTest` usa `chat/SuggestedQuestions.kt`, `chat/OfflineResponder.kt` e
  `chat/CrisisDetector.kt`; `PhoneValidationTest` usa `util/PhoneValidation.kt`; `GuidedTreeTest` usa `chat/GuidedTree.kt`.
  Para ver os acentos certos no console, rode o `.jar` com `java -Dfile.encoding=UTF-8 -jar t.jar`.
- Proxy (com a OpenAI simulada, 59 verificações): `pip install flask requests && python3 -I tests/test_proxy.py`
- Total das suítes desta versão: 1721 verificações (28 + 40 do modo offline + 9 + 24 + 1528 do roteiro + 6 das perguntas prontas + 59 do proxy + 27 da IA no aparelho).
- Os testes do Android (JUnit em `app/src/test/`, com `./gradlew test`) ainda não existem: os acima rodam fora do Android.

## Rodar o app

Abra a pasta no Android Studio, aguarde o Gradle sincronizar e clique em Run (▶).
A primeira compilação baixa as dependências (inclui `androidx.work`).
Sem o Android Studio: JDK 17 + Android SDK, `gradle wrapper`, `./gradlew assembleDebug`.

## Som da tela SOS (opcional)

Coloque `calm_loop.mp3` em `app/src/main/res/raw/`. Sem o arquivo, o app funciona, só sem som.

## v1.2 — Contatos de confiança

3 contatos (nome + telefone) cadastrados na aba Dicas. Cada um tem dois botões:

- **Ligar** — abre o discador com o número já preenchido (`ACTION_DIAL`, sem permissão).
- **SMS** — abre o app de mensagens com o texto "Oi, pode me ligar? Preciso conversar." já
  preenchido; o usuário revê e confirma antes de enviar (`ACTION_SENDTO`, sem permissão).

Nenhum dos dois liga ou envia sozinho — é sempre o usuário quem confirma no app padrão do
celular. Dá para trocar por envio silencioso com `SmsManager.sendTextMessage()`, mas isso pede
a permissão perigosa `SEND_SMS` e manda a mensagem sem o usuário ver antes; mantive o padrão
mais seguro, igual ao `Dialer` que já existia.

Novos arquivos: `data/EmergencyContact.kt`, `data/EmergencyContactDao.kt`,
`EmergencyContactsActivity.kt` (tela de cadastro), `util/Sms.kt`, `util/PhoneValidation.kt`
(testado em `tests/logica/PhoneValidationTest.kt`). Banco sobe para `version = 3`
(`MIGRATION_2_3`, tabela `emergency_contacts`), testada com SQLite real do mesmo jeito que a
migração 1→2 (histórico de humor e conversa preservados).

## v1.3 — novo visual (tema escuro) + técnica 4-7-8

Redesenho baseado numa referência que vocês mandaram: tema escuro roxo, 5 abas no menu
inferior (Início / SOS / Conversa / Dicas / Histórico — o SOS deixou de ser um botão
flutuante), Home com saudação + frase do dia + 5 estados de humor + grade de "Acesso
rápido", e a respiração guiada trocou o ciclo simples de 4s/6s pela técnica real **4-7-8**
(inspire 4s, segure 7s, expire 8s).

**O que NÃO veio nesta leva** (fica para a próxima, senão o escopo ficava grande demais
para revisar de uma vez): as telas de Dicas e Histórico ainda usam a estrutura de cards
antiga, só com as cores novas — a referência tem chips de filtro e uma tela de detalhe por
dica, que ainda não implementei. A tela de Conversa também só herdou as cores novas, sem os
avatares/balões mais trabalhados da referência.

### Humor: 3 estados → 5

`MoodEntry.mood` passa a guardar `"otimo" | "bem" | "normal" | "triste" | "muito_mal"`
(antes era `"bem" | "ansiedade" | "panico"`). **Não precisou de migração do Room** — a coluna
já era TEXT livre, só mudam os valores aceitos. HistoryFragment/HistoryAdapter foram
atualizados para os 5. O chat guiado (ChatTree.kt) não mudou nada: ele só usa o enum
`ChatAction`, e é o `ChatFragment` quem traduz cada ação para um dos 5 valores novos
(`LOG_BEM→"bem"`, `LOG_ANSIEDADE→"triste"`, `LOG_PANICO→"muito_mal"`).

### SOS: a lógica de fase virou uma classe testável

Extraí a matemática da respiração para `sos/BreathCycle.kt` — uma função pura
`stateAt(elapsedMs)` que devolve fase, segundos restantes e escala do círculo. A
`SosActivity` ficou só "lê o relógio a cada 50ms e desenha o que o BreathCycle manda".
24 testes cobrem as transições exatas entre as 3 fases (`tests/logica/BreathCycleTest.kt`).
Um bug pequeno do ciclo antigo foi corrigido de quebra: o contador mostrava "5,4,3,2,1" num
intervalo de 4 segundos (um número a mais); agora mostra exatamente "4,3,2,1".

### SOS agora é uma aba, não mais um botão flutuante

`MainActivity` trata o item "SOS" do menu como um caso especial: abre a `SosActivity` por
cima da tela atual e devolve `false` no listener, então o menu continua marcando a aba de
conteúdo que já estava selecionada (o SOS nunca "fica marcado" depois de fechar). Os
ladrilhos de "Acesso rápido" da Home chamam `(activity as? MainActivity)?.selectTab(id)`
para trocar de aba por código.

### Cores aproximadas da referência

Extraí a paleta olhando a captura de tela, não com conta-gotas de pixel — se algum tom
destoar do que vocês querem, é só pedir o ajuste do hex específico. Tema fixo em escuro
agora (removi a variação clara/escura automática, já que a referência é sempre escura).

## v1.4 — correção de contraste (bugs reais, achados por auditoria de cor)

Você reportou "não consigo ver as letras direito" e "não consigo acessar o chat pra
conversar". Encontrei e corrigi, com números — calculei a razão de contraste (fórmula do
WCAG) de todo par texto/fundo do app:

| Onde | Antes | Depois |
|---|---|---|
| **Botões de resposta do chat guiado** (`item_chat_option.xml`) | texto quase preto sobre fundo roxo-escuro: **1.14:1** | branco sobre roxo cheio: **4.86:1** |
| Fase/contador por cima do círculo de respiração | **3.00:1** (no limite mínimo, e o círculo ainda anima) | **4.38:1** |
| Botões de ação primária em geral (Salvar, Ligar, Estou melhor, Enviar) | **3.33:1** (abaixo do recomendado p/ texto normal) | **4.86:1** |

(Referência: WCAG pede ≥4.5:1 para texto normal e ≥3:1 para texto grande/negrito.)

A causa raiz dos botões do chat: o estilo `MoodButton` (antigo, dos 3 botões de humor que a
v1.3 aposentou) continuou sendo reaproveitado só por esses botões, com uma cor de texto
pensada pra fundo claro (`ink_on_tile`, bem escura) — mas o fundo deles (`primary_soft`) virou
roxo escuro no tema novo. Texto escuro sobre fundo escuro, basicamente invisível. Resultado:
dava pra abrir o chat, mas não dava pra ver em que opção tocar — por isso parecia
"inacessível". Renomeei o estilo pra `ChatOptionButton` e refeito com as cores certas, porque
reaproveitar um nome de estilo pra outra coisa é exatamente o tipo de coisa que gera esse bug.

Além disso, escureci um pouco o roxo principal (`primary`: de `#8B7CF6` para `#6C5CE7`) e o
círculo da respiração (`orb`, mesma cor nova) — resolve os outros dois contrastes de uma vez,
porque ambos usam essa mesma cor de fundo.

### Sobre o app Aura que você mandou (.apk)

Abri o pacote: é `com.aura.app`, construído em **.NET MAUI** (não é Kotlin/Java nativo — dá
pra ver pelos nomes `MainApplication`, `mono.MonoRuntimeProvider`, bibliotecas da Microsoft).
É um app comercial de verdade. Por isso não fui atrás de extrair as cores exatas de dentro do
pacote — não é uma prática legítima de benchmarking usar o binário de um app comercial pra
copiar os valores internos dele, mesmo num trabalho de faculdade. Continuei usando só a
referência visual (a mesma screenshot de antes) pra calibrar a paleta, agora com contraste
calculado de verdade, não só "aproximado".

## v1.5 — chat virou conversa livre (sem botões)

Você pediu pra tirar os botões de resposta rápida ("Estou bem", "Pânico" etc.) e deixar só a
conversa de texto, como na referência. Feito — e isso mudou a arquitetura do chat, não só a
tela:

- **Removido:** `chat/ChatTree.kt` (a árvore de nós/botões), `item_chat_option.xml`, o estilo
  `ChatOptionButton`, e as funções `chatNode`/`setChatNode` do `Prefs.kt` (só existiam pra
  lembrar "em que nó da árvore" a conversa estava).
- **Novo:** `chat/OfflineResponder.kt` — não é IA, é um conjunto de respostas fixas escolhidas
  por palavra-chave (ansiedade, pânico, tristeza, "quero respirar", humor positivo, saudação
  pura, e uma resposta genérica de fallback). É quem responde por padrão, sem precisar de
  internet nem de proxy configurado — mantém a regra original do projeto ("funciona 100%
  offline") mesmo com o chat agora sendo livre.
- O `CrisisDetector` continua rodando **antes** de tudo: texto de crise nunca chega no
  `OfflineResponder` nem na IA, vai direto pra resposta fixa com CVV/SAMU.
- A IA online (opcional, com proxy configurado) continua existindo do mesmo jeito de antes —
  agora ela só substitui o `OfflineResponder`, não substitui mais um "modo", já que não tem
  mais modo nenhum de botão pra trocar.
- **Efeito colateral que vale saber:** antes, tocar em "Estou bem" no chat também registrava
  o humor no Histórico. Sem botões, essa ponte sumiu — o Histórico só é alimentado pela Home
  agora. Se quiser esse registro de volta a partir da conversa livre, dá pra fazer o
  `OfflineResponder` sugerir "quer registrar esse humor?" como próximo passo, mas prefiro não
  inferir e salvar sozinho sem confirmação sua.
- 12 testes novos no `OfflineResponderTest.kt`, cobrindo cada categoria de palavra-chave e o
  caso de borda que quase me pegou: "Bom dia, tudo bem?" não pode ser lido como "humor
  positivo" só porque tem "bem" dentro — tratei saudação pura como categoria própria.

## v1.6 — visual no nível da referência + IA mais inteligente

Você mandou a imagem de referência (telas: Início, SOS, Conversa, Dicas, Dica, Histórico) e
pediu pra levar o app pra esse nível de acabamento, além de deixar a IA do chat mais
inteligente. As pendências que ficaram em aberto desde a v1.3 ("telas de Dicas e Histórico
ainda usam a estrutura antiga", "Conversa só herdou as cores novas") são resolvidas aqui.

### Telas novas / redesenhadas

- **Boas-vindas** (`WelcomeActivity`, nova): primeira tela ao abrir o app — fundo em degradê
  escuro, brilho atrás do logo, nome do app e "Começar". Só uma porta de entrada, sem lógica
  nem estado — por isso não tem teste próprio. `MainActivity` deixou de ser a tela de
  LAUNCHER (virou `exported="false"`, só é aberta de dentro do app).
- **SOS** (`activity_sos.xml` + `SosActivity.kt`, reescritas): agora tem 3 estados — parado
  (card da técnica 4-7-8 + "Começar"), em andamento (anel de progresso
  `CircularProgressIndicator` mostrando quanto falta da fase atual, 4 pontos marcando o
  ciclo, "Pausar") e pausado ("Continuar"). O círculo que só mudava de escala (`bg_orb`) virou
  só o estado parado; o anel ativo é o que a referência mostra ("Expire / 8s" com a barra
  circular). "Pausar" de verdade pausa o cronômetro (não só a tela) — sair e voltar ao app no
  meio da respiração retoma de onde parou, sem contar o tempo em segundo plano.
- **Conversa** (`fragment_chat.xml` + `ChatFragment.kt`): cabeçalho enxuto (← Conversa · SOS ·
  ⋮) — "IA online" e "Limpar conversa" viraram itens do menu ⋮ em vez de ocupar uma chave e
  um botão fixos na tela. Balões do bot ganharam avatar.
- **Dicas** (`fragment_tips.xml` + `TipsFragment.kt`, `TipsAdapter` novo): abas **Todos /
  Saúde Mental / Bem-estar** filtram a lista; cada dica agora é uma linha com ícone + resumo
  + seta, e tocar abre uma **tela de detalhe** (`TipDetailActivity`, nova) com ícone grande,
  texto de abertura e passos numerados — exatamente o padrão "lista → detalhe" da referência.
  O conteúdo das 8 dicas (texto que já existia) foi reorganizado em `data/Tip.kt` com um passo
  a passo pra cada uma, sem inventar conteúdo novo.
- **Histórico** (`fragment_history.xml` + `HistoryFragment.kt` + `HistoryAdapter` novo): além
  dos cartões de resumo dos últimos 7 dias (que já existiam), a lista agora é uma **linha do
  tempo de verdade** — respiração, conversa e dica lida, agrupadas por "Hoje, `<data>`" /
  "Ontem, `<data>`" / data completa, cada linha com um ícone colorido por tipo (pulmão azul =
  respiração, balão verde = conversa, lâmpada âmbar = dica) e duração quando faz sentido
  ("14:32 · 4 min"). Tocar numa linha abre os detalhes num diálogo.
- Ícones vetoriais próprios (`ic_home`, `ic_sos`, `ic_chat`, `ic_tips`, `ic_history`,
  `ic_back`, `ic_more`, `ic_send`, `ic_chevron_right`, `ic_bot`, `ic_logo_heart`) substituem os
  ícones padrão do Android no menu inferior e os emojis nos ladrilhos da Home — mesma ideia
  visual da referência (traço fino, monocromático, tingido pela cor do contexto).

### Histórico agora é alimentado por 3 fontes, não só pelo humor

Nova tabela `activity_entries(id, type, label, timestamp, durationSec)` — Room sobe pra
**version = 4** (`MIGRATION_3_4`, só cria a tabela nova; nada é apagado). `SosActivity` grava
um registro ao sair (se a respiração ativa durou 5s ou mais), `TipDetailActivity` grava ao
abrir uma dica, e `ChatFragment` grava ao sair da aba Conversa (se pelo menos uma mensagem foi
enviada). `MoodEntry` continua existindo só para os cartões de resumo de humor da Home — as
duas tabelas não se misturam.

**Bug que encontrei e corrigi nessa auditoria:** o `ChatFragment` já tinha os campos
`sessionStart`/`sentAnyThisSession` e um comentário apontando pro `onDestroyView` como o lugar
que devia gravar a conversa no Histórico — mas o `onDestroyView` só limpava o binding, nunca
chamava o `activityDao`. Resultado: nenhuma conversa aparecia no Histórico, mesmo depois de
mandar várias mensagens. Implementei o `logSessionIfNeeded()` que faltava e marquei
`sentAnyThisSession = true` no envio da mensagem — agora a conversa é registrada como as
outras duas atividades.

### IA do chat: mais inteligente, não só "ligada"

- **Modelo padrão trocado** de `claude-haiku-4-5-20251001` para `claude-sonnet-5`
  (`proxy/server.py`, variável `AI_MODEL`) — Sonnet entende nuance e contexto bem melhor que
  Haiku, o que importa numa conversa de apoio emocional. Pra mais qualidade ainda (com mais
  custo/latência), dá pra trocar por `claude-opus-5-5` só mudando a variável de ambiente
  `AI_MODEL`, sem mexer em código.
- **Prompt de sistema reescrito** para parecer menos um roteiro genérico: agora pede
  explicitamente pra validar o que a pessoa sente antes de sugerir qualquer coisa, fazer no
  máximo uma pergunta por resposta (em vez de bombardear com perguntas), e mencionar o SOS ou
  as Dicas só quando fizer sentido, não em toda resposta. As regras de segurança (nunca dizer
  que é profissional de saúde, nunca diagnosticar, sempre indicar CVV/SAMU em crise, não pedir
  dados pessoais) continuam exatamente como antes.
- **Contexto do humor do dia** (opcional): se a pessoa registrou o humor na Home hoje, o
  `ChatFragment` manda um resuminho de uma linha (ex.: "Humor que a pessoa registrou hoje no
  app: triste.") junto com a mensagem — o proxy acrescenta isso ao prompt de sistema só como
  pano de fundo pro tom da resposta, nunca como instrução, e nunca é tratado como dado vindo
  da pessoa. Nenhum outro dado (nome, histórico completo, mensagens antigas além das 12 já
  enviadas) sai do aparelho — mesma regra de privacidade de antes.

### Ferramental: faltava o Gradle Wrapper

Reparei que nenhum dos zips anteriores (v1.1 a v1.5) incluía `gradlew` / `gradlew.bat` /
`gradle/wrapper/` — o Android Studio provavelmente gerava isso sozinho ao abrir o projeto, mas
sem eles o projeto não builda por linha de comando nem num ambiente sem Android Studio. Gerei
o wrapper completo (Gradle 8.7, a versão recomendada pra o AGP 8.5.0 que o projeto já usa) e
incluí desta vez. Também sincronizei o `build.gradle.kts` raiz com a versão do Kotlin/KSP que
o Android Studio de vocês já estava usando (2.0.21 / 2.0.21-1.0.25, a julgar pelos arquivos que
vocês mandaram) — antes estava em 1.9.24, o que podia gerar um projeto com versões
inconsistentes dependendo de quem abrisse.

### O que NÃO veio nesta leva

- A tela de **Contatos de confiança** (`EmergencyContactsActivity`) manteve o visual mais
  simples (cabeçalho com texto "Fechar" em vez do padrão ← + título das outras telas) — não é
  uma das telas da referência, então não priorizei redesenhá-la agora. Funciona normalmente.
- O anel de progresso da respiração ainda usa o componente padrão do Material Design (cor e
  espessura customizadas); a referência parece ter um gradiente sutil no traço — dá pra
  refinar depois com um `Canvas` customizado, se quiserem esse detalhe.
- Testes novos (`tests/logica/*`) não foram adicionados para o código desta leva (adapters e
  telas dependem do Android, fora do escopo dos testes de lógica pura que o projeto já tem).

## v1.7 — renomeado para Inspire-se + logo novo + reconstrução do .rar corrompido

Você pediu pra trocar o nome de "Respira" pra **"Inspire-se"** (não "Respire") e aplicar a
logo nova (as três "pétalas" azul/azul/roxo convergindo num ponto, com o ponto branco perto
do topo) que está na imagem que você mandou. Também mandou o `.rar` do projeto do jeito que
está no seu Android Studio, pra eu usar como base.

### Troca de nome

Nome é mais do que o texto que aparece na tela — mexi em tudo que carrega a marca:

| Onde | Antes | Agora |
|---|---|---|
| Nome exibido (launcher, dentro do app) | `Respira` (`strings.xml`, `app_name`) | `Inspire-se` |
| `rootProject.name` (`settings.gradle.kts`) | `"Respira"` | `"Inspire-se"` |
| Pasta de pacote Kotlin / `namespace` / `applicationId` | `com.example.respira` | `com.example.inspirese` |
| Tema (`themes.xml`, `AndroidManifest.xml`) | `Theme.Respira` | `Theme.InspireSe` |
| `versionCode` / `versionName` (`app/build.gradle.kts`) | `7` / `"1.6"` | `8` / `"1.7"` |

Sobre o `applicationId`: isso é o identificador único do app pro Android (tipo um CPF do
app), não só um texto — **mudei de propósito**, porque "Inspire-se" é um app diferente do
"Respira" na visão do sistema agora. Na prática isso quer dizer que instalar este APK não vai
*atualizar* uma instalação antiga do Respira no celular de ninguém — vai instalar como um app
separado, do lado do antigo. Se vocês tinham o Respira instalado em algum aparelho de teste e
querem só um dos dois, desinstalem o antigo manualmente. Pareceu a escolha certa pra uma
troca de identidade "de verdade" (é literalmente um rebranding, não só um ajuste de texto),
mas é uma decisão que dá pra reverter (mantendo `com.example.respira`) se preferirem que uma
instalação nova simplesmente substitua a antiga.

### Logo nova

A logo veio como ícone adaptativo (`ic_launcher_foreground.xml` + `ic_launcher_background.xml`,
usados tanto no ícone do launcher quanto na tela de Boas-vindas) — recriei como vetor,
porque um ícone adaptativo de verdade (que o Android recorta em círculo, quadrado arredondado
etc., dependendo do celular) precisa ser vetor, não uma imagem fixa colada. É a mesma
composição da arte que você mandou: duas pétalas azuis nas laterais (gradiente `#2979FF` →
`#4FC3F7`) e uma pétala roxa maior no centro, por cima (gradiente `#7C4DFF` → `#B388FF`), com
um ponto branco perto do topo da pétala central. Esse mesmo drawable é reusado na tela de
Boas-vindas, então a logo do ícone e a logo dentro do app são sempre a mesma arte — muda a
logo num lugar só (`ic_launcher_foreground.xml`) se precisar ajustar de novo.

### O `.rar` estava certo: o erro foi na minha extração

Na primeira leitura do `.rar` (`5036d9fd-Inspire-se_android.rar`) eu usei ferramentas de extração
que não leem bem este formato (RAR5), como `unar` e `7z`. Vários arquivos saíram **vazios**
(0 bytes). Eu atribuí isso ao Windows sem verificar, e estava errado. Extraí de novo com `unrar`,
e todos os arquivos vieram com o tamanho certo (`unrar t` deu "All OK"). O `.rar` estava íntegro.

Por causa desse erro, na v1.7 eu **reconstruí à mão** sete arquivos que não precisavam ser
reconstruídos: `values/colors.xml`, `layout/activity_welcome.xml`, `adapter/HistoryAdapter.kt`,
`data/ChatDao.kt`, `data/ChatMessage.kt`, `tests/logica/OfflineResponderTest.kt` e
`tests/logica/PhoneValidationTest.kt`. Eram palpites, e podiam divergir do projeto de vocês.
Na v1.8 substituí os sete pelos originais extraídos com `unrar`. A verificação de recursos da
v1.7 também foi feita em cima dos palpites, então não valia para o projeto real. Refiz em cima
dos originais (ver "Verificação" na v1.8).

### Testes de lógica pura: rodei de verdade dessa vez

Na sessão anterior, tentei rodar os testes com um compilador Kotlin "montado na mão" a partir
dos `.jar` que vêm dentro do Gradle (porque não tinha um `kotlinc` de verdade disponível) e
bati num erro interno do compilador (`BackendException` durante a geração de código) que não
consegui resolver a tempo. Desta vez baixei o `kotlinc` 1.9.24 oficial (zip de ~90MB,
direto do GitHub) — o problema era mesmo a montagem manual do classpath, porque com o
compilador de verdade os quatro conjuntos de teste compilam e passam limpos:

```
BreathCycleTest.kt        24 verificações — TODOS OS TESTES PASSARAM
OfflineResponderTest.kt   13 verificações — TODOS OS TESTES PASSARAM
PhoneValidationTest.kt    11 verificações — TODOS OS TESTES PASSARAM
CrisisAndReminderTest.kt  17 verificações — TODOS OS TESTES PASSARAM
```

Isso cobre toda a lógica que não depende do framework do Android (respiração 4-7-8, detector
de crise, validação de telefone, cálculo do horário do lembrete, respostas do chat offline).
Na v1.8 eles foram rodados de novo, com os arquivos originais, e continuam passando.

**O que continua fora do alcance deste sandbox:** uma build completa do Gradle/Android. Testei
de novo o acesso a `dl.google.com`, `maven.google.com` e `repo.maven.apache.org` — os três
continuam devolvendo HTTP 403 aqui, então não dá pra baixar as dependências do Android
(AndroidX, Room, Material etc.) nem gerar um APK de verdade neste ambiente. O caminho real
pra build/instalação continua sendo o Android Studio de vocês — que, pelos arquivos que
mandaram (`local.properties`, `gradle.properties`), já está configurado e funcionando.

### O que fazer no Android Studio

Depois de abrir este projeto (substituindo a pasta antiga ou como projeto novo), um **Build >
Clean Project** seguido de **Build > Rebuild Project** é recomendado, já que o `applicationId`
mudou — evita qualquer cache antigo do Gradle confundindo "Respira" com "Inspire-se". Se
tiverem uma instalação antiga do Respira no celular/emulador de teste, desinstalem ela antes
de instalar esta build, pelo motivo explicado acima (são dois apps diferentes agora, não uma
atualização).

## v1.8 — IA do chat melhor, WhatsApp nos contatos, Acesso rápido removido

Pedidos desta versão: melhorar a IA integrada do chat, remover o "Acesso rápido" da Home,
pôr um botão de WhatsApp nos 3 contatos de emergência e gerar o APK.

### Acesso rápido removido

Na v1.7 os ladrilhos já tinham saído do layout e do código da Home. Sobravam estilos
(`QuickTile`, `QuickTileLabel`), um drawable (`drawable/bg_tile.xml`) e um comentário que
falava nos ladrilhos. Removi tudo isso. Se o seu celular ainda mostra o Acesso rápido, ele
está com um APK mais antigo que esta fonte: instale de novo a partir desta versão.

### WhatsApp nos contatos

Cada um dos 3 contatos (aba Dicas) agora tem **Ligar · SMS · WhatsApp**. O botão WhatsApp é
outlined, com texto e contorno verdes `#25D366`. Sobre o fundo do cartão (`#1E1B2E`) o contraste
é 8,5:1, acima dos 4,5:1 do WCAG. O rótulo "🟢 WhatsApp" cabe em uma linha e encolhe sozinho em
celulares pequenos. O botão abre `api.whatsapp.com/send` com a mensagem
"Oi, pode me ligar? Preciso conversar." e, se o número tiver 10 ou 11 dígitos sem `+`, põe o DDI 55.

### IA do chat

Mudanças no servidor (`proxy/server.py`):

- **Modelo padrão** `claude-sonnet-5-5` (antes `claude-sonnet-5`). Troque com a variável `AI_MODEL`.
- **Prompt reescrito**, com estas regras: acolher antes de aconselhar; retomar o que a pessoa já
  contou (continuidade); uma ideia e no máximo uma pergunta por resposta; texto corrido de até
  4 frases, sem listas (é uma tela de celular); uma técnica simples por vez (respiração lenta ou
  5 sentidos); citar só as abas que o app tem (SOS e Dicas); lembrar UBS e CAPS pelo SUS quando a
  angústia é longa; nunca se apresentar como psicólogo ou médico, nunca diagnosticar, nunca
  indicar remédio, nunca pedir dados pessoais; em crise, CVV 188 e SAMU 192.
- **Limite de resposta** em `MAX_TOKENS = 500` (o teste agora bate com o código).
- **Log de erro** do servidor: status da IA e um trecho do erro. Nunca a chave nem a conversa.

Mudanças no app:

- **Configurar o servidor pelo app.** Antes, nada gravava o endereço do proxy, e o item
  "Ligar IA online" só aparece com um servidor configurado. Resultado: a IA nunca ligava.
  Agora: ⋮ › **Configurar servidor da IA**. A chave da API continua só no servidor.

Não testei a IA ao vivo: aqui não há acesso à API da Anthropic. O teste real é rodar o proxy com
a sua chave e conversar pelo emulador ou por um endereço HTTPS.

### Logo na tela de Boas-vindas

O logo ficou em um disco claro (`drawable/bg_welcome_logo_disc.xml`), não no roxo da marca. As
pétalas são azuis e roxas, então no roxo elas perdiam contraste. A tagline voltou a ser
"Um lugar para ser ouvido." e o nome vem de `@string/app_name`.

### Verificação

- **Recursos:** 64 arquivos XML bem formados. Todas as referências `@color/`, `@string/`,
  `@style/`, `@drawable/`, `@layout/`, `@id/` estão definidas. Nenhum resto de Acesso rápido.
- **Lógica pura:** os quatro conjuntos (`BreathCycleTest`, `OfflineResponderTest`,
  `PhoneValidationTest`, `CrisisAndReminderTest`) compilam com o `kotlinc` oficial e passam.
- **Proxy:** `python3 -I tests/test_proxy.py`, 31 verificações, todas passando.
- **Não compilei o app Android aqui.** `ChatFragment.kt`, `HomeFragment.kt` e `MainActivity.kt`
  precisam do SDK do Android, que não está neste ambiente. O que falta é o build no Android Studio.

### Gerar o APK

No Android Studio: **Build › Build Bundle(s) / APK(s) › Build APK(s)**. O arquivo sai em
`app/build/outputs/apk/debug/app-debug.apk`. Para um APK assinado de release:
**Build › Generate Signed Bundle / APK**.

## v1.9 — árvore de respostas guiadas, texto grande, SOS a um toque, ícone e documentos

Esta versão parte do zip `Inspire-se-v1.8.zip` enviado e aplica o que foi pedido nas conversas
anteriores. O que mudou, em ordem de importância:

### Conversa: árvore offline, desligada por padrão

- `chat/GuidedTree.kt` é a árvore de respostas guiadas: cada passo tem uma mensagem e até três
  botões. Um botão leva a outro passo e pode disparar uma ação (SOS, Dicas, Início para registrar
  o humor, ligar para o CVV 188 ou para o SAMU 192). Funciona sem internet.
- A árvore fica **desligada por padrão**. Você pediu na v1.5 que o chat fosse só de texto, então
  ligar os botões virou uma opção: **Ajustes › Respostas com botões**. Se você preferir os botões
  sempre ligados, é uma linha em `Prefs.guidedReplies` (ver a pergunta em `docs/01`, seção 7, D8).
- Com a opção ligada, o menu ⋮ ganha **Mostrar opções**, que volta ao início da árvore.
- O último passo fica salvo, então os botões continuam certos ao trocar de aba.
- Texto de crise continua sendo tratado antes de tudo. Com a IA ligada, a conversa é livre.

### Acessibilidade para 50+

- Botões principais com 56 dp (SOS nos cabeçalhos, Ajustes, botões da conversa, Ligar, SMS,
  WhatsApp, enviar e as respostas da árvore).
- A tela de SOS agora tem o título "SOS / Respiração", como na imagem de referência.

### Ícone e layout

- Ícone adaptativo com o emblema da logo enviada, sobre `#9DB1FA`. O vetor antigo foi removido
  para não haver dois ícones de primeiro plano.
- Boas-vindas com o emblema e a frase "Cuidar da sua mente também é importante.".
- Histórico com filtros, Dicas com título "Dicas", campo "Digite sua mensagem..." e Início com
  "Olá! ☀️". "Acesso rápido" continua removido (pedido de 15/10).

### Documentos

- `docs/01-arquitetura-do-chat-e-decisoes.md`: camadas do chat, o que sai do aparelho, árvore,
  decisões D1 a D12 e o que foi cortado.
- `docs/02-navegacao-acessibilidade-e-layout.md`: mapa das 5 abas, caminho até o SOS em cada tela,
  contrastes medidos e comparação com a referência.
- `docs/03-privacidade-validacao-e-testes.md`: dados e LGPD, testes automatizados (197), roteiro
  de testes manuais F1 a F10 e o plano de teste com 2 ou 3 pessoas de 50+.

### Verificação desta versão

- Lógica pura compilada com o `kotlinc` oficial e passando: CrisisAndReminder (28), OfflineResponder
  (12), PhoneValidation (9), BreathCycle (24), GuidedTree (90).
- Proxy: `python3 -I tests/test_proxy.py`, 34 verificações, todas passando.
- **Não compilei o app Android aqui.** O ambiente não tem o SDK do Android nem alcança os servidores
  do Google, então o build precisa ser feito no Android Studio. Os arquivos XML foram conferidos
  (todos bem formados) e os arquivos Kotlin foram checados por sintaxe, sem erro de parse.

## v1.10 — chat offline com variações e perguntas prontas

Esta versão parte da v1.9. O pedido foi que o chat deixasse de parecer robotizado, porque a versão
anterior repetia a mesma frase para qualquer mensagem que tivesse uma palavra-chave.

### Modo offline (padrão, sem internet)

- `chat/OfflineResponder.kt` responde pelo assunto da mensagem: pânico, preocupação, tristeza, sono,
  pedido de ajuda, respiração, algo positivo ou cumprimento. Cada assunto tem 2 a 3 variações.
- Preocupação com entrevista, prova ou apresentação, e com contas, tem textos próprios.
- Respostas curtas ("não sei", "sim") continuam o assunto anterior.
- Cada resposta faz no máximo uma pergunta.
- Continua sendo um roteiro escrito pela equipe, não IA. Para uma conversa livre, a IA online precisa
  de servidor e chave (ver `docs/01`, seção 3).

### Perguntas prontas

- `chat/SuggestedQuestions.kt` traz três perguntas que aparecem como botões no início da conversa.
- Tocar numa delas envia o texto como se fosse digitado: passa pelo detector de crise antes de tudo.
- As perguntas somem quando a pessoa manda a primeira mensagem. Voltam pelo menu ⋮ › Mostrar perguntas sugeridas.

### Versão

- `versionCode = 11`, `versionName = "1.10"`.

### Verificação desta versão

- `OfflineResponderTest`: 40 verificações (eram 12). `SuggestedQuestionsTest`: 6 verificações (nova).
- Total das suítes: 231. Todas passando, compiladas com o `kotlinc` oficial.
- O `ChatFragment.kt` foi checado por sintaxe. O app não foi compilado aqui, porque o ambiente não tem o
  Android SDK. Compile no Android Studio antes de testar no celular.

### Não feito

- Integração do repositório `UnB-CIS/Chatbot-Psicologia`: ele não tem código do chatbot (só README, licença
  MIT e um site de documentação). Ver D15 em `docs/01`.
- Código de IA aberta de terceiros: vai ser avaliado quando for enviado, com conferência de licença,
  dos dados que sairiam do aparelho e do modelo.

## v1.11 — tom mais natural na conversa

Pedido: o chat precisava responder de forma mais natural e humana. O modo offline é um roteiro, então
o ganho de naturalidade veio de duas partes.

### IA online (proxy)

- `proxy/server.py`: o prompt do sistema foi reescrito. Deixou de ser uma lista numerada de regras, que
  soava como manual, e passou a descrever uma conversa calma e direta, com:
  - exemplos de tom (sem copiar as palavras);
  - uma lista de frases prontas a evitar ("sinto muito que você esteja assim", "estou aqui com você", "entendo");
  - no máximo uma pergunta por resposta, e nem toda resposta precisa de uma;
  - nenhum conselho sem ser pedido.
- As regras de segurança não mudaram: a IA diz que é uma IA, não se apresenta como psicólogo, não indica
  remédio, informa CVV 188 e SAMU 192 em crise e não pede dados pessoais.
- Para mais naturalidade, o modelo importa muito. O 8B padrão soa mais mecânico. Um modelo maior, como o
  Llama 3.3 70B (openrouter.ai/models/meta-llama/llama-3.3-70b-instruct), costuma ser melhor, com custo maior.
  Confira o preço antes de trocar `OPEN_MODEL`.

### Modo offline

- Frases mais diretas nas variações: saíram "Sinto muito que você esteja assim", "Estou te ouvindo" e
  "Entendo", e entraram falas mais curtas. A lógica de assuntos e continuidade é a mesma da v1.10.

### Versão e testes

- `versionCode = 12`, `versionName = "1.11"`.
- Total: 232 verificações. O proxy ganhou uma verificação sobre as regras de tom do prompt (35).
- Os testes conferem o prompt e o caminho da chamada. A qualidade real da IA só aparece com o servidor e
  a chave ligados (teste F12 em `docs/03`).

## v1.12 — IA do chat: ChatGPT (API da OpenAI)

Pedido: o chat estava "burro". As respostas que apareceram na tela vinham do modo offline, porque a IA
online não estava ligada. O pedido: usar a IA do ChatGPT.

### O que mudou

- `proxy/server.py`: o proxy agora chama a API da OpenAI (`https://api.openai.com/v1/chat/completions`),
  com o modelo padrão `gpt-5.6-terra`. A variável da chave passou a se chamar `OPENAI_API_KEY`.
- Parâmetros enviados: `max_completion_tokens` (700), no lugar do `max_tokens` antigo. `reasoning_effort`
  = `low`, para resposta rápida. `temperature` só vai se for configurada.
- `proxy/requirements.txt`: inclui `gunicorn`, para publicar o proxy na nuvem.
- App: o aviso de consentimento diz que os dados vão à OpenAI e são processados fora do Brasil. Ao salvar
  o endereço do servidor, o app oferece ligar a IA na hora. A barra da conversa diz "IA ligada (ChatGPT)".
  A espera do app passou para 15 s de conexão e 45 s de resposta, porque um servidor na nuvem pode demorar
  a responder na primeira mensagem.
- Modo offline: sem mudanças. Continua sendo usado com a IA desligada, sem internet ou sem servidor.

### Verificação desta versão

- `tests/test_proxy.py`: 48 verificações (eram 35). Cobrem o corpo enviado à OpenAI, o prazo da chamada, o
  erro 400 da API tratado como 502, e a chave que não aparece em nenhuma resposta.
- Teste de ponta a ponta com um servidor falso que imita a API da OpenAI: o proxy envia o Bearer, o modelo,
  `max_completion_tokens: 700` e `reasoning_effort: low`, sem `temperature`, e devolve o texto da resposta.
  Quando o "provedor" recusa um parâmetro, o proxy responde 502 e o terminal mostra o motivo, sem a chave.
- Total: 245 verificações (28 + 40 + 9 + 24 + 90 + 6 + 48).
- **Não verificado aqui:** a chamada real à OpenAI. Esta versão não tem a chave, e a conferência precisa
  ser feita com a chave e o servidor ligados (curl do passo 4 acima, depois o teste F12 em `docs/03`).
  O app também não foi compilado aqui (não há Android SDK nesta máquina).

### Riscos

- Quem tiver o endereço do servidor pode gastar os seus créditos. Defina um limite de gasto no painel da
  OpenAI e não publique o endereço.
- O proxy ainda não tem autenticação nem limite por pessoa (ver "Antes de publicar de verdade" acima).

### Versão

- `versionCode = 13`, `versionName = "1.12"`.

## v1.13 — a IA funciona direto, sem configurar nada

Pedido: a pessoa não quer configurar nada no app para conversar com a IA. Quer conversar com a IA diretamente.

### O que mudou no app

- O endereço do servidor saiu da tela. Ele fica na constante `AiConfig.PROXY_URL`, e vai dentro do APK. Sumiram
  do menu ⋮ o "Configurar servidor da IA" e o campo de endereço.
- Na primeira mensagem, aparece uma vez o aviso "Conversar com a IA?". Com "Aceito", a conversa é respondida pela
  IA. Com "Agora não", as respostas simples, e o aviso não volta. Quem recusou pode ligar depois em ⋮.
- Com servidor e aceite, a IA responde por padrão. ⋮ › "Desligar IA online" continua disponível.
- Se a internet cair ou o servidor não responder, a resposta simples entra naquela mensagem, com um aviso curto.
  A IA não é desligada por isso: a próxima mensagem tenta de novo.
- Sem `PROXY_URL` preenchido, o app não mostra nada de IA: só as respostas simples.

### O que mudou no servidor

- Freio de uso: 60 chamadas por IP por hora e 2000 por dia (`OPENAI_MAX_PER_IP_HOUR` e `OPENAI_MAX_PER_DAY`).
  Passou disso, responde 429, e o app usa a resposta simples. Motivo: o endereço agora é público, porque vai no APK.
- O IP vem do último item de `X-Forwarded-For`, para funcionar atrás de balanceador (Render, Railway).
- Publicação com `gunicorn -w 1 --threads 4`: um único worker, porque o freio fica na memória do processo.

### Verificação desta versão

- `tests/test_proxy.py`: 59 verificações (eram 48). Novas: limite por IP na hora, limite do dia, reset no dia
  seguinte, IP real pelo `X-Forwarded-For`, e 429 sem chamar a OpenAI.
- Total: 256 verificações (28 + 40 + 9 + 24 + 90 + 6 + 59).
- **Não compilado no Android Studio**, porque esta máquina não tem o SDK. As mudanças em `ChatFragment.kt`,
  `AiClient.kt`, `AiConfig.kt` e `Prefs.kt` foram revisadas à mão. Compile antes de testar.
- **Não testado com a chave real** da OpenAI.

### Versão

- `versionCode = 14`, `versionName = "1.13"`.


## v1.14 — roteiro do assistente: perguntas prontas, sem psicólogo e sem servidor

### O que mudou no app
- `chat/GuidedTree.kt` virou um roteiro com 22 passos: perguntas e respostas prontas, escritas pela equipe, para ansiedade (com triagem de dor no peito e falta de ar), tristeza (com tempo de duração), sono (adormecer, acordar no meio da noite, acordar cansado), registro do momento, conversa livre e ajuda (CVV 188, SAMU 192, UBS, CAPS e UPA pelo SUS).
- Sem botões (padrão), a pessoa escolhe uma opção de dois jeitos: pelo número ("2") ou por uma palavra ("triste", "peito"). Com "Respostas com botões" ligada em Ajustes, as opções aparecem como botões, como antes.
- Texto livre que combina com um passo de entrada ("estou ansioso", "não consigo dormir") começa o roteiro daquele passo. Texto que não combina recebe a resposta simples do `OfflineResponder`, e a tela lembra os números da opção atual.
- A apresentação diz que o assistente é automático, não é psicólogo nem médico e não faz diagnóstico. Nenhuma resposta se apresenta como psicólogo.
- Texto de crise continua passando antes de tudo (CVV 188 e SAMU 192), como antes.
- Menu ⋮ › "Mostrar opções" funciona nos dois modos.

### Decisão: nada de personagem de psicólogo, nada de servidor para o roteiro
- Psicólogo é profissão regulamentada no Brasil. Um bot que se passa por um, para uma pessoa fragilizada, dá uma confiança que ele não tem. Por isso o roteiro se apresenta como assistente automático e encaminha para um psicólogo, em vez de se passar por um.
- O roteiro é texto fixo dentro do APK. Não precisa de servidor, funciona sem internet e não depende de o servidor estar no ar durante a apresentação.

### Verificação desta versão
- `tests/logica/GuidedTreeTest.kt`: 319 verificações, compiladas com o `kotlinc` 2.0.21 e passando. É a primeira vez que a lógica Kotlin foi compilada neste ambiente. Cobre: integridade dos passos, alcance de todos a partir do início, ajuda de crise a até 3 toques, escolha por número, escolha por palavra-chave, e início por texto livre.
- Total das suítes: 485 verificações, todas passando.
- **Não compilado:** `ChatFragment.kt` depende do SDK do Android, que não existe neste ambiente. Foi conferido só o equilíbrio de chaves e parênteses. Compile antes de qualquer demonstração.
- Testes manuais novos no `docs/03`: F14 (escolher pelo número), F15 (escolher pela palavra) e F16 (texto livre começa o roteiro).

### Versão
- `versionCode = 15`, `versionName = "1.14"`.

## v1.15 — roteiro com 100 passos

### O que mudou
- O roteiro (`chat/GuidedTree.kt` e os arquivos `chat/Roteiro*.kt`) passou de 22 para **100 passos**, em oito temas: início e registro (`RoteiroInicio`), ansiedade em duas partes (`RoteiroAnsiedade` e `RoteiroAnsiedadeSituacoes`), respiração e relaxamento (`RoteiroRespiracao`), tristeza (`RoteiroTristeza`), sono (`RoteiroSono`), rotina e autocuidado (`RoteiroRotina`), relações e solidão (`RoteiroRelacoes`) e ajuda e emergência (`RoteiroAjuda`).
- Cada passo tem uma pergunta ou resposta pronta de até 300 caracteres e até quatro opções. As opções funcionam por botão, por número ou por palavra-chave, como na v1.14.
- Pensamentos de não querer viver aparecem como opção nos passos de tristeza e levam a um passo de ajuda com CVV 188 e SAMU 192, em até três toques.
- Nenhum passo indica remédio, dose ou comprimido. Os encaminhamentos são UBS, CAPS, UPA, psicólogo, CVV 188 e SAMU 192.

### Como o roteiro foi montado
- Os textos foram escritos pela equipe, sem revisão de psicólogo. Antes de uso com pessoas reais, essa revisão é obrigatória (ver `docs/03`, seção 6).
- Os passos foram escritos em arquivos por tema, de propósito, para que cada arquivo seja curto e fácil de revisar por quem não é programador.

### Verificação desta versão
- `tests/logica/GuidedTreeTest.kt`: **1528 verificações**, compiladas com o `kotlinc` 2.0.21 e passando. Cobre: exatamente 100 passos, ids únicos, alcance de todos os passos a partir do início, até quatro opções por passo, rótulos de até 45 caracteres, palavras-chave de pelo menos três letras e sem acento, ajuda de crise a até três toques, escolha por número, por palavra e por texto livre, e ausência de dose de remédio no texto.
- Total das suítes: 1694 verificações, todas passando.
- **Não compilado:** `ChatFragment.kt`, que não mudou nesta versão, mas usa o roteiro. O compilador do Android precisa conferir antes da apresentação.

### Versão
- `versionCode = 16`, `versionName = "1.15"`.

## v1.16 — IA no aparelho (Gemma 3 1B, sem internet)

### O que mudou
- Nova opção no chat, no menu ⋮: **Usar IA no aparelho**. Ela só aparece se a equipe definiu o endereço do modelo em `LocalAiConfig.MODEL_URL` (`chat/LocalAi.kt`). Enquanto esse endereço estiver vazio, o recurso fica escondido e o app funciona como antes.
- Ao ligar pela primeira vez, o app avisa o tamanho (cerca de 1 GB), pede Wi-Fi e pergunta se pode baixar. O download tem progresso e pode ser cancelado; o arquivo só vai para o lugar definitivo quando termina.
- Depois de baixado, a IA responde o texto livre sem internet. Ordem das respostas: crise (CVV e SAMU, sempre primeiro), roteiro (números, palavras e temas), IA online (se estiver ligada), IA no aparelho e, por último, a resposta simples.
- Menu ⋮ › **Apagar IA do aparelho** libera o espaço.
- A resposta do modelo passa por `chat/LocalPrompt.kt`: tira listas e negrito, corta em até 400 letras e recusa termos como "sou psicólogo", "diagnóstico", "dose" ou "comprimido". Nesses casos, cai no roteiro.

### Decisões
- Gemma 3 1B pela biblioteca LiteRT-LM: gratuito, funciona offline depois do download e é o que o Google indica hoje para Android (o MediaPipe LLM Inference está em manutenção).
- O modelo não vai dentro do APK, porque é grande demais. A equipe hospeda o arquivo `.litertlm` e define o endereço. Quem hospeda precisa incluir os termos do Gemma e o aviso "Built with Gemma" ([termos](https://ai.google.dev/gemma/terms)).
- Não usei o espelho de terceiros do modelo que apareceu na busca: é uma conta desconhecida e o arquivo está em outro formato (`.task`).

### O que NÃO foi verificado (confirmar no Android Studio antes da apresentação)
- **Não compilou aqui:** `LocalAi.kt`, as partes novas do `ChatFragment.kt` e o `Prefs.kt` dependem do SDK do Android, que não existe neste ambiente. A API do LiteRT-LM foi seguida da documentação oficial (`Engine`, `EngineConfig`, `initialize()`, `createConversation`, `sendMessage(...).text`).
- **Versão da biblioteca:** o `build.gradle.kts` usa `latest.release`, como a documentação indica. Fixe uma versão antes de gerar o APK final.
- **Versão mínima do Android:** a documentação não informa. O app tem `minSdk = 26`; se a biblioteca exigir mais, é preciso subir o `minSdk`.
- **Backend GPU:** o código pede `Backend.GPU()`. Em aparelho sem GPU compatível, o motor falha, o erro é capturado e o chat usa o roteiro. Não confirmei se existe `Backend.CPU()` para usar como alternativa.
- **Modelo:** nunca rodou. Não há teste de qualidade em português nem de segurança das respostas.

### Verificação desta versão
- `tests/logica/LocalPromptTest.kt`: 27 verificações, compiladas com o `kotlinc` 2.0.21 e passando. Cobre o texto do sistema, o histórico, a limpeza das respostas e os termos recusados.
- Total das suítes: 1721 verificações, todas passando.

### Versão
- `versionCode = 17`, `versionName = "1.16"`.
- Dependência nova: `com.google.ai.edge.litertlm:litertlm-android` (fixar a versão antes do APK final).
