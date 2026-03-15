
🌤️ Bot de Clima no Telegram com n8n + IA
Workflow n8n que recebe mensagens de um bot do Telegram, identifica a cidade enviada pelo usuário, consulta a temperatura em tempo real via OpenWeatherMap e responde com uma mensagem formatada por IA.

📋 Visão Geral
O usuário envia uma cidade pelo Telegram (ex: São Paulo, SP). O bot responde com a temperatura atual e uma descrição amigável do clima, formatada por um modelo de linguagem (Google Gemini).
O fluxo implementa um mecanismo de debounce via Redis: aguarda 15 segundos após a última mensagem antes de processar, permitindo que o usuário corrija ou complemente o que digitou sem disparar múltiplas consultas.

🔁 Fluxo Detalhado
1. Entrada e Fila (Redis)

Telegram Trigger — recebe toda mensagem enviada ao bot
Edit Fields3 — extrai from.id e os dados da mensagem
AddToQueue — empurra a mensagem numa lista Redis no formato {userId}_messages
GetMessages — lê a fila atual do usuário

2. Lógica de Debounce (Switch)
O nó Switch avalia o estado da fila e direciona o fluxo para um de três caminhos:
SaídaCondiçãoAçãoDo nothingPrimeira mensagem da filaKeepWaiting — filtra, não processaContinueÚltima mensagem tem mais de 15sEraseQueue → processaWaitFila ativa, ainda dentro do prazoWait 15s → volta para GetMessages
3. Sanitização com IA

Edit Fields2 — concatena todas as mensagens da fila em um único texto
Basic LLM Chain (Gemini) — lê o texto, corrige erros de digitação, identifica cidade e UF brasileira, e retorna um JSON no formato { "query": "São Paulo,BR" }

4. Consulta ao OpenWeatherMap

HTTP Request — chama api.openweathermap.org/data/2.5/weather com units=metric e lang=pt
If — verifica se o código de resposta é 200 (sucesso)

5. Formatação e Resposta
Caminho de sucesso:

AI Agent1 (Gemini) — recebe temperatura, descrição e localidade; formata uma mensagem amigável para WhatsApp/Telegram
Edit Fields1 — salva a mensagem formatada no campo temperatura

Caminho de erro:

Edit Fields — define uma mensagem de erro pedindo que o usuário informe a localidade novamente

Por fim, Send a text message envia a resposta ao chat de origem no Telegram.

⚙️ Pré-requisitos
ServiçoUsoTelegram BotReceber e enviar mensagensRedisArmazenar a fila de mensagens por usuárioGoogle Gemini APISanitização de input e formatação da respostaOpenWeatherMap APIDados de clima em tempo real

🔐 Credenciais Necessárias
Configure as seguintes credenciais no n8n:

telegramApi — Token do bot do Telegram
Redis — Conexão com instância Redis local ou remota
googlePalmApi — Chave da API Google Gemini
httpQueryAuth (OpenWeather) — API Key da OpenWeatherMap (enviada como query param)


🧪 Como Testar

Inicie o workflow no n8n
Abra o bot no Telegram e envie uma mensagem como:

   São Paulo SP
ou
   Sampa
(o agente LLM corrige automaticamente)
3. Aguarde a resposta com a temperatura atual

🗒️ Observações

O debounce de 15 segundos evita múltiplas requisições quando o usuário digita em várias partes. Esse valor pode ser ajustado no nó Wait.
O nó RemoveQueue (desconectado no fluxo principal) pode ser usado manualmente para limpar a fila de um usuário específico durante testes.
O modelo Google Gemini está configurado com temperature: 0.1 para respostas mais determinísticas na sanitização.
A API do OpenWeatherMap retorna dados em Celsius (units=metric) e em português (lang=pt).
