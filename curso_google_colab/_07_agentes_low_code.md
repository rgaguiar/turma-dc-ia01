# Aula Prática: N8N + WAHA — Do Docker ao Agente de WhatsApp com IA

**Módulo:** 5 — Automação de Fluxos de Trabalho  
**Ambiente:** Docker Desktop + N8N + WAHA  
**Última revisão:** 2025-05  

---

Nas aulas anteriores, toda a automação foi feita com código Python no Google Colab. Nesta aula, usamos uma abordagem completamente diferente: **automação visual com N8N**.

Em vez de escrever código, você arrasta, conecta e configura blocos numa interface visual. O resultado final é o mesmo — um agente de IA respondendo no WhatsApp — mas o caminho é muito mais acessível para profissionais de negócios.

> ⚠️ **Aviso importante:** o WhatsApp não permite oficialmente bots ou clientes não oficiais. O WAHA simula o WhatsApp Web. Para uso pessoal e de estudo funciona bem. Para uso comercial em escala, a alternativa oficial é a [Meta Business API](https://business.whatsapp.com/).

---

## 1. Conceitos Fundamentais

### O que é o N8N?

N8N é uma ferramenta de automação visual — você conecta blocos para criar fluxos de trabalho automáticos.

> 💡 **Analogia:** N8N é como um gerente de processos que nunca dorme. Você descreve o fluxo uma vez e ele executa sozinho, para sempre, sem precisar de você.

| N8N (esta aula) | Python + Colab (aulas anteriores) |
| --- | --- |
| Arrasta e conecta blocos | Escreve código Python |
| Roda no seu computador 24h | Para quando fecha o Colab |
| Interface visual — qualquer pessoa entende | Requer conhecimento de programação |
| Deploy local ou em VPS | Deploy no HuggingFace |

### Nó (Node) — A Caixa

O nó é o bloco básico do N8N. Cada nó executa **uma** ação específica. Existem dois tipos:

**Trigger (Gatilho):** inicia o fluxo. Fica escutando e dispara quando algo acontece.
- `When chat message received` — chat interno do N8N
- `WAHA Trigger` — mensagem chega no WhatsApp
- `Schedule` — dispara num horário definido
- `Webhook` — recebe uma requisição HTTP

**Ação:** executa algo depois que o trigger disparou.
- `AI Agent` — orquestra o modelo de IA e as ferramentas
- `Edit Fields (Set)` — organiza e transforma os dados
- `WAHA Send Message` — envia mensagem pelo WhatsApp
- `Google Sheets` — lê ou escreve numa planilha

> 💡 **Analogia:** o trigger é a campainha do escritório. Os nós de ação são os departamentos que fazem o trabalho depois que ela toca.

### Workflow — O Fluxo

Sequência de nós conectados por linhas. A execução sempre começa no trigger e segue pelas conexões.

```
[WAHA Trigger] → [Edit Fields] → [AI Agent] → [WAHA Send Message]
```

### Dados — JSON

Os nós trocam dados em formato JSON. Para acessar dados de nós anteriores:

```
{{ $json.payload.from }}    ← número de quem enviou a mensagem
{{ $json.payload.body }}    ← texto da mensagem
{{ $json.session }}         ← sessão do WAHA
```

### Execuções

Cada vez que o trigger dispara, o N8N cria uma **execução** — um registro completo do que aconteceu, com os dados em cada etapa. Use o painel **Executions** para debugar erros.

### Webhook

Webhook é um "aviso automático". Em vez do bot ficar perguntando "chegou mensagem?", o WAHA avisa o N8N automaticamente toda vez que uma mensagem chega — via uma chamada HTTP para uma URL que você configura.

> 💡 **Analogia:** webhook é como a campainha do escritório. Quem chega (mensagem do WhatsApp) aperta a campainha (envia um POST HTTP) e o N8N atende.

---

## 2. O Que Você Vai Precisar

| Ferramenta | O que é | Link |
| --- | --- | --- |
| **Docker Desktop** | Roda o N8N e o WAHA na sua máquina | [docker.com/products/docker-desktop](https://docker.com/products/docker-desktop) |
| **Conta Groq** | Chave de API gratuita para o LLM | [console.groq.com](https://console.groq.com) |
| **Conta OpenAI** (opcional) | Chave de API para modelos GPT | [platform.openai.com](https://platform.openai.com) |
| **WhatsApp no celular** | O número que será conectado ao bot | — |

---

## 3. Passo a Passo Completo

### Passo 1: Instalar o N8N via Docker Desktop

1. Abrir o **Docker Desktop**
2. Na barra de busca, digitar `n8n`
3. Selecionar `n8nio/n8n` → clicar em **Pull** → depois em **Run**
4. Expandir **Optional settings** e configurar:

| Campo | Valor |
| --- | --- |
| Container name | `n8n-dev` |
| Host port | `5678` |
| Container path | `/home/node/.n8n` |
| Host path | *(escolha uma pasta no seu computador)* |
| `GENERIC_TIMEZONE` | `America/Sao_Paulo` |

5. Clicar em **Run**
6. Após iniciar, clicar no link **5678:5678** no Docker Desktop
7. Acessar `localhost:5678` no navegador → criar conta owner

> 📌 O **Host path** é onde os dados do N8N são salvos no seu computador. Se o container for recriado, seus workflows não se perdem.

---

### Passo 2: Primeiro Agente no N8N (sem WhatsApp ainda)

Antes de conectar o WhatsApp, vamos entender como o AI Agent funciona.

**Fluxo:**
```
[When chat message received] → [AI Agent]
```

**Sub-nós do AI Agent:**

| Sub-nó | Função | Configuração |
| --- | --- | --- |
| **Groq Chat Model** | O modelo de linguagem | Model: `llama-3.3-70b-versatile` |
| **Window Buffer Memory** | Memória da conversa por sessão | Session Key: `{{ $json.sessionId }}` |
| **Calculator** | Ferramenta de cálculo (exemplo) | — configuração automática |

**Como testar:**
- Clicar em **Chat** no canto inferior esquerdo do workflow
- Digitar uma mensagem e ver a resposta do agente

> 💡 **O AI Agent é o nó central.** Ele orquestra os sub-nós: o modelo de linguagem (LLM), a memória (histórico da conversa) e as ferramentas (calculadora, busca na web, etc.). É a mesma lógica do `Agent` no Agno — só que visual.

---

### Passo 3: Instalar a Extensão WAHA no N8N

O N8N não vem com suporte ao WAHA por padrão. Precisamos instalar um node da comunidade.

1. No N8N → menu lateral → **Settings**
2. Clicar em **Community nodes**
3. Clicar em **Install**
4. No campo **npm Package Name**, digitar:

```
@devlikeapro/n8n-nodes-waha
```

5. Marcar o checkbox de confirmação de risco
6. Clicar em **Install**

**Resultado esperado:** aparece na lista de Community nodes:
```
@devlikeapro/n8n-nodes-waha  v2025.x.x
2 nodes: WAHA · WAHA Trigger
```

---

### Passo 4: Instalar o WAHA no Docker Desktop

Mesmo processo do N8N — mas com variáveis de ambiente específicas.

1. No Docker Desktop, buscar por `waha`
2. Selecionar `devlikeapro/waha` → **Pull** → **Run**
3. Expandir **Optional settings** e configurar:

| Variável | Valor | Para que serve |
| --- | --- | --- |
| Container name | `waha-dev` | Nome do container |
| Host port | `3000` | Porta do dashboard |
| `WHATSAPP_DEFAULT_ENGINE` | `NOWEB` | Engine sem browser (mais leve) |
| `WHATSAPP_HOOK_EVENTS` | `message` | Quais eventos enviar ao N8N |
| `WHATSAPP_HOOK_URL` | `http://host.docker.internal:5678/webhook/SEU-WEBHOOK` | URL do N8N para receber eventos |
| `WAHA_NO_API_KEY` | `True` | Sem autenticação por API key |
| `WAHA_DASHBOARD_NO_PASSWC` | `True` | Dashboard sem senha |
| `WHATSAPP_SWAGGER_NO_PASS` | `True` | Swagger sem senha |

4. Clicar em **Run**
5. Após iniciar, clicar no link **3000:3000** no Docker Desktop

> ⚠️ **Por que `host.docker.internal` e não `localhost`?**  
> O WAHA roda dentro do Docker — para ele, `localhost` é o próprio container, não o seu computador. O endereço especial `host.docker.internal` permite que um container acesse serviços rodando na máquina host (onde o N8N está).

> 📌 **A URL do webhook:** você pega essa URL no N8N → nó WAHA Trigger → aba **Parameters** → copiar a **Production URL**.

---

### Passo 5: Conectar o WhatsApp ao WAHA

1. Acessar o dashboard do WAHA em `localhost:3000`
2. Clicar em **Connect** na seção Workers
3. Em **Sessions** → na linha `default` → clicar no ícone de QR code
4. Abrir o WhatsApp no celular:
   - **Configurações** → **Dispositivos conectados** → **Conectar dispositivo**
   - Escanear o QR code exibido no dashboard
5. Aguardar o status mudar para **WORKING**

> 📌 O QR code expira em ~30 segundos. Se expirar antes de escanear, clique no ícone de QR de novo para gerar um novo.

---

### Passo 6: Workflow Completo — WAHA + AI Agent

Agora substituímos o `When chat message received` pelo **WAHA Trigger**. A arquitetura é a mesma — só muda o canal de entrada.

**Fluxo:**
```
[WAHA Trigger] → [Edit Fields (Set)] → [AI Agent] → [WAHA Send Message]
```

#### Configuração de cada nó

**WAHA Trigger**

| Campo | Valor |
| --- | --- |
| Session | `default` |
| Events | `message` |

**Edit Fields (Set)** — organiza os dados que chegam do WAHA

| Campo criado | Expressão |
| --- | --- |
| `from` | `{{ $json.payload.from }}` |
| `mensagem` | `{{ $json.payload.body }}` |
| `session` | `{{ $json.payload.session }}` |
| `id_mensagem` | `{{ $json.payload.id }}` |

**AI Agent**

| Campo | Valor |
| --- | --- |
| Prompt | `{{ $json.mensagem }}` |
| LLM | Groq Chat Model — `llama-3.3-70b-versatile` |
| Memory | Window Buffer Memory |
| Session Key | `{{ $json.from }}` (cada número tem sua própria memória) |
| System prompt | *Instruções do agente — adapte para seu caso de uso* |

**WAHA Send Message**

| Campo | Valor |
| --- | --- |
| Session | `{{ $json.session }}` |
| Chat ID | `{{ $json.from }}` |
| Message | `{{ $('AI Agent').item.json.output }}` |

> 📌 **Ativar o workflow:** clique no toggle **Inactive → Active** no canto superior direito. Sem isso, o WAHA Trigger não recebe mensagens mesmo com tudo configurado.

---

## 4. Testando o Sistema

Com o workflow ativo, mande uma mensagem no WhatsApp para o número conectado ao WAHA:

```
# TESTE 1 — Resposta simples
Você: "Olá, tudo bem?"
Esperado: agente responde no WhatsApp

# TESTE 2 — Memória
Você: "Meu nome é Rafael"
[aguarda resposta]
Você: "Qual é o meu nome?"
Esperado: agente lembra o nome mencionado anteriormente

# TESTE 3 — Cálculo (se Calculator estiver configurado como tool)
Você: "Quanto é 15% de 3500?"
Esperado: agente usa a calculadora e responde corretamente
```

Verifique o painel **Executions** no N8N para ver o fluxo de cada mensagem — com os dados em cada etapa.

---

## 5. Problemas Comuns

| Problema | Causa | Solução |
| --- | --- | --- |
| N8N não abre em `localhost:5678` | Container não iniciou | Verifique os logs no Docker Desktop |
| WAHA não recebe mensagens | Webhook URL incorreta | Copie a Production URL do WAHA Trigger e configure no container WAHA |
| QR code expirou | Demora para escanear | Clique no ícone de QR de novo no dashboard |
| Bot não responde | Workflow inativo | Ative o toggle Inactive → Active |
| `host.docker.internal` não funciona (Linux) | Linux não suporta nativamente | Use o IP do host: `172.17.0.1` |
| Memória não persiste entre reinicializações | Window Buffer Memory é em RAM | Para persistência use Postgres Chat Memory com container PostgreSQL |

---

## 6. Estrutura dos Containers

```
Docker Desktop
├── n8n-dev     (porta 5678)  ← interface do N8N
└── waha-dev    (porta 3000)  ← dashboard do WAHA + API WhatsApp
```

> 📌 **Comunicação entre containers:**  
> O WAHA usa `http://host.docker.internal:5678` para chamar o N8N.  
> O N8N usa `http://localhost:3000` para chamar o WAHA (quando necessário).

---

## 7. Próxima Aula — Parte 2

Na próxima semana vamos expandir o sistema com:

- **RAG com ChromaDB** — agente lê documentos PDF antes de responder
- **Google Calendar** — automação de agenda
- **Google Sheets** — registro automático de conversas
- **Pipeline completo** — agente + WhatsApp + dados integrados

---

## Gabarito — Resumo das Configurações

### Container N8N (Docker Desktop)

```
Image:            n8nio/n8n:latest
Container name:   n8n-dev
Host port:        5678
Container path:   /home/node/.n8n
Host path:        /sua/pasta/local/n8n

Environment variables:
  GENERIC_TIMEZONE = America/Sao_Paulo
```

### Container WAHA (Docker Desktop)

```
Image:            devlikeapro/waha:latest
Container name:   waha-dev
Host port:        3000
Container path:   /tmp/

Environment variables:
  WHATSAPP_DEFAULT_ENGINE  = NOWEB
  WHATSAPP_HOOK_EVENTS     = message
  WHATSAPP_HOOK_URL        = http://host.docker.internal:5678/webhook/SEU-WEBHOOK-ID/waha
  WAHA_NO_API_KEY          = True
  WAHA_DASHBOARD_NO_PASSWC = True
  WHATSAPP_SWAGGER_NO_PASS = True
```

### Expressões úteis no N8N

```javascript
// Dados que chegam do WAHA Trigger
$json.payload.from       // número do remetente (ex: 5585912345678@c.us)
$json.payload.body       // texto da mensagem
$json.payload.session    // sessão do WAHA (ex: "default")
$json.payload.id         // ID único da mensagem

// Após o Edit Fields (Set)
$json.from               // número organizado
$json.mensagem           // texto organizado
$json.session            // sessão organizada

// Output do AI Agent
$('AI Agent').item.json.output   // resposta do agente
```

### Pacote da comunidade WAHA

```
@devlikeapro/n8n-nodes-waha
```

Instalar em: **Settings → Community nodes → Install**
