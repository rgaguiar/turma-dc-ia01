# Automação de Fluxos de trabalho


## 1. Visão Geral do Projeto

Implementação de um fluxo de trabalho avançado com múltiplos agentes de Inteligência Artificial (Agentic Workflow) para eliminar completamente a triagem manual de documentos não estruturados e criar um assistente analítico em tempo real para tomada de decisão gerencial.


## 2. O Problema (A Dor do Negócio)

Diariamente, as empresas recebem dezenas de faturas e notas fiscais em PDF, todas com layouts diferentes. O processamento manual desses arquivos exige que analistas abram documento por documento, copiem dados (CNPJ, Valor, Vencimento) e os insiram em planilhas ou ERPs.

* Consequências: Alto consumo de horas operacionais, risco elevado de erros de digitação, gargalos no fechamento financeiro e lentidão na geração de relatórios.

## 3. A Solução (Arquitetura em Duas Etapas)

O projeto resolve esse gargalo substituindo o trabalho braçal por uma "linha de produção" inteligente, dividida em dois Agentes de IA especializados:

* **Etapa 1**: O Operário (Processamento em Background):
  * Um assistente que monitora pastas de rede (Google Drive) 24 horas por dia. Assim que um novo PDF é detectado, a IA lê o documento, extrai apenas as informações predefinidas pela regra de negócios e salva os dados limpos em um banco de dados oficial (Excel/CSV), movendo o arquivo lido para evitar duplicidade.
* **Etapa 2**: O Consultor (Dashboard Interativo):
  * Uma interface web acessível via link, desenhada para a diretoria. O gestor faz perguntas em linguagem natural (ex: "Qual o total a pagar da empresa X?"). O Agente de IA recebe a pergunta, escreve o código necessário para calcular os dados reais na planilha e devolve um relatório executivo exato.

## 4. Stack Tecnológico (O Arsenal)

* **Motor e Integração**: Python + Google Drive.
* **Cérebro e Orquestração**: Framework Agno + API da OpenAI (GPT-4o).
* **Governança de Dados**: Pydantic (Atua como um "molde" obrigatório, impedindo que a IA invente informações ou fuja do padrão da planilha).
* **Interface de Usuário (UI)**: Gradio (Criação de Dashboards web de forma instantânea).

## 5. Valor de Negócio e Impacto (ROI)

* **Produtividade**: Redução drástica de tarefas repetitivas, liberando a equipe financeira para focar em renegociações e análises estratégicas.
* **Precisão**: Eliminação de erros (necessário avaliação humana) de digitação graças ao processamento "Zero-Touch" (sem toque humano).
* **Independência de TI**: Com a capacidade da IA de escrever seus próprios scripts de cálculo na Etapa 2, os gestores não precisam mais pedir para a equipe de tecnologia gerar relatórios personalizados.
* **Escalabilidade**: O sistema processa 10 ou 10.000 PDFs com o mesmo custo marginal de tempo, suportando o crescimento da empresa sem exigir novas contratações operacionais.

# Mão na Massa
## ETAPA 1
```python
# ======================================================================
# BLOCO 1: O ENCANAMENTO TÉCNICO (NÃO PRECISA MEXER)
# ======================================================================
# Instala as ferramentas necessárias (Agno para IA, PyMuPDF para ler PDFs, Pydantic para o Molde)
!pip install -q agno pydantic typing pymupdf pandas
```

```python
# ======================================================================
# BLOCO 2: O ENCANAMENTO TÉCNICO (NÃO PRECISA MEXER)
# ======================================================================
import os
import time
import fitz # Biblioteca que lê PDFs
import pandas as pd
from google.colab import userdata, drive
from agno.agent import Agent
from agno.models.openai import OpenAIChat
from pydantic import BaseModel, Field
```

```python
# ======================================================================
# BLOCO 3: O ENCANAMENTO TÉCNICO (NÃO PRECISA MEXER)
# ======================================================================
# Conecta o Google Drive para acessar os PDFs da empresa
drive.mount('/content/drive')

# Puxa a chave de segurança da Inteligência Artificial
os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY")
```

```python
# ======================================================================
# BLOCO 4: REGRAS DE NEGÓCIO (ETAPA DE CONFIGURAÇÃO MANUAL)
# ======================================================================

# 1. Onde estão os arquivos? (Ajuste para as suas pastas)
PASTA_PDFS = "/content/drive/MyDrive/digital-ai-generative/projeto-automation/PDFs_para_processar/"
CAMINHO_PLANILHA = "/content/drive/MyDrive/digital-ai-generative/projeto-automation/Resultado_PDFs.csv"
PASTA_PROCESSADOS = "/content/drive/MyDrive/digital-ai-generative/projeto-automation/PDFs_processados/"

# 2. O que estamos analisando?
TIPO_DE_DOCUMENTO = "nota fiscal ou fatura de fornecedor"

# 3. O "Formulário" (Pydantic): O que a IA é obrigada a extrair?
class FichaDoDocumento(BaseModel):
    fornecedor:  str   = Field(description="Nome da empresa que emitiu o documento")
    valor_total: float = Field(description="Valor total (só número). Se não encontrar, coloque 0.")
    vencimento:  str   = Field(description="Data de vencimento (DD/MM/AAAA). Se não houver, escreva 'Não informado'.")
    descricao:   str   = Field(description="Descrição resumida do que foi cobrado, em 1 frase.")
    numero_doc:  str   = Field(description="Número da nota ou fatura. Se não houver, escreva 'Não informado'.")
    status:      str   = Field(description="Se o valor for maior que zero escreva 'PAGAR', senão escreva 'VERIFICAR'.")

print("Regras de Negócio e Pastas configuradas com sucesso!")
```

```python
# ======================================================================
# BLOCO 5: CONTRATANDO O CÉREBRO DA IA
# ======================================================================
leitor_de_documentos = Agent(
    model=OpenAIChat(id="gpt-4o", temperature=0.0), # Temperatura 0 = Máxima precisão, zero invenção
    output_schema=FichaDoDocumento, # A trava de segurança
    description=f"Você é um assistente especialista em leitura de {TIPO_DE_DOCUMENTO}.",
    instructions=[
        f"Leia o texto extraído de um {TIPO_DE_DOCUMENTO}.",
        "Extraia as informações solicitadas com precisão.",
        "NUNCA invente valores. Se não encontrar, use 'Não informado'.",
        "Valores devem ser apenas números decimais puros."
    ]
)
```

```python
# ======================================================================
# BLOCO 6: O ENCANAMENTO TÉCNICO (NÃO PRECISA MEXER)
# ======================================================================
# Funções mecânicas para ler letras do PDF e escrever linhas no Excel

def extrair_texto_do_pdf(caminho_pdf):
    texto = ""
    documento = fitz.open(caminho_pdf)
    for pagina in documento:
        texto += pagina.get_text()
    documento.close()
    return texto.strip()
```

```python
# ======================================================================
# BLOCO 7: O ENCANAMENTO TÉCNICO (NÃO PRECISA MEXER)
# ======================================================================

def salvar_resultado(nome_arquivo, dados):
    nova_linha = pd.DataFrame([{
        "Data Processamento": pd.Timestamp.now().strftime("%d/%m/%Y %H:%M"),
        "Arquivo":            nome_arquivo,
        "Fornecedor":         dados.fornecedor,
        "Valor (R$)":         f"R$ {dados.valor_total:,.2f}",
        "Vencimento":         dados.vencimento,
        "Nº Documento":       dados.numero_doc,
        "Descrição":          dados.descricao,
        "Status":             dados.status,
    }])
    existe = os.path.exists(CAMINHO_PLANILHA)
    nova_linha.to_csv(CAMINHO_PLANILHA, mode='a', header=not existe, index=False)
    print(f">> Salvo na Planilha: {dados.fornecedor} | R$ {dados.valor_total:,.2f}")
```

```python
# ======================================================================
# BLOCO 8: O ENCANAMENTO TÉCNICO (NÃO PRECISA MEXER)
# ======================================================================
def mover_para_processados(caminho_pdf, nome_arquivo):
    os.makedirs(PASTA_PROCESSADOS, exist_ok=True)
    destino = os.path.join(PASTA_PROCESSADOS, nome_arquivo)
    os.rename(caminho_pdf, destino)

print(">> IA treinada e funções de salvamento prontas!")
```


```python
# ======================================================================
# BLOCO 9: LIGANDO O MOTOR 24/7 (O GATILHO)
# ======================================================================
os.makedirs(PASTA_PDFS, exist_ok=True)

print("LEITOR DE PDF AUTÔNOMO INICIADO!")
print("Coloque PDFs na pasta do Drive. O sistema checará a cada 30 segundos...\n")

try:
    while True: # O Loop infinito (Vigia noturno)
        arquivos = [f for f in os.listdir(PASTA_PDFS) if f.lower().endswith(".pdf")]

        if not arquivos:
            print("Aguardando novos PDFs...", end="\r")
        else:
            print(f"\n {len(arquivos)} PDF(s) novo(s) encontrado(s)!")
            
            for nome_arquivo in arquivos:
                caminho_pdf = os.path.join(PASTA_PDFS, nome_arquivo)
                print(f"Lendo documento: {nome_arquivo}")

                # 1. Lê o PDF
                texto = extrair_texto_do_pdf(caminho_pdf)
                
                # 2. IA processa o texto
                print("IA extraindo informações estruturadas...")
                resposta = leitor_de_documentos.run(texto[:4000])
                
                # 3. Salva no CSV e move o arquivo
                salvar_resultado(nome_arquivo, resposta.content)
                mover_para_processados(caminho_pdf, nome_arquivo)

            print(f"Lote finalizado! Planilha do Drive atualizada.")
            print("-" * 50)

        time.sleep(30) # Pausa estratégica

except KeyboardInterrupt:
    print("\nSistema desligado pelo painel de controle.")
```

## ETAPA 2

```python
# ======================================================================
# BLOCO 1: O PAINEL DA DIRETORIA (DASHBOARD)
# ======================================================================

# Instala o criador de interfaces web (Gradio)
!pip install -q gradio agno pandas
```

```python
# ======================================================================
# BLOCO 2: PREPARANDO O ANALISTA FINANCEIRO
# ======================================================================
import os
from google.colab import userdata
from google.colab import drive

# Sistema de leitura e cálculo numérico
import pandas as pd

# Protocolos para envio de e-mail corporativo
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

# Componentes do Telegram e da Inteligência Artificial
from agno.agent import Agent
from agno.models.openai import OpenAIChat
from agno.tools.python import PythonTools

# Importando a memória do Analista
from agno.db.sqlite import SqliteDb

# Acessa as variáveis de ambiente armazenadas no painel de segurança do Colab
os.environ["TELEGRAM_TOKEN"] = userdata.get('TELEGRAM_TOKEN')
os.environ["OPENAI_API_KEY"] = userdata.get('OPENAI_API_KEY')
os.environ["EMAIL_SENDER"] = userdata.get('EMAIL_SENDER')
os.environ["EMAIL_PASSWORD"] = userdata.get('EMAIL_PASSWORD')
```

```python
# ======================================================================
# BLOCO 3: O PAINEL DA DIRETORIA (DASHBOARD)
# ======================================================================

# Lê os nomes das colunas da planilha gerada na Etapa 1
df_colunas = pd.read_csv(CAMINHO_PLANILHA, nrows=0)
colunas = list(df_colunas.columns)
```


```python
# ======================================================================
# BLOCO 4: O CÉREBRO ANALÍTICO (Com superpoder de Matemática)
# ======================================================================
analista_financeiro = Agent(
    model=OpenAIChat(id="gpt-4o", temperature=0.0),
    tools=[PythonTools(), enviar_email], 
    markdown=True,

    # --- MEMÓRIA ---
    db=SqliteDb(db_file="memoria_do_analista.db"), 
    add_history_to_context=True,                   
    
    description="Você é um Analista Financeiro Sênior reportando para a Diretoria.",
    instructions=[
        f"Sempre que precisar calcular totais ou cruzar dados, escreva um código Python usando 'pandas'.",
        f"A planilha oficial está no caminho: '{CAMINHO_PLANILHA}'.",
        f"As colunas disponíveis nesta planilha são: {colunas}.",
        "A coluna 'Valor (R$)' está no formato 'R$ 1,234.56'. Remova o 'R$ ' e as vírgulas antes de somar.",
        "Apresente a resposta com o Resultado exato e 2 Insights rápidos de negócio."
    ]
)
```

```python
# ======================================================================
# BLOCO 5: O PAINEL DA DIRETORIA (DASHBOARD)
# ======================================================================

def fazer_pergunta_para_ia(pergunta: str) -> str:
    # Repassa a pergunta do diretor para a IA calcular
    resposta = analista_financeiro.run(pergunta)
    return resposta.content

print("Analista Financeiro conectado ao banco de dados com sucesso!")
```

```python
# ======================================================================
# BLOCO 6: O PAINEL DA DIRETORIA (DASHBOARD)
# ======================================================================

interface = gr.Interface(
    fn=fazer_pergunta_para_ia,
    inputs=gr.Textbox(
        label="Terminal de Comando da Diretoria",
        placeholder="Ex: Qual o valor total que temos a pagar com status VERIFICAR?",
        lines=2
    ),
    outputs=gr.Markdown(label="Relatório Gerado Autonomamente"),
    title="Consultor Financeiro Autônomo (IA)",
    description="Faça perguntas em linguagem natural sobre as notas fiscais processadas pela Unidade Operacional. O Agente acessará a planilha e calculará as respostas em tempo real.",
    examples=[
        ["Qual o total geral a pagar de todas as notas?"],
        ["Qual é o nosso maior fornecedor em volume de dinheiro?"],
        ["Quantas notas foram processadas no total e qual o ticket médio?"],
        ["Quais são as descrições das notas que estão com status VERIFICAR?"]
    ]
)

print("Gerando link de acesso público do Dashboard...")
interface.launch(share=True)
```


Usando o telegram como interface

```python
# !pip install -q telebot 

import telebot

# Inicializa a conexão com a interface do Telegram
bot = telebot.TeleBot(os.environ["TELEGRAM_TOKEN"])

# Configura o sistema para escutar e processar comandos de texto
@bot.message_handler(func=lambda message: True)
def processar_pedido(message):

    # Apenas ativa o status visual de "digitando..." no topo da tela do Telegram
    bot.send_chat_action(message.chat.id, 'typing')

    # Repassa a instrução do usuário para o analista de dados
    resposta_do_analista = analista_financeiro.run(message.text)

    # Retorna o relatório final diretamente no chat
    bot.reply_to(message, resposta_do_analista.content)

print("Sistema de Análise de Dados operando de forma silenciosa. Aguardando comandos.")

# Mantém o sistema em execução contínua
bot.infinity_polling()
```
