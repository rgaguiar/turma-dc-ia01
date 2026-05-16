# Aula Prática: N8N + WAHA — Do Docker ao Agente de WhatsApp com IA

**Módulo:** 5 — Automação de Fluxos de Trabalho  
**Ambiente:** Docker Desktop + N8N + WAHA  
**Última revisão:** 2025-05  

---

Nas aulas anteriores, toda a automação foi feita com código Python no Google Colab. Nesta aula, usamos uma abordagem completamente diferente: **automação visual com N8N**.

Em vez de escrever código, você arrasta, conecta e configura blocos numa interface visual. O resultado final é o mesmo — um agente de IA respondendo no WhatsApp — mas o caminho é muito mais acessível para profissionais de negócios.

> ⚠️ **Aviso importante:** o WhatsApp não permite oficialmente bots ou clientes não oficiais. O WAHA simula o WhatsApp Web. Para uso pessoal e de estudo funciona bem. Para uso comercial em escala, a alternativa oficial é a [Meta Business API](https://business.whatsapp.com/).

---

## 1. Formas de Executar o N8N

Antes de instalar, é importante entender as opções disponíveis — cada uma tem um contexto adequado.

| Modalidade | Custo | Para quem | Status |
| --- | --- | --- | --- |
| **Desktop App** | Gratuito | — | ❌ Descontinuado |
| **Node Platform** | Gratuito | Desenvolvedores | ✅ Disponível |
| **Docker** | Gratuito | Estudo e experimentos | ✅ **Usaremos nesta aula** |
| **N8N Cloud** | Pago | Produção profissional | ✅ Disponível |
| **Self-host (VPS)** | Custo do servidor | Automações próprias | ✅ Disponível |

### Desktop App
Versão com interface nativa para Windows e Mac. Foi **descontinuada** pelo time do N8N — não recebe mais atualizações e não é recomendada para novos projetos.

### Node Platform
Instala o N8N como um pacote Node.js via terminal (`npm install -g n8n`). Requer conhecimento mínimo de Node.js e gerenciamento de processos. Não é a opção mais acessível para profissionais de negócios.

### Docker ← usaremos nesta aula

| ✅ Vantagens | ⚠️ Desvantagens |
| --- | --- |
| Zero configuração de ambiente | Requer Docker Desktop instalado |
| Funciona igual em qualquer sistema | Para quando o computador desliga |
| Fácil de recriar se algo quebrar | Não indicado para produção em escala |
| Ideal para aprender e experimentar | Dados ficam no computador local |
| Gratuito sem limite de workflows | Sem suporte oficial |

> 💡 **Por que Docker para esta aula?** É a forma mais rápida de ter o N8N e o WAHA funcionando sem precisar configurar ambiente, instalar dependências ou criar conta. Em 10 minutos você tem tudo rodando — e se algo der errado, basta recriar o container. É a escolha certa para aprender.

### N8N Cloud

Versão hospedada e gerenciada pelo próprio N8N. Melhor performance, atualizações automáticas, suporte e infraestrutura por conta deles.

| ✅ Vantagens | ⚠️ Desvantagens |
| --- | --- |
| Sem gerenciar servidor | Pago (a partir de ~$20/mês) |
| Alta disponibilidade 24h | Dados ficam nos servidores deles |
| Suporte oficial | Custo recorrente |
| Atualizações automáticas | — |

> 📌 **Quando migrar para N8N Cloud?** Quando o projeto sair do experimento e virar uma automação que outras pessoas dependem — clientes, equipe, processos críticos. Para uso pessoal e aprendizado, Docker é suficiente.

### Self-host (VPS)

Instalar o N8N num servidor próprio (DigitalOcean, Hostinger, AWS). Você tem controle total, mas é responsável por tudo — atualizações, backups, disponibilidade.

| ✅ Vantagens | ⚠️ Desvantagens |
| --- | --- |
| Roda 24h sem depender do seu computador | Você gerencia o servidor |
| Custo menor que N8N Cloud (~R$25/mês) | Sem suporte oficial do N8N |
| Dados ficam no seu servidor | Requer conhecimento mínimo de VPS |
| Controle total | Você é responsável por backups |

> 💡 **Caminho natural de evolução:** Docker (aprender) → Self-host VPS (automações próprias) → N8N Cloud (quando precisar de escala e suporte).

---

## 2. Conceitos Fundamentais

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

## 3. O Que Você Vai Precisar

| Ferramenta | O que é | Link |
| --- | --- | --- |
| **Docker Desktop** | Roda o N8N e o WAHA na sua máquina | [docker.com/products/docker-desktop](https://docker.com/products/docker-desktop) |
| **Conta Groq** | Chave de API gratuita para o LLM | [console.groq.com](https://console.groq.com) |
| **Conta OpenAI** (opcional) | Chave de API para modelos GPT | [platform.openai.com](https://platform.openai.com) |
| **WhatsApp no celular** | O número que será conectado ao bot | — |

---

## 4. Passo a Passo Completo

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

### Passo 6: Primeiro Projeto com WhatsApp — Ping Pong

Antes de adicionar IA, vamos confirmar que o WAHA está funcionando com um projeto simples: você manda uma mensagem e o WhatsApp responde automaticamente com o que você digitou.

> 💡 **Por que fazer isso primeiro?** Porque separa dois problemas distintos. Se o ping pong funciona, o WAHA e o N8N estão conectados corretamente. Se depois o agente com IA não funciona, o problema está na configuração da IA — não na infraestrutura.

**Fluxo:**
```
[WAHA Trigger] → [Edit Fields (Set)] → [Send Seen] → [Send a Text Message]
```

#### Os 4 nós do Ping Pong

**WAHA Trigger** — recebe a mensagem do WhatsApp

| Campo | Valor |
| --- | --- |
| Session | `default` |
| Events | `message` |

**Edit Fields (Set)** — organiza os dados para os próximos nós

| Campo criado | Expressão |
| --- | --- |
| `from` | `{{ $json.payload.from }}` |
| `mensagem` | `{{ $json.payload.body }}` |
| `session` | `{{ $json.payload.session }}` |
| `id_mensagem` | `{{ $json.payload.id }}` |

**Send Seen** — marca a mensagem como lida (o "✓✓ azul" no WhatsApp)

| Campo | Valor |
| --- | --- |
| Session | `{{ $json.session }}` |
| Chat ID | `{{ $json.from }}` |
| Message ID | `{{ $json.id_mensagem }}` |

> 📌 **Por que o Send Seen?** É uma boa prática — indica para o cliente que a mensagem foi recebida enquanto o agente processa. Em bots profissionais, esse nó sempre aparece antes da resposta.

**Send a Text Message** — envia a resposta de volta

| Campo | Valor |
| --- | --- |
| Session | `{{ $json.session }}` |
| Chat ID | `{{ $json.from }}` |
| Message | `Você digitou: {{ $json.mensagem }}` |

> 🔥 **Lição em sala:** quando funcionar, você vai ver o WhatsApp receber "Você digitou: olá" em menos de 2 segundos.

---

### Passo 7: Workflow Completo — WAHA + AI Agent

Com o ping pong funcionando, adicionamos inteligência. Substituímos o **Send a Text Message** pelo **AI Agent** — o restante do fluxo permanece igual.

**Fluxo:**
```
[WAHA Trigger] → [Edit Fields (Set)] → [Send Seen] → [AI Agent] → [Send a Text Message]
```

#### Configuração dos nós novos

**AI Agent**

| Campo | Valor |
| --- | --- |
| Prompt | `{{ $json.mensagem }}` |
| LLM | Groq Chat Model — `llama-3.3-70b-versatile` |
| Memory | Window Buffer Memory |
| Session Key | `{{ $json.from }}` |
| System prompt | *Instruções do agente — adapte para seu caso de uso* |

**Send a Text Message** — agora envia a resposta do agente

| Campo | Valor |
| --- | --- |
| Session | `{{ $json.session }}` |
| Chat ID | `{{ $json.from }}` |
| Message | `{{ $('AI Agent').item.json.output }}` |

> 📌 **Ativar o workflow:** clique no toggle **Inactive → Active** no canto superior direito. Sem isso, o WAHA Trigger não recebe mensagens mesmo com tudo configurado.


---

## 5. Testando o Sistema

### Teste do Ping Pong (Passo 6)

Com o workflow de ping pong ativo, mande qualquer mensagem:

```
# TESTE 1 — Ping pong básico
Você: "olá"
Esperado: "Você digitou: olá"

# TESTE 2 — Confirmação do Send Seen
Você: qualquer mensagem
Esperado: os dois checks ficam azuis (✓✓) antes da resposta chegar
```

Se funcionar, a infraestrutura está 100% correta. Prossiga para o Passo 7.

### Teste do Agente com IA (Passo 7)

```
# TESTE 3 — Resposta simples
Você: "Olá, tudo bem?"
Esperado: agente responde com uma saudação

# TESTE 4 — Memória
Você: "Meu nome é Rafael"
[aguarda resposta]
Você: "Qual é o meu nome?"
Esperado: agente lembra o nome mencionado anteriormente

# TESTE 5 — Cálculo (se Calculator estiver configurado como tool)
Você: "Quanto é 15% de 3500?"
Esperado: agente usa a calculadora e responde corretamente
```

Verifique o painel **Executions** no N8N para ver o fluxo de cada mensagem.

---

## 6. Problemas Comuns

| Problema | Causa | Solução |
| --- | --- | --- |
| N8N não abre em `localhost:5678` | Container não iniciou | Verifique os logs no Docker Desktop |
| WAHA não recebe mensagens | Webhook URL incorreta | Copie a Production URL do WAHA Trigger e configure no container WAHA |
| QR code expirou | Demora para escanear | Clique no ícone de QR de novo no dashboard |
| Bot não responde | Workflow inativo | Ative o toggle Inactive → Active |
| `host.docker.internal` não funciona (Linux) | Linux não suporta nativamente | Use o IP do host: `172.17.0.1` |
| Memória não persiste entre reinicializações | Window Buffer Memory é em RAM | Para persistência use Postgres Chat Memory com container PostgreSQL |

---

## 7. Estrutura dos Containers

```
Docker Desktop
├── n8n-dev     (porta 5678)  ← interface do N8N
└── waha-dev    (porta 3000)  ← dashboard do WAHA + API WhatsApp
```

> 📌 **Comunicação entre containers:**  
> O WAHA usa `http://host.docker.internal:5678` para chamar o N8N.  
> O N8N usa `http://localhost:3000` para chamar o WAHA (quando necessário).

---

## 8. Próxima Aula — Parte 2

Na próxima semana vamos expandir o sistema com:

- **RAG com ChromaDB** — agente lê documentos PDF antes de responder
- **Google Calendar** — automação de agenda
- **Google Sheets** — registro automático de conversas
- **Pipeline completo** — agente + WhatsApp + dados integrados

---

## 9. Gabarito — Resumo das Configurações

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
