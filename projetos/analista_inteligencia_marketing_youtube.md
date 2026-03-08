# Analista de Inteligência Competitiva (Vídeos)

### Ador do Negócio
A informação de mercado e os anúncios de concorrentes ocorrem cada vez mais em formatos audiovisuais longos (webinars, keynotes, reuniões de resultados). O problema é que ninguém no time de Vendas ou Marketing possui o tempo operacional necessário 
para assistir a essas transmissões na íntegra. Consumir vídeo é lento, não indexável e gera um gargalo de inteligência competitiva nas empresas, atrasando a formulação de estratégias de resposta.

### Contextualização das Ferramentas
* **Agno e Groq**: O motor lógico da aplicação.
* **YouTubeTools**: Uma ferramenta nativa do Agno que possibilita ao Agente conectar-se a uma URL e extrair a transcrição completa do vídeo de forma automatizada.
* **Gradio**: Biblioteca para o desenvolvimento da interface com o usuário final.

### Arquitetura do Sistema 
A arquitetura utilizada é a de Injeção de Contexto via Ferramenta (Tool-Use):
1. O usuário fornece a URL do vídeo e uma instrução de negócios.
2. O Agente aciona autonomamente a ferramenta YouTubeTools.
3. A ferramenta processa a extração dos dados em texto e os devolve ao Agente.
4. O Agente aplica o framework analítico sobre o texto recuperado e gera a resposta.

## O Processo de Construção e Explicação do código por Blocos
#### BLOCO 1: FERRAMENTAS E AUTENTICAÇÃO

Iniciamos o projeto configurando o acesso às APIs e importando a ferramenta YouTubeTools. Esta biblioteca específica fornece ao nosso sistema a capacidade técnica de contornar a limitação de dados estáticos, permitindo a extração autônoma de transcrições de URLs externas na internet.

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
```

#### BLOCO 2: O AGENTE ESPECIALISTA (COM TOOLS)

Nos projetos anteriores, utilizávamos uma **knowledge** base como ferramenta, que permitia ao agente acessar uma base de conhecimento externa enviada pelo usuário.

Neste exemplo, o diferencial técnico está na integração do parâmetro `tools=[YouTubeTools()]`.
Com isso, o comportamento do modelo deixa de ser apenas passivo. O agente passa a ter a capacidade de realizar chamadas de função de forma autônoma para capturar as informações diretamente do vídeo.

Além disso, a forma como estruturamos o prompt (no passo 5) força a IA a converter um conteúdo originalmente não estruturado em um relatório executivo.
Na prática, isso transforma o simples consumo de um vídeo em insights e planos de ação comerciais imediatos.

```python
# ===============================
# BLOCO 2: O AGENTE ESPECIALISTA (COM TOOLS)
# ===============================
agent = Agent(
    model=Groq(id="llama-3.3-70b-versatile", temperature=0.5), 
    tools=[YouTubeTools()], 
    show_tool_calls=True,
    
    # 1. Cargo / Persona
    role="Estrategista de Inteligência de Mercado",
    
    # 2. Missão Principal
    description="Analisar transcrições de vídeos do YouTube e extrair insights executivos baseados APENAS no que foi falado.",
    
    # Regras de Execução (O Manual de Conduta)
    instructions=[
        # 3. Público
        "Seu público são executivos e líderes que precisam extrair o conhecimento central do vídeo rapidamente.",
        
        # 4. Profundidade
        "Mantenha a fidelidade absoluta ao que o apresentador falou, capturando as nuances mais importantes e estratégicas.",
        
        # 5. Estrutura Obrigatória
        "Sempre estruture o seu relatório no seguinte formato:\n**1. Visão Geral do Vídeo:** (Qual é o tema central?)\n**2. Tópicos Abordados:** (Divida o conteúdo em tópicos lógicos)\n**3. Conceitos-Chave e Argumentos:** (Destaque ideias principais ou argumentos de venda)\n**4. Plano de Ação:** (Crie passos de ação baseados no vídeo para o usuário aplicar no negócio)",
        
        # 6. Limite de Escopo (GUARDRAIL DE COMPLIANCE)
        "REGRA DE OURO: Você deve SEMPRE usar a ferramenta do YouTube para ler a transcrição antes de responder. Jamais invente informações fora do vídeo ou use seu conhecimento prévio.",
        
        # 7. Estilo
        "Seja analítico e direto ao ponto."
    ]
)
```
Configuração do Agente: Os Porquês

* `show_tool_calls=True`: Essencial para a transparência do processo. Permite auditar o exato momento em que o Agente aciona a ferramenta para ler o vídeo, garantindo que a resposta seja fundamentada na fonte.
* **Modelo** (`llama-3.3-70b-versatile`): Escolhido pela sua alta capacidade de compreensão e geração de texto, aliado à baixa latência proporcionada pela infraestrutura da Groq. Esse modelo é especialmente eficiente para processar grandes volumes de texto,
* como transcrições e legendas de vídeos, mantendo boa qualidade de resposta e desempenho.

Explicação das Etapas do Prompt

1. **Cargo & Missão**: Orienta o modelo para uma postura de análise crítica e extração de valor corporativo.
2. **Público e Profundidade**: Direcionado a tomadores de decisão, focando em nuances estratégicas.
3. **Estrutura Obrigatória**: Impõe a criação de um "Plano de Ação", transformando a análise em diretrizes aplicáveis.
4. **Limite de Escopo**: Bloqueia o uso de dados de treinamento prévios do modelo.

#### BLOCO 3 e 4: O PROCEDIMENTO DE ATENDIMENTO E A INTERFACE DO USUÁRIO (UI)


```python
# ===============================
# BLOCO 3: O PROCEDIMENTO DE ATENDIMENTO
# ===============================
def analisar_video(url_do_video, objetivo_analise):
    if not url_do_video:
        return "Cole o link de um vídeo válido."
    
    comando = f"Acesse as legendas deste vídeo ({url_do_video}). Focando neste objetivo: '{objetivo_analise}', gere o relatório estruturado."
    resposta = agent.run(comando)
    return resposta.content

# ===============================
# BLOCO 4: INTERFACE DO USUÁRIO (UI)
# ===============================
interface = gr.Interface(
    fn=analisar_video,
    inputs=[
        gr.Textbox(label="1. Cole o Link do Vídeo (YouTube)"),
        gr.Textbox(label="2. Qual o seu objetivo com este vídeo?", placeholder="Ex: Extrair argumentos de venda...", lines=2)
    ],
    outputs=gr.Markdown(label="Inteligência Extraída do Vídeo"),
    title="Analista de Inteligência Competitiva"
)

interface.launch(share=True)
```

O **Bloco 3** estabelece o procedimento de execução: ele formata o comando de entrada combinando o link do vídeo e o objetivo de negócios da área de marketing para guiar a análise da IA. O **Bloco 4** implementa a interface de usuário (front-end), 
entregando uma ferramenta de monitoramento de mercado pronta para uso pelas equipes estratégicas.

### Valor Estratégico do Projeto

Resolve a assimetria de informação. Multiplica a capacidade analítica da equipe, reduzindo horas de consumo de vídeo a segundos de processamento, o que acelera o tempo de formulação de estratégias de resposta ao mercado.

### Possibilidades de Aplicação
* **Marketing**: Monitoramento de eventos e lançamentos da concorrência para identificação de vulnerabilidades.
* **Operações**: Extração de atas e definição de responsáveis a partir de gravações de reuniões gerenciais.


## Código completo

```python
# Instalar as ferramentas
!pip install -q agno
!pip install -q groq
```

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

# ===============================
# BLOCO 2: O AGENTE ESPECIALISTA (COM TOOLS)
# ===============================
agent = Agent(
    model=Groq(id="llama-3.3-70b-versatile", temperature=0.5), 
    tools=[YouTubeTools()], 
    show_tool_calls=True,
    markdown=True,
    
    # 1. Cargo / Persona
    role="Estrategista de Inteligência de Mercado",
    
    # 2. Missão Principal
    description="Analisar transcrições de vídeos do YouTube e extrair insights executivos baseados APENAS no que foi falado.",
    
    # Regras de Execução (O Manual de Conduta)
    instructions=[
        # 3. Público
        "Seu público são executivos e líderes que precisam extrair o conhecimento central do vídeo rapidamente.",
        
        # 4. Profundidade
        "Mantenha a fidelidade absoluta ao que o apresentador falou, capturando as nuances mais importantes e estratégicas.",
        
        # 5. Estrutura Obrigatória
        "Sempre estruture o seu relatório no seguinte formato:\n**1. Visão Geral do Vídeo:** (Qual é o tema central?)\n**2. Tópicos Abordados:** (Divida o conteúdo em tópicos lógicos)\n**3. Conceitos-Chave e Argumentos:** (Destaque ideias principais ou argumentos de venda)\n**4. Plano de Ação:** (Crie passos de ação baseados no vídeo para o usuário aplicar no negócio)",
        
        # 6. Limite de Escopo (GUARDRAIL DE COMPLIANCE)
        "REGRA DE OURO: Você deve SEMPRE usar a ferramenta do YouTube para ler a transcrição antes de responder. Jamais invente informações fora do vídeo ou use seu conhecimento prévio.",
        
        # 7. Estilo
        "Seja analítico e direto ao ponto."
    ]
)

# ===============================
# BLOCO 3: O PROCEDIMENTO DE ATENDIMENTO
# ===============================
def analisar_video(url_do_video, objetivo_analise):
    if not url_do_video:
        return "Cole o link de um vídeo válido."
    
    comando = f"Acesse as legendas deste vídeo ({url_do_video}). Focando neste objetivo: '{objetivo_analise}', gere o relatório estruturado."
    resposta = agent.run(comando)
    return resposta.content

# ===============================
# BLOCO 4: INTERFACE DO USUÁRIO (UI)
# ===============================
interface = gr.Interface(
    fn=analisar_video,
    inputs=[
        gr.Textbox(label="1. Cole o Link do Vídeo (YouTube)"),
        gr.Textbox(label="2. Qual o seu objetivo com este vídeo?", placeholder="Ex: Extrair argumentos de venda...", lines=2)
    ],
    outputs=gr.Markdown(label="Inteligência Extraída do Vídeo"),
    title="Analista de Inteligência Competitiva"
)

interface.launch(share=True)


```

