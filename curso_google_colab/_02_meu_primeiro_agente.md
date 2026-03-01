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

### Bloco 3: O Cérebro da Operação (Regras de Negócio)
A qualidade de uma IA para o mercado de trabalho depende diretamente de como você a instrui. Em vez de escrevermos um texto solto, vamos dividir as instruções da nossa IA como se estivéssemos preenchendo a **Descrição de Cargo** (Job Description) de um novo funcionário. 

Veja como os comentários (as linhas com `#`) organizam o nosso pensamento:

```python
# ===============================
# AGENTE ESPECIALISTA
# ===============================
agente_consultor_juridico = Agent(
    model=Groq(id="llama-3.3-70b-versatile"),
    
    # Cargo da IA
    role="Consultor Sênior de Estratégia para Escritórios de Advocacia",
    
    instructions=[
        # 1. Quem é o agente (Persona)
        "Você é um consultor sênior especializado em estratégia e gestão de escritórios de advocacia.",
        
        # 2. Qual problema ele resolve (Objetivo)
        "Seu objetivo é ajudar escritórios jurídicos a melhorar posicionamento, rentabilidade, organização interna e crescimento sustentável.",
        
        # 3. Para qual público ele responde (Tom de voz e adequação)
        "Você responde principalmente a sócios, gestores jurídicos e advogados responsáveis pela administração do escritório.",
        
        # 4. Nível de profundidade (Complexidade)
        "Forneça respostas em nível estratégico e executivo, evitando explicações excessivamente básicas.",
        
        # 5. Estrutura obrigatória (Formatação de saída)
        "Organize sempre a resposta nos seguintes tópicos:",
        "- Contexto",
        "- Diagnóstico",
        "- Análise de riscos",
        "- Estratégias recomendadas",
        "- Próximos passos",
        
        # 6. Limite de escopo (Barreira de Segurança / Guardrail)
        "Se a pergunta estiver fora do escopo jurídico-empresarial, informe educadamente que o tema não faz parte da sua especialização.",
        
        # 7. Estilo de comunicação
        "Seja objetivo, estratégico e orientado à tomada de decisão."
    ]
)
```
**Entendendo os Parâmetros de Configuração:**

* **model= (O Motor Lógico)**: Define qual cérebro vamos usar. No nosso caso, apontamos para os servidores da Groq e escolhemos o modelo llama-3.3-70b-versatile. É como escolher o nível de senioridade e a capacidade de processamento do seu consultor (este modelo específico é excelente para raciocínios complexos).

* **role= (O Cargo Oficial)**: É o "crachá" do agente. É a primeira diretriz de contexto que a IA lê para assumir a sua persona antes mesmo de ler as regras.

* **instructions= (As Regras de Negócio)**: Note que este parâmetro usa colchetes [...] para criar uma lista. É a verdadeira Engenharia de Prompt no Código. O uso da cerquilha (#) no Python permite que você deixe anotações para você mesmo (o computador ignora essas linhas).

>**💡 Dica de Negócios**: O item 6 (Limite de escopo) é o que chamamos tecnicamente de Guardrail (barreira de segurança). É ele que impede que um colaborador use essa ferramenta corporativa para pedir receitas de bolo ou planejar o roteiro de férias, mantendo o custo e o uso do sistema estritamente profissionais.

### Bloco 4: O Procedimento Operacional Padrão (POP)

Temos o especialista, mas precisamos de um processo para que a demanda chegue até ele.

```python
# ===============================
# PROCEDIMENTO OP. PADRÃO (POP)
# ===============================
def consultar_assistente(pergunta_do_usuario):
    resposta = agente_consultor_juridico.run(pergunta_do_usuario)
    return resposta.content
```

Criamos a função consultar_assistente. Ela atua como uma triagem interna: pega o documento (`a pergunta`), entrega na mesa do especialista (`agente_consultor_juridico`), aguarda a elaboração do plano e devolve apenas o texto útil (`resposta.content`).

### Bloco 5: A Recepção (Interface Visual)

A ferramenta precisa ter "cara de software".

```python
# ===============================
# INTERFACE GRADIO
# ===============================
interface = gr.Interface(
    fn=consultar_assistente,
    inputs=gr.Textbox(lines=25, placeholder="Descreva o cenário ou desafio da sua empresa..."),
    outputs=gr.Markdown(),
    title="Portal do Consultor Estratégico",
    description="Insira sua dúvida de negócios abaixo para receber orientações estruturadas da IA."
)
interface.launch(share=True)
```

Conectamos nosso procedimento (`fn=consultar_assistente`) a uma tela web. Como consultas jurídicas e de negócios costumam ser longas, alteramos o lines=25 para dar uma caixa de texto grande e confortável ao usuário. O share=True gera o link público para você enviar para sua equipe testar no navegador deles.

### Bloco 6: Homologação (Testando na Prática)

Todo software precisa ser validado. Use as perguntas abaixo para testar a sua interface:

```python
# ===============================
# PERGUNTAS PARA TESTE
# ===============================
# 1. Como um escritório de advocacia pode aumentar sua captação de clientes empresariais?
# 2. Tenho um escritório com 12 advogados e queda de 25% no faturamento nos últimos 8 meses. O que devo fazer?
# 3. Escreva um poema sobre justiça.
```

**Testando a Arquitetura:**

* **Pergunta 1**: Testa a capacidade de planejamento.  
* **Pergunta 2**: É um "teste de estresse" executivo. Avalia se a IA vai aplicar corretamente os tópicos obrigatórios (Diagnóstico, Riscos, Estratégia) em um cenário de crise real. 
* **Pergunta 3**: Testa a nossa regra de segurança (Guardrail). O sistema deve negar o pedido educadamente.

  

# O Resultado Final: IA Aplicada à Realidade do Negócio

O que acabamos de construir não é apenas um script de código, mas um **produto digital funcional e de alto valor estratégico**. Ao encapsularmos o raciocínio avançado de um modelo de linguagem em uma interface acessível, criamos uma ferramenta pronta para ser adotada por qualquer departamento da empresa.

<img width="1819" height="958" alt="Interface do Portal do Consultor Estratégico" src="https://github.com/user-attachments/assets/52998123-967f-4b24-b20f-3051d80ef9f6" />

**Por que esse projeto tem um alto valor estratégico?**

* **Autonomia Operacional:** Gestores, sócios e analistas que não sabem programar agora têm acesso direto a um "consultor sênior" disponível 24/7, diretamente pelo navegador.
* **Padronização de Entregas:** Graças à nossa Engenharia de Prompt no código, a IA não dá respostas genéricas. Toda análise gerada segue um padrão corporativo rigoroso (Contexto, Diagnóstico, Riscos e Próximos Passos).
* **Segurança e Foco:** As regras de limite de escopo (*guardrails*) garantem que a ferramenta seja utilizada exclusivamente para resolver problemas de negócios, evitando o uso indevido da infraestrutura da empresa.
* **Escalabilidade:** O código base que criamos pode ser facilmente replicado. Mudando apenas o `role` e as `instructions`, você pode criar um "Auditor Fiscal", um "Especialista em SEO" ou um "Revisor de Contratos" usando a mesma arquitetura.

Este é o verdadeiro poder da Inteligência Artificial quando aliada a uma boa visão de negócios: transformar conhecimento técnico em **vantagem competitiva imediata**.  


### Expandindo a Arquitetura: Casos de Uso para Outras Áreas

A grande beleza da arquitetura que acabamos de construir é a sua **escalabilidade**. O código base (a infraestrutura) permanece exatamente o mesmo. Para criar uma nova ferramenta para outro departamento, você só precisa alterar o "Manual de Conduta" (os parâmetros `role` e `instructions` do Agente).

Abaixo, apresentamos como você pode adaptar esse mesmo código para resolver problemas reais em diferentes áreas do negócio:

#### 1. Para o Time de Marketing e Vendas
**Objetivo:** Criar um assistente focado em conversão, aquisição de clientes e posicionamento de marca.
* **`role`:** `"Head de Growth e Marketing Digital"`
* **`instructions` (Exemplos de regras):**
  * "Seu objetivo é estruturar campanhas de marketing, otimizar o Custo de Aquisição de Clientes (CAC) e aumentar o Life Time Value (LTV)."
  * "Ao receber um produto ou serviço, crie uma estratégia de lançamento em 3 fases: Atração, Engajamento e Conversão."
  * "Sempre inclua sugestões de gatilhos mentais e canais de distribuição (Tráfego Pago, SEO, Redes Sociais)."

#### 2. Para o Time de Recursos Humanos (RH)
**Objetivo:** Desenvolver um consultor para mediação de conflitos, cultura organizacional e avaliações de desempenho.
* **`role`:** `"Consultor Sênior de Desenvolvimento Organizacional e RH"`
* **`instructions` (Exemplos de regras):**
  * "Você ajuda líderes e gestores a lidar com desafios de gestão de pessoas, retenção de talentos e engajamento."
  * "Ao receber um relato de conflito ou baixo desempenho, forneça um roteiro prático para uma reunião de feedback construtivo (1:1)."
  * "Mantenha um tom empático, neutro e estritamente alinhado às boas práticas de compliance trabalhista e ética corporativa."

#### 3. Para o Time de Seguros e Logística
**Objetivo:** Estruturar um analista focado em mitigação de riscos e análise de apólices complexas.
* **`role`:** `"Especialista em Subscrição e Seguros de Transporte (Nacional e Internacional)"`
* **`instructions` (Exemplos de regras):**
  * "Sua função é avaliar cenários logísticos e identificar gargalos de segurança e riscos de avaria ou sinistro em cargas."
  * "Sempre estruture a resposta apontando: Riscos da Rota, Cláusulas Sugeridas para a Apólice e Medidas de Prevenção de Perdas."
  * "Se o usuário perguntar sobre rotas marítimas internacionais, destaque as regras de compliance aduaneiro e os incoterms aplicáveis."

#### 4. Para o Time Financeiro e Controladoria
**Objetivo:** Um assistente analítico para revisão de custos e saúde financeira.
* **`role`:** `"Analista Sênior de Controladoria e Finanças Corporativas"`
* **`instructions` (Exemplos de regras):**
  * "Você analisa cenários de fluxo de caixa, redução de custos operacionais (OPEX) e viabilidade de investimentos (CAPEX)."
  * "Diante de um cenário de queda de margem de lucro, proponha um plano de ação focado em eficiência imediata e renegociação de contratos."
  * "Nunca forneça conselhos de investimentos em bolsa de valores; restrinja-se à saúde financeira interna da empresa."

**O seu próximo passo:** Copie o código original da nossa aula, escolha uma das áreas acima que mais se aproxima da sua realidade profissional e modifique os parâmetros. Execute a célula e veja o seu novo assistente ganhar vida!


# Gabarito: Código completo da aplicação do Assistente Jurídico


```python
# instalando as ferramentas necessárias
!pip install -q agno
!pip install -q groq
```


```python
# ===============================
# FERRAMENTAS
# ===============================

import gradio as gr
from agno.agent import Agent
from agno.models.groq import Groq
from google.colab import userdata
import os

# ===============================
# AUTENTICAÇÃO API_KEY_GROQ
# ===============================

chave_groq = userdata.get('API_KEY_GROQ')
os.environ["GROQ_API_KEY"] = chave_groq

# ===============================
# AGENTE ESPECIALISTA
# ===============================

agente_consultor_juridico = Agent(
    model=Groq(id="llama-3.3-70b-versatile"),
    
    # persona
    role="Consultor Sênior de Estratégia para Escritórios de Advocacia",
    
    instructions=[
        # Quem é o agente
        "Você é um consultor sênior especializado em estratégia e gestão de escritórios de advocacia.",
        
        # Qual problema ele resolve
        "Seu objetivo é ajudar escritórios jurídicos a melhorar posicionamento, rentabilidade, organização interna e crescimento sustentável.",
        
        # Para qual público responde
        "Você responde principalmente a sócios, gestores jurídicos e advogados responsáveis pela administração do escritório.",
        
        # Nível de profundidade
        "Forneça respostas em nível estratégico e executivo, evitando explicações excessivamente básicas.",
        
        # Estrutura obrigatória
        "Organize sempre a resposta nos seguintes tópicos:",
        "- Contexto",
        "- Diagnóstico",
        "- Análise de riscos",
        "- Estratégias recomendadas",
        "- Próximos passos",
        
        # Limite de escopo
        "Se a pergunta estiver fora do escopo jurídico-empresarial, informe educadamente que o tema não faz parte da sua especialização.",
        
        # Estilo
        "Seja objetivo, estratégico e orientado à tomada de decisão."
    ]
)

# ===============================
# PROCEDIMENTO OP. PADRÃO (POP)
# ===============================

def consultar_assistente(pergunta_do_usuario):
    # O agente processa a pergunta e retorna a resposta gerada
    resposta = agente_consultor_juridico.run(pergunta_do_usuario)
    return resposta.content


# ===============================
# INTERFACE GRADIO
# ===============================

# Construindo e inicializando a Interface Visual
interface = gr.Interface(
    fn=consultar_assistente,
    inputs=gr.Textbox(lines=25, placeholder="Descreva o cenário ou desafio da sua empresa..."),
    outputs=gr.Markdown(),
    title="Portal do Consultor Estratégico",
    description="Insira sua dúvida de negócios abaixo para receber orientações estruturadas da IA."
)

# Inicia a aplicação para visualização no Colab e gera um link público para compartilhamento
interface.launch(share=True)


# ===============================
# PERGUNTA PARA TESTE
# ===============================

# - Como um escritório de advocacia pode aumentar sua captação de clientes empresariais?
# - Tenho um escritório com 12 advogados e queda de 25% no faturamento nos últimos 8 meses. O que devo fazer?
# - Escreva um poema sobre justiça.



```

