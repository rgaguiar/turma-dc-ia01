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
