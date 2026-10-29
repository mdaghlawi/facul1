# 🌤️ Chatbot de Temperatura no Telegram com n8n + OpenWeather

Chatbot do Telegram feito no **n8n**. O usuário manda o nome de uma cidade e o bot responde com a temperatura atual, consultada na API da **OpenWeather**.

```
Usuário: Belo Horizonte,MG,BR
Bot:     🌤️ A temperatura em Belo Horizonte é de 25°C.
```

---

## 📁 Estrutura do repositório

| Arquivo | Descrição |
|---|---|
| `workflow-chatbot-telegram.json` | Workflow exportado do n8n (arquivo principal da entrega) |
| `workflow-telegram-chatbot.json` | Cópia idêntica do workflow (o enunciado cita os dois nomes) |
| `docker-compose.yml` | Sobe o n8n localmente com Docker |
| `.env.example` | Modelo das variáveis de ambiente (sem valores reais) |
| `README.md` | Esta documentação |

---

## 🔄 Como o workflow funciona

```
Telegram Trigger
   └─► Formatar Entrada (Set: cria a variável "queue")
         └─► OpenWeather - Clima Atual (HTTP Request)
               └─► Resposta Válida? (IF)
                     ├─ true  ─► Formatar Mensagem (Fallback) ─► Gemini - Melhorar Mensagem (opcional) ─► Mensagem Final ─► Enviar Temperatura (Telegram)
                     └─ false ─► Enviar Erro (Telegram)
```

1. **Telegram Trigger**: recebe as mensagens de texto enviadas ao bot.
2. **Formatar Entrada** (Set node): grava o texto na variável **`queue`** já normalizado:
   - remove espaços no início e no fim, espaços repetidos e espaços em volta das vírgulas;
   - remove acentos (`São Paulo` → `sao paulo`) e converte tudo para minúsculas;
   - se o usuário mandar só o nome da cidade, adiciona `,br` (`Curitiba` → `curitiba,br`), seguindo o formato `cidade,estado,país` da OpenWeather.
   - também guarda o `chat_id` para responder ao usuário certo.
3. **OpenWeather - Clima Atual** (HTTP Request): `GET https://api.openweathermap.org/data/2.5/weather` com **Send Query Parameters** ativado:

   | Parâmetro | Valor |
   |---|---|
   | `q` | `{{ $json.queue }}` (o parâmetro que a OpenWeather de fato lê) |
   | `queue` | `{{ $json.queue }}` (incluído porque o enunciado pede esse nome; a API ignora) |
   | `units` | `metric` (graus Celsius) |
   | `lang` | `pt_br` (português brasileiro) |
   | `appid` | `={{ $env.OPENWEATHER_API_KEY }}` |

   O nó está com *Include Response Headers and Status* e *Never Error* ativados, então o código HTTP (`statusCode`) chega ao IF mesmo quando a cidade não existe (404).
4. **Resposta Válida?** (IF node): segue pelo ramo `true` apenas se `statusCode == 200`, se `body.main.temp` existe e se `body.name` não está vazio. Qualquer outro caso (cidade não encontrada, chave inválida, entrada vazia) vai para o ramo `false`.
5. **Formatar Mensagem (Fallback)** (Code node): extrai `name` e `main.temp`, arredonda a temperatura com `Math.round` e monta a mensagem de forma determinística.
6. **Gemini - Melhorar Mensagem** (opcional, **desativado por padrão**): reescreve a mensagem de forma mais natural. Veja a seção do Gemini abaixo.
7. **Mensagem Final** (Code node): usa a resposta do Gemini se ela for um JSON válido `{"message": "...", "ok": true}`. Se não for, usa a mensagem do fallback.
8. **Enviar Temperatura** / **Enviar Erro** (Telegram Send Message):
   - Sucesso: `🌤️ A temperatura em Belo Horizonte é de 25°C.`
   - Erro: `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).`

---

## 🔑 Variáveis e credenciais necessárias

| Variável | Para que serve | Onde conseguir |
|---|---|---|
| `OPENWEATHER_API_KEY` | Chave da API OpenWeather, lida pelo workflow via `$env.OPENWEATHER_API_KEY` | https://home.openweathermap.org/api_keys (chaves novas podem levar até ~2h para ativar) |
| `TELEGRAM_BOT_TOKEN` | Token do bot, usado na credencial **Telegram API** do n8n | Converse com o [@BotFather](https://t.me/BotFather) no Telegram e use `/newbot` |
| `WEBHOOK_URL` | URL pública HTTPS do n8n (o Telegram só envia webhooks para HTTPS) | ngrok, cloudflared ou um servidor com domínio |
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | Deve ser `false` para o n8n permitir `$env` nas expressões | Já vem definido no `docker-compose.yml` |
| `GEMINI_API_KEY` *(opcional)* | Chave do Google Gemini, usada na credencial do n8n | https://aistudio.google.com/apikey |

> ⚠️ **Nenhuma chave ou token real está neste repositório.** O JSON do workflow não tem credenciais embutidas: os nós do Telegram vêm sem credencial selecionada e a chave da OpenWeather é lida da variável de ambiente.

---

## 🐳 1. Subindo o n8n com Docker

```bash
# 1. Copie o modelo de variáveis e preencha com os seus valores
cp .env.example .env          # no Windows (PowerShell): copy .env.example .env

# 2. Exponha a porta 5678 com HTTPS (em outro terminal)
ngrok http 5678
#    Copie a URL https gerada (ex.: https://abcd-1234.ngrok-free.app) para WEBHOOK_URL no .env

# 3. Suba o n8n
docker compose up -d

# 4. Acesse http://localhost:5678 e crie a conta de administrador
```

Versão recomendada: n8n **1.x** ou mais recente (imagem `docker.n8n.io/n8nio/n8n`).

**Sem Docker (Node.js 20+):** defina as variáveis e rode `npx n8n`:

```powershell
$env:OPENWEATHER_API_KEY="sua_chave"
$env:WEBHOOK_URL="https://sua-url.ngrok-free.app/"
$env:N8N_BLOCK_ENV_ACCESS_IN_NODE="false"
npx n8n
```

---

## 📥 2. Importando o workflow no n8n

1. No n8n, clique em **Create Workflow** (ou **+** → *Workflow*).
2. No menu **⋯** (canto superior direito), escolha **Import from File...**.
3. Selecione o arquivo `workflow-chatbot-telegram.json`.
4. Clique em **Save**.

---

## 🔐 3. Configurando as credenciais

### Telegram (`TELEGRAM_BOT_TOKEN`)
1. Abra o nó **Telegram Trigger**.
2. Em **Credential to connect with**, clique em **Create New Credential**.
3. No campo **Access Token**, cole o valor do seu `TELEGRAM_BOT_TOKEN` (o token dado pelo @BotFather, no formato `123456789:ABC...`).
4. Salve a credencial.
5. Selecione **a mesma credencial** nos nós **Enviar Temperatura** e **Enviar Erro**.

### OpenWeather (`OPENWEATHER_API_KEY`)
Não é preciso criar credencial no n8n. O nó **OpenWeather - Clima Atual** lê a chave com a expressão:

```
={{ $env.OPENWEATHER_API_KEY }}
```

Basta a variável `OPENWEATHER_API_KEY` estar definida no ambiente do n8n (arquivo `.env` usado pelo `docker-compose.yml`) e `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`. Se mudar o `.env`, reinicie o container com `docker compose up -d --force-recreate`.

---

## ▶️ 4. Executando o chatbot

1. Com o workflow salvo e as credenciais configuradas, ligue o botão **Active** (canto superior direito). O n8n registra o webhook no Telegram automaticamente.
   - Para testar sem ativar, clique em **Test workflow** e envie uma mensagem ao bot em até 2 minutos.
2. No Telegram, abra o seu bot e envie uma cidade.

### Exemplos de teste

| Mensagem enviada | Resposta esperada |
|---|---|
| `Belo Horizonte,MG,BR` | `🌤️ A temperatura em Belo Horizonte é de 25°C.` |
| `São Paulo,SP,BR` | `🌤️ A temperatura em São Paulo é de 22°C.` |
| `rio de janeiro` | `🌤️ A temperatura em Rio de Janeiro é de 28°C.` |
| `Curitiba, PR, BR` | `🌤️ A temperatura em Curitiba é de 17°C.` |
| `Cidadeinexistente123` | `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).` |

*(As temperaturas acima são ilustrativas; o valor real depende do clima no momento.)*

---

## ✨ Opcional: melhoria da mensagem com Google Gemini

**Onde está:** entre **Formatar Mensagem (Fallback)** e **Mensagem Final**, no ramo de sucesso do IF:
- **Gemini - Melhorar Mensagem** (*Basic LLM Chain*): pede ao modelo para reescrever a mensagem em português, de forma natural, respondendo só no formato `{"message":"...","ok":true}`.
- **Google Gemini Chat Model**: modelo `models/gemini-2.5-flash` com **temperatura 0.1** (respostas determinísticas).

Os dois nós vêm **desativados**. Assim o workflow funciona sem custo e sem credencial do Gemini.

**Como ativar:**
1. Gere uma chave em https://aistudio.google.com/apikey (`GEMINI_API_KEY`).
2. No n8n, abra o nó **Google Gemini Chat Model** → **Create New Credential** → tipo **Google Gemini(PaLM) Api**:
   - **Host:** `https://generativelanguage.googleapis.com`
   - **API Key:** cole a sua `GEMINI_API_KEY`
3. Selecione um modelo disponível na lista (ex.: `gemini-2.5-flash`).
4. Clique com o botão direito em cada um dos dois nós → **Activate** (ou selecione e aperte `D`).
5. Salve o workflow.

**Fallback (obrigatório):** o nó **Formatar Mensagem (Fallback)** sempre gera a mesma `message` de forma determinística. O nó **Mensagem Final** só usa o texto do Gemini se ele for um JSON válido com `ok: true`. Se o Gemini estiver desativado, sem credencial, der erro (o nó usa *On Error → Continue*) ou responder fora do formato, a mensagem do fallback é enviada. O campo `fonte` na saída do **Mensagem Final** mostra qual foi usada (`gemini` ou `fallback`).

---

## ✅ Checklist da entrega

- [x] Workflow começa com **Telegram Trigger**
- [x] Texto capturado na variável **`queue`** (Set node) e normalizado (espaços, acentos, minúsculas)
- [x] **HTTP Request** para `/data/2.5/weather` com *Send Query Parameters*, `units=metric`, `lang=pt_br`, `appid={{ $env.OPENWEATHER_API_KEY }}`
- [x] Temperatura extraída e arredondada, mensagem com cidade e temperatura
- [x] **IF node** valida status HTTP 200 e os campos `main.temp` e `name`
- [x] Mensagem de erro enviada quando a cidade não é encontrada
- [x] Respostas enviadas pelo **Telegram Send Message**
- [x] JSON sem credenciais ou tokens embutidos
- [x] (Opcional) Gemini com temperatura baixa e fallback determinístico em Code node
- [x] (Opcional) `docker-compose.yml`

Para conferir que não há segredos no JSON:

```bash
grep -nE "[0-9]{8,10}:[A-Za-z0-9_-]{35}|[a-f0-9]{32}|AIza" workflow-chatbot-telegram.json || echo "Nenhum segredo encontrado"
```
