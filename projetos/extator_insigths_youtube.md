# Extrator de Insights de Vídeos do YouTube com IA

### Ador do Negócio
A informação de mercado e os anúncios de concorrentes ocorrem cada vez mais em formatos audiovisuais longos (webinars, keynotes, reuniões de resultados). O problema é que ninguém no time de Vendas ou Marketing possui o tempo 
operacional necessário para assistir a essas transmissões na íntegra. Consumir vídeo é lento, não indexável e gera um gargalo de inteligência competitiva nas empresas, atrasando a formulação de estratégias de resposta.

### Contextualização das Ferramentas
* **Agno e Groq**: O motor lógico da aplicação. Usaremos o modelo Llama 3.3 via Groq, famoso por sua altíssima velocidade de processamento.
* **YouTubeTools**: Uma biblioteca de extração que possibilita ao sistema conectar-se a uma URL e baixar a transcrição completa do vídeo de forma automatizada.
* **Gradio**: Biblioteca para o desenvolvimento da interface com o usuário final.

### Arquitetura do Sistema 
A arquitetura utilizada é a de Injeção de Contexto (Context Injection) com tratamento de dados:

1. O usuário fornece a URL do vídeo e uma instrução de negócios.
2. O sistema aciona a ferramenta YouTubeTools em Python para extrair as legendas.
3. O texto extraído recebe tratamento (limite de tamanho para evitar falhas de memória).
4. O texto é injetado diretamente no Agente junto com a instrução do usuário.
5. O Agente aplica o framework analítico sobre o texto recuperado e gera a resposta.

## O Processo de Construção e Explicação do código por Blocos
#### BLOCO 1: FERRAMENTAS E AUTENTICAÇÃO

Iniciamos o projeto configurando o acesso às APIs da Groq e importando a ferramenta YouTubeTools. A grande diferença arquitetural aqui é que parametrizamos a ferramenta para extrair dados em múltiplos idiomas (PT e EN), 
permitindo o monitoramento global de concorrentes.

```python
# ===============================
# BLOCO 1: FERRAMENTAS E AUTENTICAÇÃO
# ===============================
import os
import gradio as gr
from agno.agent import Agent
from agno.models.groq import Groq
from agno.tools.youtube import YouTubeTools
from google.colab import userdata

os.environ["GROQ_API_KEY"] = userdata.get("GROQ_API_KEY")

# Tool do YouTube configurada para o ambiente corporativo
youtube_tool = YouTubeTools(
    enable_get_video_captions=True,
    languages=["pt","en"]
)
```

#### BLOCO 2: O AGENTE ESPECIALISTA

Neste exemplo, a inteligência artificial não buscará a resposta em sua memória base, mas será rigorosamente parametrizada para atuar apenas sobre o texto injetado. A forma como estruturamos o prompt força a IA a converter 
um conteúdo massivo e não estruturado em um relatório executivo de rápida leitura.

```python
# ===============================
# BLOCO 2: O AGENTE ESPECIALISTA
# ===============================
agent = Agent(
    model=Groq(id="llama-3.3-70b-versatile", temperature=0.5),

    # 1. Cargo / Persona
    role="Estrategista de Inteligência de Mercado",

    # 2. Missão Principal
    description="Analisar transcrições de vídeos do YouTube e extrair insights executivos baseados no conteúdo do vídeo.",

    # Regras de Execução
    instructions=[
        # 3. Público
        "Seu público são executivos que precisam extrair rapidamente o conhecimento central do vídeo.",
        
        # 4. Profundidade
        "Analise a transcrição recebida e gere um relatório estruturado capturando nuances importantes.",
        
        # 5. Estrutura Obrigatória
        "Analise a transcrição e gere um relatório estruturado no formato:\n**1. Visão Geral do Vídeo**\n**2. Tópicos Abordados**\n**3. Conceitos-Chave e Argumentos**\n**4.Plano de Ação**",
        
        # 6. Limite de Escopo (GUARDRAIL DE COMPLIANCE)
        "REGRA DE OURO: Baseie sua resposta APENAS na transcrição do vídeo fornecida na mensagem. Não invente dados.",
        
        # 7. Estilo
        "Seja analítico, objetivo e fiel ao conteúdo."
    ]
)
```
Configuração do Agente: Os Porquês

* **Modelo** (`llama-3.3-70b-versatile`): Escolhido pela sua alta capacidade de compreensão aliada à baixíssima latência da infraestrutura da Groq, sendo vital para processar rapidamente as milhares de palavras de um webinar longo.
* **Temperatura** (`0.5`): Balanceada para garantir fidelidade ao texto original, mas mantendo a fluência necessária para criar Planos de Ação aplicáveis.

Explicação das Etapas do Prompt

1. **Cargo & Missão**: Orienta o modelo para uma postura crítica voltada a negócios.
2. **Público e Profundidade**: Focado em diretores e gestores com restrição de tempo.
3. **Estrutura Obrigatória**: Impõe a conversão do consumo de vídeo passivo em "Planos de Ação" ativos.
4. **Limite de Escopo**: Bloqueia categoricamente a alucinação de dados não mencionados na transmissão.

#### BLOCO 3 e 4: O PROCEDIMENTO DE ATENDIMENTO E A INTERFACE DO USUÁRIO (UI)

O **Bloco 3** concentra a lógica de estabilidade do sistema. Implementamos tratamento de erros (`try/except`) e um limite de volume de dados (`texto[:12000]`) para garantir que o sistema não trave se processar 
um vídeo muito longo. O Bloco 4 entrega o dashboard analítico para o usuário final.

```python
# ===============================
# BLOCO 3: PROCEDIMENTO DE ANÁLISE E TRATAMENTO
# ===============================
def analisar_video(url_do_video, objetivo_analise):
    if not url_do_video:
        return "Cole um link válido do YouTube."

    try:
        # A máquina extrai as legendas de forma estável
        texto = youtube_tool.get_youtube_video_captions(url=url_do_video)

        if not texto or len(texto) < 50:
            return "Não foi possível obter uma transcrição válida. O vídeo pode não ter legendas."

        # Limitar tamanho da transcrição (Proteção de Memória)
        texto = texto[:12000]

    except Exception as e:
        return f"Erro ao obter legendas: {str(e)}"

    # Injetando o texto extraído no prompt do modelo
    prompt = f"""
    Objetivo da análise:
    {objetivo_analise}

    Transcrição do vídeo:
    {texto}
    """
    resposta = agent.run(prompt)
    return resposta.content

# ===============================
# BLOCO 4: INTERFACE DO USUÁRIO (UI)
# ===============================
interface = gr.Interface(
    fn=analisar_video,
    inputs=[
        gr.Textbox(label="1. Cole o Link do Vídeo (YouTube)"),
        gr.Textbox(label="2. Qual o seu objetivo com este vídeo?", placeholder="Ex: Extrair argumentos de venda", lines=2)
    ],
    outputs=gr.Markdown(label="Inteligência Extraída do Vídeo"),
    title="Extrator de Insights de Vídeos do YouTube com IA",
    description="Cole um link de vídeo do YouTube e a IA irá extrair os principais insights, conceitos e planos de ação."
)

interface.launch(share=True)
```

### Valor Estratégico do Projeto

Resolve a assimetria de informação corporativa. Multiplica a capacidade analítica da equipe, reduzindo horas de monitoramento a segundos de processamento de texto e acelerando a formulação de estratégias de mercado.

### Possibilidades de Aplicação

* **Marketing**: Analisar vídeos de concorrentes no YouTube para identificar posicionamento de produto, argumentos de venda e tendências de mercado.
* **Educação e Treinamento**: Extrair os principais conceitos de aulas, palestras e tutoriais publicados no YouTube, transformando o conteúdo em resumos estruturados e roteiros de estudo.
* **Pesquisa de Tecnologia**: Acompanhar lançamentos e demonstrações técnicas apresentados em vídeos, permitindo identificar rapidamente novas ferramentas, frameworks e abordagens.
* **Inteligência de Conteúdo**: Transformar vídeos longos, como entrevistas ou podcasts, em relatórios executivos com os principais pontos discutidos.

## Código completo

```python
# Instalar as ferramentas
!pip install -q agno groq gradio youtube-transcript-api
```

```python
import os
import gradio as gr
from agno.agent import Agent
from agno.models.groq import Groq
from agno.tools.youtube import YouTubeTools
from google.colab import userdata

os.environ["GROQ_API_KEY"] = userdata.get("GROQ_API_KEY")

youtube_tool = YouTubeTools(
    enable_get_video_captions=True,
    languages=["pt","en"]
)

agent = Agent(
    model=Groq(id="llama-3.3-70b-versatile", temperature=0.5),

    # 1. Cargo / Persona
    role="Estrategista de Inteligência de Mercado",

    # 2. Missão Principal
    description="Analisar transcrições de vídeos do YouTube e extrair insights executivos baseados no conteúdo do vídeo.",

    # Regras de Execução
    instructions=[
        # 3. Público
        "Seu público são executivos que precisam extrair rapidamente o conhecimento central do vídeo.",
        
        # 4. Profundidade
        "Analise a transcrição recebida e gere um relatório estruturado capturando nuances importantes.",
        
        # 5. Estrutura Obrigatória
        "Analise a transcrição e gere um relatório estruturado no formato:\n**1. Visão Geral do Vídeo**\n**2. Tópicos Abordados**\n**3. Conceitos-Chave e Argumentos**\n**4.Plano de Ação**",
        
        # 6. Limite de Escopo (GUARDRAIL DE COMPLIANCE)
        "REGRA DE OURO: Baseie sua resposta APENAS na transcrição do vídeo fornecida na mensagem. Não invente dados.",
        
        # 7. Estilo
        "Seja analítico, objetivo e fiel ao conteúdo."
    ]
)

def analisar_video(url_do_video, objetivo_analise):
    if not url_do_video:
        return "Cole um link válido do YouTube."
    try:
        texto = youtube_tool.get_youtube_video_captions(url=url_do_video)
        if not texto or len(texto) < 50:
            return "Não foi possível obter uma transcrição válida deste vídeo."
        texto = texto[:12000]
    except Exception as e:
        return f"Erro ao obter legendas: {str(e)}"

    prompt = f"Objetivo da análise:\n{objetivo_analise}\n\nTranscrição do vídeo:\n{texto}"
    resposta = agent.run(prompt)
    return resposta.content

interface = gr.Interface(
    fn=analisar_video,
    inputs=[
        gr.Textbox(label="1. Cole o Link do Vídeo (YouTube)"),
        gr.Textbox(label="2. Qual o seu objetivo com este vídeo?", lines=2)
    ],
    outputs=gr.Markdown(label="Inteligência Extraída do Vídeo"),
    title="Extrator de Insights de Vídeos do YouTube com IA",
    description="Cole um link de vídeo do YouTube e a IA irá extrair os principais insights, conceitos e planos de ação."
)

interface.launch(share=True)


```

## Resultado do Produto Extrator de Insights de Vídeos do YouTube com IA

<img width="2488" height="1365" alt="image" src="https://github.com/user-attachments/assets/2764246e-e15d-490b-b14b-ed58aba29349" />


