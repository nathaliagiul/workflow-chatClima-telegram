# Telegram Weather Chatbot

Este projeto é um chatbot para Telegram construído no **n8n**. Ele permite que os usuários consultem a temperatura atual de qualquer cidade de forma rápida e intuitiva. 

O fluxo integra a **OpenWeather API** para buscar os dados meteorológicos e utiliza o **Google Gemini** para formatar a resposta com linguagem natural e amigável. Caso a IA falhe ou não possua credenciais configuradas, o sistema possui uma rota de **fallback determinístico**, garantindo que o usuário sempre receba a sua resposta.

## Tecnologias e Pré-requisitos

* **n8n** (Instância local via Docker)
* **Telegram Bot Token** (Criado via BotFather no Telegram)
* **OpenWeather API Key** (Conta gratuita na OpenWeatherMap)
* **Google Gemini API Key** (Opcional, para melhoria da mensagem de saída)

## Preparação do ambiente local (Docker)

Se você deseja rodar o n8n localmente, este repositório inclui um arquivo `docker-compose.yml`.

1. Clone o repositório.
2. Renomeie o arquivo `.env.example` para `.env`.
3. Edite o arquivo `.env` e insira a sua chave real da OpenWeather na variável esperada:
   
   OPENWEATHER_API_KEY=sua_chave_real_aqui
   N8N_ENV_VARS_ALLOW_LIST=OPENWEATHER_API_KEY

4. No terminal, execute o comando para iniciar o n8n:
   bash
   docker-compose up -d

5. Acesse o n8n no seu navegador: `http://localhost:5678`

## Passos para importar o workflow no n8n

1. Abra a sua interface do n8n.
2. No menu lateral esquerdo, clique em **Workflows** e depois no botão **Add workflow**.
3. No canto superior direito, clique no menu de três pontos (opções) e selecione **Import from File**.
4. Selecione o arquivo `workflow-chatbot-telegram.json` incluído neste repositório.
5. O fluxo será carregado na sua tela.

## Configuração das Credenciais no n8n

Para que o workflow funcione, você precisa configurar as credenciais nos nós específicos.

### 1. Telegram (TELEGRAM_BOT_TOKEN)
* No workflow, clique no nó inicial **Telegram Trigger**.
* Em *Credential to connect with*, clique em *Create New Credential*.
* Insira o seu `TELEGRAM_BOT_TOKEN` gerado pelo BotFather e salve.
* **Importante:** Repita a seleção dessa credencial nos nós finais de envio de mensagem (**Telegram: Erro**, **Telegram: Erro1** e **Clima atual**).

### 2. OpenWeather (OPENWEATHER_API_KEY)
* Este workflow foi projetado com boas práticas de segurança. A chave da OpenWeather **não** é inserida diretamente na interface do nó.
* Ela é lida a partir das variáveis de ambiente do sistema operacional ou do Docker (arquivo `.env`).
* Certifique-se de que a variável `OPENWEATHER_API_KEY` está configurada no seu ambiente e declarada na lista de permissões do n8n (`N8N_ENV_VARS_ALLOW_LIST`). O nó **OpenWeather API** já está pré-configurado para ler a expressão `={{ $env.OPENWEATHER_API_KEY }}`.

### 3. Google Gemini (Opcional)
* O workflow possui um nó do **Google Gemini** (LangChain) localizado logo após o nó "Validação de Resposta", na rota de sucesso.
* Para ativá-lo, clique no nó **Message a model**.
* Crie ou selecione uma credencial do tipo *Google Gemini(PaLM) Api account* e insira a sua API Key do Google AI Studio.
* *Nota de Fallback:* Se você não configurar a chave do Gemini, o fluxo sofrerá um erro nesse nó, mas continuará automaticamente (graças à configuração `Continue On Fail`) para o nó de **Fallback** (Code node), garantindo o funcionamento do bot sem custos.

### Configuração de Webhook

O Telegram exige que o bot receba atualizações através de uma URL pública (HTTPS). Se você estiver rodando o n8n localmente (localhost), precisará expor a porta `5678` para a internet usando uma ferramenta como o [ngrok](https://ngrok.com/) ou Cloudflare Tunnels.

**Como configurar com o ngrok:**
1. Em um novo terminal, inicie o ngrok apontando para a porta do n8n:
   ```bash
   ngrok http 5678
   ```
2. Copie a URL `Forwarding` gerada pelo ngrok (ex: `https://abcd-12-34.ngrok-free.dev`).
3. Abra o arquivo `docker-compose.yml` e substitua o valor genérico da variável `WEBHOOK_URL` pela sua URL do ngrok gerada no passo anterior:
   ```yaml
   WEBHOOK_URL=[https://abcd-12-34.ngrok-free.dev](https://abcd-12-34.ngrok-free.dev)
   ```
4. Após atualizar a URL, suba o container novamente com `docker-compose up -d`.
```

## Como executar e testar o Chatbot

Com o workflow ativo (botão superior direito `Active` habilitado) e as credenciais configuradas, abra o Telegram e inicie uma conversa com o seu bot.

**Cenário de Sucesso:**
1. Envie uma mensagem no formato `Cidade,UF,BR`. Exemplo:
   > `São Paulo,SP,BR`
2. O bot processará o pedido e retornará:
   > `🌤️ A temperatura em São Paulo (SP) é de 25°C.`

**Cenários de Erro (Tratamento de Exceções):**
1. **Formato inválido:** Envie apenas `São Paulo` (sem a vírgula e o estado).
   * O bot responderá: `⚠️ Formato inválido. Por favor, envie a mensagem no padrão: Cidade,UF,BR (Exemplo: São Paulo,SP,BR).`
2. **Cidade inexistente:** Envie `CidadeInventadaXYZ, SP`.
   * O bot responderá: `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).`
