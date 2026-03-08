# Assistente Automatizado de Onboarding e Compliance (PDF)



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

### O Processo de Construção e Explicação do código por Blocos
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


