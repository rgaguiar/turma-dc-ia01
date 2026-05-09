# Aula Prática: Chat do Corretor de Imóveis — Roteador · RAG · Memória · Deploy

**Módulo:** 4 — Times de Agentes  
**Aula:** 16 (Aula 4 do módulo de Times)  
**Versão testada:** agno==2.6.5 · openai==2.32.0 · groq==1.2.0 · chromadb==1.5.9 · gradio==5.50.0  
**Última revisão:** 2025-05  

---

Nas três aulas anteriores, você aprendeu cada peça separadamente: times de agentes, ferramentas de busca, transcrição de áudio e leitura de documentos com RAG.

Nesta aula, montamos o projeto integrador — e com um tema próximo da realidade de negócios: um **chat de corretor de imóveis**. O sistema identifica a intenção do cliente, aciona o especialista certo e responde com precisão, usando dados reais de um catálogo, um manual de SAC e pesquisa na internet.

Ao final da aula, fazemos o deploy no HuggingFace Spaces com link permanente — o projeto sai do Colab e vai para o ar de verdade.

---

## 1. O Projeto: Chat do Corretor de Imóveis

### O Problema que Vamos Resolver

Uma imobiliária recebe perguntas de três tipos completamente diferentes:

> *"Tem apartamento de 2 quartos para alugar em Curitiba até R$ 4.000?"*

> *"Como funciona o caução? Posso usar FGTS para comprar?"*

> *"Tem boa escola particular perto de Fortaleza?"*

Cada pergunta exige uma fonte diferente: a primeira precisa do catálogo de imóveis. A segunda, do manual de processos da imobiliária. A terceira, de pesquisa atualizada na web. Um agente único tentando responder os três tipos confunde as fontes e perde precisão.

A solução é um **roteador** que lê a intenção do cliente e aciona o especialista certo.

### O Time

```
[Cliente digita a mensagem]
          │
          ▼
[Roteador]  ← Groq · Llama 3 (gratuito · sem memória)
    Classifica: IMOVEIS · LOCACAO · VIZINHANCA
          │
          ├──▶ [Agente de Imóveis]
          │       PythonTools escreve pandas → consulta imoveis.csv
          │       GPT-4o · temperature=0.0 · com memória
          │
          ├──▶ [Agente de Locação]
          │       RAG busca no sac_locacao.pdf
          │       GPT-4o mini · com memória
          │
          └──▶ [Agente de Vizinhança]
                  WebSearchTools pesquisa na web
                  GPT-4o mini · sem memória
                        │
                        ▼
              [Resposta no chat Gradio]
```

### Decisão de Modelos e Memória

| Agente | Modelo | Tool / Fonte | Memória | Por quê |
| --- | --- | --- | --- | --- |
| **Roteador** | Groq Llama 3 | — | ❌ Não | Só classifica — gratuito e rápido |
| **Corretor de Imóveis** | GPT-4o `temp=0.0` | PythonTools + CSV | ✅ Sim | Escreve pandas: precisa do modelo mais capaz. `temp=0.0` para dados exatos |
| **Especialista em Locação** | GPT-4o mini | RAG no PDF | ✅ Sim | Cliente faz follow-up: "e sobre o fiador?" — precisa lembrar do contexto |
| **Consultor de Vizinhança** | GPT-4o mini | WebSearchTools | ❌ Não | Cada consulta de cidade é independente — memória não agrega valor |

---

## 2. Os Arquivos do Projeto

Três arquivos precisam estar na mesma pasta — no Colab e no HuggingFace:

| Arquivo | O que é |
| --- | --- |
| `imoveis.csv` | 100 imóveis fictícios para locação e venda em capitais brasileiras |
| `sac_locacao.pdf` | Manual de SAC da ImoveisConnect — processos, caução, contratos |
| `script_aula.py` / `app.py` | O código (versão Colab ou versão HF Spaces) |

> 📌 **No Colab:** faça upload dos dois arquivos no painel lateral (aba Arquivos) antes de executar.

---

## 3. Construindo o Projeto Bloco a Bloco

### Bloco 1: Instalação

```python
# ===============================
# INSTALAÇÃO DAS DEPENDÊNCIAS
# ===============================
!pip install -q agno groq openai pypdf chromadb gradio ddgs pandas
```

| Pacote | Função |
| --- | --- |
| `pypdf` | Leitura do PDF de SAC pelo `PDFReader` do Agno |
| `chromadb` | Banco vetorial para indexar e buscar o PDF |
| `pandas` | Leitura do CSV de imóveis pelo agente com PythonTools |
| `ddgs` | Motor de busca usado internamente pelo `WebSearchTools` |
| Os demais | Já conhecidos das aulas anteriores |

---

### Bloco 2: Imports e Autenticação

```python
# ===============================
# FERRAMENTAS
# ===============================
import os
import pandas as pd
import gradio as gr
from google.colab import userdata

from agno.agent import Agent
from agno.models.groq import Groq
from agno.models.openai import OpenAIChat
from agno.tools.websearch import WebSearchTools
from agno.tools.python import PythonTools
from agno.db.sqlite import SqliteDb                 # ← import correto na versão atual do Agno

from agno.knowledge.knowledge import Knowledge
from agno.knowledge.reader.pdf_reader import PDFReader
from agno.knowledge.embedder.sentence_transformer import SentenceTransformerEmbedder
from agno.vectordb.chroma import ChromaDb

# ===============================
# AUTENTICAÇÃO — COLAB
# ===============================
os.environ["GROQ_API_KEY"]   = userdata.get('GROQ_API_KEY')
os.environ["OPENAI_API_KEY"] = userdata.get('OPENAI_API_KEY')
APP_PASSWORD = userdata.get('APP_PASSWORD')

print("✅ Autenticação concluída.")
```

> 📌 **`from agno.db.sqlite import SqliteDb`** — este é o import correto na versão 2.x do Agno. Versões anteriores usavam `agno.memory.db.sqlite` ou `agno.memory.agent`, que foram descontinuados. Se aparecer `ModuleNotFoundError`, confirme que está usando `agno.db.sqlite`.

---

### Bloco 3: Dataset de Imóveis

```python
# ===============================
# DATASET DE IMÓVEIS
# ===============================
# Lê APENAS o cabeçalho — zero linhas de dados
# O agente escreve pandas e consulta o arquivo com PythonTools
df_imoveis      = pd.read_csv("imoveis.csv", nrows=0)
colunas_imoveis = list(df_imoveis.columns)

print(f"✅ Estrutura do catálogo mapeada.")
print(f"   Colunas: {colunas_imoveis}")
```

> 💡 **`nrows=0` — por que ler só o cabeçalho?** Passamos o caminho do arquivo e os nomes das colunas nas `instructions`. O agente escreve o código pandas e executa com `PythonTools` na hora — mais preciso do que injetar o CSV inteiro no contexto.

---

### Bloco 4: RAG com o PDF de SAC

```python
# ===============================
# RAG — MANUAL DE SAC (PDF)
# ===============================
embedder = SentenceTransformerEmbedder(dimensions=384)

vector_db = ChromaDb(
    collection="sac_imoveis",
    path="./chromadb_sac",
    persistent_client=True,
    embedder=embedder,
)

knowledge_sac = Knowledge(vector_db=vector_db)

knowledge_sac.insert(
    path="sac_locacao.pdf",
    reader=PDFReader(chunk_size=1000, chunk_overlap=200)
)

print("✅ SAC de locação indexado com sucesso.")
```

> 💡 **Por que RAG no PDF e PythonTools no CSV?** São naturezas diferentes. O PDF é um documento narrativo — ideal para busca semântica. O CSV é uma tabela estruturada — ideal para código pandas. Cada ferramenta onde ela é mais forte.

---

### Bloco 5: Criando os Agentes

#### Roteador

```python
# ===============================
# AGENTE ROTEADOR
# ===============================
roteador = Agent(
    name="Roteador",
    model=Groq(id="llama-3.3-70b-versatile"),
    # Sem memória: classifica a intenção e retorna uma palavra — só isso
    instructions=[
        "Você analisa mensagens de clientes de uma imobiliária.",
        "Retorne APENAS uma das palavras abaixo, sem explicação:",
        "",
        "IMOVEIS    — busca de imóveis, preços, disponibilidade, quantidade, tipos, cidades",
        "LOCACAO    — caução, garantias, contrato, rescisão, documentos, FGTS, financiamento",
        "VIZINHANCA — infraestrutura da cidade: escolas, academias, hospitais, restaurantes, transporte",
        "",
        "Exemplos:",
        "'Tem ap de 2 quartos em Curitiba?' → IMOVEIS",
        "'Como funciona o caução?' → LOCACAO",
        "'Tem escola boa perto de Florianópolis?' → VIZINHANCA",
        "'Qual o melhor restaurante perto do Ibirapuera?' → VIZINHANCA",
        "",
        "Retorne SOMENTE a palavra. Sem ponto, sem explicação."
    ]
)
```

---

#### Agente de Imóveis

```python
# ===============================
# AGENTE DE IMÓVEIS
# ===============================
agente_imoveis = Agent(
    name="Corretor de Imóveis",
    role="Corretor de imóveis simpático e consultivo da ImoveisConnect",

    # GPT-4o com temperature=0.0 — dados precisam ser exatos, sem criatividade
    model=OpenAIChat(id="gpt-4o", temperature=0.0),

    # PythonTools: o agente escreve e executa pandas para consultar o CSV
    tools=[PythonTools()],

    # Memória por sessão: cliente pode perguntar "qual desses tem mais quartos?"
    db=SqliteDb(db_file="memoria_do_analista.db"),
    add_history_to_context=True,
    markdown=True,

    instructions=[
        "Você é o corretor virtual da ImoveisConnect — simpático, consultivo e preciso.",
        "Na PRIMEIRA mensagem do cliente, se apresente brevemente e pergunte o que ele procura.",
        "Sempre que precisar consultar imóveis, escreva um código Python usando PANDAS.",
        f"O arquivo do catálogo está em: 'imoveis.csv'",
        f"As colunas disponíveis são: {colunas_imoveis}",
        "REGRA CRÍTICA: filtre SEMPRE por disponivel == 'Sim' antes de apresentar imóveis ao cliente.",
        "Para estatísticas (preço médio, quantidade total), use o dataset completo.",
        "Apresente no máximo 3 imóveis por resposta — destaque os pontos fortes de cada um.",
        "Se o cliente não especificou cidade, tipo ou faixa de preço, pergunte para refinar.",
        "Nunca invente imóveis — use apenas os dados do arquivo.",
        "Se não houver resultado, sugira flexibilizar os critérios com empatia.",
        "Finalize sempre com: 'Posso agendar uma visita?' ou 'Quer ver mais opções?'",
    ]
)
```

> ⚠️ **Por que `gpt-4o` e não `gpt-4o-mini`?** O agente escreve código pandas com `PythonTools`. Código malformado trava o agente em loop de erro. O `gpt-4o` escreve pandas correto na primeira tentativa — o custo marginal vale a confiabilidade.

---

#### Agente de Locação

```python
# ===============================
# AGENTE DE LOCAÇÃO/SAC
# ===============================
agente_locacao = Agent(
    name="Especialista em Processos",
    role="Especialista em processos de locação, venda e documentação imobiliária",
    model=OpenAIChat(id="gpt-4o-mini"),
    knowledge=knowledge_sac,   # RAG nativo do Agno
    search_knowledge=True,     # busca no PDF automaticamente

    # Memória: o cliente faz follow-up — "e sobre o fiador?" precisa de contexto
    db=SqliteDb(db_file="memoria_do_analista.db"),
    add_history_to_context=True,
    markdown=True,

    instructions=[
        "Você é o especialista em processos da ImoveisConnect.",
        "Sempre pesquise na base de conhecimento antes de responder.",
        "Baseie respostas EXCLUSIVAMENTE no documento de SAC.",
        "Seja direto e use linguagem simples — máximo 3 parágrafos por resposta.",
        "Se não encontrar no documento: 'Não encontrei no nosso manual. Quer falar com um consultor?'",
        "Cite a seção do documento que embasou sua resposta.",
        "Se o cliente estiver em dúvida, ofereça: 'Posso explicar mais alguma etapa?'",
    ]
)
```

---

#### Agente de Vizinhança

```python
# ===============================
# AGENTE DE VIZINHANÇA
# ===============================
agente_vizinhanca = Agent(
    name="Consultor de Vizinhança",
    role="Especialista em infraestrutura urbana e qualidade de vida por cidade",
    model=OpenAIChat(id="gpt-4o-mini"),
    tools=[WebSearchTools(backend="bing")],

    # Sem memória: cada consulta de cidade é independente
    # Não há follow-up esperado — "me fale mais sobre Fortaleza" é nova busca
    markdown=True,

    instructions=[
        "Você é o consultor de vizinhança da ImoveisConnect.",
        "SEMPRE pesquise na web antes de responder. Nunca responda de memória.",

        # Guardrail crítico: evita resultados de outras cidades
        "REGRA DE BUSCA: inclua SEMPRE o nome exato da cidade em todas as suas pesquisas.",
        "Exemplo correto: 'melhores escolas particulares Fortaleza CE'",
        "Exemplo errado: 'melhores escolas particulares' (sem cidade → resultados de qualquer lugar)",
        "Se um resultado não mencionar explicitamente a cidade pedida, ignore e busque outro.",

        "Responda de forma concisa — máximo 5 itens por categoria.",
        "Cubra apenas o que o cliente perguntou: escolas, academias, hospitais, restaurantes ou transporte.",
        "Use linguagem leve e conversacional.",
        "Finalize com: 'Quer que eu mostre imóveis disponíveis em [cidade]?'",
    ]
)
```

> 🔥 **Lição Aprendida em Sala:** o `WebSearchTools` pode retornar resultados de outras cidades quando a busca é genérica. Uma escola pesquisada como "melhores escolas particulares" pode trazer resultados de São Paulo mesmo que o cliente tenha perguntado sobre Fortaleza. A instrução `"inclua SEMPRE o nome exato da cidade"` é o guardrail que resolve isso.

---

### Bloco 6: Função de Roteamento

```python
# ===============================
# FUNÇÃO DE ROTEAMENTO
# ===============================
def rotear_e_responder(mensagem, historico, session_id):
    # Passa as últimas 2 trocas do chat ao roteador para ele entender mensagens curtas
    # Exemplo: "curitiba" sozinho só faz sentido com o contexto "como é a infraestrutura?"
    contexto = ""
    if historico:
        for user_msg, bot_msg in historico[-2:]:
            contexto += f"Usuário: {user_msg}\nAssistente: {str(bot_msg)[:100]}...\n"

    prompt_roteador = f"{contexto}Nova mensagem: {mensagem}" if contexto else mensagem
    destino = roteador.run(prompt_roteador).content.strip().upper()

    if "IMOVEIS" in destino:
        resposta = agente_imoveis.run(mensagem, session_id=session_id).content
        icone    = "🏠 Corretor de Imóveis"

    elif "VIZINHANCA" in destino:
        resposta = agente_vizinhanca.run(mensagem, session_id=session_id).content
        icone    = "📍 Consultor de Vizinhança"

    else:
        resposta = agente_locacao.run(mensagem, session_id=session_id).content
        icone    = "📋 Especialista em Processos"

    return f"*{icone}*\n\n---\n\n{resposta}"
```

> 🔥 **Lição Aprendida em Sala:** mensagens curtas como "curitiba" ou "sim" sem contexto confundem o roteador — ele não sabe que é uma resposta à pergunta anterior sobre infraestrutura. Passar as últimas 2 trocas do histórico resolve isso sem custo significativo de tokens.

---

### Bloco 7: Interface Gradio com Chat e Senha

```python
# ===============================
# INTERFACE GRADIO COM CHAT E SENHA
# ===============================
with gr.Blocks(theme=gr.themes.Base(), title="Chat Corretor de Imóveis") as app:

    session_state = gr.State(value="")

    gr.Markdown("""
    # 🏠 ImoveisConnect — Chat do Corretor
    *Encontre o imóvel ideal ou tire dúvidas sobre locação e compra*
    """)

    with gr.Group(visible=True) as tela_login:
        gr.Markdown("### 🔐 Acesso ao Chat")
        senha_input = gr.Textbox(type="password", label="Senha de acesso",
                                  placeholder="Digite a senha para continuar...")
        btn_entrar  = gr.Button("Entrar", variant="primary")
        msg_erro    = gr.Markdown(visible=False)

    with gr.Group(visible=False) as interface_principal:
        chatbox = gr.Chatbot(label="Chat com o Corretor", height=450, bubble_full_width=False)
        with gr.Row():
            texto_input = gr.Textbox(
                placeholder="Ex: 'Tem ap 2 quartos em Curitiba?' ou 'Como funciona o caução?'",
                label="Sua mensagem", scale=5
            )
            btn_enviar = gr.Button("Enviar ▶", variant="primary", scale=2)
        gr.Markdown("---\n💡 *Imóveis por cidade · Processo de locação · Infraestrutura da cidade*")

    def verificar_senha(senha):
        if senha == APP_PASSWORD:
            return gr.update(visible=False), gr.update(visible=True), gr.update(visible=False), senha
        return gr.update(visible=True), gr.update(visible=False), gr.update(visible=True, value="❌ Senha incorreta."), ""

    def responder(mensagem, historico, session_id):
        if not mensagem.strip():
            return historico, ""
        resposta  = rotear_e_responder(mensagem, historico, session_id)  # ← passa historico
        historico = historico or []
        historico.append((mensagem, resposta))
        return historico, ""

    btn_entrar.click(fn=verificar_senha, inputs=[senha_input],
                     outputs=[tela_login, interface_principal, msg_erro, session_state])
    btn_enviar.click(fn=responder, inputs=[texto_input, chatbox, session_state],
                     outputs=[chatbox, texto_input])
    texto_input.submit(fn=responder, inputs=[texto_input, chatbox, session_state],
                       outputs=[chatbox, texto_input])

# ===============================
# INICIALIZANDO — COLAB
# ===============================
app.launch(share=True, debug=True)
```

---

## 4. Deploy no HuggingFace Spaces

### As Duas Diferenças entre Colab e HuggingFace

Ao fazer o deploy, você ajusta exatamente **dois pontos**:

| Ponto | Colab | HuggingFace Spaces |
| --- | --- | --- |
| **Import** | `from google.colab import userdata` | Remove essa linha |
| **Autenticação** | `userdata.get('CHAVE')` | `os.environ["CHAVE"]` via Secrets do painel |
| **Launch** | `app.launch(share=True, debug=True)` | `app.launch()` |

**Bloco de autenticação no `app.py`:**

```python
# ===============================
# AUTENTICAÇÃO — HUGGING FACE
# ===============================
# As chaves são lidas automaticamente dos Secrets do painel
# Não importar google.colab aqui
APP_PASSWORD = os.environ["APP_PASSWORD"]
```

**Launch no `app.py`:**

```python
# ===============================
# INICIALIZANDO — HUGGING FACE
# ===============================
app.launch()
```

### Arquivos para o HuggingFace Spaces

```
seu-space/
├── app.py              ← código com as alterações acima
├── requirements.txt    ← versões fixadas
├── imoveis.csv         ← dataset de imóveis
└── sac_locacao.pdf     ← manual de SAC
```

### `requirements.txt`

```
agno==2.6.5
groq==1.2.0
openai==2.32.0
pypdf==6.10.2
chromadb==1.5.9
gradio==5.50.0
pandas==2.2.2
```

> 📌 **Versões fixadas:** sempre fixe as versões no HuggingFace. Sem versão fixada, o Spaces instala a última versão disponível — que pode quebrar o código se o Agno lançar uma atualização.

### Configurar Secrets no HuggingFace

Em **Settings → Variables and Secrets**:

| Nome | Valor |
| --- | --- |
| `GROQ_API_KEY` | Sua chave da Groq |
| `OPENAI_API_KEY` | Sua chave da OpenAI |
| `APP_PASSWORD` | A senha que você quer usar |

---

## 5. Casos de Teste

```python
# TESTE 1 — Agente de Imóveis: filtro com pandas
# "Tem apartamento disponível para locação em Fortaleza até R$ 3.500?"
# Esperado: 🏠 Corretor — executa pandas com filtro cidade + finalidade + valor

# TESTE 2 — Agente de Imóveis: estatística
# "Qual o preço médio dos imóveis para venda no portfólio?"
# Esperado: 🏠 Corretor — pandas sem filtro de disponibilidade, retorna média geral

# TESTE 3 — Agente de Imóveis: memória entre turnos
# [Turno 1] "Me mostra apartamentos em Belo Horizonte para locação"
# [Turno 2] "Qual desses tem mais quartos?"
# Esperado: no turno 2 o agente lembra os imóveis apresentados no turno 1

# TESTE 4 — Agente de Locação: RAG no PDF
# "Como funciona o caução? Qual o valor máximo que a imobiliária pode cobrar?"
# Esperado: 📋 Especialista — seção 2.1 do SAC, resposta direta e concisa

# TESTE 5 — Agente de Locação: follow-up com memória
# [Turno 1] "Quais são as modalidades de garantia para locação?"
# [Turno 2] "E qual delas você recomenda para quem não tem fiador?"
# Esperado: no turno 2 o agente usa o contexto do turno 1

# TESTE 6 — Agente de Vizinhança: busca com cidade forçada
# "Quais as melhores escolas particulares em Fortaleza?"
# Esperado: 📍 Consultor — busca 'melhores escolas particulares Fortaleza CE'
#           Todos os resultados devem mencionar Fortaleza explicitamente

# TESTE 7 — Agente de Vizinhança: conexão com imóveis
# "Como é a infraestrutura de Curitiba para quem tem filhos?"
# Esperado: 📍 Consultor — escolas e parques de Curitiba + convite para ver imóveis

# TESTE 8 — Roteamento: pergunta ambígua
# "Quero saber sobre contratos e também ver imóveis em Salvador"
# Esperado: sistema não quebra — roteador escolhe o destino mais provável
```

---

## Gabarito: Código Completo para o Colab

```python
# ===============================
# INSTALAÇÃO
# ===============================
!pip install -q agno groq openai pypdf chromadb gradio ddgs pandas
```

```python
# ===============================
# FERRAMENTAS
# ===============================
import os
import pandas as pd
import gradio as gr
from google.colab import userdata

from agno.agent import Agent
from agno.models.groq import Groq
from agno.models.openai import OpenAIChat
from agno.tools.websearch import WebSearchTools
from agno.tools.python import PythonTools
from agno.db.sqlite import SqliteDb

from agno.knowledge.knowledge import Knowledge
from agno.knowledge.reader.pdf_reader import PDFReader
from agno.knowledge.embedder.sentence_transformer import SentenceTransformerEmbedder
from agno.vectordb.chroma import ChromaDb

# ===============================
# AUTENTICAÇÃO — COLAB
# ===============================
os.environ["GROQ_API_KEY"]   = userdata.get('GROQ_API_KEY')
os.environ["OPENAI_API_KEY"] = userdata.get('OPENAI_API_KEY')
APP_PASSWORD = userdata.get('APP_PASSWORD')
print("✅ Autenticação concluída.")

# ===============================
# DATASET DE IMÓVEIS
# ===============================
df_imoveis      = pd.read_csv("imoveis.csv", nrows=0)
colunas_imoveis = list(df_imoveis.columns)
print(f"✅ Colunas: {colunas_imoveis}")

# ===============================
# RAG — MANUAL DE SAC
# ===============================
embedder  = SentenceTransformerEmbedder(dimensions=384)
vector_db = ChromaDb(
    collection="sac_imoveis",
    path="./chromadb_sac",
    persistent_client=True,
    embedder=embedder,
)
knowledge_sac = Knowledge(vector_db=vector_db)
knowledge_sac.insert(
    path="sac_locacao.pdf",
    reader=PDFReader(chunk_size=1000, chunk_overlap=200)
)
print("✅ SAC indexado.")

# ===============================
# AGENTES
# ===============================
roteador = Agent(
    name="Roteador",
    model=Groq(id="llama-3.3-70b-versatile"),
    instructions=[
        "Analise a mensagem e retorne APENAS uma palavra:",
        "IMOVEIS    — busca de imóveis, preços, disponibilidade, tipos, cidades, quantidade",
        "LOCACAO    — caução, garantias, contrato, rescisão, documentos, FGTS, financiamento",
        "VIZINHANCA — infraestrutura da cidade: escolas, academias, hospitais, restaurantes, transporte",
        "Retorne SOMENTE a palavra, sem ponto ou explicação."
    ]
)

agente_imoveis = Agent(
    name="Corretor de Imóveis",
    role="Corretor de imóveis simpático e consultivo da ImoveisConnect",
    model=OpenAIChat(id="gpt-4o", temperature=0.0),
    tools=[PythonTools()],
    db=SqliteDb(db_file="memoria_do_analista.db"),
    add_history_to_context=True,
    markdown=True,
    instructions=[
        "Você é o corretor virtual da ImoveisConnect — simpático, consultivo e preciso.",
        "Na PRIMEIRA mensagem do cliente, se apresente brevemente e pergunte o que ele procura.",
        "Sempre que precisar consultar imóveis, escreva um código Python usando PANDAS.",
        f"O arquivo do catálogo está em: 'imoveis.csv'",
        f"As colunas disponíveis são: {colunas_imoveis}",
        "REGRA CRÍTICA: filtre SEMPRE por disponivel == 'Sim' antes de apresentar imóveis ao cliente.",
        "Para estatísticas (preço médio, quantidade total), use o dataset completo.",
        "Apresente no máximo 3 imóveis por resposta — destaque os pontos fortes de cada um.",
        "Se o cliente não especificou cidade, tipo ou faixa de preço, pergunte para refinar.",
        "Nunca invente imóveis — use apenas os dados do arquivo.",
        "Se não houver resultado, sugira flexibilizar os critérios com empatia.",
        "Finalize sempre com: 'Posso agendar uma visita?' ou 'Quer ver mais opções?'",
    ]
)

agente_locacao = Agent(
    name="Especialista em Processos",
    role="Especialista em processos de locação, venda e documentação imobiliária",
    model=OpenAIChat(id="gpt-4o-mini"),
    knowledge=knowledge_sac,
    search_knowledge=True,
    db=SqliteDb(db_file="memoria_do_analista.db"),
    add_history_to_context=True,
    markdown=True,
    instructions=[
        "Você é o especialista em processos da ImoveisConnect.",
        "Sempre pesquise na base de conhecimento antes de responder.",
        "Baseie respostas EXCLUSIVAMENTE no documento de SAC.",
        "Seja direto e use linguagem simples — máximo 3 parágrafos por resposta.",
        "Se não encontrar no documento: 'Não encontrei no nosso manual. Quer falar com um consultor?'",
        "Cite a seção do documento que embasou sua resposta.",
        "Se o cliente estiver em dúvida, ofereça: 'Posso explicar mais alguma etapa?'",
    ]
)

agente_vizinhanca = Agent(
    name="Consultor de Vizinhança",
    role="Especialista em infraestrutura urbana e qualidade de vida por cidade",
    model=OpenAIChat(id="gpt-4o-mini"),
    tools=[WebSearchTools(backend="bing")],
    markdown=True,
    instructions=[
        "Você é o consultor de vizinhança da ImoveisConnect.",
        "SEMPRE pesquise na web antes de responder. Nunca responda de memória.",
        "REGRA DE BUSCA: inclua SEMPRE o nome exato da cidade em todas as suas pesquisas.",
        "Exemplo correto: 'melhores escolas particulares Fortaleza CE'",
        "Exemplo errado: 'melhores escolas particulares' (sem cidade — resultados de qualquer lugar)",
        "Se um resultado não mencionar explicitamente a cidade pedida, ignore e busque outro.",
        "Responda de forma concisa — máximo 5 itens por categoria.",
        "Cubra apenas o que o cliente perguntou: escolas, academias, hospitais, restaurantes ou transporte.",
        "Use linguagem leve e conversacional.",
        "Finalize com: 'Quer que eu mostre imóveis disponíveis em [cidade]?'",
    ]
)

# ===============================
# ROTEAMENTO
# ===============================
def rotear_e_responder(mensagem, historico, session_id):
    contexto = ""
    if historico:
        for user_msg, bot_msg in historico[-2:]:
            contexto += f"Usuário: {user_msg}\nAssistente: {str(bot_msg)[:100]}...\n"

    prompt_roteador = f"{contexto}Nova mensagem: {mensagem}" if contexto else mensagem
    destino = roteador.run(prompt_roteador).content.strip().upper()

    if "IMOVEIS" in destino:
        resposta = agente_imoveis.run(mensagem, session_id=session_id).content
        icone    = "🏠 Corretor de Imóveis"
    elif "VIZINHANCA" in destino:
        resposta = agente_vizinhanca.run(mensagem, session_id=session_id).content
        icone    = "📍 Consultor de Vizinhança"
    else:
        resposta = agente_locacao.run(mensagem, session_id=session_id).content
        icone    = "📋 Especialista em Processos"
    return f"*{icone}*\n\n---\n\n{resposta}"

# ===============================
# INTERFACE GRADIO
# ===============================
with gr.Blocks(theme=gr.themes.Base(), title="Chat Corretor de Imóveis") as app:

    session_state = gr.State(value="")
    gr.Markdown("# 🏠 ImoveisConnect — Chat do Corretor\n*Encontre o imóvel ideal ou tire dúvidas sobre locação e compra*")

    with gr.Group(visible=True) as tela_login:
        gr.Markdown("### 🔐 Acesso ao Chat")
        senha_input = gr.Textbox(type="password", label="Senha de acesso", placeholder="Digite a senha...")
        btn_entrar  = gr.Button("Entrar", variant="primary")
        msg_erro    = gr.Markdown(visible=False)

    with gr.Group(visible=False) as interface_principal:
        chatbox = gr.Chatbot(label="Chat com o Corretor", height=450, bubble_full_width=False)
        with gr.Row():
            texto_input = gr.Textbox(
                placeholder="Ex: 'Tem ap 2 quartos em Curitiba?' ou 'Como funciona o caução?'",
                label="Sua mensagem", scale=5
            )
            btn_enviar = gr.Button("Enviar ▶", variant="primary", scale=2)
        gr.Markdown("---\n💡 *Imóveis por cidade · Processo de locação · Infraestrutura da cidade*")

    def verificar_senha(senha):
        if senha == APP_PASSWORD:
            return gr.update(visible=False), gr.update(visible=True), gr.update(visible=False), senha
        return gr.update(visible=True), gr.update(visible=False), gr.update(visible=True, value="❌ Senha incorreta."), ""

    def responder(mensagem, historico, session_id):
        if not mensagem.strip():
            return historico, ""
        resposta  = rotear_e_responder(mensagem, historico, session_id)  # ← passa historico
        historico = historico or []
        historico.append((mensagem, resposta))
        return historico, ""

    btn_entrar.click(fn=verificar_senha, inputs=[senha_input],
                     outputs=[tela_login, interface_principal, msg_erro, session_state])
    btn_enviar.click(fn=responder, inputs=[texto_input, chatbox, session_state],
                     outputs=[chatbox, texto_input])
    texto_input.submit(fn=responder, inputs=[texto_input, chatbox, session_state],
                       outputs=[chatbox, texto_input])

# ===============================
# INICIALIZANDO — COLAB
# ===============================
app.launch(share=True, debug=True)
```

```python
# ===============================
# requirements.txt — para o HuggingFace Spaces
# Execute essa célula para gerar o arquivo
# ===============================
%%writefile requirements.txt
agno==2.6.5
groq==1.2.0
openai==2.32.0
pypdf==6.10.2
chromadb==1.5.9
gradio==5.50.0
pandas==2.2.2
```
