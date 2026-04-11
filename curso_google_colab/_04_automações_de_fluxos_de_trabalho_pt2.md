# Automatização de Workflows com IA

---

## 1. Visão Geral do Projeto

Implementação de um fluxo de trabalho com um Agente de Inteligência Artificial para eliminar completamente o processamento manual de documentos financeiros recebidos por e-mail — lendo, interpretando e registrando automaticamente as informações em uma planilha de negócios.

---

## 2. O Problema (A Dor do Negócio)

Diariamente, empresas recebem dezenas de notas fiscais, cupons e recibos por e-mail — todos com layouts diferentes. O processamento manual exige que analistas abram cada e-mail, identifiquem os anexos, extraiam dados e os registrem em planilhas ou ERPs.

**Consequências:**

- **Inexistência de Escalabilidade:** Se o volume de notas dobra, a equipe precisa dobrar.
- **Risco de Compliance:** Erros de digitação em valores e CNPJs geram multas e problemas fiscais.
- **Custo Operacional:** Profissionais qualificados perdendo tempo com digitação de dados desestruturados.
- **Gargalos no Fechamento:** Documentos inválidos ou duplicados contaminam a base e atrasam relatórios.

---

## 3. A Solução — O Fluxo em Três Filtros

O projeto substitui o trabalho braçal por uma **linha de produção inteligente** com três filtros sequenciais:

**Filtro Estratégico (Assunto do E-mail):**
O sistema não lê todos os e-mails da caixa de entrada — apenas os não lidos com palavras-chave relevantes no assunto (`nota`, `cupom`, `recibo`). Isso economiza tempo de processamento e custo de API.

**Visão da IA (GPT-4o Multimodal):**
O agente recebe a imagem do documento — seja uma foto, um scan ou um PDF convertido — e extrai as informações com precisão visual. Ele também decide se o documento é válido ou não.

**Planilha de Negócios (Destino Limpo):**
Apenas documentos validados pela IA chegam ao CSV final. Documentos inválidos são descartados antes de contaminar a base de dados.

---

## 4. Valor de Negócio e Impacto (ROI)

**Produtividade:** O agente processa toda a caixa de entrada de forma autônoma, liberando a equipe financeira para análises estratégicas e renegociações.

**Auditoria e Compliance:** Cada linha no CSV pode ser conferida com o anexo original. O rastro digital é criado automaticamente — sem esforço humano.

**Precisão de Dados (Zero Erro de Digitação):** Ao eliminar o fator humano na transposição de números, a confiança nos relatórios financeiros sobe para níveis próximos de 100%.

**Human-in-the-Loop:** O analista deixa de ser um "digitador" e passa a ser um "auditor". Ele apenas revisa o CSV final, focando nas exceções e anomalias identificadas pela IA.

**Escalabilidade:** O sistema processa 10 ou 10.000 e-mails com o mesmo custo marginal de tempo — sem novas contratações operacionais.

---

## 5. Por que Python e Agentes? (O Diferencial Tecnológico)

Muitas empresas tentam resolver esse problema usando o ChatGPT manualmente ou ferramentas de Low-Code. Veja por que a abordagem via código e Agentes é superior:

**Versus Uso Manual do ChatGPT:**
Nossa solução é autônoma — ela vigia a caixa de entrada e trabalha enquanto você dorme, eliminando o erro humano e o trabalho braçal de upload.

**Versus Ferramentas Low-Code (Zapier, Make):**
Em Python, você tem controle total sobre a lógica de filtragem e conversão de arquivos, sem custos de licenciamento por execução.

**Processamento de Dados Complexos:**
O Agno permite lidar com fluxos onde a IA precisa decidir o que fazer com base no conteúdo visual — algo que ferramentas de "clica e arrasta" têm dificuldade em processar com precisão.

---

## 6. Stack Tecnológico

| Componente | Função |
|---|---|
| **Python** | Linguagem principal — orquestra todo o fluxo |
| **Agno** | Framework de Agentes IA — define comportamento e schema de saída |
| **GPT-4o** | Modelo multimodal — lê imagens de documentos e extrai dados |
| **Pydantic** | Define o molde obrigatório dos dados extraídos pela IA |
| **imaplib** | Protocolo de acesso ao Gmail |
| **pdf2image** | Converte PDFs em imagens para a visão do modelo |
| **csv** | Grava os dados estruturados na planilha de negócios |
| **Google Colab** | Ambiente de execução — sem instalar nada localmente |

---

# Mão na Massa

## Contexto Geral do Projeto

Nesta aula, estamos construindo um **Agente Auditor Financeiro** capaz de monitorar uma caixa de entrada do Gmail, identificar e-mails com documentos financeiros, ler as imagens com Inteligência Artificial e registrar automaticamente os dados em uma planilha.

Na prática, estamos automatizando um processo que normalmente seria feito por uma pessoa:

1. Abrir a caixa de entrada
2. Identificar e-mails com notas fiscais ou recibos
3. Baixar o anexo
4. Ler o documento e extrair fornecedor, CNPJ, valor e data
5. Registrar na planilha
6. Descartar documentos inválidos

**Agora, quem faz isso é a Inteligência Artificial.**

---

### Bloco 0: Instalando as Ferramentas

Antes de qualquer coisa, precisamos instalar as bibliotecas que o projeto vai usar. Este bloco também instala o **Poppler** — um componente do sistema operacional necessário para converter PDFs em imagens.

```python
# Instala o framework de agentes, a conversão de PDF e o processamento de imagens
!pip install -U agno pdf2image pillow

# Instala o Poppler — motor de renderização de PDF usado pelo pdf2image
!apt-get install -y poppler-utils
```

**O que cada ferramenta faz:**

- **Agno** — framework para criar e orquestrar o agente de IA.
- **pdf2image** — converte páginas de PDF em imagens JPEG, necessário porque o GPT-4o trabalha com visão (imagem), não com texto de PDF.
- **Pillow** — biblioteca de processamento de imagens, usada internamente pelo pdf2image.
- **poppler-utils** — motor que executa a renderização do PDF. Sem ele, o pdf2image não funciona.

---

### Bloco 1: Importações e Configurações de Segurança

Aqui importamos os módulos e carregamos as credenciais de forma segura. As chaves de acesso nunca ficam escritas no código — elas são lidas do cofre de segurança do Google Colab.

```python
# =====================================================================
# PROJETO: AGENTE AUDITOR FINANCEIRO (VERSÃO FINAL PARA AULA)
# FLUXO: Filtro Estratégico -> Visão IA -> Planilha de Negócios
# =====================================================================

import os       # Acesso a variáveis de ambiente e sistema de arquivos
import imaplib  # Protocolo de acesso à caixa de entrada do Gmail
import email    # Leitura e interpretação de e-mails
import csv      # Leitura e escrita de arquivos CSV (planilha)

from pydantic import BaseModel, Field       # Definição do molde de dados (schema)
from pdf2image import convert_from_path     # Conversão de PDF para imagem
from google.colab import userdata           # Acesso seguro às credenciais do Colab
from agno.agent import Agent                # Classe principal do agente de IA
from agno.models.openai import OpenAIChat   # Conexão com o modelo GPT-4o
from agno.media import Image                # Envia imagens para o agente

# CONFIGURAÇÕES INICIAIS (SEGURANÇA)
# Lê as credenciais do cofre do Colab — nunca escreva senhas diretamente no código
os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY")
EMAIL_USER = userdata.get("EMAIL_SENDER")   # Endereço de e-mail
EMAIL_PASS = userdata.get("EMAIL_PASSWORD") # App Password do Gmail
```

**Por que usar Secrets do Colab?**

Credenciais escritas diretamente no código representam um risco de segurança — qualquer pessoa com acesso ao notebook veria as senhas. O `userdata.get()` lê as variáveis de um cofre criptografado, mantendo o código seguro para compartilhar.

**O que é o App Password do Gmail?**
É uma senha gerada pelo Google especificamente para aplicações externas — diferente da senha da conta e pode ser revogada a qualquer momento sem afetar o acesso normal. Para gerar: *Conta Google → Segurança → Verificação em duas etapas → Senhas de app*.

---

### Bloco 2: O Contrato de Dados (Schema Pydantic)

Este é um dos blocos mais importantes do projeto. Aqui definimos **exatamente o que a IA deve entregar** — os campos obrigatórios, os tipos de dados e as instruções de preenchimento quando uma informação não for encontrada.

```python
# CONTRATO DE DADOS (O QUE A IA DEVE ENTREGAR)
# O schema Pydantic funciona como um formulário obrigatório:
# a IA só pode responder preenchendo esses campos, nesse formato exato
class FichaDoDocumento(BaseModel):
    fornecedor:   str   = Field(description="Nome da empresa ou 'INVÁLIDO'")
    cnpj:         str   = Field(description="CNPJ encontrado")
    valor_total:  float = Field(description="Valor numérico (ex: 150.50)")
    data_emissao: str   = Field(description="Data DD/MM/AAAA")
```

**Por que o schema Pydantic é essencial?**

Sem ele, a IA responderia com texto livre — e cada documento teria um formato diferente, impossível de importar automaticamente para uma planilha. O schema funciona como um formulário padronizado entregue na mão do analista: ele só pode devolver o trabalho preenchendo os campos corretos, nos tipos corretos.

**Campos definidos:**

| Campo | Tipo | Instrução para a IA |
|---|---|---|
| `fornecedor` | texto | Nome da empresa emissora — ou `'INVÁLIDO'` se não for documento financeiro |
| `cnpj` | texto | CNPJ encontrado no documento |
| `valor_total` | número | Apenas o número decimal — sem `R$` ou formatação |
| `data_emissao` | texto | Data no formato DD/MM/AAAA |

**O campo `fornecedor` como filtro de qualidade:**
Repare que o campo `fornecedor` tem duas funções: guarda o nome da empresa *e* serve como sinal de documento inválido. Quando a IA não reconhece um documento financeiro, ela escreve `'INVÁLIDO'` nesse campo — e o código usa esse valor para decidir se salva ou descarta o registro.

---

### Bloco 3: Criando o Agente de IA

Aqui contratamos oficialmente o nosso Auditor Digital. Este bloco cria o agente de Inteligência Artificial com suas instruções de trabalho e a trava de segurança do schema.

```python
# O AGENTE (INTELIGÊNCIA COM FILTRO DE QUALIDADE)
agente = Agent(
    # O modelo escolhido: GPT-4o com capacidade de visão (lê imagens)
    model=OpenAIChat(id="gpt-4o"),

    # A trava de segurança: obriga a IA a devolver os dados no formato do schema
    output_schema=FichaDoDocumento,

    # O manual de operações: as duas regras que o agente deve sempre seguir
    instructions="""
    Analise a imagem. 
    1. Se for Nota Fiscal, Cupom ou Recibo, extraia os dados.
    2. Se NÃO for um documento financeiro, escreva 'INVÁLIDO' no campo fornecedor.
    """
)
```

**O que está sendo definido aqui:**

**O Modelo (`model`):**
Escolhemos o GPT-4o — modelo com capacidade nativa de visão. Ele lê imagens de documentos, identifica campos como CNPJ e valor, e interpreta layouts diferentes de notas fiscais sem nenhum treinamento específico.

**A Trava de Segurança (`output_schema`):**
É o schema Pydantic do bloco anterior. Ele impede que o agente responda com texto livre ou formatos inesperados. A saída sempre será um objeto estruturado com os quatro campos esperados pela planilha.

**O Manual de Operações (`instructions`):**
São as duas únicas regras do agente — intencionalmente simples. Regra 1: se for documento financeiro, extraia. Regra 2: se não for, marque como `'INVÁLIDO'`. Essa simplicidade garante comportamento previsível em produção.

---

### Bloco 4: Conectando ao Gmail com Filtro Estratégico

Abrimos a conexão com a caixa de entrada do Gmail e aplicamos o primeiro filtro do fluxo — o **Filtro Estratégico de Assunto**. Em vez de processar todos os e-mails, o sistema seleciona apenas os relevantes.

```python
# ACESSO AO GMAIL COM FILTRO DE ASSUNTO (ECONOMIA DE PROCESSAMENTO)
print("🔌 Conectando ao Gmail...")

# Abre uma conexão segura com o servidor de e-mail do Gmail
mail = imaplib.IMAP4_SSL("imap.gmail.com")

# Autentica com o endereço e o App Password
mail.login(EMAIL_USER, EMAIL_PASS)

# Seleciona a caixa de entrada
mail.select("inbox")

# FILTRO ESTRATÉGICO: Apenas e-mails não lidos com palavras-chave no assunto
# Isso evita processar e-mails irrelevantes e economiza chamadas à API da OpenAI
criterio_busca = '(UNSEEN OR SUBJECT "nota" OR SUBJECT "cupom" SUBJECT "recibo")'
_, mensagens = mail.search(None, criterio_busca)
ids_emails = mensagens[0].split()

print(f"📩 {len(ids_emails)} e-mails qualificados encontrados.")
```

**Por que o filtro estratégico é importante?**

Cada chamada ao GPT-4o tem um custo — tanto em tempo quanto em dinheiro. Processar toda a caixa de entrada sem filtro significaria enviar à IA newsletters, notificações e promoções que jamais conterão uma nota fiscal.

O filtro de assunto funciona como uma triagem humana: só chegam ao agente os e-mails com real chance de conter um documento financeiro. Menos chamadas à API, menos custo, mais velocidade.

---

### Bloco 5: O Loop de Automação

Este é o coração do projeto. Para cada e-mail selecionado, o sistema percorre os anexos, aplica filtros de formato, converte PDFs em imagens, aciona a IA e decide se os dados devem ser salvos ou descartados.

```python
# LOOP DE AUTOMAÇÃO
for num in ids_emails:

    # Baixa o e-mail completo
    _, data = mail.fetch(num, "(RFC822)")
    msg = email.message_from_bytes(data[0][1])
    print(f"\n--- Analisando: {msg['Subject']} ---")

    # Percorre todas as partes do e-mail (corpo, anexos, imagens inline)
    for part in msg.walk():
        nome_arquivo = part.get_filename()

        # FILTRO DE FORMATO: Apenas PDFs e imagens — ignora texto, HTML e outros
        if nome_arquivo and nome_arquivo.lower().endswith((".pdf", ".png", ".jpg", ".jpeg")):
            caminho = f"/content/{nome_arquivo}"

            # Salva o anexo no ambiente do Colab
            with open(caminho, "wb") as f:
                f.write(part.get_payload(decode=True))

            # CONVERSÃO DE PDF PARA IMAGEM
            # O GPT-4o trabalha com visão — PDFs precisam virar imagem primeiro
            if caminho.lower().endswith(".pdf"):
                print("🔄 Convertendo PDF para Imagem...")
                imagens = convert_from_path(caminho, first_page=1, last_page=1)
                caminho = caminho.replace(".pdf", ".jpg")
                imagens[0].save(caminho, "JPEG")

            # IA ANALISA O DOCUMENTO
            print("🧠 IA processando imagem...")
            resposta = agente.run("Extraia os dados", images=[Image(filepath=caminho)])
            ficha = resposta.content

            # FILTRO DE SEGURANÇA: Só salva se a IA validou o documento
            if ficha.fornecedor != "INVÁLIDO":
                print(f"💾 Salvando Dados: {ficha.fornecedor} | R$ {ficha.valor_total}")

                # Verifica se a planilha já existe para não duplicar o cabeçalho
                arquivo_existe = os.path.exists("relatorio_final.csv")

                with open("relatorio_final.csv", "a", newline="", encoding="utf-8") as f:
                    writer = csv.writer(f)
                    if not arquivo_existe:
                        writer.writerow(["Fornecedor", "CNPJ", "Valor", "Data"])
                    writer.writerow([ficha.fornecedor, ficha.cnpj, ficha.valor_total, ficha.data_emissao])
            else:
                print("🚫 Arquivo ignorado: Não é um documento financeiro.")

mail.logout()
print("\n✅ Processamento concluído com sucesso!")
```

**O que acontece em cada passagem do loop — em linguagem de negócio:**

| Etapa | O que o sistema faz | Por que importa |
|---|---|---|
| **Busca o e-mail** | Baixa o conteúdo completo, incluindo anexos | Sem isso, só teríamos o assunto |
| **Filtra o formato** | Aceita só PDF, PNG, JPG — rejeita Word, Excel, HTML | Evita que arquivos errados cheguem à IA |
| **Converte PDF** | Transforma a primeira página em imagem JPEG | O GPT-4o enxerga imagens, não texto de PDF |
| **Aciona a IA** | Envia a imagem e recebe a ficha preenchida | É aqui que a inteligência acontece |
| **Filtra o resultado** | Só salva se `fornecedor != 'INVÁLIDO'` | Documentos irrelevantes não chegam à planilha |
| **Grava no CSV** | Adiciona uma linha sem sobrescrever as anteriores | A planilha cresce a cada execução, sem perda de dados |
| **Encerra a sessão** | Fecha a conexão com o Gmail | Boa prática de segurança — libera o acesso |

---

## Fluxo Completo em Resumo

```
Entrada → E-mail com anexo PDF ou imagem
    │
    ▼ Filtro de Assunto: contém "nota", "cupom" ou "recibo"?
    │
    ▼ Filtro de Formato: é PDF, PNG, JPG ou JPEG?
    │
    ▼ Conversão: PDF → Imagem JPEG (se necessário)
    │
    ▼ Visão da IA: GPT-4o analisa e preenche a FichaDoDocumento
    │
    ▼ Filtro de Segurança: fornecedor != "INVÁLIDO"?
              │
              ├── SIM → Salva no relatorio_final.csv
              └── NÃO → Descarta silenciosamente

Saída → Uma linha limpa na planilha, pronta para ERP ou BI
```

---

## Exercício

**Customização do Schema:** Adicione campos como `vencimento` ou `categoria` no Bloco 2 e observe como o agente passa a extrair automaticamente essas informações sem nenhuma outra alteração no código.

---

## Próximos Passos

**1 — Configure as credenciais:**
Acesse os Secrets do Colab (ícone de chave no menu lateral) e adicione `OPENAI_API_KEY`, `EMAIL_SENDER` e `EMAIL_PASSWORD`.

**2 — Gere o App Password do Gmail:**
Conta Google → Segurança → Verificação em duas etapas → Senhas de app. Use essa senha, não a da conta.

**3 — Teste com e-mails reais:**
Envie para si mesmo alguns e-mails com notas fiscais em PDF ou imagem com a palavra "nota" no assunto. Execute o código e verifique se o `relatorio_final.csv` foi gerado corretamente.

**4 — Adapte o schema:**
Precisa de mais campos? Adicione à `FichaDoDocumento` — por exemplo, `numero_nf`, `vencimento` ou `categoria`. O agente se adapta automaticamente às novas instruções.

**5 — Expanda o filtro de assunto:**
Adicione palavras-chave do seu setor: `"fatura"`, `"boleto"`, `"NF-e"` — e o sistema passa a capturar todos os padrões da sua empresa.

---

*Comece pequeno, valide rapidamente, escale com confiança.*
