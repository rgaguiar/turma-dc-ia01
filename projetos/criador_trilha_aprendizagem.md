# Gerador de Trilhas de Aprendizagem (Agente Estratégico + RAG) - Usando Agno

<img width="1536" height="1024" alt="ChatGPT Image 4 de mar  de 2026, 22_39_31" src="https://github.com/user-attachments/assets/92f1f85f-d584-4f9f-b80d-075772082928" />
  
* Neste projeto, vamos unir o conhecimento de agente e RAG. Vamos construir um Designer Instrucional de IA.

## Contextualização do Projeto
Gerador de Trilhas de Aprendizagem com Agente de IA + RAG

Este projeto implementa uma aplicação de IA aplicada à educação corporativa. O sistema recebe um documento PDF (manual, apostila ou política interna) e gera automaticamente uma trilha estruturada de estudos baseada nesse material.

O objetivo é automatizar a criação de roteiros de aprendizagem, permitindo que empresas ou professores transformem rapidamente documentos em planos de estudo organizados.

A solução combina três componentes principais:

* **Large Language Model (LLM)** – responsável por gerar o roteiro de estudos
* **RAG (Retrieval Augmented Generation)** – mecanismo que conecta o modelo ao conteúdo do documento
* **Interface Web** – camada que permite ao usuário interagir com o sistema

### Arquitetura do Sistema

O sistema será dividido em quatro camadas.

### 1. Interface do usuário

Responsável pela interação com o usuário.

O usuário:
* envia um PDF
* informa qual tema deseja estudar dentro do material

A interface que usaremos para interagir com o usuário será construída com **Gradio**.

### 2. Processamento do documento

O documento enviado pelo usuário precisa ser transformado em um formato que a IA consiga pesquisar.

Para isso ocorre o seguinte fluxo:

1. O PDF é lido
2. O texto é dividido em partes menores (chunks)
3. Cada parte é transformada em um vetor numérico
4. Esses vetores são armazenados em um banco vetorial

Esse processo permite que a IA encontre rapidamente os trechos relevantes do documento.

### 3. Recuperação de informação (RAG)

RAG significa Retrieval Augmented Generation.

O conceito é simples:

1. o usuário faz uma pergunta ou pedido
2. o sistema busca no banco vetorial os trechos mais relevantes
3. esses trechos são enviados ao modelo de linguagem
4. o modelo gera a resposta usando esse contexto

Isso garante que o modelo responda com base no documento enviado, e não apenas em conhecimento geral.

### 4. Agente de IA

O agente define como o modelo deve se comportar.

Ele contém:

* papel do agente
* objetivo da tarefa
* regras de execução
* formato obrigatório da resposta

Isso é uma prática chamada engenharia de prompt estruturada.

No projeto foi utilizado um framework de 7 passos, que define:

1. cargo do agente
2. missão
3. público
4. profundidade da resposta
5. estrutura obrigatória
6. limite de escopo
7. estilo de comunicação

## Explicação do Código

```python
import os
import gradio as gr
from agno.agent import Agent
from agno.knowledge.embedder.sentence_transformer import SentenceTransformerEmbedder
from agno.knowledge.knowledge import Knowledge
from agno.knowledge.reader.pdf_reader import PDFReader
from agno.models.openai import OpenAIChat
from agno.vectordb.chroma import ChromaDb
from google.colab import userdata

# ===============================
# 1. INFRAESTRUTURA E AUTENTICAÇÃO
# ===============================
os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY")
os.environ["HF_TOKEN"] = userdata.get("HF_TK")

embedder = SentenceTransformerEmbedder(dimensions=384)

vector_db = ChromaDb(
    collection="trilhas_de_aprendizagem",
    path="./chromadb_trilhas",
    persistent_client=True,
    embedder=embedder,
)

knowledge = Knowledge(vector_db=vector_db)

# ===============================
# 2. ENGENHARIA DO AGENTE (DESIGNER INSTRUCIONAL)
# ===============================
agent = Agent(
    model=OpenAIChat(id="gpt-4o-mini", temperature=0.5), # Temperatura menor para roteiros mais precisos e menos "criativos"
    knowledge=knowledge,   
    search_knowledge=True, 
    markdown=True,
    
    # 1. Cargo / Persona
    role="Designer Instrucional Sênior e Especialista em Educação Corporativa",
    
    # 2. Missão Principal
    description="Criar trilhas de aprendizagem e roteiros de estudo estruturados, baseando-se estritamente na base de conhecimento (PDF) fornecida.",
    
    # Regras de Execução (O Manual de Conduta)
    instructions=[
        # 3. Público
        "Seu público são colaboradores em onboarding, estudantes ou profissionais buscando capacitação rápida e eficiente sobre o tema solicitado.",
        
        # 4. Profundidade
        "A progressão do estudo deve ser didática: comece pelos conceitos fundamentais e avance gradativamente para as aplicações complexas contidas no material.",
        
        # 5. Estrutura Obrigatória
        "Sempre estruture o roteiro de estudos no seguinte formato:",
        "**1. Visão Geral do Tema:** (Breve resumo do que será aprendido)",
        "**2. Módulos de Estudo:** (Divida a leitura do documento em 3 a 5 módulos lógicos. Cite as páginas ou seções do PDF, se possível)",
        "**3. Conceitos-Chave:** (Destaque os termos técnicos mais importantes do material)",
        "**4. Questões para Fixação:** (Crie 3 perguntas práticas baseadas no texto para o usuário testar seu conhecimento)",
        
        # 6. Limite de Escopo (GUARDRAIL DE COMPLIANCE)
        "REGRA DE OURO: Baseie o roteiro APENAS no documento fornecido. Se o usuário pedir um roteiro sobre um tema que NÃO está no PDF, recuse educadamente e informe que o material não aborda esse assunto. Jamais inclua informações externas ao documento.",
        
        # 7. Estilo
        "Mantenha um tom encorajador, acadêmico, estruturado e altamente profissional."
    ]
)

# ===============================
# 3. INTEGRAÇÃO E INTERFACE (GRADIO)
# ===============================
def gerar_roteiro_de_estudos(arquivo_pdf, tema_estudo):
    if arquivo_pdf is None:
        return "Por favor, faça o upload de uma apostila ou manual (PDF) para iniciar."
    
    if not tema_estudo:
        return "Por favor, digite o tema que você deseja estudar."
        
    # Processamento dinâmico do documento inserido via UI
    knowledge.insert(
        path=arquivo_pdf.name, 
        reader=PDFReader(chunk_size=1000, chunk_overlap=200)
    )
    
    # Cria o comando unindo o pedido do usuário com a missão do agente
    comando = f"Crie um roteiro de estudos detalhado sobre o tema: '{tema_estudo}'."
    
    resposta = agent.run(comando)
    return resposta.content

# Interface do Usuário (UI)
interface = gr.Interface(
    fn=gerar_roteiro_de_estudos,
    inputs=[
        gr.File(label="1. Faça o upload da Apostila / Manual (PDF)", file_types=[".pdf"]),
        gr.Textbox(label="2. Qual tema específico deste material você quer estudar?", placeholder="Ex: Regras de Compliance, Capítulo 3, Metodologia de Vendas...", lines=2)
    ],
    outputs=gr.Markdown(label="Roteiro de Estudos Estruturado"),
    title="Plataforma de Trilhas de Aprendizagem (IA)",
    description="Automatize seu treinamento. Faça o upload de um documento corporativo e a IA extrairá um roteiro de estudos pedagógico com conceitos-chave e questões de fixação."
)

interface.launch(share=True)
```
