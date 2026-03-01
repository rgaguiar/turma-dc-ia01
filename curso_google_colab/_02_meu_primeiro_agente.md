# Aula Prática: Arquitetura de um Agente Especialista – Do Código à Interface Visual

Nesta aula, vamos elevar o nível da nossa aplicação. Sairamos da criação de um assistente genérico de terminal para construir um **Consultor Sênior de Estratégia para Escritórios de Advocacia** com uma interface visual completa.

Antes de colocarmos a mão na massa com o código, precisamos dar um passo atrás e olhar para a "foto de satélite" do nosso projeto. Para profissionais de negócios, entender a arquitetura de uma solução de IA é tão importante quanto saber programar.

## 1. Visão Geral da Arquitetura: A Foto do Satélite

Construir um agente de IA é muito parecido com estruturar um novo departamento na sua empresa. Precisamos de insumos estruturados, processos bem definidos e um balcão de atendimento amigável.

Podemos dividir a nossa arquitetura em três camadas principais:

<img width="2816" height="1536" alt="Gemini_Generated_Image_ua0b7pua0b7pua0b" src="https://github.com/user-attachments/assets/e85d3200-dfbe-4b14-b359-d4d13efbac75" />

### A. A Camada de Insumos (O que você fornece)
Todo projeto de IA começa com três elementos fundamentais que você, como gestor da ferramenta, deve estruturar:
* **O Código-Fonte:** É a "receita" ou o roteiro de processos da nossa aplicação.
* **Os Prompts (Instruções):** É o *Job Description* (Descrição de Cargo) do agente. É aqui que definimos a persona e as regras rígidas de negócios que a IA deve seguir.
* **A Chave de API (API Key):** É o crachá de segurança. Ela autoriza nossa aplicação local a se comunicar com os servidores de inteligência mais poderosos do mundo.

### B. A Camada de Processamento (O Gestor e o Cérebro)
Aqui é onde a mágica acontece, transformando a sua regra de negócio em inteligência aplicável:
* **O Orquestrador (Biblioteca Agno):** O Agno atua como o **gerente do projeto**. Ele não "pensa" sozinho, mas gerencia o fluxo: pega a pergunta do usuário, aplica as suas regras de negócio e encaminha para quem sabe resolver.
* **O Motor de Raciocínio (Groq/Llama):** Este é o verdadeiro "cérebro" ou o "consultor externo" contratado. Ele recebe a demanda processada pelo Agno, executa o raciocínio complexo em altíssima velocidade e devolve a resposta estratégica.

### C. A Camada de Entrega (O Balcão de Atendimento)
* **A Interface (Biblioteca Gradio):** O código mais brilhante do mundo não gera valor se o cliente final (um sócio, um cliente ou analista de marketing) não souber usar. O Gradio pega toda a complexidade técnica e a transforma em uma interface visual web, com caixas de texto e botões amigáveis.

---

## 2. Desconstruindo o Código: Passo a Passo

Agora que entendemos a teoria, vamos analisar como essa arquitetura ganha vida no código. Dividimos a construção da nossa ferramenta em **6 blocos operacionais**.

### Bloco 1: Convocando a Equipe (Ferramentas)
Antes de iniciar o projeto, precisamos reunir a equipe. Na programação, fazemos isso através das importações (`import`).

```python
# ===============================
# FERRAMENTAS
# ===============================
import gradio as gr
from agno.agent import Agent
from agno.models.groq import Groq
from google.colab import userdata
import os
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

### Bloco 2: O Crachá de Acesso (Autenticação)  

Fomos até o cofre de "Segredos" do Colab, resgatamos a senha de forma criptografada e a entregamos ao sistema. Agora temos autorização para usar a infraestrutura da Groq.

```python
# ===============================
# AUTENTICAÇÃO API_KEY_GROQ
# ===============================
chave_groq = userdata.get('API_KEY_GROQ')
os.environ["GROQ_API_KEY"] = chave_groq
```

> 📌 **Atenção:**  
> Consulte o item **“Cadastrando a Chave no Cofre de Senhas do Google Colab”**, no tópico  
> **“Criando seu primeiro agente com Agno”**, disponível no material de [**Introdução ao Google Colab**](https://github.com/rgaguiar/turma-dc-ia01/blob/main/curso_google_colab/_01_intro_colab.md#5-criando-seu-primeiro-agente-com-agno).
