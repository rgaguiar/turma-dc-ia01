# Aula Prática: Times de Agentes com RAG — O Time que Lê, Ouve e Compara

**Módulo:** 4 — Times de Agentes  
**Aula:** 13 (Aula 3 do módulo de Times)  
**Última revisão:** 2025-05  

---

Nas duas aulas anteriores, você construiu times que pensam e agem. O Co-piloto de RH leu anotações e gerou diagnósticos. O Assistente Jurídico pesquisou legislação e calculou verbas. Os dois recebem texto puro como entrada.

Mas a realidade do trabalho é diferente. Reuniões são gravadas em áudio. Templates de relatório ficam em PDF. Ninguém para uma reunião para digitar anotações estruturadas — as pessoas falam, e depois precisam de um registro.

Nesta aula, vamos resolver isso. Vamos ensinar o time a **ouvir** e a **ler documentos** — e a usar um template como referência para auditar o que aconteceu em uma reunião.

O projeto do dia é o **Auditor de Reuniões** — um time que recebe a gravação ou a transcrição de uma reunião, compara com o template de como ela deveria ter sido conduzida, e entrega um diagnóstico executivo com pontos fortes, pontos fracos e lacunas não abordadas.

---
<img width="1188" height="1346" alt="image" src="https://github.com/user-attachments/assets/481f0f15-453c-46aa-a5c5-9198c914368c" />


## 1. Os Dois Conceitos Novos desta Aula

### RAG: O Agente que Lê Documentos de Referência

Nas aulas anteriores, todo o contexto vinha do usuário — ele colava o texto, o agente processava. O problema é que contextos importantes ficam em documentos externos: o template de relatório da empresa, a política de reuniões, o roteiro de feedback estruturado.

RAG (Retrieval-Augmented Generation) é a técnica que permite ao agente **consultar um documento de referência antes de responder**. Em vez de depender só do que foi treinado, ele vai ao documento, recupera as partes relevantes e usa isso para ancorar a análise.

A analogia é simples: é a diferença entre um consultor que responde de memória e um consultor que abre o manual da empresa antes de falar.

| Sem RAG | Com RAG |
| --- | --- |
| Agente usa só o que foi treinado | Agente consulta o documento fornecido |
| Resposta genérica, sem ancoragem na realidade da empresa | Resposta baseada no template e nas políticas reais |
| Template fixo no código — difícil de atualizar | Template externo — o usuário atualiza sem mexer no código |

### Whisper: O Agente que Ouve

O Whisper é o modelo de transcrição de áudio da OpenAI. Ele recebe um arquivo de áudio (MP3, MP4, WAV, M4A) e devolve o texto transcrito com alta precisão, inclusive em português.

Nesta aula usamos o **`whisper-1` via API da OpenAI** — a mesma chave que você já usa desde a Aula 1. Sem instalar nada novo, sem configurar GPU, sem esperar modelo carregar. O arquivo é enviado para os servidores da OpenAI e o texto transcrito volta em segundos.

No nosso projeto, o Whisper é a primeira etapa do pipeline: se o usuário enviar um áudio, ele transcreve antes de qualquer análise. Se o usuário enviar texto direto, o Whisper é pulado.

```
[Áudio da Reunião]  →  whisper-1 API OpenAI  →  [Transcrição em Texto]  →  Time de Agentes
[Texto da Reunião]  →  (pula o Whisper)  →  Time de Agentes
```

> 💡 **Por que a API OpenAI e não o modelo HuggingFace?** Para arquivos de até 25MB (equivalente a ~30 min de áudio), a API é a opção mais simples e rápida — ~30 a 60 segundos de espera, sem GPU, sem dependência extra. O custo é ~$0.006 por minuto: uma reunião de 20 minutos custa ~$0.12, menos que R$ 1,00.

> 📌 **Alternativa gratuita:** para quem não quer custo de API, o modelo `jlondonobo/whisper-medium-pt` (HuggingFace, fine-tuned para PT-BR) é uma excelente opção — mas requer GPU T4 ativada no Colab e ~5 minutos para carregar na primeira vez.

---

## 2. A Arquitetura do Projeto

### O Problema que Vamos Resolver

Toda empresa tem um jeito certo de conduzir reuniões — mas raramente verifica se esse jeito está sendo seguido. Um One-on-One de RH deveria cobrir desenvolvimento, metas e bem-estar. Uma reunião de vendas deveria cobrir pipeline, objeções e próximos passos. Uma reunião de vistoria de sinistro deveria cobrir todos os campos do laudo.

Na prática, as reuniões acontecem, as gravações ficam na nuvem e ninguém audita se os pontos obrigatórios foram abordados.

O Auditor de Reuniões resolve isso: ele lê (ou ouve) a reunião, consulta o template do que deveria ter sido coberto, e entrega um diagnóstico em segundos.

### O Time do Auditor de Reuniões

```
[Usuário]
    │
    ├── Envia ÁUDIO da reunião  →  [Whisper]  →  Transcrição em texto
    │
    ├── Envia TEXTO da reunião  →  (direto para o time)
    │
    └── Envia TEMPLATE (PDF)  →  pdfplumber  →  Texto do template
        OU usa o template padrão fixo no código
                │
                ▼
[Coordenador de Auditoria]  ← Líder do Time (GPT-4o mini)
                │
                ├──▶ [Transcritor / Organizador]    → Estrutura a reunião em tópicos        (Llama 3 via Groq)
                │
                ├──▶ [Auditor de Conformidade]      → Compara com o template e pontua       (GPT-4o mini + RAG)
                │
                └──▶ [Gerador de Diagnóstico]       → Produz pontos fortes, fracos e gaps   (Llama 3 via Groq)
                                │
                                ▼
                    [Relatório de Auditoria em Markdown]
```

### Decisão de Modelos

| Agente | Modelo | Justificativa |
| --- | --- | --- |
| Transcritor / Organizador | `Groq / Llama 3` | Estruturação de texto — tarefa leve, gratuita |
| Auditor de Conformidade | `GPT-4o mini` | Precisa raciocinar sobre dois documentos simultaneamente |
| Gerador de Diagnóstico | `Groq / Llama 3` | Síntese estruturada — rápida e gratuita |
| Coordenador (Líder) | `GPT-4o mini` | Coordenação e formatação do relatório final |

> 💡 **Por que o Auditor de Conformidade usa GPT-4o mini?** Ele precisa comparar a transcrição organizada com o template e raciocinar sobre o que está presente, o que está ausente e o que está parcialmente coberto. Esse nível de raciocínio relacional é onde o GPT-4o mini se destaca em relação ao Llama 3.

---

## 3. Pré-requisitos do Projeto

Mesmas credenciais das aulas anteriores — mais uma biblioteca nova:

| Secret no Colab | Plataforma | Custo |
| --- | --- | --- |
| `GROQ_API_KEY` | [console.groq.com](https://console.groq.com) | Gratuito |
| `OPENAI_API_KEY` | [platform.openai.com](https://platform.openai.com) | ~R$ 0,20 por auditoria completa |

**Biblioteca nova: pdfplumber**

Usada para extrair o texto de templates em PDF enviados pelo usuário. Sem necessidade de chave de API — é uma biblioteca Python pura.

> 📌 **Prepare para a aula:** grave um áudio de 2-3 minutos simulando uma reunião — pode ser você mesmo falando sobre qualquer assunto profissional. Formatos aceitos: MP3, MP4, WAV ou M4A.

---

## 4. Construindo o Projeto Bloco a Bloco

### Bloco 1: Instalando as Dependências

```python
# ===============================
# INSTALAÇÃO DAS DEPENDÊNCIAS
# ===============================
!pip install -q -U agno groq openai ddgs gradio pdfplumber
```

**O que é novo em relação à Aula 2:**

| Pacote | Função |
| --- | --- |
| `pdfplumber` | Extrai texto de arquivos PDF (o template de reunião do usuário) |
| Os demais | Já conhecidos das aulas anteriores |

> 📌 **Sem novidades de infraestrutura:** a transcrição de áudio usa a mesma `OPENAI_API_KEY` que você já configurou na Aula 1. Nenhuma GPU, nenhum modelo para baixar.

---

### Bloco 2: Importando as Ferramentas e Autenticando

```python
# ===============================
# FERRAMENTAS
# ===============================
import os
import gradio as gr
import pdfplumber
from openai import OpenAI
from google.colab import userdata

from agno.agent import Agent
from agno.team import Team
from agno.models.groq import Groq
from agno.models.openai import OpenAIChat

# ===============================
# AUTENTICAÇÃO
# ===============================
os.environ["GROQ_API_KEY"]   = userdata.get('GROQ_API_KEY')
os.environ["OPENAI_API_KEY"] = userdata.get('OPENAI_API_KEY')

# Cliente OpenAI — usado tanto para transcrição quanto para os agentes
cliente_openai = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

print("✅ Autenticação concluída.")
```

> 📌 **Uma chave, dois usos:** o mesmo `OPENAI_API_KEY` autentica tanto a transcrição de áudio (Whisper) quanto os agentes GPT-4o mini. Nenhuma configuração extra necessária.

---

### Bloco 3: O Template Padrão de Reunião

Este é o coração do RAG desta aula. O template define **o que uma boa reunião deve cobrir**. Ele é o documento de referência contra o qual a reunião real será auditada.

O sistema aceita dois tipos de template:

1. **Template padrão fixo** — definido aqui no código, serve para qualquer setor
2. **Template personalizado** — o usuário faz upload de um PDF com o template da própria empresa

```python
# ===============================
# TEMPLATE PADRÃO DE REUNIÃO
# ===============================

# Este template será usado quando o usuário não enviar um PDF personalizado.
# Ele representa os tópicos que uma reunião executiva bem conduzida deve cobrir.
# O professor (ou o aluno na personalização) pode editar este texto livremente.

TEMPLATE_PADRAO = """
# Template de Reunião Executiva

## 1. Abertura e Alinhamento de Pauta
- Confirmar os participantes e seus papéis
- Revisar a pauta e o objetivo da reunião
- Definir o tempo disponível

## 2. Revisão de Pendências Anteriores
- Status de ações da reunião anterior
- O que foi concluído, o que está em andamento, o que foi bloqueado

## 3. Pauta Principal
- Apresentação do tema central
- Discussão e contribuições dos participantes
- Dados ou evidências apresentados

## 4. Tomada de Decisão
- Decisões tomadas durante a reunião
- Responsáveis por cada decisão
- Critérios usados para decidir

## 5. Plano de Ação
- Ações definidas (o quê)
- Responsáveis (quem)
- Prazos (quando)

## 6. Encerramento
- Resumo das decisões e ações
- Data e pauta da próxima reunião
- Avaliação da efetividade da reunião pelos participantes
"""

print("Template padrão carregado.")
print(f"Tópicos obrigatórios: {TEMPLATE_PADRAO.count('##')} seções principais")
```

> 💡 **Por que o template fica no código?** Para que o professor possa adaptá-lo ao vivo durante a personalização guiada. O aluno de RH muda para o template de One-on-One. O aluno de vendas muda para o template de pipeline. O código não muda — só o texto do template.

---

### Bloco 4: As Funções de Entrada (Texto, Áudio e PDF)

Este bloco cria as três funções que processam os insumos antes de chegarem ao time de agentes.

```python
# ===============================
# FUNÇÃO 1: TRANSCRIÇÃO DE ÁUDIO
# ===============================
def transcrever_audio(arquivo_audio):
    """
    Recebe um arquivo de áudio e usa o whisper-1 (API OpenAI)
    para transcrever o conteúdo em português.

    Formatos aceitos: MP3, MP4, WAV, M4A, WEBM
    Limite: 25MB por arquivo (~30 min de áudio em MP3 típico)
    Custo: ~$0.006 por minuto de áudio
    """
    with open(arquivo_audio, "rb") as f:
        transcricao = cliente_openai.audio.transcriptions.create(
            model="whisper-1",
            file=f,
            language="pt"    # força português — melhora a precisão
        )
    return transcricao.text


# ===============================
# FUNÇÃO 2: EXTRAÇÃO DE TEXTO DO TEMPLATE (PDF)
# ===============================
def extrair_texto_pdf(arquivo_pdf):
    """
    Recebe um arquivo PDF (o template de reunião da empresa)
    e extrai o texto usando pdfplumber.

    Retorna o template padrão se nenhum PDF for enviado.
    """
    if arquivo_pdf is None:
        return TEMPLATE_PADRAO

    texto_extraido = ""
    with pdfplumber.open(arquivo_pdf) as pdf:
        for pagina in pdf.pages:
            texto_extraido += pagina.extract_text() or ""

    # Fallback: se o PDF estiver vazio ou corrompido, usa o template padrão
    if not texto_extraido.strip():
        print("⚠️ PDF vazio ou ilegível — usando template padrão.")
        return TEMPLATE_PADRAO

    return texto_extraido


# ===============================
# FUNÇÃO 3: PREPARAR O CONTEÚDO DA REUNIÃO
# ===============================
def preparar_conteudo_reuniao(texto_digitado, arquivo_audio):
    """
    Decide qual fonte usar como conteúdo da reunião:
    - Se o usuário enviou um áudio → transcreve com Whisper
    - Se o usuário digitou texto → usa o texto diretamente
    - Se os dois foram enviados → o áudio tem prioridade
    """
    if arquivo_audio is not None:
        print("🎙️ Áudio detectado — iniciando transcrição com Whisper...")
        conteudo = transcrever_audio(arquivo_audio)
        print(f"✅ Transcrição concluída: {len(conteudo.split())} palavras")
        return conteudo

    if texto_digitado and texto_digitado.strip():
        return texto_digitado

    return None
```

> 💡 **Por que `language="pt"`?** Sem esse parâmetro, o Whisper detecta o idioma automaticamente — o que funciona bem, mas em reuniões com muitos termos técnicos ou sotaques regionais, forçar o português melhora a precisão.

> ⚠️ **Por que o áudio tem prioridade sobre o texto?** Em reuniões reais, o áudio é a fonte mais completa e fidedigna. Se o usuário enviou os dois, provavelmente o texto é um rascunho e o áudio é o registro oficial.

> 📌 **Alternativa gratuita para arquivos grandes:** se o áudio passar de 25MB ou você quiser evitar custo de API, use o `faster-whisper` (SYSTRAN) com o modelo `medium` — processa áudios de qualquer tamanho na GPU do Colab em ~2–4 minutos.

---

### Bloco 5: Criando os Agentes Membros do Time

#### Agente 1 — Transcritor e Organizador

Este agente recebe o conteúdo bruto da reunião (que pode ser uma transcrição de áudio, cheia de vícios de linguagem, repetições e frases incompletas) e o organiza em tópicos limpos e estruturados.

```python
# ===============================
# AGENTE 1: TRANSCRITOR E ORGANIZADOR
# ===============================
transcritor_organizador = Agent(
    name="Transcritor e Organizador",

    # Cargo do agente dentro do time
    role="Especialista em organização e estruturação de conteúdo de reuniões",

    # Groq com Llama 3 — tarefa de limpeza e estruturação de texto
    model=Groq(id="llama-3.3-70b-versatile"),

    instructions=[
        # O contexto de trabalho
        "Você receberá o conteúdo bruto de uma reunião — pode ser uma transcrição de áudio ou texto digitado.",

        # O que ele deve fazer
        "Leia o conteúdo e organize-o em tópicos claros, eliminando:",
        "- Repetições e vícios de linguagem ('né', 'tipo', 'então', 'hum')",
        "- Interrupções e falas incompletas",
        "- Conversas paralelas fora do tema principal",

        # O que ele deve entregar
        "Retorne a reunião organizada com a seguinte estrutura:",
        "- **Participantes identificados** (se mencionados)",
        "- **Tópicos discutidos** (em ordem cronológica)",
        "- **Decisões mencionadas** (se houver)",
        "- **Ações citadas** (se houver)",

        # Limite de escopo
        "Não avalie a qualidade da reunião — apenas organize o conteúdo.",
        "Mantenha fidelidade ao que foi dito. Não invente informações ausentes."
    ]
)
```

---

#### Agente 2 — Auditor de Conformidade

Este é o agente de RAG. Ele recebe a reunião organizada e o template, compara os dois sistematicamente e calcula o **Score de Qualidade** diretamente nas instruções — sem arquivos, sem ambiguidade.

> 🔥 **Lição Aprendida em Sala:** tentamos usar `PythonTools` aqui para calcular o score. O que aconteceu: o modelo tentou salvar o conteúdo da reunião em arquivos temporários (`/content/reuniao.txt`) que não existem no Colab — e travou em loop de `FileNotFoundError`. Use `PythonTools` apenas quando o dado vem de uma fonte externa real. Quando o dado já está no contexto da mensagem, instrução direta é mais confiável e mais simples.

```python
# ===============================
# AGENTE 2: AUDITOR DE CONFORMIDADE
# ===============================
auditor_conformidade = Agent(
    name="Auditor de Conformidade",

    # Cargo do agente
    role="Especialista em auditoria de qualidade de reuniões executivas",

    # GPT-4o mini — raciocínio sobre dois documentos simultâneos
    model=OpenAIChat(id="gpt-4o-mini"),

    # Sem PythonTools: score calculado diretamente pela instrução matemática
    instructions=[
        # O contexto de trabalho — este agente recebe dois documentos
        "Você receberá dois documentos:",
        "DOCUMENTO 1: A reunião organizada pelo seu colega Transcritor",
        "DOCUMENTO 2: O template de como a reunião deveria ter sido conduzida",

        # O que ele deve fazer — a comparação sistemática (o núcleo do RAG)
        "Para cada seção do TEMPLATE, verifique se a reunião real a cobriu:",
        "- COBERTO: o tópico foi claramente discutido na reunião",
        "- PARCIAL: o tópico foi mencionado mas não desenvolvido adequadamente",
        "- AUSENTE: o tópico não foi abordado em nenhum momento",

        # O que ele deve entregar — tabela de conformidade
        "Retorne uma tabela Markdown com as colunas:",
        "Seção do Template | Status | Evidência (trecho da reunião ou 'não encontrado')",

        # Cálculo do score — instrução matemática direta, sem arquivos
        "Após a tabela, conte as seções por status e calcule o Score de Qualidade:",
        "score = (cobertas * 2 + parciais * 1) / (total_secoes * 2) * 100",
        "Classifique: 80-100 = Excelente | 60-79 = Boa | 40-59 = Regular | 0-39 = Crítica",
        "Apresente como: Score de Qualidade: XX/100 — Classificação",

        # Tom e estilo
        "Seja objetivo e baseado em evidências. Cite trechos reais da reunião.",
        "Não julgue intenções — apenas registre o que foi ou não coberto."
    ]
)
```

> 💡 **Este é o RAG desta aula.** O Auditor de Conformidade recebe o template como documento de referência no contexto da mensagem e compara com a reunião real — o agente analisa dois documentos simultaneamente e produz um diagnóstico baseado na diferença entre eles.

---

#### Agente 3 — Gerador de Diagnóstico

Este agente recebe a tabela de conformidade e transforma em um diagnóstico executivo — a linguagem que um gestor espera receber.

```python
# ===============================
# AGENTE 3: GERADOR DE DIAGNÓSTICO
# ===============================
gerador_diagnostico = Agent(
    name="Gerador de Diagnóstico",

    # Cargo do agente
    role="Especialista em comunicação executiva e feedback construtivo",

    # Groq com Llama 3 — síntese estruturada, rápida e gratuita
    model=Groq(id="llama-3.3-70b-versatile"),

    instructions=[
        # O contexto de trabalho
        "Você receberá a tabela de conformidade produzida pelo Auditor.",

        # O que ele deve produzir
        "Com base nessa tabela, gere um diagnóstico executivo com as seguintes seções:",

        "### ✅ Pontos Fortes",
        "Liste os tópicos que foram bem cobertos na reunião.",
        "Para cada ponto, explique brevemente por que isso é positivo.",

        "### ⚠️ Pontos de Atenção",
        "Liste os tópicos cobertos apenas parcialmente.",
        "Para cada um, sugira como aprofundar na próxima reunião.",

        "### ❌ Lacunas Críticas",
        "Liste os tópicos completamente ausentes.",
        "Para cada lacuna, explique o risco de não abordar esse ponto.",

        "### 📋 Recomendações para a Próxima Reunião",
        "Liste 3 ações concretas para melhorar a condução da próxima reunião.",

        # Tom e estilo
        "Use linguagem executiva, direta e construtiva.",
        "O gestor que vai ler esse relatório precisa de clareza, não de julgamento.",
        "Máximo de 3 itens por seção — priorize os mais impactantes."
    ]
)
```

---

### Bloco 6: Criando o Líder do Time

```python
# ===============================
# LÍDER DO TIME: COORDENADOR DE AUDITORIA
# ===============================
coordenador_auditoria = Team(
    name="Coordenador de Auditoria",

    # Cargo do líder
    role="Coordenador de auditoria de qualidade de reuniões",

    # GPT-4o mini — coordenação e formatação do relatório final
    model=OpenAIChat(id="gpt-4o-mini"),

    # O time completo
    members=[transcritor_organizador, auditor_conformidade, gerador_diagnostico],

    instructions=[
        # O script de coordenação
        "Você coordena um time de especialistas para auditar a qualidade de reuniões.",
        "Você receberá dois insumos: o conteúdo da reunião e o template de referência.",
        "Siga exatamente esta sequência de trabalho:",
        "1. Envie o conteúdo da reunião ao Transcritor e Organizador para estruturar.",
        "2. Envie a reunião organizada E o template ao Auditor de Conformidade para comparar.",
        "3. Envie a tabela de conformidade ao Gerador de Diagnóstico para produzir o relatório.",

        # O que ele deve entregar
        "Após receber o diagnóstico do Gerador, produza APENAS o Relatório Final.",
        "Não repita os processos intermediários. Entregue diretamente o documento executivo.",

        # Estrutura obrigatória do relatório final
        "O Relatório Final deve seguir obrigatoriamente esta estrutura:",
        "## Relatório de Auditoria de Reunião",
        "### Score de Qualidade (calculado pelo Auditor)",
        "### Resumo Executivo (3 linhas)",
        "### Conformidade com o Template (tabela do Auditor)",
        "### Diagnóstico Detalhado (seções do Gerador)",
        "### Próximos Passos",

        # Tom e estilo
        "Use linguagem objetiva e construtiva.",
        "O relatório deve ser útil para quem conduziu a reunião, não punitivo."
    ],

    markdown=True
)
```

---

### Bloco 7: A Ponte entre o Gradio e o Agno

```python
# ===============================
# PROCEDIMENTO OP. PADRÃO (POP)
# ===============================
def auditar_reuniao(texto_digitado, arquivo_audio, arquivo_template_pdf):
    """
    Recebe os insumos do usuário (texto ou áudio + template opcional),
    prepara o conteúdo e aciona o time de auditoria.
    """
    # ── Etapa 1: preparar o conteúdo da reunião ──────────────────────────────
    conteudo_reuniao = preparar_conteudo_reuniao(texto_digitado, arquivo_audio)

    if not conteudo_reuniao:
        return "⚠️ Por favor, insira o texto da reunião ou faça upload de um arquivo de áudio."

    # ── Etapa 2: preparar o template de referência ───────────────────────────
    template = extrair_texto_pdf(arquivo_template_pdf)

    # ── Etapa 3: montar o contexto completo para o time ──────────────────────
    contexto = f"""
CONTEÚDO DA REUNIÃO:
{conteudo_reuniao}

---

TEMPLATE DE REFERÊNCIA (como a reunião deveria ter sido conduzida):
{template}
"""

    # ── Etapa 4: acionar o time de auditoria ─────────────────────────────────
    resposta = coordenador_auditoria.run(
        f"Audite esta reunião com base no template fornecido: {contexto}"
    )

    return resposta.content
```

---

### Bloco 8: Interface Visual com Gradio

```python
# ===============================
# INTERFACE GRADIO
# ===============================
with gr.Blocks(theme=gr.themes.Base(), title="Auditor de Reuniões") as app_auditor:

    gr.Markdown("""
    # Auditor de Reuniões
    **Envie a gravação ou transcrição da sua reunião e receba um diagnóstico completo de qualidade.**

    O sistema compara a reunião com o template de como ela deveria ter sido conduzida
    e identifica pontos fortes, lacunas e recomendações.
    """)

    with gr.Row():

        # ── Coluna de entrada ─────────────────────────────────────────────────
        with gr.Column(scale=1):

            gr.Markdown("### 1. Conteúdo da Reunião")
            gr.Markdown("*Escolha uma das opções abaixo — áudio tem prioridade sobre texto.*")

            audio_input = gr.Audio(
                sources=["upload", "microphone"],
                type="filepath",
                label="🎙️ Upload de áudio da reunião (MP3, MP4, WAV, M4A)"
            )

            texto_input = gr.Textbox(
                lines=8,
                placeholder=(
                    "Ou cole aqui a transcrição ou anotações da reunião...\n\n"
                    "Exemplo:\n"
                    "Reunião de segunda-feira com o time de produto. João apresentou o "
                    "novo roadmap. Maria questionou os prazos do Q3. Decidimos adiar o "
                    "lançamento para outubro. Próximos passos ficaram em aberto."
                ),
                label="📝 Texto da reunião (se não tiver áudio)"
            )

            gr.Markdown("### 2. Template de Referência")
            gr.Markdown("*Opcional — se não enviar, usamos o template padrão.*")

            pdf_input = gr.File(
                file_types=[".pdf"],
                label="📄 Upload do template da sua empresa (PDF) — opcional"
            )

            btn_auditar = gr.Button(
                "🔍 Auditar Reunião",
                variant="primary",
                size="lg"
            )

        # ── Coluna de saída ───────────────────────────────────────────────────
        with gr.Column(scale=2):
            relatorio_output = gr.Markdown(
                label="📊 Relatório de Auditoria"
            )

    btn_auditar.click(
        fn=auditar_reuniao,
        inputs=[texto_input, audio_input, pdf_input],
        outputs=[relatorio_output]
    )

    gr.Markdown("""
    ---
    *Desenvolvido com Agno Framework + GPT-4o mini + LLaMA 3.3 70B + Whisper*
    """)

# ===============================
# INICIALIZANDO O SISTEMA
# ===============================
app_auditor.launch(
    share=True,
    debug=True
)
```

---

### Bloco 9: Testando o Sistema

```python
# ===============================
# CASOS DE TESTE
# ===============================

# TESTE 1 — Reunião bem conduzida (poucos gaps)
"""
[Texto para colar no campo de entrada]

Reunião de planejamento trimestral — participantes: Ana (gerente), Bruno (produto),
Carla (design). Objetivo: definir prioridades do Q3.
Revisamos as pendências do Q2: 3 de 5 ações concluídas, 2 em andamento.
Bruno apresentou o roadmap com dados de NPS e churn. Discutimos as 3 principais
funcionalidades a priorizar. Decidimos focar em onboarding (Ana responsável),
melhorias de performance (Bruno) e redesign do dashboard (Carla).
Prazos: onboarding até 15/07, performance até 30/07, dashboard até 10/08.
Próxima reunião: segunda-feira seguinte. A reunião foi avaliada como produtiva
pelos participantes.
"""
# O que esperar: maioria das seções COBERTO, diagnóstico com poucos gaps

# TESTE 2 — Reunião com lacunas graves
"""
[Texto para colar no campo de entrada]

Reunião rápida antes do almoço. O pessoal discutiu o projeto novo, falaram de
alguns problemas com o cliente. Ficou meio indefinido o que cada um vai fazer.
Não tinha pauta clara. Saímos sem grandes conclusões. Foi mais um alinhamento
mesmo. Durou uns 40 minutos.
"""
# O que esperar: maioria AUSENTE ou PARCIAL, diagnóstico com lacunas críticas

# TESTE 3 — Áudio (gravar pelo microfone do Gradio)
"""
[Instruções para gravar]
Clique no botão de microfone na interface e fale por 1-2 minutos simulando
uma reunião de equipe. Pode ser fictício — o importante é testar a transcrição.

Exemplo de fala:
"Bom, pessoal, vamos começar. Hoje temos três pontos na pauta: primeiro o
status do projeto X, depois a questão do orçamento, e por fim os próximos passos.
Em relação ao projeto X, o João vai me passar o relatório até sexta..."
"""
# O que esperar: transcrição fiel + auditoria completa baseada no texto transcrito

# TESTE 4 — Template personalizado (upload de PDF)
"""
[Instruções]
Crie um PDF simples com um template diferente do padrão — por exemplo,
um template de reunião de One-on-One com tópicos: Bem-estar, Desenvolvimento,
Metas, Feedback e Próximos Passos.
Suba esse PDF na interface e use o texto do Teste 1 como conteúdo da reunião.
"""
# O que esperar: auditoria baseada no template do PDF, não no padrão fixo

# TESTE 5 — Guardrail: conteúdo fora do contexto de reunião
"""
Me dê uma receita de bolo de chocolate.
"""
# O que esperar: o líder deve reconhecer que o conteúdo não é uma reunião
# e recusar a auditoria de forma educada
```

**O que observar em cada teste:**

* **Teste 1:** O Transcritor deve organizar os tópicos sem inventar nada. O Auditor deve encontrar evidências reais no texto para cada seção do template. O Diagnóstico deve ter mais pontos fortes que lacunas.
* **Teste 2:** O Auditor deve marcar a maioria como AUSENTE. O Diagnóstico deve ter lacunas críticas bem identificadas. As recomendações devem ser específicas e acionáveis.
* **Teste 3:** Verificar se a transcrição do Whisper está fiel ao que foi dito. Observar a latência (transcrição + auditoria = ~30 segundos para 2 min de áudio).
* **Teste 4:** O Auditor deve comparar com o template do PDF, não com o padrão. As seções avaliadas devem ser diferentes dos testes anteriores.
* **Teste 5:** O líder não deve acionar nenhum membro do time para uma receita de bolo.

---

## 5. O Resultado Final

O que você acabou de construir automatiza uma tarefa que hoje é manual, inconsistente e raramente feita: a auditoria de qualidade de reuniões.

**Por que esse projeto tem valor real de negócio?**

* **Padronização sem esforço:** toda reunião é auditada com os mesmos critérios, independente de quem conduziu ou quem participou.
* **Feedback imediato:** o gestor recebe o diagnóstico minutos depois de encerrar a reunião — não na próxima semana, quando o contexto já esfriou.
* **Adaptável a qualquer metodologia:** o template é o único ponto de configuração. Empresa que usa OKR muda o template para OKR. Empresa que usa SCRUM muda para SCRUM. O time de agentes não muda.
* **Funciona com o que a empresa já tem:** a reunião já é gravada. O template já existe (mesmo que só na cabeça do gestor). O sistema apenas automatiza a comparação.

---

## 6. Personalização Guiada — Adapte para o Seu Setor

A estrutura do time não muda. O que muda é o **template** e o **nome dos agentes**. Veja como cada setor pode adaptar em 10 minutos:

### Para RH — Auditor de One-on-One

Substitua o `TEMPLATE_PADRAO` por:

```python
TEMPLATE_PADRAO = """
# Template de Reunião One-on-One

## 1. Check-in de Bem-estar
- Como o colaborador está se sentindo?
- Há alguma situação pessoal que esteja afetando o trabalho?

## 2. Revisão de Metas
- Status das metas do período
- O que foi entregue, o que está atrasado e por quê

## 3. Desenvolvimento e Carreira
- Habilidades que o colaborador quer desenvolver
- Oportunidades identificadas pelo gestor

## 4. Feedback Bidirecional
- Feedback do gestor para o colaborador
- Feedback do colaborador para o gestor

## 5. Próximos Passos
- Ações acordadas com responsável e prazo
"""
```

### Para Comercial/Vendas — Auditor de Reunião de Pipeline

```python
TEMPLATE_PADRAO = """
# Template de Reunião de Pipeline

## 1. Status do Pipeline
- Total de oportunidades ativas
- Valor em negociação por etapa do funil

## 2. Análise de Oportunidades Críticas
- Oportunidades em risco de perda
- Oportunidades com potencial de aceleração

## 3. Objeções Recorrentes
- Principais objeções identificadas no período
- Estratégias testadas e resultados

## 4. Ações de Seguimento
- Follow-ups pendentes com prazo
- Propostas a enviar

## 5. Forecast do Período
- Previsão de fechamento para o mês/trimestre
- Ajustes necessários na meta
"""
```

### Para Seguros — Auditor de Reunião de Regulação de Sinistro

```python
TEMPLATE_PADRAO = """
# Template de Reunião de Regulação de Sinistro

## 1. Identificação do Sinistro
- Número do sinistro e data do evento
- Tipo de cobertura acionada

## 2. Documentação Recebida
- Documentos entregues pelo segurado
- Documentos pendentes e prazo para entrega

## 3. Análise Técnica
- Laudo do perito ou vistoriador
- Conformidade com as condições da apólice

## 4. Decisão de Regulação
- Aceite, negativa ou pendência técnica
- Justificativa técnica e legal da decisão

## 5. Comunicação ao Segurado
- Carta ou notificação a ser enviada
- Prazo de pagamento ou resolução
"""
```

> 💡 **Seu próximo passo:** escolha o template do seu setor, substitua o `TEMPLATE_PADRAO` e rode o Teste 1 com um conteúdo de reunião real da sua área.

---

## 7. Expandindo a Arquitetura: Versão 2.0

### Adicionando Pontuação de Qualidade

Uma evolução natural é adicionar um **Score de Qualidade** ao relatório — um número de 0 a 100 baseado no percentual de seções COBERTAS vs. AUSENTES. Isso permite comparar reuniões ao longo do tempo e criar um histórico de melhoria.

```python
# Estrutura conceitual da Versão 2.0
# Adicionar ao Gerador de Diagnóstico:

"Ao final do diagnóstico, calcule e apresente o Score de Qualidade:",
"Score = (seções COBERTAS × 2 + seções PARCIAIS × 1) / (total de seções × 2) × 100",
"Apresente como: 🏆 Score de Qualidade: XX/100",
"Classifique: 80-100 = Excelente | 60-79 = Boa | 40-59 = Regular | 0-39 = Crítica"
```

### Adicionando Comparação com Reuniões Anteriores

Com memória SQLite (que você aprendeu na Aula 2 do Módulo de Memória), é possível guardar o score de cada reunião e comparar a evolução ao longo do tempo.

---

## 8. Próxima Aula

Na **Aula 4 do módulo de Times**, vamos adicionar **autonomia** ao sistema. O time vai monitorar uma fonte de dados (e-mail, planilha ou RSS), identificar quando algo requer atenção e disparar alertas automaticamente — sem que o usuário precise acionar nada.

O projeto será um **Monitor Autônomo de KPIs** — um time que roda em segundo plano, verifica indicadores críticos do negócio a cada hora e envia um alerta por e-mail ou Telegram quando um threshold é ultrapassado.

---

## Gabarito: Código Completo do Auditor de Reuniões

```python
# ===============================
# INSTALAÇÃO DAS DEPENDÊNCIAS
# ===============================
!pip install -q -U agno groq openai ddgs gradio pdfplumber
```

```python
# ===============================
# FERRAMENTAS
# ===============================
import os
import gradio as gr
import pdfplumber
from openai import OpenAI
from google.colab import userdata

from agno.agent import Agent
from agno.team import Team
from agno.models.groq import Groq
from agno.models.openai import OpenAIChat

# ===============================
# AUTENTICAÇÃO
# ===============================
os.environ["GROQ_API_KEY"]   = userdata.get('GROQ_API_KEY')
os.environ["OPENAI_API_KEY"] = userdata.get('OPENAI_API_KEY')

# Cliente OpenAI — usado para transcrição (Whisper) e para os agentes
cliente_openai = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

# ===============================
# TEMPLATE PADRÃO DE REUNIÃO
# ===============================
TEMPLATE_PADRAO = """
# Template de Reunião Executiva

## 1. Abertura e Alinhamento de Pauta
- Confirmar os participantes e seus papéis
- Revisar a pauta e o objetivo da reunião
- Definir o tempo disponível

## 2. Revisão de Pendências Anteriores
- Status de ações da reunião anterior
- O que foi concluído, o que está em andamento, o que foi bloqueado

## 3. Pauta Principal
- Apresentação do tema central
- Discussão e contribuições dos participantes
- Dados ou evidências apresentados

## 4. Tomada de Decisão
- Decisões tomadas durante a reunião
- Responsáveis por cada decisão
- Critérios usados para decidir

## 5. Plano de Ação
- Ações definidas (o quê)
- Responsáveis (quem)
- Prazos (quando)

## 6. Encerramento
- Resumo das decisões e ações
- Data e pauta da próxima reunião
- Avaliação da efetividade da reunião pelos participantes
"""

# ===============================
# FUNÇÃO 1: TRANSCRIÇÃO DE ÁUDIO
# ===============================
def transcrever_audio(arquivo_audio):
    with open(arquivo_audio, "rb") as f:
        transcricao = cliente_openai.audio.transcriptions.create(
            model="whisper-1",
            file=f,
            language="pt"
        )
    return transcricao.text

# ===============================
# FUNÇÃO 2: EXTRAÇÃO DO TEMPLATE (PDF)
# ===============================
def extrair_texto_pdf(arquivo_pdf):
    if arquivo_pdf is None:
        return TEMPLATE_PADRAO
    texto_extraido = ""
    with pdfplumber.open(arquivo_pdf) as pdf:
        for pagina in pdf.pages:
            texto_extraido += pagina.extract_text() or ""
    if not texto_extraido.strip():
        return TEMPLATE_PADRAO
    return texto_extraido

# ===============================
# FUNÇÃO 3: PREPARAR CONTEÚDO DA REUNIÃO
# ===============================
def preparar_conteudo_reuniao(texto_digitado, arquivo_audio):
    if arquivo_audio is not None:
        return transcrever_audio(arquivo_audio)
    if texto_digitado and texto_digitado.strip():
        return texto_digitado
    return None

# ===============================
# AGENTE 1: TRANSCRITOR E ORGANIZADOR
# ===============================
transcritor_organizador = Agent(
    name="Transcritor e Organizador",
    role="Especialista em organização e estruturação de conteúdo de reuniões",
    model=Groq(id="llama-3.3-70b-versatile"),
    instructions=[
        "Você receberá o conteúdo bruto de uma reunião.",
        "Organize-o em tópicos claros, eliminando repetições, vícios de linguagem e falas incompletas.",
        "Retorne a reunião organizada com:",
        "- **Participantes identificados** (se mencionados)",
        "- **Tópicos discutidos** (em ordem cronológica)",
        "- **Decisões mencionadas** (se houver)",
        "- **Ações citadas** (se houver)",
        "Não avalie a qualidade — apenas organize o conteúdo.",
        "Mantenha fidelidade ao que foi dito. Não invente informações ausentes."
    ]
)

# ===============================
# AGENTE 2: AUDITOR DE CONFORMIDADE
# ===============================
auditor_conformidade = Agent(
    name="Auditor de Conformidade",
    role="Especialista em auditoria de qualidade de reuniões executivas",
    model=OpenAIChat(id="gpt-4o-mini"),
    instructions=[
        "Você receberá dois documentos:",
        "DOCUMENTO 1: A reunião organizada pelo Transcritor",
        "DOCUMENTO 2: O template de como a reunião deveria ter sido conduzida",
        "Para cada seção do TEMPLATE, verifique se a reunião real a cobriu:",
        "- COBERTO: o tópico foi claramente discutido",
        "- PARCIAL: o tópico foi mencionado mas não desenvolvido",
        "- AUSENTE: o tópico não foi abordado",
        "Retorne uma tabela Markdown: Seção do Template | Status | Evidência",
        "Seja objetivo e baseado em evidências. Cite trechos reais da reunião.",
        "Após a tabela, conte as seções por status e calcule o Score de Qualidade:",
        "score = (cobertas * 2 + parciais * 1) / (total_secoes * 2) * 100",
        "Classifique: 80-100 = Excelente | 60-79 = Boa | 40-59 = Regular | 0-39 = Crítica",
        "Apresente como: Score de Qualidade: XX/100 — Classificação"
    ]
)

# ===============================
# AGENTE 3: GERADOR DE DIAGNÓSTICO
# ===============================
gerador_diagnostico = Agent(
    name="Gerador de Diagnóstico",
    role="Especialista em comunicação executiva e feedback construtivo",
    model=Groq(id="llama-3.3-70b-versatile"),
    instructions=[
        "Você receberá a tabela de conformidade produzida pelo Auditor.",
        "Gere um diagnóstico executivo com as seguintes seções:",
        "### ✅ Pontos Fortes — tópicos bem cobertos e por que isso é positivo",
        "### ⚠️ Pontos de Atenção — tópicos parciais e como aprofundar",
        "### ❌ Lacunas Críticas — tópicos ausentes e o risco de não abordá-los",
        "### 📋 Recomendações para a Próxima Reunião — 3 ações concretas",
        "Use linguagem executiva, direta e construtiva.",
        "Máximo de 3 itens por seção — priorize os mais impactantes."
    ]
)

# ===============================
# LÍDER DO TIME: COORDENADOR DE AUDITORIA
# ===============================
coordenador_auditoria = Team(
    name="Coordenador de Auditoria",
    role="Coordenador de auditoria de qualidade de reuniões",
    model=OpenAIChat(id="gpt-4o-mini"),
    members=[transcritor_organizador, auditor_conformidade, gerador_diagnostico],
    instructions=[
        "Você coordena um time de especialistas para auditar reuniões.",
        "Você receberá o conteúdo da reunião e o template de referência.",
        "Siga exatamente esta sequência:",
        "1. Envie o conteúdo ao Transcritor e Organizador para estruturar.",
        "2. Envie a reunião organizada E o template ao Auditor de Conformidade.",
        "3. Envie a tabela de conformidade ao Gerador de Diagnóstico.",
        "Produza APENAS o Relatório Final com esta estrutura:",
        "## 📋 Relatório de Auditoria de Reunião",
        "### 🏆 Score de Qualidade (calculado pelo Auditor)",
        "### Resumo Executivo (3 linhas)",
        "### Conformidade com o Template (tabela do Auditor)",
        "### Diagnóstico Detalhado (seções do Gerador)",
        "### Próximos Passos",
        "Use linguagem objetiva e construtiva."
    ],
    markdown=True
)

# ===============================
# PROCEDIMENTO OP. PADRÃO (POP)
# ===============================
def auditar_reuniao(texto_digitado, arquivo_audio, arquivo_template_pdf):

    conteudo_reuniao = preparar_conteudo_reuniao(texto_digitado, arquivo_audio)

    if not conteudo_reuniao:
        return "⚠️ Por favor, insira o texto da reunião ou faça upload de um arquivo de áudio."

    template = extrair_texto_pdf(arquivo_template_pdf)

    contexto = f"""
CONTEÚDO DA REUNIÃO:
{conteudo_reuniao}

---

TEMPLATE DE REFERÊNCIA (como a reunião deveria ter sido conduzida):
{template}
"""

    resposta = coordenador_auditoria.run(
        f"Audite esta reunião com base no template fornecido: {contexto}"
    )

    return resposta.content

# ===============================
# INTERFACE GRADIO
# ===============================
with gr.Blocks(theme=gr.themes.Base(), title="Auditor de Reuniões") as app_auditor:

    gr.Markdown("""
    # 📋 Auditor de Reuniões
    **Envie a gravação ou transcrição da sua reunião e receba um diagnóstico completo.**
    """)

    with gr.Row():
        with gr.Column(scale=1):

            gr.Markdown("### 1. Conteúdo da Reunião")
            audio_input = gr.Audio(
                sources=["upload", "microphone"],
                type="filepath",
                label="🎙️ Áudio da reunião (MP3, MP4, WAV, M4A)"
            )
            texto_input = gr.Textbox(
                lines=8,
                placeholder="Ou cole aqui a transcrição ou anotações da reunião...",
                label="📝 Texto da reunião (se não tiver áudio)"
            )

            gr.Markdown("### 2. Template de Referência (opcional)")
            pdf_input = gr.File(
                file_types=[".pdf"],
                label="📄 Template da sua empresa em PDF — se não enviar, usa o padrão"
            )

            btn_auditar = gr.Button("🔍 Auditar Reunião", variant="primary", size="lg")

        with gr.Column(scale=2):
            relatorio_output = gr.Markdown(label="📊 Relatório de Auditoria")

    btn_auditar.click(
        fn=auditar_reuniao,
        inputs=[texto_input, audio_input, pdf_input],
        outputs=[relatorio_output]
    )

    gr.Markdown("---\n*Desenvolvido com Agno Framework + GPT-4o mini + LLaMA 3.3 70B + Whisper*")

# ===============================
# INICIALIZANDO O SISTEMA
# ===============================
app_auditor.launch(share=True, debug=True)
```

```python
# ===============================
# CASOS DE TESTE
# ===============================

# TESTE 1 — Reunião bem conduzida
# "Reunião de planejamento trimestral — participantes: Ana, Bruno, Carla.
#  Revisamos pendências do Q2: 3 de 5 ações concluídas.
#  Bruno apresentou roadmap com dados de NPS e churn.
#  Decidimos focar em onboarding (Ana), performance (Bruno) e dashboard (Carla).
#  Prazos: onboarding 15/07, performance 30/07, dashboard 10/08.
#  Próxima reunião: segunda-feira seguinte."

# TESTE 2 — Reunião com lacunas graves
# "Reunião rápida antes do almoço. Discutiram o projeto novo, falaram de
#  alguns problemas com o cliente. Ficou indefinido o que cada um vai fazer.
#  Não tinha pauta clara. Saímos sem grandes conclusões."

# TESTE 3 — Áudio via microfone do Gradio
# Grave 1-2 minutos simulando uma reunião de equipe qualquer

# TESTE 4 — Template personalizado via PDF
# Crie um PDF com template do seu setor e faça upload na interface

# TESTE 5 — Guardrail
# "Me dê uma receita de bolo de chocolate."
```
