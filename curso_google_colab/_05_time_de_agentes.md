# Aula Prática: Times de Agentes de IA — Do Agente Solo ao Time que Trabalha por Você

Nas aulas anteriores, você aprendeu a criar um único agente especialista: dar a ele um cargo, regras de conduta e ferramentas. Ele funciona bem quando o problema é simples e linear.

Mas e quando a tarefa é complexa demais para um só especialista? Quando ela exige analisar sentimentos **e** auditar uma pauta **e** propor um plano de ação, tudo junto, e de forma coordenada?

Nesta aula, vamos dar um salto arquitetural: sair do agente solo e construir um **time de agentes**, onde cada membro tem um papel distinto e um agente líder coordena o trabalho de todos. O projeto do dia é o **Co-piloto Executivo de RH**, uma ferramenta que transforma as anotações de uma reunião de One-on-One em um diagnóstico completo de clima organizacional.

---

## 1. O Que é um Time de Agentes? A Visão de Negócios

Antes de escrever uma linha de código, precisamos entender o conceito. Vamos usar a linguagem que você já conhece: a de gestão de pessoas.

### A Analogia com a Equipe Humana

Imagine que você é um Diretor de RH e precisa fazer o diagnóstico de clima de um time depois de várias reuniões individuais. Como você resolveria isso com uma equipe humana?

Você não faria tudo sozinho. Você convocaria especialistas:

| Profissional | Responsabilidade |
|---|---|
| **Analista de Clima** | Lê as anotações e mapeia temas e sentimentos |
| **Consultor de RH** | Audita se todos os pilares da reunião foram cobertos |
| **Estrategista** | Consolida o diagnóstico e propõe o plano de ação |
| **Diretor de RH** | Coordena os três, revisa e entrega o relatório final |

Um time de agentes de IA replica exatamente essa lógica. Em vez de contratar pessoas, você **cria agentes especializados** e define quem coordena quem.

### Por que um Time é Melhor que um Agente Solo?

Um único agente tentando fazer tudo isso ao mesmo tempo tende a misturar as perspectivas, perder profundidade em cada análise e gerar respostas genéricas.

Com um time, cada agente:

- **Foca em uma tarefa específica** → mais precisão e profundidade
- **Usa o modelo mais adequado para seu trabalho** → custo otimizado
- **Pode ser auditado por outro agente** → maior confiabilidade do resultado final

> 💡 **Dica de Negócios:** Pense em cada agente como um colaborador com um Job Description muito claro. Quanto mais específico o cargo, melhor o desempenho — tanto em humanos quanto em IAs.

---

## 2. Conceito-Chave: Como o Agno Orquestra um Time

O framework Agno tem um mecanismo nativo para isso. Um agente pode receber um parâmetro chamado `team`, que é simplesmente uma lista de outros agentes.

```python
# Estrutura conceitual de um time no Agno
agente_lider = Agent(
    name="Diretor",
    team=[agente_a, agente_b, agente_c],  # ← aqui está o time
    model=...,
    instructions=[...]
)
```

Quando você chama `agente_lider.run("sua pergunta")`, o Agno entrega a tarefa ao líder, que por sua vez aciona os membros do time na ordem que suas instruções mandam, coleta os resultados e produz a entrega final.

### Os Papéis no Time

| Papel | O que faz no Agno |
|---|---|
| **Agente Membro** | Recebe uma subtarefa, a executa e retorna o resultado para o líder |
| **Agente Líder** | Coordena os membros via `team=[...]`, consolida e entrega o relatório final |

> ⚠️ **Importante:** O agente líder não precisa ser o mais caro ou poderoso. Sua função é coordenar e formatar a entrega. Os membros é que fazem o trabalho pesado analítico.

---

## 3. A Arquitetura do Nosso Projeto

Agora que você entende o conceito, veja a foto de satélite do que vamos construir hoje.

### O Time do Co-piloto Executivo de RH

```
[Gestor]
   │
   │ Cola as anotações da reunião na interface Gradio
   ▼
[Diretor de RH]  ← Agente Líder (GPT-4o mini)
   │
   ├──▶ [Mapeador de Sinais]    → Extrai temas e classifica sentimentos (Llama 3 via Groq)
   │
   ├──▶ [Auditor de Pauta]      → Verifica lacunas nos 5 pilares de RH (Llama 3 via Groq)
   │
   └──▶ [Estrategista de Risco] → Analisa riscos e propõe 3 insights acionáveis (GPT-4o mini)
           │
           ▼
   [Relatório Final em Markdown]
```

### Por que esse time usa dois modelos diferentes?

Repare que usamos **Groq (Llama 3)** para os membros analistas e **OpenAI (GPT-4o mini)** para o estrategista e o líder. Isso é uma decisão de arquitetura deliberada:

| Agente | Modelo | Justificativa |
|---|---|---|
| Mapeador de Sinais | `Groq / Llama 3` | Tarefa de extração e classificação — rápida e gratuita |
| Auditor de Pauta | `Groq / Llama 3` | Verificação estruturada contra uma lista fixa — gratuita |
| Estrategista de Risco | `OpenAI / GPT-4o mini` | Raciocínio estratégico mais elaborado — custo baixo |
| Diretor de RH (Líder) | `OpenAI / GPT-4o` | Coordenação e formatação final do relatório executivo |

> 💡 **Dica de Arquitetura:** Use modelos gratuitos para tarefas de triagem e classificação. Reserve o GPT-4o mini (pago, mas econômico) para onde o raciocínio executivo faz diferença. Isso reduz o custo operacional do sistema em até 70%.

---

## 4. Pré-requisitos do Projeto

### Contas e Credenciais Necessárias

Você precisará de chaves de acesso em duas plataformas:

**Groq (gratuito)**

1. Acesse [console.groq.com](https://console.groq.com)
2. Crie uma conta gratuita
3. Vá em **API Keys → Create API Key**
4. Copie a chave gerada

**OpenAI (pago — ~USD$ 5,00 (~R$ 30,00) de crédito já é suficiente para toda a aula)**

1. Acesse [platform.openai.com](https://platform.openai.com)
2. Vá em **Billing → Add to credit balance** e adicione um crédito mínimo
3. Vá em **API Keys → Create new secret key**
4. Copie a chave gerada

### Configurando o Cofre de Senhas do Google Colab

Por segurança, **nunca cole suas chaves diretamente no código**. Use o cofre de Secrets do Colab.

No menu lateral esquerdo do Google Colab, clique no ícone de chave 🔑 **(Secrets)** e adicione:

| Nome do Secret | O que é |
|---|---|
| `GROQ_API_KEY` | Sua chave da Groq |
| `OPENAI_API_KEY` | Sua chave da OpenAI |

> 📌 **Lembre-se:** Ative o botão (interruptor) ao lado de cada secret para que o notebook possa acessá-las. Veja o tutorial completo em [Introdução ao Google Colab](https://github.com/rgaguiar/turma-dc-ia01/blob/main/curso_google_colab/_01_intro_colab.md).

---

## 5. Construindo o Projeto Bloco a Bloco

### Bloco 1: Instalando as Dependências

Como o Google Colab começa "zerado" a cada sessão, precisamos instalar os pacotes antes de qualquer coisa.

```python
# ===============================
# INSTALAÇÃO DAS DEPENDÊNCIAS
# ===============================
!pip install -U agno groq openai gradio
```

**Por que cada pacote?**

| Pacote | Função no Projeto |
|---|---|
| `agno` | O gerente do time — cria e orquestra os agentes |
| `groq` | Conecta o Agno aos modelos Llama 3 (gratuito e rápido) |
| `openai` | Conecta o Agno ao GPT-4o mini para o estrategista e o líder |
| `gradio` | Cria a interface web visual para o gestor usar |

---

### Bloco 2: Importando as Ferramentas e Autenticando

```python
# ===============================
# FERRAMENTAS
# ===============================
import os
import gradio as gr
from google.colab import userdata

from agno.agent import Agent
from agno.models.groq import Groq
from agno.models.openai import OpenAIChat

# ===============================
# AUTENTICAÇÃO
# ===============================
os.environ["GROQ_API_KEY"]   = userdata.get('GROQ_API_KEY')
os.environ["OPENAI_API_KEY"] = userdata.get('OPENAI_API_KEY')
```

> 📌 **Atenção:** O `userdata.get()` busca a chave diretamente no cofre de Secrets do Colab. Assim, as suas senhas nunca ficam expostas no código — nem nos prints, nem nos logs.

---

### Bloco 3: Criando os Agentes Membros do Time

Este é o bloco mais importante conceitualmente. Aqui você vai perceber que **criar um agente membro é idêntico a criar qualquer agente** — a diferença é que depois vamos entregá-los ao líder via `team=[...]`.

#### Agente 1 — Mapeador de Sinais

```python
# ===============================
# AGENTE 1: MAPEADOR DE SINAIS
# ===============================
mapeador_sinais = Agent(
    name="Mapeador de Sinais",

    # Cargo do agente dentro do time
    role="Especialista em extração de temas e análise de sentimentos",

    # Modelo: Groq com Llama 3 — rápido e gratuito para tarefas de classificação
    model=Groq(id="llama-3.3-70b-versatile"),

    instructions=[
        # O que ele deve fazer
        "Você receberá anotações brutas de uma reunião de One-on-One ou de equipe.",

        # Como ele deve estruturar a saída
        "Leia atentamente e extraia TODOS os temas mencionados, explícita ou implicitamente.",
        "Para cada tema identificado, classifique o sentimento predominante como POSITIVO, NEGATIVO ou NEUTRO.",

        # Formato obrigatório da entrega
        "Organize sua análise em uma tabela Markdown com as colunas: Tema | Classificação | Trecho de Referência.",
        "Seja objetivo. Não inclua recomendações — apenas o mapeamento."
    ]
)
```

> 💡 **Por que `llama-3.3-70b-versatile`?** Para tarefas de leitura e classificação estruturada, o Llama 3 da Groq entrega resultados excelentes com latência baixíssima e custo zero. Guarde o GPT-4o para onde ele realmente faz a diferença.

---

#### Agente 2 — Auditor de Pauta

```python
# ===============================
# AGENTE 2: AUDITOR DE PAUTA
# ===============================
auditor_pauta = Agent(
    name="Auditor de Pauta de RH",

    # Cargo do agente
    role="Especialista em diagnóstico de qualidade de reuniões de RH",

    # Modelo: Groq com Llama 3 — verificação estruturada contra uma lista fixa
    model=Groq(id="llama-3.3-70b-versatile"),

    instructions=[
        # O contexto de trabalho
        "Você receberá a análise de sentimentos de uma reunião já processada pelo Mapeador de Sinais.",

        # A base de referência que ele deve usar
        "Uma reunião de diagnóstico de RH de qualidade deve cobrir obrigatoriamente os 5 Pilares:",
        "1. Liderança e relacionamento com a chefia",
        "2. Remuneração e reconhecimento",
        "3. Jornada de trabalho e carga",
        "4. Ferramentas e infraestrutura",
        "5. Relacionamento com a equipe e clima",

        # O que ele deve entregar
        "Verifique quais desses 5 pilares NÃO foram abordados nas anotações.",
        "Liste os pilares ausentes e, para cada um, explique o risco de deixá-lo sem diagnóstico.",
        "Use linguagem direta e executiva."
    ]
)
```

---

#### Agente 3 — Estrategista de Risco

```python
# ===============================
# AGENTE 3: ESTRATEGISTA DE RISCO
# ===============================
analista_risco = Agent(
    name="Estrategista de Risco",

    # Cargo do agente
    role="Especialista em gestão de riscos de pessoas e planos de ação executivos",

    # Modelo: GPT-4o mini — raciocínio estratégico mais elaborado
    model=OpenAIChat(id="gpt-4o-mini"),

    instructions=[
        # O contexto de trabalho
        "Você receberá o mapeamento de temas e a auditoria de pauta produzidos pelos seus colegas de time.",

        # O que ele deve analisar
        "Com base nessas informações, faça uma Análise de Risco focada em:",
        "- Risco de Burnout: avalie os sinais de sobrecarga emocional ou de trabalho",
        "- Risco de Turnover: avalie os sinais de desengajamento ou intenção de saída",
        "Classifique cada risco como ALTO, MÉDIO ou BAIXO com justificativa.",

        # O que ele deve propor
        "Ao final, proponha exatamente 3 Insights Acionáveis — ações práticas que o gestor pode tomar nos próximos 30 dias.",
        "Cada insight deve ser específico, mensurável e viável para um gestor de equipe.",

        # Estilo da entrega
        "Use linguagem executiva. Sem introduções longas. Vá direto ao ponto."
    ]
)
```

---

### Bloco 4: Criando o Agente Líder

O agente líder é quem recebe o pedido do usuário, aciona os membros do time na sequência correta e entrega o relatório final consolidado. Ele é definido **depois** dos membros, pois precisa referenciá-los no parâmetro `team`.

```python
# ===============================
# AGENTE LÍDER: DIRETOR DE RH
# ===============================
diretor_rh = Team(
    name="Diretor de RH",

    # Cargo do agente líder
    role="Coordenador de diagnóstico de clima organizacional",

    # Modelo: GPT-4o mini — coordenação e formatação executiva do relatório final
    model=OpenAIChat(id="gpt-4o-mini"),

    # ← Aqui está o coração do time: a lista de membros
    members=[mapeador_sinais, auditor_pauta, analista_risco],

    instructions=[
        # O script de coordenação — o líder sabe a ordem de acionamento
        "Você coordena um time de especialistas de RH para fazer o diagnóstico de clima organizacional.",
        "Siga exatamente esta sequência de trabalho:",
        "1. Acione o Mapeador de Sinais para identificar temas e sentimentos das anotações.",
        "2. Acione o Auditor de Pauta para verificar lacunas nos 5 pilares.",
        "3. Acione o Estrategista de Risco para consolidar riscos e propor insights.",

        # O que ele deve entregar — apenas o relatório final, sem ruído
        "Após receber as análises dos três especialistas, produza APENAS o Relatório Final.",
        "Não repita os processos intermediários. Entregue diretamente o documento executivo.",

        # Estrutura obrigatória do relatório final
        "O Relatório Final deve seguir obrigatoriamente esta estrutura:",
        "## 📊 Diagnóstico de Clima Organizacional",
        "### 1. Mapeamento de Temas e Sentimentos",
        "### 2. Lacunas de Pauta Identificadas",
        "### 3. Análise de Risco (Burnout e Turnover)",
        "### 4. Plano de Ação — 3 Insights para os Próximos 30 Dias",

        # Tom e estilo
        "Use linguagem objetiva, profissional e orientada à tomada de decisão.",
        "O gestor que vai ler esse relatório não tem tempo. Seja direto e preciso."
    ],

    # Ativa a formatação Markdown no relatório final
    markdown=True
)
```

> ⚠️ **Importante:** Note que o `team=[...]` recebe as **variáveis** dos agentes, não strings com seus nomes. O Agno usa essas referências diretamente para acionar cada membro. Por isso os membros precisam ser criados **antes** do líder.

---

### Bloco 5: Criando a Ponte entre o Gradio e o Agno

Esta função é simples, mas fundamental: ela é a "ponte" que conecta o que o usuário digita na tela ao time de agentes.

```python
# ===============================
# PROCEDIMENTO OP. PADRÃO (POP)
# ===============================
def analisar_clima_organizacional(anotacoes_reuniao):
    """
    Recebe as anotações do gestor, aciona o time de agentes
    e retorna o relatório final formatado.
    """
    # Validação básica: impede que o sistema rode com campo vazio
    if not anotacoes_reuniao.strip():
        return "⚠️ Por favor, insira as anotações da reunião antes de continuar."

    # Envia as anotações para o Diretor de RH (agente líder)
    # O .run() aciona toda a cadeia do time automaticamente
    resposta = diretor_rh.run(
        f"Analise as anotações desta reunião de clima organizacional: {anotacoes_reuniao}"
    )

    # Retorna apenas o conteúdo textual do relatório final
    return resposta.content
```

---

### Bloco 6: Construindo a Interface Visual com Gradio

Agora vamos criar a "recepção" do sistema — a tela que o gestor vai usar, sem precisar saber nada de código.

```python
# ===============================
# INTERFACE GRADIO
# ===============================
app_rh = gr.Interface(
    fn=analisar_clima_organizacional,

    inputs=gr.Textbox(
        lines=10,
        placeholder="Cole aqui as anotações da sua reunião de One-on-One ou de equipe...",
        label="Anotações do Gestor"
    ),

    outputs=gr.Markdown(
        label="Diagnóstico Oficial e Plano de Ação"
    ),

    title="Co-piloto Executivo de RH",

    description="Transforme anotações rápidas de reuniões em um diagnóstico completo de clima organizacional e gestão de riscos."
)

app_rh.launch(
    share=True,
    debug=True,
    theme=gr.themes.Soft()
)

# ===============================
# INICIALIZANDO O SISTEMA
# ===============================
app_rh.launch(share=True, debug=True)
```

---

### Bloco 7: Testando o Sistema

Com a interface no ar, use as anotações abaixo para validar o funcionamento do time.

```python
# ===============================
# CASOS DE TESTE
# ===============================

# TESTE 1 — Cenário de risco alto (burnout e falhas de pauta)
"""
João chegou visivelmente cansado. Disse que a carga está muito pesada ultimamente
e que sente que está sempre apagando incêndio. Mencionou que as ferramentas do sistema
travam com frequência e isso atrasa tudo. Não falou nada sobre salário nem sobre a equipe.
Saiu rápido, disse que tinha muita coisa para entregar ainda hoje.
"""

# TESTE 2 — Cenário de risco moderado (equipe engajada, mas pontos de atenção)
"""
Maria está animada com o projeto novo. Elogiou bastante o suporte da liderança.
Mas mencionou que o relacionamento com um colega específico está tenso há semanas
e que prefere evitar reuniões em conjunto. Salário não foi pauta. Disse que gosta do
trabalho mas está curiosa sobre oportunidades de crescimento na empresa.
"""

# TESTE 3 — Guardrail: assunto fora do escopo
"""
Você pode me sugerir um bom restaurante para o almoço de equipe de sexta-feira?
"""
```

**O que observar em cada teste:**

- **Teste 1:** O Mapeador deve identificar os sinais de sobrecarga. O Auditor deve apontar que Remuneração, Liderança e Relacionamento com a equipe não foram cobertos. O Estrategista deve classificar o risco de Burnout como ALTO.
- **Teste 2:** O time deve identificar o risco de clima interpessoal como ponto de atenção e o risco de Turnover como MÉDIO pela pergunta sobre crescimento.
- **Teste 3:** O líder deve reconhecer que a solicitação está fora do escopo e informar isso de forma educada, sem acionar o time.

---

## 6. O Resultado Final

O que você acabou de construir é um **departamento de diagnóstico de RH funcional**, operando em segundos, com custo marginal por análise e disponível 24 horas via link no navegador.

**Por que esse projeto tem valor real de negócio?**

- **Autonomia para gestores não-técnicos:** Qualquer líder de equipe usa o sistema sem saber programar. Ele só precisa colar as anotações e clicar em "Enviar".
- **Padronização de entregas:** Todo relatório segue a mesma estrutura — Mapeamento, Auditoria, Risco e Plano de Ação. Sem análises genéricas ou inconsistentes.
- **Custo otimizado por arquitetura:** Os agentes de classificação rodam gratuitamente via Groq. O custo com OpenAI fica restrito às análises que realmente precisam de raciocínio avançado.
- **Escalável para qualquer área:** Mudando os membros do time e as instruções, a mesma arquitetura serve para Jurídico, Marketing, Financeiro ou Atendimento.

---

## 7. Expandindo a Arquitetura: Adaptações por Área

A estrutura de time que você aprendeu hoje é reutilizável. Veja como adaptar para outras realidades:

### Para o Time Jurídico

- **Membro 1 — Analisador de Risco Legal:** Lê o caso e identifica os pontos de vulnerabilidade jurídica
- **Membro 2 — Pesquisador de Jurisprudência:** Busca precedentes relevantes (com ferramenta de busca)
- **Membro 3 — Redator de Parecer:** Consolida em linguagem formal para o cliente
- **Líder — Sócio Coordenador:** Revisa e entrega o parecer final

### Para o Time de Marketing

- **Membro 1 — Analista de Persona:** Identifica o perfil do público-alvo no briefing
- **Membro 2 — Estrategista de Canais:** Define os canais e a abordagem mais eficaz
- **Membro 3 — Redator de Copy:** Gera os textos para cada canal
- **Líder — Head de Marketing:** Consolida a estratégia e a entrega em um plano acionável

### Para o Time de Análise de Dados

- **Membro 1 — Analista de Qualidade:** Verifica a consistência e completude dos dados recebidos
- **Membro 2 — Analista Estatístico:** Identifica padrões, tendências e anomalias
- **Membro 3 — Comunicador de Insights:** Traduz os achados em linguagem executiva
- **Líder — Data Lead:** Consolida e formata o relatório final para a diretoria# ===============================
# INSTALAÇÃO DAS DEPENDÊNCIAS
# ===============================
!pip install -U agno groq openai gradio

> 💡 **Seu próximo passo:** Escolha uma das áreas acima, ou pense em um problema real do seu trabalho, e adapte o time. Os blocos de código são os mesmos — o que muda são os `name`, `role` e `instructions` de cada agente.

---

## 8. Próxima Aula

Na **Aula 2**, vamos aumentar a complexidade do time:

- **Memória persistente:** o time vai lembrar de conversas anteriores entre sessões
- **Ferramenta de busca:** um agente vai pesquisar informações na internet em tempo real
- **Envio de e-mail:** o relatório final será entregue automaticamente na caixa do gestor

O projeto será um **Assistente Jurídico com busca em legislação**, construído para analistas de compliance, advogados e gestores de contratos.

---

## Gabarito: Código Completo do Co-piloto Executivo de RH

```python
# ===============================
# INSTALAÇÃO DAS DEPENDÊNCIAS
# ===============================
!pip install -U agno groq openai gradio
```

```python
# ===============================
# FERRAMENTAS
# ===============================
import os
import gradio as gr
from google.colab import userdata
from agno.team import Team
from agno.agent import Agent
from agno.models.groq import Groq
from agno.models.openai import OpenAIChat

# ===============================
# AUTENTICAÇÃO
# ===============================
os.environ["GROQ_API_KEY"]   = userdata.get('GROQ_API_KEY')
os.environ["OPENAI_API_KEY"] = userdata.get('OPENAI_API_KEY')

# ===============================
# AGENTE 1: MAPEADOR DE SINAIS
# ===============================
mapeador_sinais = Agent(
    name="Mapeador de Sinais",
    role="Especialista em extração de temas e análise de sentimentos",
    model=Groq(id="llama-3.3-70b-versatile"),
    instructions=[
        "Você receberá anotações brutas de uma reunião de One-on-One ou de equipe.",
        "Leia atentamente e extraia TODOS os temas mencionados, explícita ou implicitamente.",
        "Para cada tema identificado, classifique o sentimento predominante como POSITIVO, NEGATIVO ou NEUTRO.",
        "Organize sua análise em uma tabela Markdown com as colunas: Tema | Classificação | Trecho de Referência.",
        "Seja objetivo. Não inclua recomendações — apenas o mapeamento."
    ]
)

# ===============================
# AGENTE 2: AUDITOR DE PAUTA
# ===============================
auditor_pauta = Agent(
    name="Auditor de Pauta de RH",
    role="Especialista em diagnóstico de qualidade de reuniões de RH",
    model=Groq(id="llama-3.3-70b-versatile"),
    instructions=[
        "Você receberá a análise de sentimentos de uma reunião já processada pelo Mapeador de Sinais.",
        "Uma reunião de diagnóstico de RH de qualidade deve cobrir obrigatoriamente os 5 Pilares:",
        "1. Liderança e relacionamento com a chefia",
        "2. Remuneração e reconhecimento",
        "3. Jornada de trabalho e carga",
        "4. Ferramentas e infraestrutura",
        "5. Relacionamento com a equipe e clima",
        "Verifique quais desses 5 pilares NÃO foram abordados nas anotações.",
        "Liste os pilares ausentes e, para cada um, explique o risco de deixá-lo sem diagnóstico.",
        "Use linguagem direta e executiva."
    ]
)

# ===============================
# AGENTE 3: ESTRATEGISTA DE RISCO
# ===============================
analista_risco = Agent(
    name="Estrategista de Risco",
    role="Especialista em gestão de riscos de pessoas e planos de ação executivos",
    model=OpenAIChat(id="gpt-4o-mini"),
    instructions=[
        "Você receberá o mapeamento de temas e a auditoria de pauta produzidos pelos seus colegas de time.",
        "Com base nessas informações, faça uma Análise de Risco focada em:",
        "- Risco de Burnout: avalie os sinais de sobrecarga emocional ou de trabalho",
        "- Risco de Turnover: avalie os sinais de desengajamento ou intenção de saída",
        "Classifique cada risco como ALTO, MÉDIO ou BAIXO com justificativa.",
        "Ao final, proponha exatamente 3 Insights Acionáveis — ações práticas que o gestor pode tomar nos próximos 30 dias.",
        "Cada insight deve ser específico, mensurável e viável para um gestor de equipe.",
        "Use linguagem executiva. Sem introduções longas. Vá direto ao ponto."
    ]
)

# ===============================
# AGENTE LÍDER: DIRETOR DE RH
# ===============================
diretor_rh = Team(
    name="Diretor de RH",
    role="Coordenador de diagnóstico de clima organizacional",
    model=OpenAIChat(id="gpt-4o-mini"),
    members=[mapeador_sinais, auditor_pauta, analista_risco],
    instructions=[
        "Você coordena um time de especialistas de RH para fazer o diagnóstico de clima organizacional.",
        "Siga exatamente esta sequência de trabalho:",
        "1. Acione o Mapeador de Sinais para identificar temas e sentimentos das anotações.",
        "2. Acione o Auditor de Pauta para verificar lacunas nos 5 pilares.",
        "3. Acione o Estrategista de Risco para consolidar riscos e propor insights.",
        "Após receber as análises dos três especialistas, produza APENAS o Relatório Final.",
        "Não repita os processos intermediários. Entregue diretamente o documento executivo.",
        "O Relatório Final deve seguir obrigatoriamente esta estrutura:",
        "## 📊 Diagnóstico de Clima Organizacional",
        "### 1. Mapeamento de Temas e Sentimentos",
        "### 2. Lacunas de Pauta Identificadas",
        "### 3. Análise de Risco (Burnout e Turnover)",
        "### 4. Plano de Ação — 3 Insights para os Próximos 30 Dias",
        "Use linguagem objetiva, profissional e orientada à tomada de decisão.",
        "O gestor que vai ler esse relatório não tem tempo. Seja direto e preciso."
    ],
    markdown=True
)

# ===============================
# PROCEDIMENTO OP. PADRÃO (POP)
# ===============================
def analisar_clima_organizacional(anotacoes_reuniao):
    if not anotacoes_reuniao.strip():
        return "⚠️ Por favor, insira as anotações da reunião antes de continuar."

    resposta = diretor_rh.run(
        f"Analise as anotações desta reunião de clima organizacional: {anotacoes_reuniao}"
    )
    return resposta.content

# ===============================
# INTERFACE GRADIO
# ===============================
app_rh = gr.Interface(
    fn=analisar_clima_organizacional,
    inputs=gr.Textbox(
        lines=10,
        placeholder="Cole aqui as anotações da sua reunião de One-on-One ou de equipe...",
        label="📝 Anotações do Gestor"
    ),
    outputs=gr.Markdown(label="📋 Diagnóstico Oficial e Plano de Ação"),
    title="📊 Co-piloto Executivo de RH",
    description="Transforme anotações rápidas de reuniões em um diagnóstico completo de clima organizacional e gestão de riscos.",
    theme=gr.themes.Soft(),
    allow_flagging="never"
)

# ===============================
# INICIALIZANDO O SISTEMA
# ===============================
app_rh.launch(share=True, debug=True)
```

```python
# ===============================
# CASOS DE TESTE
# ===============================

# TESTE 1 — Risco alto (burnout + lacunas de pauta)
# "João chegou visivelmente cansado. Disse que a carga está muito pesada e
#  que as ferramentas travam com frequência. Não falou sobre salário nem
#  sobre a equipe. Saiu rápido."

# TESTE 2 — Risco moderado (engajamento com ponto de atenção)
# "Maria está animada com o projeto novo e elogiou a liderança.
#  Mas mencionou tensão com um colega. Está curiosa sobre crescimento na empresa."

# TESTE 3 — Guardrail: fora do escopo
# "Você pode me sugerir um bom restaurante para o almoço de equipe?"
```
