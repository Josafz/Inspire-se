# Inspire-se

Aplicativo Android de apoio em momentos de ansiedade, pensado para pessoas com 50 anos ou mais. Tem respiração guiada, registro de humor, dicas, contatos de confiança, lembrete diário e uma conversa que funciona sem internet. A IA da conversa é opcional.

> **Aviso:** o Inspire-se não substitui psicólogo, psiquiatra ou médico. Em emergência, ligue **192 (SAMU)**. Se você está pensando em se machucar, ligue **188 (CVV)**: a ligação é gratuita e funciona 24 horas.

**Versão 1.16** · Kotlin · Android 8.0 ou superior (API 26) · Sem login

**Sumário:** [Funcionalidades](#funcionalidades) · [Como rodar](#como-rodar) · [Configurar a IA](#configurar-a-ia-opcional) · [Privacidade](#privacidade-e-segurança) · [Testes](#testes) · [Estrutura](#estrutura-do-projeto) · [Status](#status-e-limitações) · [Documentação](#documentação) · [Créditos e licença](#créditos-e-licença) · [Histórico de versões](#histórico-de-versões)

## Funcionalidades

| Função | Como funciona | Precisa de internet? |
|---|---|---|
| **Respiração 4-7-8** (aba SOS) | Inspira 4 s, segura 7 s, expira 8 s, com contador e anel de progresso. Pode pausar e retomar. | Não |
| **Registro de humor** (Início) | Cinco estados: ótimo, bem, normal, triste e muito mal. | Não |
| **Conversa com roteiro** | 100 passos com perguntas e respostas prontas. A pessoa responde pelo número, por uma palavra ou por botão (opcional, em Ajustes). | Não |
| **Conversa de texto livre** | Respostas simples, escolhidas por palavras-chave. | Não |
| **IA online** (opcional) | Conversa livre com o ChatGPT, por um servidor próprio. Pede consentimento antes. | Sim |
| **IA no aparelho** (opcional, experimental) | Modelo Gemma 3 1B rodando no celular, depois de baixado. | Só para baixar |
| **Detecção de crise** | Textos com sinais de crise recebem CVV 188 e SAMU 192 antes de qualquer outra resposta. | Não |
| **Dicas** | 8 dicas de saúde mental e bem-estar, com passo a passo. | Não |
| **Contatos de confiança** | Até 3 contatos, com Ligar, SMS e WhatsApp. Nada é enviado sem a pessoa confirmar no app do celular. | Não |
| **Histórico** | Linha do tempo de respirações, conversas e dicas, e resumo dos últimos 7 dias. | Não |
| **Lembrete diário** | Notificação no horário escolhido. Tocar nela abre a respiração guiada. | Não |

Feito para 50+: botões de 56 dp, contraste de texto conforme o padrão WCAG, texto grande em Ajustes e tema escuro fixo.

## Como rodar

### Para usar

Ainda não há APK publicado. Gere o APK pelo Android Studio (veja abaixo) e instale em um celular com Android 8.0 ou superior. Ao instalar um APK fora da loja, o Android pede permissão para instalar apps de fontes desconhecidas.

### Para desenvolver

Requisitos: Android Studio recente, JDK 17 e Android SDK 34.

1. Clone o repositório:
   ```bash
   git clone https://github.com/SEU-USUARIO/inspire-se.git
   ```
2. No Android Studio, escolha **File › Open** e abra a pasta do projeto. Espere o Gradle sincronizar. Na primeira vez, ele baixa as dependências.
3. Para rodar no emulador ou no celular, clique em **Run ▶**.
4. Para gerar o APK, use **Build › Build Bundle(s) / APK(s) › Build APK(s)**. O arquivo sai em `app/build/outputs/apk/debug/app-debug.apk`.

**Som da respiração (opcional):** coloque um arquivo `calm_loop.mp3` em `app/src/main/res/raw/` (crie a pasta, se não existir). Sem ele, o app funciona sem som.

## Configurar a IA (opcional)

O código sai com as duas IAs desligadas: os endereços vêm vazios. Nesse estado, o app usa só o roteiro e as respostas simples, sem servidor e sem internet. Para ligar uma IA, a equipe define o endereço antes de gerar o APK:

| IA | Constante | Arquivo |
|---|---|---|
| IA online (ChatGPT) | `AiConfig.PROXY_URL` | `app/src/main/java/com/example/inspirese/chat/AiConfig.kt` |
| IA no aparelho (Gemma) | `LocalAiConfig.MODEL_URL` | `app/src/main/java/com/example/inspirese/chat/LocalAi.kt` |

### IA online (ChatGPT)

Um servidor intermediário (pasta `proxy/`, em Flask) chama a API da OpenAI. A chave fica só no servidor, nunca no APK. Quem usa o app não configura nada: na primeira mensagem, o app pergunta uma vez se pode usar a IA.

Para testar no computador:

```bash
cd proxy
pip install -r requirements.txt
export OPENAI_API_KEY="sua-chave"
flask --app server run --host 0.0.0.0 --port 8000
```

- **Emulador Android:** `PROXY_URL = "http://10.0.2.2:8000"`. O emulador acessa o computador por esse endereço.
- **Celular de verdade ou versão de apresentação:** use um endereço HTTPS público, de um serviço como Render ou Railway, sem barra no final.

Para publicar o servidor (Render, Railway ou similar):

- Pasta raiz: `proxy`
- Comando de início: `gunicorn -w 1 --threads 4 -b 0.0.0.0:$PORT server:app`
- Variável `OPENAI_API_KEY` nas configurações do serviço.
- Use um plano que não entre em repouso. No plano gratuito do Render, a primeira mensagem pode demorar, e o app espera até 45 segundos.
- Use um único worker (`-w 1`), porque o freio de uso fica na memória do processo.

Para conferir o servidor no ar:

```bash
curl -s -X POST https://SEU-ENDERECO/chat \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"oi, tudo bem?"}]}'
```

Deve voltar `{"reply": "..."}`, com uma frase em português.

**Variáveis do servidor**

| Variável | Padrão | Função |
|---|---|---|
| `OPENAI_API_KEY` | (obrigatória) | Chave da API |
| `OPENAI_MODEL` | `gpt-5.6-terra` | Modelo usado. `gpt-5.6-luna` tem custo menor |
| `OPENAI_REASONING_EFFORT` | `low` | Se o modelo recusar o parâmetro (erro 400), deixe a variável vazia |
| `OPENAI_TEMPERATURE` | não definida | Deixe sem definir: os modelos GPT-5 aceitam só o padrão |
| `OPENAI_API_URL` | API oficial | Só para testes com um servidor falso |
| `OPENAI_MAX_PER_IP_HOUR` | `60` | Chamadas por IP por hora |
| `OPENAI_MAX_PER_DAY` | `2000` | Chamadas por dia, no total |

Acima de um limite, o servidor responde 429 e o app usa as respostas simples.

A API da OpenAI é cobrada à parte da assinatura do ChatGPT. Confira os preços no site da OpenAI e defina um **limite de gasto mensal** no painel dela. O endereço do servidor vai dentro do APK, então qualquer pessoa que o encontrar pode usar a sua cota. O freio de uso reduz o risco, mas o limite de gasto é a principal proteção. O servidor não tem autenticação de cliente.

### IA no aparelho (Gemma 3 1B, experimental)

A conversa pode usar o modelo Gemma 3 1B (formato `.litertlm`, cerca de 1 GB), executado com a biblioteca LiteRT-LM, do Google. Depois de baixado, o modelo responde sem internet.

- O modelo não vai dentro do APK, porque é grande demais. A equipe hospeda o arquivo e define `LocalAiConfig.MODEL_URL`.
- Quem hospeda ou distribui o modelo precisa seguir os [termos de uso do Gemma](https://ai.google.dev/gemma/terms) e exibir o aviso "Built with Gemma".
- No app: ⋮ › **Usar IA no aparelho**. Na primeira vez, o app avisa o tamanho, pede Wi-Fi e pergunta se pode baixar. ⋮ › **Apagar IA do aparelho** libera o espaço.
- Em aparelho sem GPU compatível, o motor falha e a conversa volta ao roteiro.

## Privacidade e segurança

Dados de saúde são dados pessoais sensíveis pela LGPD. O app mantém os dados no aparelho sempre que possível:

- **Sem login.** Não há conta nem cadastro.
- **Dados no aparelho.** Humor, conversas, histórico e contatos ficam em um banco local (Room/SQLite).
- **Sem rastreamento.** As dependências do projeto não incluem SDK de análise nem de publicidade.
- **Permissões:** internet, estado da rede e notificações.
- **IA online é opcional e pede consentimento.** Com ela ligada, as últimas mensagens e o humor do dia saem do aparelho, passam pelo servidor e chegam à OpenAI, que processa os dados fora do Brasil. Segundo a política da OpenAI consultada em 29/10/2026, dados enviados pela API não são usados para treinar modelos por padrão, e registros de uso abusivo podem ficar guardados por até 30 dias. Confira a política atual antes de usar com pessoas reais.
- **A chave da OpenAI não sai do servidor.**
- **Crise primeiro.** Textos com sinais de crise não são enviados à IA. Recebem CVV 188 e SAMU 192. O detector trabalha por trechos de texto: pode deixar passar frases que não previu e pode acionar por engano. É uma rede de segurança, não um classificador.

Antes de usar com pessoas reais, confirme com o responsável pelo projeto as exigências da LGPD: consentimento para dados de saúde, transferência internacional e política de privacidade.

## Testes

### Lógica pura

São testes em Kotlin que rodam no computador, sem emulador. Cada arquivo tem o seu próprio `main`: compile-o junto com os fontes de que depende. Os exemplos usam bash (Linux, macOS ou Git Bash no Windows) e precisam do `kotlinc` instalado.

```bash
S=app/src/main/java/com/example/inspirese
kotlinc $S/chat/CrisisDetector.kt $S/reminder/ReminderTime.kt tests/logica/CrisisAndReminderTest.kt -include-runtime -d t.jar
java -Dfile.encoding=UTF-8 -jar t.jar
```

Para o roteiro:

```bash
kotlinc $S/chat/GuidedTree.kt $S/chat/Roteiro*.kt $S/chat/OfflineResponder.kt $S/chat/CrisisDetector.kt tests/logica/GuidedTreeTest.kt -include-runtime -d t.jar
java -Dfile.encoding=UTF-8 -jar t.jar
```

| Teste | Fontes que ele compila junto | Verificações |
|---|---|---|
| `CrisisAndReminderTest.kt` | `chat/CrisisDetector.kt`, `reminder/ReminderTime.kt` | 28 |
| `OfflineResponderTest.kt` | `chat/OfflineResponder.kt`, `chat/CrisisDetector.kt` | 40 |
| `SuggestedQuestionsTest.kt` | `chat/SuggestedQuestions.kt`, `chat/OfflineResponder.kt`, `chat/CrisisDetector.kt` | 6 |
| `GuidedTreeTest.kt` | `chat/GuidedTree.kt`, `chat/Roteiro*.kt`, `chat/OfflineResponder.kt`, `chat/CrisisDetector.kt` | 1528 |
| `LocalPromptTest.kt` | `chat/LocalPrompt.kt`, `chat/CrisisDetector.kt` | 27 |
| `PhoneValidationTest.kt` | `util/PhoneValidation.kt` | 9 |
| `BreathCycleTest.kt` | `sos/BreathCycle.kt` | 24 |
| `tests/test_proxy.py` | `proxy/server.py`, com a OpenAI simulada | 59 |
| **Total** | | **1721** |

Os testes do servidor rodam com `pip install -r proxy/requirements.txt` e `python3 -I tests/test_proxy.py`.

Não há testes automatizados para as telas, o banco de dados e os fluxos do Android (JUnit em `app/src/test/`). Para esses, use o roteiro de testes manuais em `docs/03` (F1 a F19).

## Estrutura do projeto

```
app/src/main/java/com/example/inspirese/
├── MainActivity.kt               menu inferior e abertura pela notificação
├── WelcomeActivity.kt            tela de boas-vindas
├── HomeFragment.kt               saudação e registro de humor
├── ChatFragment.kt               conversa: roteiro, texto livre e IA (menu ⋮)
├── TipsFragment.kt               dicas, contatos de confiança e lembrete
├── TipDetailActivity.kt          detalhe de cada dica
├── HistoryFragment.kt            histórico e resumo de 7 dias
├── SosActivity.kt                respiração guiada 4-7-8
├── SettingsActivity.kt           ajustes (botões na conversa, texto grande, IA)
├── EmergencyContactsActivity.kt  cadastro dos contatos de confiança
├── adapter/                      listas (conversa, histórico e dicas)
├── chat/                         roteiro (GuidedTree e Roteiro*.kt), respostas simples,
│                                 detector de crise, IA online (AiClient, AiConfig)
│                                 e IA no aparelho (LocalAi, LocalPrompt)
├── data/                         Room: humor, conversa, contatos, atividades, dicas
├── reminder/                     lembrete diário (WorkManager)
├── sos/BreathCycle.kt            fases da respiração (lógica pura)
└── util/                         preferências, ligação, SMS, WhatsApp, telefone
proxy/                            servidor intermediário da IA online (Flask)
tests/logica/                     testes da lógica pura (Kotlin)
tests/test_proxy.py               testes do servidor
docs/                             arquitetura, acessibilidade, privacidade e testes
```

### Banco de dados (Room, versão 4)

| Tabela | Conteúdo |
|---|---|
| `mood_entries` | Humor registrado na tela inicial |
| `chat_messages` | Mensagens da conversa |
| `emergency_contacts` | Até 3 contatos de confiança |
| `activity_entries` | Respirações, conversas e dicas lidas (alimenta o Histórico) |

Ao mudar o banco, crie uma migração nova que só adiciona tabelas ou colunas. Não use `fallbackToDestructiveMigration`: ela apaga o histórico do usuário.

## Status e limitações

### Verificado

- A lógica pura (roteiro, respostas simples, detector de crise, respiração, validação de telefone, horário do lembrete e prompt da IA no aparelho) e o servidor: 1721 verificações passando.

### Ainda falta

- **Build e telas no Android:** não há testes de interface nem integração contínua no repositório. Gere o APK e teste no celular antes de cada uso.
- **IA no aparelho:** não testada em celular real. Qualidade do português, segurança das respostas e desempenho ainda precisam de teste.
- **Conteúdo do roteiro e das dicas:** não revisados por psicólogo. A revisão é obrigatória antes de uso com pessoas reais.
- **Serviços e telefones:** CVV 188, SAMU 192, UBS, CAPS e UPA devem ser conferidos na cidade de uso.
- **Biblioteca LiteRT-LM:** está em `latest.release`. Fixe uma versão antes de gerar o APK final.

### Limitações conhecidas

- O detector de crise trabalha por trechos de texto e não substitui avaliação profissional.
- O servidor da IA não tem autenticação de cliente.
- O lembrete diário não é exato: o Android pode atrasar alguns minutos para poupar bateria.

## Contribuir

Sugestões e correções são bem-vindas, por meio de issues. Mudanças no roteiro ou nos textos de crise precisam de revisão de um profissional de psicologia antes de entrar.

## Documentação

- [docs/01-arquitetura-do-chat-e-decisoes.md](docs/01-arquitetura-do-chat-e-decisoes.md): camadas do chat, o que sai do aparelho e as decisões de projeto.
- [docs/02-navegacao-acessibilidade-e-layout.md](docs/02-navegacao-acessibilidade-e-layout.md): navegação, acessibilidade e contrastes.
- [docs/03-privacidade-validacao-e-testes.md](docs/03-privacidade-validacao-e-testes.md): dados e LGPD, testes automatizados e roteiro de testes manuais.

## Créditos e licença

- Projeto acadêmico desenvolvido na FATEC.
- IA no aparelho: modelo Gemma e biblioteca LiteRT-LM, ambos do Google. IA online: API da OpenAI.
- Este repositório ainda não tem licença. Até que os autores escolham uma, todos os direitos sobre o código ficam reservados a eles.

## Histórico de versões

| Versão | Principais mudanças |
|---|---|
| 1.16 | IA no aparelho (Gemma 3 1B via LiteRT-LM), opcional e experimental |
| 1.15 | Roteiro ampliado para 100 passos, em oito temas |
| 1.14 | Roteiro de perguntas e respostas prontas, como assistente automático |
| 1.13 | IA online sem configuração pelo usuário: endereço no APK, aceite único e freio de uso no servidor |
| 1.12 | IA online pela API da OpenAI (ChatGPT) |
| 1.11 | Tom mais natural nas respostas |
| 1.10 | Variações nas respostas simples e perguntas prontas no início da conversa |
| 1.9 | Respostas guiadas por botões (opcional), texto grande em Ajustes, SOS com um toque |
| 1.8 | Botão de WhatsApp nos contatos |
| 1.7 | Novo nome, Inspire-se, e nova logo (antes: Respira) |
| 1.6 | Tela de boas-vindas, dicas com detalhe e histórico em linha do tempo |
| 1.5 | Conversa de texto livre |
| 1.4 | Correção de contraste nos botões e textos |
| 1.3 | Tema escuro, cinco abas, humor com cinco estados e respiração 4-7-8 |
| 1.2 | Contatos de confiança com Ligar e SMS |
| 1.1 | Conversa guiada, IA opcional e lembrete diário |
