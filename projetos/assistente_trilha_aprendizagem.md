# Assistente Automatizado de Onboarding e Compliance (PDF) usando RAG
 
<img width="1536" height="1024" alt="ChatGPT Image 4 de mar  de 2026, 22_39_31" src="https://github.com/user-attachments/assets/92f1f85f-d584-4f9f-b80d-075772082928" />



### Ador do Negócio
No dia a dia corporativo, profissionais de áreas como Direito, Recursos Humanos e Operações perdem dias de trabalho para ler, interpretar e resumir documentos extensos. Seja uma nova resolução tributária, um manual de subscrição de seguros ou a atualização do 
código de ética da empresa, o processo é sempre o mesmo: leitura exaustiva e baixa retenção de conhecimento. Quando um novo colaborador entra na empresa, o tempo de ramp-up (treinamento até ele ser produtivo) é longo e custoso, simplesmente porque o conhecimento 
está trancado em arquivos PDF de 150 páginas.

### Contextualização das Ferramentas

Para resolver esse problema de dados não estruturados, vamos utilizar um conjunto de ferramentas focado em leitura e recuperação de informação:

* **Agno**: Nosso framework principal para orquestrar o Agente e as ferramentas.
* **SentenceTransformer (Embedder)**: Um modelo matemático focado em ler o texto e transformar o significado das frases em números (vetores).
* **ChromaDB**: Um banco de dados vetorial, otimizado para armazenar e buscar essas coordenadas matemáticas em milissegundos.
* **Gradio**: A biblioteca que criará a interface visual (tela) para que o usuário final faça o upload do PDF sem precisar ver nenhuma linha de código.

### Arquitetura do Sistema 

A arquitetura que usaremos é o RAG (Retrieval-Augmented Generation). O sistema funcionará no seguinte fluxo:
* O usuário faz o upload do PDF e digita o tema que deseja estudar.
* O sistema lê o PDF, divide o texto em blocos menores (Chunking) e os converte em vetores (Embeddings), salvando no ChromaDB.
* O Agente recebe a pergunta do usuário, busca a resposta exclusivamente no banco de dados vetorial e estrutura uma trilha de aprendizagem.

## O Processo de Construção e Explicação do código por Blocos
#### BLOCO 1: FERRAMENTAS E AUTENTICAÇÃO

Esta etapa inicial é focada na infraestrutura e segurança da informação. Aqui, importamos as bibliotecas necessárias para o sistema funcionar e configuramos as chaves de acesso (API Keys). Isso estabelece uma conexão autenticada e privada com os provedores de 
Inteligência Artificial, garantindo que os manuais e documentos confidenciais da empresa sejam processados em um ambiente seguro, mitigando o risco de vazamento de dados.

```python
# ===============================
# BLOCO 1: FERRAMENTAS E AUTENTICAÇÃO
# ===============================
import os
import gradio as gr
from agno.agent import Agent
from agno.knowledge.embedder.sentence_transformer import SentenceTransformerEmbedder
from agno.knowledge.knowledge import Knowledge
from agno.knowledge.reader.pdf_reader import PDFReader
from agno.models.openai import OpenAIChat
from agno.vectordb.chroma import ChromaDb
from google.colab import userdata

os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY")
```

#### BLOCO 2: PROCESSAMENTO SEMÂNTICO

O embedder é o algoritmo responsável pelo processamento semântico. Textos jurídicos e de compliance são complexos; este modelo lê o documento corporativo e converte as frases em vetores matemáticos. Isso permite que o sistema realize buscas por similaridade de contexto e 
significado, garantindo precisão mesmo que o usuário não utilize as palavras exatas contidas no manual.

```python
# ===============================
# BLOCO 2: PROCESSAMENTO SEMÂNTICO
# ===============================
embedder = SentenceTransformerEmbedder(dimensions=384)
```

#### BLOCO 3: BANCO DE DADOS VETORIAL E CONTEXTO
O **ChromaDb** atua como nosso banco de dados vetorial local. Ele armazena as coordenadas matemáticas geradas no bloco anterior e as indexa para recuperação de alta velocidade. A função **Knowledge** integra esse repositório de dados ao agente de Inteligência Artificial, estruturando a base oficial de consulta.

```python
# ===============================
# BLOCO 3: BANCO DE DADOS VETORIAL E CONTEXTO
# ===============================
vector_db = ChromaDb(
    collection="trilhas_de_aprendizagem",
    path="./chromadb_trilhas",
    persistent_client=True,
    embedder=embedder,
)

knowledge = Knowledge(vector_db=vector_db)
```

#### BLOCO 4: O AGENTE ESPECIALISTA (COM RAG)

Neste bloco, instanciamos o Modelo de Linguagem (LLM) e definimos seus parâmetros de operação. A temperatura `0.5` calibra o modelo para priorizar respostas analíticas e coerentes. A diretriz central aqui é o "Limite de Escopo" (passo 6): ela atua como uma trava rígida de compliance, 
impedindo o agente de utilizar conhecimentos externos ao PDF. Isso elimina o risco de alucinações e garante que a IA não gere regras ou processos inexistentes na empresa.

```python
# ===============================
# BLOCO 4: O AGENTE ESPECIALISTA (COM RAG)
# ===============================
agent = Agent(
    model=OpenAIChat(id="gpt-4o-mini", temperature=0.5), 
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
        "Seu público são colaboradores em onboarding ou profissionais buscando capacitação rápida sobre o tema solicitado.",
        
        # 4. Profundidade
        "A progressão do estudo deve ser didática: comece pelos conceitos fundamentais e avance para as aplicações complexas contidas no material.",
        
        # 5. Estrutura Obrigatória
        "Sempre estruture o roteiro no seguinte formato:\n**1. Visão Geral do Tema:** (Breve resumo)\n**2. Módulos de Estudo:** (Divida a leitura do documento em 3 a 5 módulos lógicos. Cite as páginas do PDF, se possível)\n**3. Conceitos-Chave:** (Destaque os termos técnicos)\n**4. Questões para Fixação:** (Crie 3 perguntas práticas baseadas no texto para o usuário testar seu conhecimento)",
        
        # 6. Limite de Escopo (GUARDRAIL DE COMPLIANCE)
        "REGRA DE OURO: Baseie o roteiro APENAS no documento fornecido. Se o tema NÃO está no PDF, recuse educadamente. Jamais inclua informações externas.",
        
        # 7. Estilo
        "Mantenha um tom encorajador, estruturado e profissional."
    ]
)
```

Configuração do Agente: Os Porquês

* **Modelo** (`gpt-4o-mini`): Ideal para leitura e resumo de textos com excelente custo-benefício computacional.
* **Temperatura** (`0.5`): A temperatura equilibrada garante que o modelo mantenha a coerência pedagógica, sendo didático sem inventar regras inexistentes.

Explicação das Etapas do Prompt
1. **Cargo / Persona**: Define o escopo de atuação técnica da IA.
2. **Missão Principal**: Estabelece a meta primária de basear-se estritamente no documento de origem.
3. **Público**: Direciona o tom de voz para adequação corporativa.
4. **Profundidade**: Exige uma lógica de estruturação de informação (do básico ao avançado).
5. **Estrutura Obrigatória**: Padroniza a saída de dados, garantindo que o resultado seja aplicável em cenários reais.
6. **Limite de Escopo**: A trava de segurança da informação (guardrail).
7. **Estilo**: Mantém o padrão de comunicação empresarial.

#### BLOCO 5 e 6: O PROCEDIMENTO DE ATENDIMENTO E A INTERFACE DO USUÁRIO (UI)

O Bloco 5 define a função de processamento dinâmico: ele recebe o documento enviado pelo usuário, fatia o texto em blocos menores (chunk_size) para otimizar o uso da memória de curto prazo da IA, e executa a consulta. O Bloco 6 encapsula toda essa lógica de 
engenharia em uma Interface Gráfica (UI) intuitiva via Gradio, permitindo que colaboradores interajam com o sistema e consumam treinamentos sem precisarem de conhecimentos em programação.


```python
# ===============================
# BLOCO 5: O PROCEDIMENTO DE ATENDIMENTO
# ===============================
def gerar_roteiro_de_estudos(arquivo_pdf, tema_estudo):
    if arquivo_pdf is None or not tema_estudo:
        return "Faça o upload do arquivo e digite um tema."
        
    knowledge.insert(
        path=arquivo_pdf.name, 
        reader=PDFReader(chunk_size=1000, chunk_overlap=200)
    )
    
    comando = f"Crie um roteiro de estudos detalhado sobre o tema: '{tema_estudo}'."
    resposta = agent.run(comando)
    return resposta.content

# ===============================
# BLOCO 6: INTERFACE DO USUÁRIO (UI)
# ===============================
interface = gr.Interface(
    fn=gerar_roteiro_de_estudos,
    inputs=[
        gr.File(label="1. Faça o upload do Manual (PDF)", file_types=[".pdf"]),
        gr.Textbox(label="2. Qual tema deste material você quer estudar?", lines=2)
    ],
    outputs=gr.Markdown(label="Roteiro de Estudos Estruturado"),
    title="Assistente Automatizado de Onboarding",
    description="Transforme qualquer manual em uma trilha de aprendizagem."
)

interface.launch(share=True)

```

### Valor Estratégico do Projeto

Transforma o consumo passivo de documentos em uma ferramenta de treinamento ativo. Reduz drasticamente os custos operacionais de integração de novos colaboradores e assegura que o conhecimento regulatório seja disseminado com controle de qualidade.

### Possibilidades de Aplicação
* **Recursos Humanos**: Transformar o Código de Conduta ou Manuais de Benefícios em treinamentos modulares.
* **Jurídico**: Sistematizar a adequação a novos Marcos Regulatórios gerando checklists operacionais a partir da lei pura.

## Código completo
Você pode testar a aplicação usando o PDF de compliance da empresa Cobrape, disponível no [link](https://github.com/rgaguiar/turma-dc-ia01/blob/main/documentos/cobrape.pdf). 
Baixe o arquivo e, em seguida, faça o upload dele na aplicação para verificar como o agente analisa o conteúdo com o tema que você vai usar.

```python
# Instalar as ferramentas
!pip install -q agno
!pip install -q pypdf
!pip install -q chromadb
```
```python
# ===============================
# BLOCO 1: FERRAMENTAS E AUTENTICAÇÃO
# ===============================
import os
import gradio as gr
from agno.agent import Agent
from agno.knowledge.embedder.sentence_transformer import SentenceTransformerEmbedder
from agno.knowledge.knowledge import Knowledge
from agno.knowledge.reader.pdf_reader import PDFReader
from agno.models.openai import OpenAIChat
from agno.vectordb.chroma import ChromaDb
from google.colab import userdata

os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY")

# ===============================
# BLOCO 2: PROCESSAMENTO SEMÂNTICO
# ===============================
embedder = SentenceTransformerEmbedder(dimensions=384)

# ===============================
# BLOCO 3: BANCO DE DADOS VETORIAL E CONTEXTO
# ===============================
vector_db = ChromaDb(
    collection="trilhas_de_aprendizagem",
    path="./chromadb_trilhas",
    persistent_client=True,
    embedder=embedder,
)

knowledge = Knowledge(vector_db=vector_db)


# ===============================
# BLOCO 4: O AGENTE ESPECIALISTA (COM RAG)
# ===============================
agent = Agent(
    model=OpenAIChat(id="gpt-4o-mini", temperature=0.5), 
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
        "Seu público são colaboradores em onboarding ou profissionais buscando capacitação rápida sobre o tema solicitado.",
        
        # 4. Profundidade
        "A progressão do estudo deve ser didática: comece pelos conceitos fundamentais e avance para as aplicações complexas contidas no material.",
        
        # 5. Estrutura Obrigatória
        "Sempre estruture o roteiro no seguinte formato:\n**1. Visão Geral do Tema:** (Breve resumo)\n**2. Módulos de Estudo:** (Divida a leitura do documento em 3 a 5 módulos lógicos. Cite as páginas do PDF, se possível)\n**3. Conceitos-Chave:** (Destaque os termos técnicos)\n**4. Questões para Fixação:** (Crie 3 perguntas práticas baseadas no texto para o usuário testar seu conhecimento)",
        
        # 6. Limite de Escopo (GUARDRAIL DE COMPLIANCE)
        "REGRA DE OURO: Baseie o roteiro APENAS no documento fornecido. Se o tema NÃO está no PDF, recuse educadamente. Jamais inclua informações externas.",
        
        # 7. Estilo
        "Mantenha um tom encorajador, estruturado e profissional."
    ]
)

# ===============================
# BLOCO 5: O PROCEDIMENTO DE ATENDIMENTO
# ===============================
def gerar_roteiro_de_estudos(arquivo_pdf, tema_estudo):
    if arquivo_pdf is None or not tema_estudo:
        return "Faça o upload do arquivo e digite um tema."
        
    knowledge.insert(
        path=arquivo_pdf.name, 
        reader=PDFReader(chunk_size=1000, chunk_overlap=200)
    )
    
    comando = f"Crie um roteiro de estudos detalhado sobre o tema: '{tema_estudo}'."
    resposta = agent.run(comando)
    return resposta.content

# ===============================
# BLOCO 6: INTERFACE DO USUÁRIO (UI)
# ===============================
interface = gr.Interface(
    fn=gerar_roteiro_de_estudos,
    inputs=[
        gr.File(label="1. Faça o upload do Manual (PDF)", file_types=[".pdf"]),
        gr.Textbox(label="2. Qual tema deste material você quer estudar?", lines=2)
    ],
    outputs=gr.Markdown(label="Roteiro de Estudos Estruturado"),
    title="Assistente Automatizado de Onboarding",
    description="Transforme qualquer manual em uma trilha de aprendizagem."
)

interface.launch(share=True)

```
### Resultado do Produto: Assistente Automatizado de Onboarding e Compliance (PDF) usando RAG

<img width="2485" height="853" alt="image" src="https://github.com/user-attachments/assets/d39b06dd-5ec0-42a4-9a2a-a56d7ed4b0ae" />
