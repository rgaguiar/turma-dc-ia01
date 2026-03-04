# Aula Prática: Ensinando a IA a Ler os Seus Documentos (O Poder do RAG)

<img width="1338" height="903" alt="image" src="https://github.com/user-attachments/assets/672adac1-2ba7-43ac-8ded-9b9ec3319a54" />

Até agora, criamos agentes incríveis, mas eles tinham um ponto cego: eles só sabiam o que aprenderam durante o treinamento deles na internet. Se você perguntasse sobre a tabela de preços atualizada da sua empresa, as regras de RH exclusivas do seu escritório ou um contrato confidencial, a IA não saberia responder (ou pior, tentaria inventar).

Para resolver isso, a indústria criou o **RAG (Retrieval-Augmented Generation)**, que em português significa **Geração Aumentada por Recuperação**.



### O que é o RAG na prática?
Imagine que a IA é um estagiário brilhante, mas que acabou de chegar na empresa e não conhece os processos internos. Em vez de exigir que ele saiba tudo de cabeça, você entrega a ele o **Manual da Empresa** e diz: *"Toda vez que alguém te fizer uma pergunta, consulte este manual primeiro. Se a resposta estiver aí, responda. Se não estiver, diga que não sabe."*

É exatamente isso que faremos aqui. Vamos construir uma aplicação onde o usuário faz o upload de um PDF (um contrato, um edital, uma tabela de preços), e a IA usará esse documento como sua "fonte da verdade".

---

## Desconstruindo a Arquitetura RAG

Para que a IA consiga ler um PDF de 100 páginas em segundos, precisamos de uma engrenagem um pouco mais sofisticada. Vamos analisar o código em blocos:

### Bloco 1: As Ferramentas e o Crachá
Como sempre, começamos reunindo nossa equipe e garantindo a segurança.

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

# 1. Configuração das chaves de acesso
os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY")
os.environ["HF_TOKEN"] = userdata.get("HF_TK") # Usado para os modelos de leitura de texto
```
