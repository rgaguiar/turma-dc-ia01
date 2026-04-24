# Aula Prática: Times de Agentes com Ferramentas — O Time que Age no Mundo Real

Na Aula 1, você construiu um time que **pensa**: lê anotações, analisa sentimentos, audita pautas e entrega um diagnóstico executivo. Os agentes trabalhavam apenas com o texto que você colava na tela, e faziam isso muito bem.

Mas esse time tinha uma limitação clara: ele não sabe o que aconteceu ontem. Ele não consulta a legislação atualizada, não verifica a convenção coletiva vigente, não acessa nenhuma fonte externa. Se a lei mudou na semana passada, ele não sabe.

Nesta aula, vamos resolver isso. Vamos equipar o time com **ferramentas**, a capacidade de pesquisar na web em tempo real e executar cálculos precisos com código Python.

O projeto do dia é o **Assistente Jurídico Trabalhista**, um time que recebe a descrição de uma situação de RH, pesquisa a legislação vigente, verifica a conformidade legal e calcula as verbas rescisórias, entregando um parecer técnico completo.

---

## 1. O Novo Conceito: Ferramentas nos Agentes

### A Diferença entre Pensar e Agir

Na Aula 1, vimos que dividir o raciocínio em especialistas melhora muito a qualidade da análise. Mas raciocínio puro tem um teto: o modelo de IA foi treinado até uma data de corte, ele não sabe o que mudou depois disso.

Um advogado trabalhista sênior não trabalha de memória. Antes de emitir um parecer, ele **pesquisa**: consulta o texto atualizado da CLT, verifica súmulas recentes do TST, confere a convenção coletiva da categoria. Ferramentas dão exatamente essa capacidade ao agente.

| Agente sem ferramenta | Agente com ferramenta |
| --- | --- |
| Responde com o que foi treinado até a data de corte | Consulta fontes reais antes de responder |
| Pode citar artigo revogado ou percentual desatualizado | Trabalha com informação atual da web |
| Limitado ao contexto que você fornece | Expande ativamente o contexto quando necessário |
| Ideal para análise de documentos que você entrega | Ideal para pesquisa, compliance e due diligence |

> 💡 **Dica de Negócios:** Use agentes sem ferramentas quando o insumo é o documento que você tem em mãos. Use agentes com ferramentas quando o insumo precisa ser buscado, legislação, preço de mercado, notícia recente, dado público.

### Como o Agno Implementa Ferramentas

Adicionar uma ferramenta a um agente é tão simples quanto declarar uma lista no parâmetro `tools`:

```
# Estrutura conceitual — agente com ferramenta de busca
from agno.tools.duckduckgo import DuckDuckGoTools

agente_pesquisador = Agent(
    name="Pesquisador",
    model=Groq(id="llama-3.3-70b-versatile"),
    tools=[DuckDuckGoTools()],   # ← a ferramenta entra aqui
    instructions=[...]
)
```

Quando o agente entende, a partir das suas instruções, que precisa de uma informação externa, ele **aciona a ferramenta automaticamente**, processa o resultado e continua o raciocínio. Você não gerencia isso — o Agno gerencia.

---

## 2. As Ferramentas desta Aula

| Ferramenta | O que faz | Custo |
| --- | --- | --- |
| `DuckDuckGoTools` | Pesquisa na web sem chave de API | Gratuito |
| `PythonTools` | Executa código Python dentro do agente | Gratuito |

### Por que DuckDuckGo e não Google?

O DuckDuckGo não exige cadastro, não tem limite de requisições para uso moderado e indexa muito bem fontes públicas como `planalto.gov.br`, `tst.jus.br` e `jusbrasil.com.br` — exatamente onde está a legislação trabalhista brasileira que nosso agente vai precisar.

### Por que PythonTools?

Cálculos trabalhistas têm regras matemáticas precisas: avos de férias, proporcionalidade de 13º, FGTS sobre verbas. Se você pedir para a IA **calcular** sem uma ferramenta de execução de código, ela vai raciocinar sobre os valores — e pode errar. Com `PythonTools`, 
o agente **escreve e executa** o cálculo em Python, garantindo precisão matemática real.

> ⚠️ **Importante:** Ferramentas não substituem as instruções. O agente só vai usar a ferramenta se suas `instructions` mandarem ele usar. Sem instrução explícita, ele pode ignorar a ferramenta e responder de memória.

---

## 3. A Arquitetura do Projeto

### O Problema que Vamos Resolver

Uma situação comum em qualquer empresa com área de RH ou jurídico:

> *"Temos um colaborador com 14 meses de empresa, salário de R$ 4.200, CLT, jornada de 44h semanais. A empresa quer fazer uma demissão sem justa causa. Quais são as obrigações legais? Quanto deve ser pago? Existe risco de passivo trabalhista?"*

Responder isso com precisão exige pesquisar os artigos da CLT aplicáveis, verificar súmulas do TST, calcular aviso prévio proporcional, saldo de salário, férias proporcionais, 13º proporcional e FGTS com multa — tudo baseado em legislação vigente.

### O Time do Assistente Jurídico Trabalhista

```
[Advogado / Gestor de RH]
           │
           │  Descreve a situação trabalhista na interface Gradio
           ▼
[Coordenador Jurídico]  ← Líder do Time (GPT-4o mini)
           │
           ├──▶ [Pesquisador de Legislação]  → Busca CLT, súmulas TST e normas na web   (Llama 3 + DuckDuckGo)
           │
           ├──▶ [Analista de Compliance]     → Verifica conformidade e aponta riscos     (Llama 3 via Groq)
           │
           └──▶ [Calculador de Verbas]       → Calcula rescisórias e prazos em Python   (GPT-4o + PythonTools)
                        │
                        ▼
              [Parecer Técnico Completo em Markdown]
```

### Decisão de Modelos e Ferramentas

| Agente | Modelo | Ferramenta | Justificativa |
| --- | --- | --- | --- |
| Pesquisador de Legislação | `Groq / Llama 3` | `DuckDuckGoTools` | Busca web gratuita de fontes públicas de legislação |
| Analista de Compliance | `Groq / Llama 3` | Nenhuma | Analisa o que o Pesquisador trouxe — sem busca redundante |
| Calculador de Verbas | `GPT-4o` | `PythonTools` | Cálculos precisos exigem execução real de código |
| Coordenador Jurídico (Líder) | `GPT-4o mini` | Nenhuma | Coordena o time e formata o parecer final |

> 💡 **Arquitetura intencional:** O Analista de Compliance não tem ferramenta de busca propositalmente. Ele trabalha **sobre** o que o Pesquisador trouxe — isso evita buscas redundantes e mantém o custo baixo.

---

## 4. Pré-requisitos do Projeto

As credenciais são as mesmas da Aula 1. Você já tem tudo configurado no cofre de Secrets do Colab:

| Secret no Colab | Plataforma | Custo |
| --- | --- | --- |
| `GROQ_API_KEY` | [console.groq.com](https://console.groq.com) | Gratuito |
| `OPENAI_API_KEY` | [platform.openai.com](https://platform.openai.com) | ~R$ 0,15 por análise completa |

> 📌 **Lembrete:** O `DuckDuckGoTools` não consomem tokens adicionais — apenas a chamada ao modelo LLM é cobrada.

---

## 5. Construindo o Projeto Bloco a Bloco

### Bloco 1: Instalando as Dependências

```
# ===============================
# INSTALAÇÃO DAS DEPENDÊNCIAS
# ===============================
!pip install -U agno groq openai duckduckgo-search gradio
```

**O que é novo em relação à Aula 1:**

| Pacote | Função |
| --- | --- |
| `duckduckgo-search` | Motor de busca usado internamente pela `DuckDuckGoTools` |
| Os demais | Já conhecidos da Aula 1 |

---

### Bloco 2: Importando as Ferramentas e Autenticando

```
# ===============================
# FERRAMENTAS
# ===============================
import os
import gradio as gr
from google.colab import userdata

from agno.agent import Agent
from agno.team import Team
from agno.models.groq import Groq
from agno.models.openai import OpenAIChat

# Novidade desta aula: ferramentas que os agentes vão usar
from agno.tools.duckduckgo import DuckDuckGoTools
from agno.tools.python import PythonTools

# ===============================
# AUTENTICAÇÃO
# ===============================
os.environ["GROQ_API_KEY"]   = userdata.get('GROQ_API_KEY')
os.environ["OPENAI_API_KEY"] = userdata.get('OPENAI_API_KEY')
```

> 📌 **Atenção:** As ferramentas são objetos independentes, importados separadamente dos modelos. Você os instancia com `DuckDuckGoTools()` e `PythonTools()` e os entrega ao agente via `tools=[...]`. O Agno gerencia quando e como cada ferramenta é acionada durante a execução.

---

### Bloco 3: Criando os Agentes Membros do Time

#### Agente 1 — Pesquisador de Legislação

Este é o agente que **age no mundo real**. Antes de qualquer análise, ele vai à web buscar as fontes jurídicas relevantes e atualizadas.

```
# ===============================
# AGENTE 1: PESQUISADOR DE LEGISLAÇÃO
# ===============================
pesquisador_legislacao = Agent(
    name="Pesquisador de Legislação",

    # Cargo do agente dentro do time
    role="Especialista em pesquisa de legislação trabalhista e jurisprudência",

    # Modelo: Groq com Llama 3 — rápido e gratuito para busca e leitura de fontes
    model=Groq(id="llama-3.3-70b-versatile"),

    # Ferramenta de busca na web — gratuita, sem chave de API adicional
    tools=[DuckDuckGoTools()],

    instructions=[
        # O que ele deve fazer antes de qualquer coisa
        "Você receberá a descrição de uma situação trabalhista.",
        "SEMPRE pesquise na web antes de responder. Nunca responda apenas de memória.",

        # Onde pesquisar
        "Priorize as seguintes fontes oficiais:",
        "- planalto.gov.br (texto atualizado da CLT)",
        "- tst.jus.br (súmulas e jurisprudência do TST)",
        "- esocial.gov.br (obrigações acessórias)",
        "- jusbrasil.com.br (jurisprudência consolidada)",

        # O que ele deve entregar
        "Retorne uma lista estruturada com:",
        "- Os artigos da CLT aplicáveis ao caso (número + ementa resumida)",
        "- Súmulas do TST relevantes (número + enunciado)",
        "- Qualquer norma complementar identificada (portaria, instrução normativa)",

        # Limite de escopo
        "Foque exclusivamente no embasamento legal.",
        "Não faça análise de conformidade nem cálculos — isso é papel dos seus colegas de time.",
        "Se não encontrar uma fonte confiável para um ponto específico, informe explicitamente."
    ]
)
```

> 💡 **Por que instruir o agente a SEMPRE pesquisar?** Sem essa instrução explícita, o modelo pode decidir que "já sabe" a resposta e ignorar a ferramenta. A instrução `"SEMPRE pesquise na web antes de responder"` é o guardrail que garante que ele vai à fonte antes de falar.

---

#### Agente 2 — Analista de Compliance

Este agente não busca — ele **raciocina** sobre o que o Pesquisador trouxe e identifica os pontos de risco jurídico.

```
# ===============================
# AGENTE 2: ANALISTA DE COMPLIANCE
# ===============================
analista_compliance = Agent(
    name="Analista de Compliance",

    # Cargo do agente
    role="Especialista em análise de conformidade trabalhista e gestão de riscos jurídicos",

    # Modelo: Groq com Llama 3 — análise estruturada sobre as fontes fornecidas
    model=Groq(id="llama-3.3-70b-versatile"),

    # Sem ferramenta: analisa o que o Pesquisador trouxe — sem busca redundante
    instructions=[
        # O contexto de trabalho
        "Você receberá a descrição da situação trabalhista e a pesquisa de legislação feita pelo seu colega.",

        # O que ele deve verificar
        "Com base na legislação pesquisada, verifique:",
        "1. A empresa está cumprindo os prazos obrigatórios da CLT para este tipo de situação?",
        "2. Existe risco de configurar dispensa discriminatória ou nulidade do ato?",
        "3. Há obrigações acessórias que a empresa precisa cumprir (eSocial, CAGED, homologação)?",
        "4. Existem precedentes jurisprudenciais que aumentam o risco de passivo trabalhista?",

        # O que ele deve entregar
        "Para cada ponto verificado, classifique o risco como ALTO, MÉDIO ou BAIXO com justificativa.",
        "Liste as obrigações legais que a empresa deve cumprir, com prazo quando aplicável.",
        "Use linguagem direta e executiva. O leitor é um gestor de RH ou advogado."
    ]
)
```

---

#### Agente 3 — Calculador de Verbas

Este agente usa o `PythonTools` para executar os cálculos com precisão matemática real — não estimativas.

```
# ===============================
# AGENTE 3: CALCULADOR DE VERBAS
# ===============================
calculador_verbas = Agent(
    name="Calculador de Verbas",

    # Cargo do agente
    role="Especialista em cálculo de verbas rescisórias trabalhistas",

    # Modelo: GPT-4o mini — escreve e executa código Python com precisão
    model=OpenAIChat(id="gpt-4o"),

    # Ferramenta de execução de código — garante cálculo real, não estimado
    tools=[PythonTools()],

    instructions=[
        # O contexto de trabalho
        "Você receberá os dados do colaborador (salário, tempo de empresa, tipo de rescisão) e os artigos da CLT aplicáveis.",

        # Como ele deve calcular
        "SEMPRE escreva e execute código Python para calcular cada verba rescisória.",
        "Nunca faça os cálculos de cabeça ou por estimativa — use o PythonTools para executar.",

        # Quais verbas calcular em uma demissão sem justa causa
        "Para demissão sem justa causa, calcule obrigatoriamente:",
        "1. Saldo de salário (dias trabalhados no mês / 30 × salário)",
        "2. Aviso prévio proporcional (30 dias + 3 dias por ano completo, máximo 90 dias)",
        "3. Férias proporcionais + 1/3 constitucional",
        "4. Férias vencidas + 1/3 (se houver)",
        "5. 13º salário proporcional (meses trabalhados / 12 × salário)",
        "6. FGTS do período + multa de 40% sobre saldo total do FGTS",

        # Formato de entrega
        "Apresente os resultados em uma tabela Markdown com as colunas: Verba | Base de Cálculo | Valor (R$)",
        "Mostre o total geral ao final.",
        "Explique brevemente a fórmula usada para cada verba — o gestor precisa entender para validar."
    ]
)
```

> ⚠️ **Por que GPT-4o mini no Calculador?** O `PythonTools` funciona com qualquer modelo, mas o GPT-4o mini escreve código Python mais robusto e com menos erros do que os modelos menores. Para cálculos que envolvem dinheiro, vale o custo marginal.

---

### Bloco 4: Criando o Líder do Time

```
# ===============================
# LÍDER DO TIME: COORDENADOR JURÍDICO
# ===============================
coordenador_juridico = Team(
    name="Coordenador Jurídico",

    # Cargo do líder
    role="Coordenador de análise jurídica trabalhista",

    # Modelo: GPT-4o mini — coordenação e formatação do parecer final
    model=OpenAIChat(id="gpt-4o-mini"),

    # O time completo de especialistas
    members=[pesquisador_legislacao, analista_compliance, calculador_verbas],

    instructions=[
        # O script de coordenação
        "Você coordena um time de especialistas jurídicos para análise de situações trabalhistas.",
        "Siga exatamente esta sequência de trabalho:",
        "1. Acione o Pesquisador de Legislação para buscar os artigos da CLT e súmulas aplicáveis.",
        "2. Acione o Analista de Compliance para verificar riscos e obrigações com base na legislação pesquisada.",
        "3. Acione o Calculador de Verbas para apurar os valores rescisórios com precisão.",

        # O que ele deve entregar
        "Após receber as análises dos três especialistas, produza APENAS o Parecer Final.",
        "Não repita os processos intermediários. Entregue diretamente o documento técnico.",

        # Estrutura obrigatória do parecer final
        "O Parecer Final deve seguir obrigatoriamente esta estrutura:",
        "## Parecer Jurídico Trabalhista",
        "### 1. Base Legal Aplicável",
        "### 2. Análise de Conformidade e Riscos",
        "### 3. Verbas Rescisórias Apuradas",
        "### 4. Obrigações e Prazos para a Empresa",
        "### 5. Recomendação Final",

        # Tom e estilo
        "Use linguagem técnica e objetiva. O leitor é um profissional de RH ou advogado.",
        "Inclua os valores calculados e os artigos citados diretamente no parecer.",
        "Seja preciso. Cada informação deve ser rastreável à legislação pesquisada."
    ],

    markdown=True
)
```

> 📌 **Atenção à ordem dos members:** O Agno aciona os membros conforme o líder decide, guiado pelas `instructions`. A sequência descrita nas instruções é o "script de trabalho" do coordenador — ela define o fluxo de raciocínio do time.

---

### Bloco 5: A Ponte entre o Gradio e o Agno

```
# ===============================
# PROCEDIMENTO OP. PADRÃO (POP)
# ===============================
def analisar_situacao_trabalhista(descricao_caso):
    """
    Recebe a descrição da situação trabalhista, aciona o time de agentes
    e retorna o parecer técnico completo formatado.
    """
    # Validação básica
    if not descricao_caso.strip():
        return "⚠️ Por favor, descreva a situação trabalhista antes de continuar."

    # Envia para o Coordenador Jurídico (líder do time)
    # O .run() aciona toda a cadeia automaticamente
    resposta = coordenador_juridico.run(
        f"Analise esta situação trabalhista e emita um parecer técnico completo: {descricao_caso}"
    )

    return resposta.content
```

---

### Bloco 6: Interface Visual com Gradio

```
# ===============================
# INTERFACE GRADIO
# ===============================
with gr.Blocks(theme=gr.themes.Base(), title="Assistente Jurídico Trabalhista") as app_juridico:

    gr.Markdown("""
    # ⚖️ Assistente Jurídico Trabalhista
    **Descreva a situação e receba um parecer com base legal atualizada e cálculo de verbas.**

    O sistema pesquisa a legislação vigente na web, verifica conformidade e calcula os valores com precisão.
    """)

    with gr.Row():
        with gr.Column(scale=1):
            caso_input = gr.Textbox(
                lines=10,
                placeholder=(
                    "Descreva a situação trabalhista com os dados necessários para o cálculo.\n\n"
                    "Exemplo:\n"
                    "Colaborador CLT, 14 meses de empresa, salário de R$ 4.200,00, "
                    "jornada de 44h semanais. A empresa deseja realizar uma demissão "
                    "sem justa causa. Quais são as obrigações legais e o valor total das verbas rescisórias?"
                ),
                label="📋 Descrição da Situação Trabalhista"
            )
            btn_analisar = gr.Button("⚖️ Gerar Parecer Jurídico", variant="primary", size="lg")

        with gr.Column(scale=2):
            parecer_output = gr.Markdown(label="📄 Parecer Técnico Completo")

    btn_analisar.click(
        fn=analisar_situacao_trabalhista,
        inputs=[caso_input],
        outputs=[parecer_output]
    )

    gr.Markdown("---\n*Desenvolvido com Agno Framework + GPT-4o + GPT-4o mini + LLaMA 3.3 70B + DuckDuckGo Search*")

# ===============================
# INICIALIZANDO O SISTEMA
# ===============================
app_juridico.launch(
    share=True,
    debug=True
)
```

---

### Bloco 7: Testando o Sistema

```
# ===============================
# CASOS DE TESTE
# ===============================

# TESTE 1 — Demissão sem justa causa (caso completo com dados para cálculo)
"""
Colaborador CLT, admitido há 14 meses, salário de R$ 4.200,00,
jornada de 44h semanais, sem férias vencidas, FGTS recolhido regularmente.
A empresa deseja realizar demissão sem justa causa.
Quais são as obrigações legais, os prazos e o valor total das verbas rescisórias?
"""

# TESTE 2 — Afastamento por doença (conformidade sem rescisão)
"""
Colaboradora diagnosticada com tendinite, atestado médico de 20 dias.
Está há 2 anos na empresa, salário de R$ 3.800,00.
Quais são as obrigações da empresa? A partir de quando o INSS assume?
Existe risco trabalhista se a empresa não tomar nenhuma providência?
"""

# TESTE 3 — Banco de horas irregular (compliance preventivo)
"""
A empresa adota banco de horas informal, sem acordo coletivo assinado.
Os colaboradores fazem regularmente 2h extras por dia sem compensação.
Qual é o risco jurídico dessa prática? O que a CLT diz sobre isso?
"""

# TESTE 4 — Guardrail: fora do escopo trabalhista
"""
Preciso de ajuda para redigir um contrato de locação de imóvel comercial.
"""
```

**O que observar em cada teste:**

* **Teste 1:** O Pesquisador deve citar os arts. 477 e 478 da CLT e a Lei 8.036/90 (FGTS). O Calculador deve executar código Python e mostrar cada verba em tabela com o total. O Analista deve verificar prazos do eSocial e CAGED.
* **Teste 2:** O Pesquisador deve trazer o art. 60 da CLT e a Lei 8.213/91 (INSS a partir do 16º dia). O Analista deve apontar o risco de estabilidade provisória por doença ocupacional.
* **Teste 3:** O Pesquisador deve citar o art. 59 da CLT e a Súmula 85 do TST. O Analista deve classificar o risco como ALTO e listar o passivo potencial.
* **Teste 4:** O líder deve reconhecer que contratos de locação estão fora do escopo trabalhista e recusar educadamente, sem acionar nenhum membro do time.

---

## 6. O Resultado Final

O que você acabou de construir vai além de um chatbot jurídico. É um sistema que **pesquisa, analisa e calcula** — tudo automaticamente, com rastreabilidade legal.

**Por que esse projeto tem valor real de negócio?**

* **Base legal sempre atualizada:** O Pesquisador vai à web antes de cada análise. Se a lei mudou ontem, o sistema já sabe — sem necessidade de atualizar o código.
* **Cálculo auditável:** O Calculador mostra o código Python executado e a fórmula de cada verba. O gestor pode validar o resultado antes de usar.
* **Prevenção de passivo trabalhista:** O Analista de Compliance identifica riscos antes que eles virem processo — o que vale muito mais do que a hora de consultoria jurídica que substituiu.
* **Acessível para toda a empresa:** Um gestor de RH sem formação jurídica consegue rodar uma análise completa em menos de 2 minutos, sem depender da agenda do advogado.

---

## 7. Expandindo a Arquitetura: Versão 2.0 com Upload de Documentos

Assim como fizemos na Aula 1 com o Co-piloto de RH, você pode evoluir o Assistente Jurídico para aceitar documentos reais — contratos, convenções coletivas, recibos de pagamento — usando `pdfplumber` para extrair o texto e enviá-lo ao time junto com a descrição do caso.

A estrutura de time permanece exatamente a mesma. O que muda é a função de entrada:

```
# Estrutura conceitual da versão 2.0
import pdfplumber

def analisar_com_documento(arquivo, descricao_caso):
    """
    Extrai o texto do contrato ou convenção coletiva enviado
    e enriquece o contexto do time antes da análise.
    """
    texto_documento = ""

    if arquivo is not None:
        with pdfplumber.open(arquivo.name) as pdf:
            for pagina in pdf.pages:
                texto_documento += pagina.extract_text() or ""

    # Contexto enriquecido: caso + documento
    contexto_completo = f"""
    SITUAÇÃO TRABALHISTA:
    {descricao_caso}

    DOCUMENTO FORNECIDO (contrato, convenção coletiva ou recibo):
    {texto_documento}
    """

    resposta = coordenador_juridico.run(contexto_completo)
    return resposta.content
```

> 💡 **Seu próximo passo:** Teste o sistema com um caso real do seu trabalho. Se você é da área jurídica, cole uma situação de rescisão recente. Se é de RH, descreva um afastamento ou banco de horas que precise de análise.

---

## 8. Próxima Aula

Na **Aula 3**, vamos dar ao time a capacidade de **ler e analisar documentos enviados pelo usuário** — contratos, briefings, propostas comerciais — usando RAG (Retrieval-Augmented Generation) com banco de dados vetorial.

O projeto será um **Analista de Contratos e Briefings de Marketing** — um time que lê o documento, identifica cláusulas de risco, extrai os requisitos da campanha e gera um plano de ação baseado no conteúdo real do arquivo.

---

## Gabarito: Código Completo do Assistente Jurídico Trabalhista

```
# ===============================
# INSTALAÇÃO DAS DEPENDÊNCIAS
# ===============================
!pip install -U agno groq openai duckduckgo-search gradio
```

```
# ===============================
# FERRAMENTAS
# ===============================
import os
import gradio as gr
from google.colab import userdata

from agno.agent import Agent
from agno.team import Team
from agno.models.groq import Groq
from agno.models.openai import OpenAIChat
from agno.tools.duckduckgo import DuckDuckGoTools
from agno.tools.python import PythonTools

# ===============================
# AUTENTICAÇÃO
# ===============================
os.environ["GROQ_API_KEY"]   = userdata.get('GROQ_API_KEY')
os.environ["OPENAI_API_KEY"] = userdata.get('OPENAI_API_KEY')

# ===============================
# AGENTE 1: PESQUISADOR DE LEGISLAÇÃO
# ===============================
pesquisador_legislacao = Agent(
    name="Pesquisador de Legislação",
    role="Especialista em pesquisa de legislação trabalhista e jurisprudência",
    model=Groq(id="llama-3.3-70b-versatile"),
    tools=[DuckDuckGoTools()],
    instructions=[
        "Você receberá a descrição de uma situação trabalhista.",
        "SEMPRE pesquise na web antes de responder. Nunca responda apenas de memória.",
        "Priorize as seguintes fontes oficiais:",
        "- planalto.gov.br (texto atualizado da CLT)",
        "- tst.jus.br (súmulas e jurisprudência do TST)",
        "- esocial.gov.br (obrigações acessórias)",
        "- jusbrasil.com.br (jurisprudência consolidada)",
        "Retorne uma lista estruturada com:",
        "- Os artigos da CLT aplicáveis ao caso (número + ementa resumida)",
        "- Súmulas do TST relevantes (número + enunciado)",
        "- Qualquer norma complementar identificada",
        "Foque exclusivamente no embasamento legal.",
        "Não faça análise de conformidade nem cálculos — isso é papel dos seus colegas de time.",
        "Se não encontrar uma fonte confiável para um ponto específico, informe explicitamente."
    ]
)

# ===============================
# AGENTE 2: ANALISTA DE COMPLIANCE
# ===============================
analista_compliance = Agent(
    name="Analista de Compliance",
    role="Especialista em análise de conformidade trabalhista e gestão de riscos jurídicos",
    model=Groq(id="llama-3.3-70b-versatile"),
    instructions=[
        "Você receberá a descrição da situação trabalhista e a pesquisa de legislação feita pelo seu colega.",
        "Com base na legislação pesquisada, verifique:",
        "1. A empresa está cumprindo os prazos obrigatórios da CLT para este tipo de situação?",
        "2. Existe risco de configurar dispensa discriminatória ou nulidade do ato?",
        "3. Há obrigações acessórias que a empresa precisa cumprir (eSocial, CAGED, homologação)?",
        "4. Existem precedentes jurisprudenciais que aumentam o risco de passivo trabalhista?",
        "Para cada ponto verificado, classifique o risco como ALTO, MÉDIO ou BAIXO com justificativa.",
        "Liste as obrigações legais que a empresa deve cumprir, com prazo quando aplicável.",
        "Use linguagem direta e executiva."
    ]
)

# ===============================
# AGENTE 3: CALCULADOR DE VERBAS
# ===============================
calculador_verbas = Agent(
    name="Calculador de Verbas",
    role="Especialista em cálculo de verbas rescisórias trabalhistas",
    model=OpenAIChat(id="gpt-4o"),
    tools=[PythonTools()],
    instructions=[
        "Você receberá os dados do colaborador e os artigos da CLT aplicáveis.",
        "SEMPRE escreva e execute código Python para calcular cada verba rescisória.",
        "Nunca faça os cálculos de cabeça ou por estimativa — use o PythonTools para executar.",
        "Para demissão sem justa causa, calcule obrigatoriamente:",
        "1. Saldo de salário (dias trabalhados no mês / 30 × salário)",
        "2. Aviso prévio proporcional (30 dias + 3 dias por ano completo, máximo 90 dias)",
        "3. Férias proporcionais + 1/3 constitucional",
        "4. Férias vencidas + 1/3 (se houver)",
        "5. 13º salário proporcional (meses trabalhados / 12 × salário)",
        "6. FGTS do período + multa de 40% sobre saldo total do FGTS",
        "Apresente os resultados em uma tabela Markdown com as colunas: Verba | Base de Cálculo | Valor (R$)",
        "Mostre o total geral ao final.",
        "Explique brevemente a fórmula usada para cada verba."
    ]
)

# ===============================
# LÍDER DO TIME: COORDENADOR JURÍDICO
# ===============================
coordenador_juridico = Team(
    name="Coordenador Jurídico",
    role="Coordenador de análise jurídica trabalhista",
    model=OpenAIChat(id="gpt-4o-mini"),
    members=[pesquisador_legislacao, analista_compliance, calculador_verbas],
    instructions=[
        "Você coordena um time de especialistas jurídicos para análise de situações trabalhistas.",
        "Siga exatamente esta sequência de trabalho:",
        "1. Acione o Pesquisador de Legislação para buscar os artigos da CLT e súmulas aplicáveis.",
        "2. Acione o Analista de Compliance para verificar riscos e obrigações com base na legislação pesquisada.",
        "3. Acione o Calculador de Verbas para apurar os valores rescisórios com precisão.",
        "Após receber as análises dos três especialistas, produza APENAS o Parecer Final.",
        "Não repita os processos intermediários. Entregue diretamente o documento técnico.",
        "O Parecer Final deve seguir obrigatoriamente esta estrutura:",
        "## Parecer Jurídico Trabalhista",
        "### 1. Base Legal Aplicável",
        "### 2. Análise de Conformidade e Riscos",
        "### 3. Verbas Rescisórias Apuradas",
        "### 4. Obrigações e Prazos para a Empresa",
        "### 5. Recomendação Final",
        "Use linguagem técnica e objetiva.",
        "Inclua os valores calculados e os artigos citados diretamente no parecer.",
        "Seja preciso. Cada informação deve ser rastreável à legislação pesquisada."
    ],
    markdown=True
)

# ===============================
# PROCEDIMENTO OP. PADRÃO (POP)
# ===============================
def analisar_situacao_trabalhista(descricao_caso):

    if not descricao_caso.strip():
        return "⚠️ Por favor, descreva a situação trabalhista antes de continuar."

    resposta = coordenador_juridico.run(
        f"Analise esta situação trabalhista e emita um parecer técnico completo: {descricao_caso}"
    )
    return resposta.content

# ===============================
# INTERFACE GRADIO
# ===============================
with gr.Blocks(theme=gr.themes.Base(), title="Assistente Jurídico Trabalhista") as app_juridico:

    gr.Markdown("""
    # ⚖️ Assistente Jurídico Trabalhista
    **Descreva a situação e receba um parecer com base legal atualizada e cálculo de verbas.**
    """)

    with gr.Row():
        with gr.Column(scale=1):
            caso_input = gr.Textbox(
                lines=10,
                placeholder=(
                    "Descreva a situação trabalhista com os dados necessários.\n\n"
                    "Exemplo:\n"
                    "Colaborador CLT, 14 meses de empresa, salário de R$ 4.200,00, "
                    "44h semanais. A empresa deseja realizar demissão sem justa causa. "
                    "Quais são as obrigações legais e o valor total das verbas rescisórias?"
                ),
                label="📋 Descrição da Situação Trabalhista"
            )
            btn_analisar = gr.Button("⚖️ Gerar Parecer Jurídico", variant="primary", size="lg")

        with gr.Column(scale=2):
            parecer_output = gr.Markdown(label="📄 Parecer Técnico Completo")

    btn_analisar.click(
        fn=analisar_situacao_trabalhista,
        inputs=[caso_input],
        outputs=[parecer_output]
    )

    gr.Markdown("---\n*Desenvolvido com Agno Framework + GPT-4o mini + LLaMA 3.3 70B + DuckDuckGo Search*")

# ===============================
# INICIALIZANDO O SISTEMA
# ===============================
app_juridico.launch(
    share=True,
    debug=True
)
```

```
# ===============================
# CASOS DE TESTE
# ===============================

# TESTE 1 — Demissão sem justa causa (caso completo com dados para cálculo)
# "Colaborador CLT, admitido há 14 meses, salário de R$ 4.200,00,
#  jornada de 44h semanais, sem férias vencidas, FGTS recolhido regularmente.
#  A empresa deseja realizar demissão sem justa causa.
#  Quais são as obrigações legais, os prazos e o valor total das verbas rescisórias?"

# TESTE 2 — Afastamento por doença (conformidade sem rescisão)
# "Colaboradora diagnosticada com tendinite, atestado de 20 dias.
#  Está há 2 anos na empresa, salário de R$ 3.800,00.
#  Quais são as obrigações da empresa? A partir de quando o INSS assume?"

# TESTE 3 — Banco de horas irregular (compliance preventivo)
# "A empresa adota banco de horas informal, sem acordo coletivo assinado.
#  Colaboradores fazem 2h extras por dia sem compensação.
#  Qual é o risco jurídico? O que a CLT diz sobre isso?"

# TESTE 4 — Guardrail: fora do escopo trabalhista
# "Preciso de ajuda para redigir um contrato de locação de imóvel comercial."
```
