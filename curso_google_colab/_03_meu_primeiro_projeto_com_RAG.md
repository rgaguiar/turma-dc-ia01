# Aula Prática: RAG (Retrieval-Augmented Generation) – Baseando a IA em Documentos Corporativos

<img width="1338" height="903" alt="image" src="https://github.com/user-attachments/assets/672adac1-2ba7-43ac-8ded-9b9ec3319a54" />

Até agora, criamos agentes com alto poder de raciocínio, mas que possuem uma limitação: eles baseiam suas respostas apenas nos dados globais com os quais foram treinados originalmente. Se você perguntar sobre a política interna de RH da sua empresa ou sobre as cláusulas de um contrato confidencial, o modelo não saberá responder (ou, em um cenário pior, tentará gerar informações imprecisas, gerando "alucinações").

Para solucionar esse problema no ambiente corporativo, a arquitetura padrão utilizada no mercado é o RAG (Retrieval-Augmented Generation), ou Geração Aumentada por Recuperação.



### O que é o RAG na prática?
O RAG é uma técnica que obriga o modelo de Inteligência Artificial a consultar uma base de dados específica e restrita (fornecida por você) antes de formular uma resposta. Em vez de depender da sua memória global, a IA recupera a informação exata no documento corporativo e a utiliza como contexto oficial para gerar a resposta. Se a informação não estiver lá, o sistema é instruído a negar a resposta.

Vamos construir essa arquitetura em duas fases: primeiro estruturando o motor de busca, e em seguida criando a interface para o usuário final.

---

## FASE 1: A Arquitetura RAG (Lógica de Backend)

Nesta etapa, vamos configurar a infraestrutura de dados. Faremos a leitura de um documento PDF alocado no sistema e prepararemos a IA para interpretá-lo.

### Bloco 1: As Ferramentas de Acesso
Iniciamos importando as dependências do projeto e garantindo o acesso seguro via API.

```python
# instalando as ferramentas necessárias
!pip install -q agno
!pip install -q pypdf
!pip install -q chromadb
```

```python
# ===============================
# BLOCO 1: FERRAMENTAS E AUTENTICAÇÃO
# ===============================
import os
from agno.agent import Agent
from agno.knowledge.embedder.sentence_transformer import SentenceTransformerEmbedder
from agno.knowledge.knowledge import Knowledge
from agno.knowledge.reader.pdf_reader import PDFReader
from agno.models.openai import OpenAIChat
from agno.vectordb.chroma import ChromaDb
from google.colab import userdata

# Autenticação Segura
os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY")
os.environ["HF_TOKEN"] = userdata.get("HF_TK") # Usado para os modelos de leitura
```

> **⚠️ Importante (Google Colab)**
> 
> Antes de executar os `imports`, é necessário instalar as bibliotecas utilizadas no projeto.
> 
> No Google Colab, a ordem correta é:
> 
> 1. Instalar as dependências (`!pip install ...`)
> 2. Executar os `imports`
> 3. Rodar o restante do código
> 
> Se ocorrer algum erro, consulte a documentação oficial do framework para verificar possíveis atualizações ou mudanças recentes.

### Bloco 2: Processamento Semântico (Embedder)

Modelos de IA não processam texto da mesma forma que humanos; eles realizam cálculos matemáticos.

```python
# ===============================
# BLOCO 2: O TRADUTOR DE SIGNIFICADOS
# ===============================
embedder = SentenceTransformerEmbedder(dimensions=384)
```
O **Embedder** é o algoritmo responsável por processar o texto em formato numérico. Ele lê o documento e converte cada frase em um vetor matemático (uma coordenada de 384 dimensões). Isso é crucial para os negócios, pois permite que o sistema realize buscas semânticas. Ou seja, a IA encontra a resposta pelo significado e contexto da pergunta, e não apenas buscando palavras-chave exatas.  

> **Dimensão** é o número de características numéricas usadas para representar o significado de um texto.  
> 
> Exemplo simplificado:  
>
>Frase:  
>"Seguro de carro cobre colisão"  
>
>Vetor (dimensão 5 só para ilustrar): [0.21, -0.33, 0.87, 0.10, -0.44]

### Bloco 3: Armazenamento Vetorial (Vector Database)
Como nosso texto foi convertido em vetores matemáticos, bancos de dados tradicionais (como planilhas Excel ou bancos SQL) não são adequados para esse tipo de operação.  

Isso acontece porque esses bancos foram projetados para consultas exatas (por exemplo: igualdade, filtros ou relações entre tabelas). Já em aplicações com embeddings, o objetivo é realizar buscas por similaridade entre vetores, normalmente usando cálculos como distância coseno ou distância euclidiana.  

Quando a base possui milhares ou milhões de vetores, calcular essa similaridade diretamente em um banco tradicional se torna computacionalmente caro e lento, pois ele não possui estruturas de indexação otimizadas para esse tipo de busca.  

Por isso utilizamos bancos de dados vetoriais, que são projetados especificamente para armazenar vetores de alta dimensão e realizar buscas de similaridade de forma eficiente.  

```python
# ===============================
# BLOCO 3: BANCO DE DADOS VETORIAL E CONTEXTO
# ===============================
# Repositório Vetorial
vector_db = ChromaDb(
    collection="escola_do_saber",
    path="./chromadb_escola",
    persistent_client=True,
    embedder=embedder,
)

# Gerenciador de Conhecimento (Knowledge Base)
knowledge = Knowledge(vector_db=vector_db)

```
Utilizamos o **ChromaDb**, um banco de dados vetorial projetado especificamente para armazenar e realizar buscas de similaridade em altíssima velocidade. A função Knowledge atua como a ponte de integração, estruturando esses dados de forma que o Agente consiga consultá-los.   

No Google Colab, `persistent_client=True` funciona apenas durante a sessão ativa do ambiente.

### Bloco 4: Extração e Segmentação de Dados (Chunking)

```python
# ===============================
# BLOCO 4: LEITURA E SEGMENTAÇÃO DO PDF
# ===============================
knowledge.insert(
    path="/content/Escola do Saber.pdf", # Caminho do arquivo físico
    reader=PDFReader(
        chunk_size=1000, 
        chunk_overlap=200
  ),
)
```

**Conceito de Segmentação (Chunking)**   
Modelos de IA possuem um limite de memória de curto prazo (chamado de janela de contexto). Por isso, não é eficiente inserir um documento grande (como um PDF de 100 páginas) de uma única vez no processamento.  

Para resolver isso, utilizamos a técnica de chunking do tipo fixed-size, em que o texto é dividido em blocos com um tamanho definido previamente.

* **Chunk Size (1000)**: O documento é segmentado em blocos de 1000 caracteres. Essa divisão padronizada facilita o processamento e melhora a eficiência na etapa de busca semântica.
* **Chunk Overlap (200)**: Para evitar que informações relevantes fiquem separadas no limite entre dois blocos, utilizamos uma sobreposição de 200 caracteres entre chunks consecutivos. Assim, partes do contexto são repetidas entre os blocos, reduzindo o risco de perda de significado durante a recuperação da informação.

Essa estratégia garante que cada trecho do documento seja processado, indexado e recuperado com contexto suficiente durante as consultas feitas pela IA.  

Vale destacar que o **fixed-size chunking** é apenas uma das abordagens possíveis. Existem outras técnicas de segmentação, como:  

**Chunking por sentença ou parágrafo**, que respeita a estrutura natural do texto.  
**Chunking semântico**, que divide o conteúdo com base em mudanças de significado.  
**Chunking hierárquico**, que utiliza diferentes níveis de divisão (seções, subtópicos, parágrafos).  

Cada técnica possui vantagens e limitações, e a escolha depende do tipo de documento e do objetivo da aplicação.

### Bloco 5: Configuração do Agente com RAG

Por fim, integramos o Modelo de Linguagem à nossa Base de Conhecimento e aplicamos o framework de regras de negócio.

```python
# ===============================
# BLOCO 5: O AGENTE ESPECIALISTA (COM RAG)
# ===============================
agent = Agent(
    model=OpenAIChat(id="gpt-4o-mini", temperature=0.3),
    knowledge=knowledge,   # INTEGRAÇÃO RAG: Conectando a base de dados ao agente
    search_knowledge=True, # Habilitando a função de recuperação (Retrieval)
    markdown=True,
    
    # 1. Cargo / Persona
    role="Assistente Sênior de Secretaria da Escola do Saber",
    
    # 2. Missão Principal
    description="Fornecer informações precisas sobre matrículas e regras, baseando-se EXCLUSIVAMENTE nos documentos fornecidos.",
    
    # Regras de Execução
    instructions=[
        # 3. Público
        "Você responde a pais de alunos e potenciais clientes.",
        # 4. Profundidade
        "Forneça respostas claras e acessíveis.",
        # 5. Estrutura
        "Organize em tópicos para facilitar a leitura.",
        # 6. Limite de Escopo (GUARDRAIL ANTI-ALUCINAÇÃO)
        "Sempre pesquise na base de conhecimento antes de responder. Se a informação solicitada NÃO constar no PDF, informe que não possui a resposta. Jamais gere informações não documentadas.",
        # 7. Estilo
        "Seja educado e direto."
    ]
)

# ===============================
# BLOCO 6: TESTE DE HOMOLOGAÇÃO
# ===============================
print("--- Iniciando Consulta ao Agente ---")
agent.print_response("Qual o preço do ensino médio na sua escola?", stream=True)
```
>**💡 Dica de Negócios**: O item 6 (Limite de escopo) é a trava de segurança mais importante em implementações corporativas. Ele atua como um mecanismo de compliance, garantindo que o assistente não assuma compromissos financeiros ou crie regras inexistentes para os seus clientes.

## FASE 2: Interface Dinâmica de Usuário (Gradio)
