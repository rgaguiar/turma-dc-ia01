# Desenvolvendo um Analista de Dados com Agno e Telegram

<img width="1536" height="1024" alt="ChatGPT Image 20 de mar  de 2026, 17_26_28" src="https://github.com/user-attachments/assets/626a0e92-64ba-45fe-8808-3cff014beabb" />

```python
os.environ["TELEGRAM_TOKEN"] = userdata.get('TELEGRAM_TOKEN')
os.environ["OPENAI_API_KEY"] = userdata.get('OPENAI_API_KEY')
os.environ["EMAIL_SENDER"] = userdata.get('EMAIL_SENDER')
os.environ["EMAIL_PASSWORD"] = userdata.get('EMAIL_PASSWORD')
```

O projeto prático tem como objetivo integrar aplicações de IA Generativa com ferramentas de mensagens, podendo ser aplicadas ao Telegram, WhatsApp, Instagram, etc. 
A proposta é que o usuário possa criar assistentes usando frameworks de IA para realizar determinadas funções, como analisar dados, acessar e-mails, gerenciar agendas e cadastrar informações em bancos de dados.

## Pré-requisitos do Projeto

Antes de iniciarmos a construção do nosso Analista de Dados, você precisará preparar o seu ambiente de trabalho e ter em mãos algumas chaves de acesso essenciais. Pense nesta etapa como a separação dos ingredientes antes de começar a cozinhar.

### 1. Contas e Credenciais Necessárias

Você precisará criar contas gratuitas nas seguintes plataformas para gerar as suas chaves de acesso (Senhas). Copie essas chaves e guarde-as em um bloco de notas temporário:

* **Telegram (`TELEGRAM_TOKEN`)**: Necessário para criar o assistente no aplicativo. Você consegue essa chave conversando com o robô oficial chamado BotFather no próprio Telegram.
  
  1. **BotFather (Criação do Assistente e Senha de Acesso)**: O BotFather (o "Pai dos Bots") é o robô oficial do próprio Telegram responsável por criar novos assistentes virtuais. Você vai procurá-lo na barra de busca do aplicativo. É conversando com ele que você dará um nome e um perfil para o seu Analista de Dados. Ao final da criação, o BotFather te entregará o Token: uma senha única e longa que serve para conectar o nosso sistema de Inteligência Artificial ao aplicativo do Telegram.  
<img width="1086" height="370" alt="image" src="https://github.com/user-attachments/assets/4f52e04e-2231-4127-929a-5812747c9410" />


  2. **Get My ID Bot (Identificação de Segurança)**: O Get My ID é uma ferramenta de busca do Telegram que revela o seu número de identificação pessoal (o seu ID). Como o seu Analista de Dados lidará com planilhas corporativas e envios de e-mail, esse ID funciona como o seu "crachá de diretoria". Saber o seu próprio ID permite que, no futuro, nós possamos ensinar o sistema a obedecer e enviar relatórios exclusivamente para você, ignorando mensagens de pessoas desconhecidas.  
<img width="1086" height="347" alt="image" src="https://github.com/user-attachments/assets/834effb3-4a68-41da-9d84-4ae080643ee9" />


* **OpenAI (`OPENAI_API_KEY`)**: A chave de acesso ao "cérebro" do ChatGPT. Gerada no painel de desenvolvedores da OpenAI.
* **Groq (`GROQ_API_KEY`)**: (Opcional) Uma chave alternativa de Inteligência Artificial, caso você prefira usar modelos de código aberto ultrarrápidos em vez da OpenAI.
* **Gmail corporativo ou pessoal**:
  * **`EMAIL_SENDER`**: O seu endereço de e-mail completo (ex: seu.nome@gmail.com).
  * **`EMAIL_PASSWORD`**: Atenção: Esta não é a senha normal que você usa para entrar no e-mail. É uma "Senha de Aplicativo" específica de 16 letras, gerada nas configurações de segurança da sua Conta Google, que permite que o nosso sistema envie mensagens com segurança.

<img width="1388" height="838" alt="image" src="https://github.com/user-attachments/assets/e3462bd5-1b93-44d4-9c69-a80794ebcb72" />



> #### DICA: Obtendo o EMAIL_SENDER e o EMAIL_PASSWORD
> 
> **1. Acesse as configurações da Conta Google**
> * Abra o seu Gmail no computador.
> * Clique na sua foto de perfil, no canto superior direito.
> * Clique no botão "Gerenciar sua Conta do Google".
> 
> **2. O Atalho Rápido (A melhor forma de achar)**
> * Na nova tela que abriu, vai ter uma barra de pesquisa gigante no topo com uma lupa.
> * Digite exatamente assim: **Senhas de app** (ou *App passwords*, se estiver em inglês).
> * Clique na opção "Senhas de app" que aparecer na lista. (O Google pode pedir para você digitar sua senha normal agora, só para confirmar que é você mesmo).
> 
> **3. Criando a Senha do Analista**
> * Você verá uma tela pedindo para dar um nome para o aplicativo.
> * Digite algo que faça você lembrar do projeto, como: `Analista de Dados Colab`.
> * Clique no botão **Criar**.
> 
> **4. Copiando o seu Código Secreto**
> * Um quadro vai aparecer na tela com uma senha de 16 letras (geralmente com o fundo amarelo).
> * Essa é a sua `EMAIL_PASSWORD`!
> * Copie essas 16 letras juntas (não precisa copiar os espaços entre elas) e cole no cofre (Secrets) lá do Google Colab.
> 
> ⚠️ **Atenção**: O Google só mostra essa senha uma única vez. Se você perder, terá que excluir e criar uma nova.

### 2. Configurando o Cofre do Google Colab

Por questões de segurança corporativa, nós nunca digitamos senhas diretamente no código.
No menu lateral esquerdo do Google Colab, clique no ícone de chave (🔑 **Secrets / Segredos**) e adicione os cinco itens listados acima.

> **Importante**: Certifique-se de ativar o botão azul (interruptor) ao lado de cada nome para permitir que o nosso projeto acesse essas senhas, exatamente como mostra a imagem de referência.

## O Processo de Construção e Explicação do código por Blocos

### Bloco 0: Preparando o Ambiente (Instalação de Pacotes)

Como estamos rodando nosso projeto na nuvem (Google Colab), o computador virtual começa "zerado" toda vez que o abrimos. O nosso primeiro passo é instalar os pacotes de software que darão vida ao projeto.

* Copie o código abaixo, cole na primeira célula do seu Colab e clique no botão de "Play" para executar:
* Com as senhas guardadas no cofre, o primeiro passo no código é instalar os programas de Inteligência Artificial, leitura de dados e comunicação.
* Adicione o seguinte código na sua primeira célula do Google Colab e execute:

```python
# Instala os pacotes de Inteligência Artificial, Telegram e manipulação de dados
!pip install -q agno openai pyTelegramBotAPI pandas
```
#### Por que precisamos de cada uma dessas ferramentas?

Para que o sistema funcione com autonomia corporativa, ele depende de quatro pilares fundamentais. Cada pacote instalado acima tem uma função específica no nosso projeto:

* **agno (O Gerente do Sistema)**: Esta é a biblioteca mais importante do projeto. Ela funciona como o "esqueleto" ou o sistema operacional do nosso assistente. É o Agno que gerencia a memória, guarda o caderno de anotações, conecta as ferramentas (como o e-mail) e dita as regras de conduta que o nosso Analista de Dados deve seguir.

* **openai (O Motor de Raciocínio)**: É a tecnologia que conecta o nosso sistema aos modelos de linguagem avançados (como o GPT-4o). É este pacote que permite ao nosso assistente ler uma pergunta da diretoria escrita em português natural, interpretar o que foi pedido e escrever um relatório executivo bem estruturado e compreensível.

* **pyTelegramBotAPI (A Ponte de Comunicação)**: É o software que conecta o nosso código ao aplicativo de mensagens. Sem ele, o nosso Analista ficaria isolado. Esta ferramenta permite que o sistema receba instantaneamente a pergunta que o diretor digitou no celular e, minutos depois, devolva a resposta formatada diretamente na mesma conversa do Telegram.

* **pandas (O Motor Matemático)**: Inteligências Artificiais são excelentes com textos, mas costumam cometer erros ao tentar fazer cálculos de cabeça. O pandas resolve isso. Ele é um sistema focado exclusivamente em ler arquivos de dados (como a nossa planilha .csv) e executar operações matemáticas complexas com precisão absoluta. Ele garante que os valores financeiros do relatório sejam 100% reais e exatos.

---

### Bloco 1: Ativando as Ferramentas de Trabalho (Importações)

Depois de instalar os programas no computador virtual do Google, nós precisamos dizer ao nosso código quais peças exatas nós vamos usar durante o expediente. Na programação, chamamos essa ação de "importar".

Copie o código abaixo e execute na próxima célula do seu Colab:

```python
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
import telebot
from agno.agent import Agent
from agno.models.openai import OpenAIChat
from agno.tools.python import PythonTools

# Importando o "caderno de anotações" do Analista
from agno.db.sqlite import SqliteDb
```
#### Entendendo as categorias de ferramentas:

Para manter o projeto organizado, nós dividimos as ativações em pequenos grupos lógicos que representam as habilidades do nosso Analista de Dados:

* **Sistema e Segurança (os, userdata, drive)**: São os comandos que permitem ao código acessar o cofre de senhas ocultas do Google Colab e estabelecer a conexão segura com o seu Google Drive (onde a planilha oficial da empresa ficará guardada).

* **Análise de Dados (pandas)**: É a habilidade matemática. O pandas é a ferramenta que permite ao sistema abrir planilhas, ler as colunas e realizar cruzamentos de dados com precisão absoluta.

* **Envio de E-mail (smtplib, email)**: São os protocolos de rede padrão. Eles ensinam o sistema a estruturar um e-mail (com remetente, assunto e corpo do texto) e fazer a entrega através dos servidores do Gmail.

* **O Cérebro e a Memória (agno, telebot)**: Aqui ativamos o aplicativo do Telegram e a Inteligência Artificial.

* **O destaque técnico deste grupo é o PythonTools**: ele concede à Inteligência Artificial a permissão para escrever e executar códigos de cálculo sozinha, garantindo que ela não tente adivinhar os números.

* **O SqliteDb é o nosso banco de dados**. Ele funciona como um caderno físico de anotações, garantindo que o Analista tenha memória das mensagens anteriores e não esqueça o assunto da reunião a cada nova pergunta.

---

### Bloco 2: Abrindo o Arquivo Confidencial (Conexão com o Google Drive)

O Google Colab é um computador "emprestado" e temporário. Se nós colocarmos a nossa base de dados oficial solta nele, ela será apagada assim que fecharmos o navegador.   

Para garantir que o nosso Analista de Dados tenha acesso permanente à planilha financeira da empresa e consiga salvar o seu caderno de anotações (memória) de forma segura, nós precisamos conectar este ambiente virtual diretamente ao seu **Google Drive**.

Copie o código abaixo e execute na próxima célula:

```python
# Conecta o ambiente virtual ao Google Drive
drive.mount('/content/drive')
```
#### O que vai acontecer ao rodar este código?

* **Pedido de Autorização**: O Google Colab vai pausar e exibir uma janela pop-up perguntando se você permite que este notebook acesse os seus arquivos do Google Drive.

* **Confirmação de Segurança**: Você deve clicar em "Permitir", escolher a sua conta do Google e confirmar os avisos de segurança.

* **Acesso Liberado**: Após alguns segundos, aparecerá a mensagem Mounted at /content/drive. Isso significa que a gaveta foi aberta com sucesso. A partir de agora, o nosso código em Python consegue ler as suas planilhas e salvar arquivos diretamente nas suas pastas na nuvem, garantindo que o cérebro do projeto não sofra de amnésia quando o expediente acabar.

---

### Bloco 3: Localizando a Base de Dados (O Endereço do Arquivo)

No bloco anterior, nós destrancamos a sua conta do Google Drive. Agora, precisamos dizer ao código exatamente em qual "prateleira" e em qual "pasta" está o documento oficial que ele deve analisar. Na programação, nós chamamos isso de "Caminho do Arquivo" (File Path).

Copie o código abaixo para a sua próxima célula. **Mas atenção: você precisará alterar o texto entre aspas para o seu endereço real.**

```python
# Define o caminho exato onde a planilha corporativa está salva
caminho_planilha_drive = "/content/drive/MyDrive/base_de_dados/imoveis_ficticios.csv"

print("Caminho do Google Drive configurado com sucesso.")
```

#### Como encontrar o caminho correto do SEU arquivo:

É muito comum o código dar erro nesta etapa dizendo que "o arquivo não foi encontrado". Isso acontece porque o caminho `/base_de_dados/imoveis_ficticios.csv` é apenas um exemplo. Para o projeto funcionar, você precisa colocar o endereço exato de onde você salvou a sua planilha.

Acessando o arquivo pelo Google Colab:

1. No menu esquerdo do Colab, clique no ícone de Pasta (Files).
2. Clique na pasta `drive`, depois abra `MyDrive`.
3. Navegue pelas suas pastas até encontrar o seu arquivo `.csv`.
4. Clique nos três pontinhos ao lado do nome do arquivo e escolha Copiar caminho (Copy path).
5. Cole esse caminho exato dentro das aspas do código acima.

Pronto! Agora o Analista de Dados tem acesso a sua pasta e sabe exatamente qual documento abrir quando pedir um cálculo.

---
### Bloco 4: A Habilidade de Comunicação (Enviando E-mails)

Agora que o nosso Analista sabe onde a planilha está guardada, ele precisa de um meio oficial para entregar os resultados. Neste bloco, nós vamos criar uma ferramenta (uma *Tool*) que ensina o sistema a redigir e enviar um e-mail corporativo de forma totalmente autônoma através do Gmail.

Copie e execute o código abaixo na próxima célula:

```python
def enviar_email(destinatario: str, assunto: str, corpo: str) -> str:
    """Envia um e-mail com o relatório de dados usando o servidor SMTP do Gmail.

    Args:
        destinatario: O endereço de e-mail do diretor ou setor.
        assunto: O assunto do relatório financeiro/analítico.
        corpo: O texto do relatório com os insights estratégicos.
    """
    EMAIL_SENDER = os.environ.get("EMAIL_SENDER")
    EMAIL_PASSWORD = os.environ.get("EMAIL_PASSWORD")

    msg = MIMEMultipart()
    msg['From'] = EMAIL_SENDER
    msg['To'] = destinatario
    msg['Subject'] = assunto
    msg.attach(MIMEText(corpo, 'plain'))

    # Conecta no servidor SMTP e processa o envio
    server = smtplib.SMTP('smtp.gmail.com', 587)
    server.starttls()
    server.login(EMAIL_SENDER, EMAIL_PASSWORD)
    server.sendmail(EMAIL_SENDER, destinatario, msg.as_string())
    server.quit()

    return f"Relatório enviado com sucesso para {destinatario}."
```

#### Como a Inteligência Artificial usa esse código?

* **O Manual de Instruções (Docstring)**: Aquele texto longo no início do código (entre as três aspas duplas """) não é apenas um comentário para humanos. É o manual que a Inteligência Artificial vai ler! É graças a esse texto que o robô entende que, se você pedir no Telegram "Envie isso para o João", ele deve usar esta exata função.

* **Segurança Total**: As linhas `os.environ.get` garantem que o código vá buscar o seu e-mail e aquela Senha de Aplicativo de 16 letras diretamente no cofre seguro do Google Colab. As suas credenciais nunca ficam expostas no texto.

* **O Retorno de Sucesso**: A última linha do código (`return`) funciona como um aviso de entrega. O sistema do Gmail avisa a Inteligência Artificial que o e-mail foi despachado, e a IA, por sua vez, pode te avisar no Telegram: "Chefe, e-mail enviado com sucesso!".

---
### Bloco 5: Mapeando a Base de Dados (O Índice do Arquivo)

Antes de contratar o nosso Analista, precisamos entender o que tem dentro daquela planilha do Google Drive. Porém, planilhas corporativas podem ter milhões de linhas. Se o sistema tentar ler tudo de uma vez, ele vai travar e consumir muitos recursos financeiros. 

A solução inteligente é ler **apenas o cabeçalho** (os nomes das colunas). É como ler o índice de um livro em vez de ler todas as páginas.

Copie o código abaixo e execute na próxima célula:

```python
# Mapeia a estrutura da planilha uma única vez (Lê apenas os títulos das colunas)
df = pd.read_csv(caminho_planilha_drive, nrows=0, sep=',')
colunas_da_planilha = list(df.columns)

print(f"Estrutura da planilha mapeada com sucesso!")
print(f"Colunas encontradas: {colunas_da_planilha}")
```

O que esse código faz?

* O comando `nrows=0` garante que o sistema baixe exatamente zero linhas de dados, puxando apenas a primeira linha lá de cima (os títulos).
* O print final vai exibir na tela do Colab os nomes das colunas que ele encontrou (ex: Preço, Categoria, Quartos, etc.), provando que a conexão com o arquivo está funcionando perfeitamente.

---
### Bloco 6: Contratando o Analista e Ativando a Memória (O Agente)

Agora que sabemos quais dados temos, é hora de dar vida ao nosso funcionário digital. Aqui nós criamos o Agent (O Agente de Inteligência Artificial). Nós vamos dar a ele uma profissão, entregar as ferramentas matemáticas e de e-mail, apresentar as colunas da planilha e dar a ele um caderno de anotações para que ele tenha memória das reuniões.

Copie o código abaixo e execute na próxima célula:

```python
# Cria o analista AQUI FORA para garantir que ele lembre das mensagens
analista_de_dados = Agent(
    model=OpenAIChat(id="gpt-4o", temperature=0.0),
    tools=[PythonTools(), enviar_email],
    markdown=True,

    # --- AS DUAS LINHAS DA MEMÓRIA ---
    db=SqliteDb(db_file="memoria_do_analista.db"), # Entrega o caderninho físico
    add_history_to_context=True,                   # Dá a ordem para ele ler o caderninho

    role="Cientista de Dados Sênior e Analista Financeiro",
    description="Analisar bases de dados executando scripts em Python para responder a perguntas corporativas.",
    instructions=[
        "Seu público são Diretores Executivos que precisam de decisões baseadas em dados exatos.",
        f"Sempre que precisar calcular dados, escreva um código Python usando 'pandas'. O arquivo está no caminho: '{caminho_planilha_drive}'.",
        f"As colunas numéricas e de texto disponíveis nesta planilha são: {colunas_da_planilha}.",
        "Estruture relatórios analíticos em:\n**1. Resumo da Análise:**\n**2. Resultado Executivo:**\n**3. Insights Estratégicos:**",
        "REGRA DE PROTEÇÃO: Nunca imprima a base de dados inteira no sistema. Extraia e retorne apenas os números finais.",
        "Se o usuário pedir para enviar a análise por e-mail, NÃO recalcule nada. Apenas pegue o texto da sua última resposta no histórico e use a ferramenta de e-mail."
    ]
)

print("Analista de Dados configurado, com caderninho na mão e memória ativada.")
```
#### Dissecando o Cérebro do Analista

* **A Mente Fria (`temperature=0.0`)**: Modelos de IA têm uma configuração de "Criatividade" que vai de 0 a 1. Como lidamos com dados exatos, travamos a temperatura no zero absoluto. Isso impede que a IA invente números (alucinação) e a força a ser estritamente analítica.

* **O Caderno de Anotações (`SqliteDb`)**: Por padrão, as IAs têm amnésia. Ao adicionar o banco de dados e a regra add_history, damos ao Analista uma memória. É isso que permite que você peça um cálculo e, depois, diga apenas "Envie isso por e-mail", e ele saberá do que você está falando.

* **As Instruções (`A Regra de Ouro`)**: Definimos o tom de voz corporativo, blindamos o sistema contra travamentos ("Nunca imprima a base inteira") e ensinamos a estrutura exata do relatório que a diretoria espera receber. Além disso, passamos a variável colunas_da_planilha para que ele saiba exatamente o que pode pesquisar.

---
### Bloco 7: Colocando o Analista Online (A Conexão com o Telegram)

Chegamos à última etapa do projeto! Nosso funcionário digital já está treinado e com acesso aos dados. Agora, precisamos ligar o canal de comunicação para que ele possa receber as perguntas e enviar os relatórios diretamente para o seu celular.

Neste bloco, vamos configurar a interface do Telegram e criar um comando de limpeza de memória para iniciar novas reuniões sem misturar os assuntos.

Copie o código abaixo e execute na sua última célula:

```python
# Inicializa a conexão com a interface do Telegram usando a senha do cofre
bot = telebot.TeleBot(os.environ["TELEGRAM_TOKEN"])

# --- REGRA 1: O botão de Reset (Comando /limpar) ---
@bot.message_handler(commands=['limpar'])
def limpar_memoria(message):
    # O comando abaixo apaga as anotações recentes da mente do Analista
    analista_de_dados.memory.clear()
    bot.reply_to(message, "Memória limpa com sucesso! O contexto anterior foi apagado. O que vamos analisar agora?")

# --- REGRA 2: O atendimento normal (As perguntas da diretoria) ---
@bot.message_handler(func=lambda message: True)
def processar_pedido(message):
    # Apenas ativa o status visual de "digitando..." no topo da tela do Telegram
    bot.send_chat_action(message.chat.id, 'typing')

    # Repassa a instrução do usuário para o analista de dados pensar e calcular
    resposta_do_analista = analista_de_dados.run(message.text)

    # Retorna o relatório final formatado diretamente no chat
    bot.reply_to(message, resposta_do_analista.content)

print("Painel da Diretoria online! Digite /limpar no Telegram para zerar a reunião ou faça uma pergunta.")

# Mantém o sistema em execução contínua, aguardando mensagens
bot.infinity_polling()
```

#### Entendendo a Central de Atendimento:

* **O Status de "Digitando..." (`send_chat_action`)**: Em vez de enviar mensagens robóticas de "processando", nós usamos uma função nativa que faz aparecer a palavra digitando... no topo do Telegram. Isso cria uma experiência de usuário (UX) muito mais natural, indicando que o sistema está calculando os dados sem poluir a tela.

* **A Passagem de Bastão (`analista_de_dados.run`)**: O Telegram em si não é inteligente; ele é apenas o mensageiro. Esta linha de código pega a sua pergunta em português e a entrega para o cérebro da nossa Inteligência Artificial resolver.

* **A Gestão de Memória (`/limpar`)**: Como o nosso sistema tem memória de longo prazo, ele acumula o contexto da conversa. Em um ambiente corporativo, se terminamos de analisar o "Preço dos Imóveis" e vamos passar para "Despesas de Marketing", não queremos que a IA misture os dados. O comando `/limpar` arranca as páginas antigas do caderno de anotações e permite que a próxima análise comece 100% focada.

* **O Plantão 24h (`infinity_polling`)**: É o comando que impede o programa de desligar. Ele avisa ao Google Colab: "Fique com os ouvidos abertos indefinidamente, aguardando a próxima notificação do celular".

Exemplo:
1. *Mande no Telegram:* "Qual o preço médio dos imóveis?"
2. *Depois mande:* "Envie este relatório para [seu email]"
3. *Por fim mande:* "/limpar"

### Resultado

<img width="1083" height="978" alt="image" src="https://github.com/user-attachments/assets/401fd8d6-993e-4174-b18f-a73d1ca8db54" />

### Bloco Auxiliar: Upgrade de Memória (Salvando no Google Drive)

No formato padrão, o nosso Analista guarda o caderno de anotações (memoria_do_analista.db) na pasta temporária do Google Colab. Isso significa que, se você fechar a aba do navegador, a equipe de limpeza do Google apaga o arquivo e o seu bot perde toda a memória das reuniões passadas.

Para um ambiente corporativo, nós queremos que essa memória seja permanente. Neste bloco auxiliar, nós vamos atualizar o código do nosso Agente para que ele guarde o caderno diretamente dentro da gaveta segura do seu Google Drive.

```python
# 1. Definimos o caminho do caderninho direto no Google Drive
# ATENÇÃO: Crie a pasta 'Assistente_Escola' no seu Drive ou mude o nome abaixo para uma pasta sua.
caminho_memoria_drive = "/content/drive/MyDrive/Assistente_Escola/memoria_do_analista.db"

# Mapeia a estrutura da planilha
df = pd.read_csv(caminho_planilha_drive, nrows=0, sep=',')
colunas_da_planilha = list(df.columns)

# Cria o analista com memória persistente
analista_de_dados = Agent(
    model=OpenAIChat(id="gpt-4o", temperature=0.0),
    tools=[PythonTools(), enviar_email],
    markdown=True,

    # 2. Guardamos o banco de dados na pasta do Drive!
    db=SqliteDb(db_file=caminho_memoria_drive),
    add_history_to_context=True,

    role="Cientista de Dados Sênior e Analista Financeiro",
    description="Analisar bases de dados executando scripts em Python para responder a perguntas corporativas.",
    instructions=[
        "Seu público são Diretores Executivos que precisam de decisões baseadas em dados exatos.",
        f"Sempre que precisar calcular dados, escreva um código Python usando 'pandas'. O arquivo está no caminho: '{caminho_planilha_drive}'.",
        f"As colunas numéricas e de texto disponíveis nesta planilha são: {colunas_da_planilha}.",
        "Estruture relatórios analíticos em:\n**1. Resumo da Análise:**\n**2. Resultado Executivo:**\n**3. Insights Estratégicos:**",
        "REGRA DE PROTEÇÃO: Nunca imprima a base de dados inteira no sistema. Extraia e retorne apenas os números finais.",
        "Se o usuário pedir para enviar a análise por e-mail, NÃO recalcule nada. Apenas pegue o texto da sua última resposta no histórico e use a ferramenta de e-mail."
    ]
)

print("Analista de Dados configurado e memória salva permanentemente no Google Drive.")
```

#### Entendendo a principal mudança

* **A Variável de Caminho (caminho_memoria_drive)**: Em vez de deixar o arquivo solto, nós indicamos o endereço exato do Google Drive onde o arquivo .db deve ser criado e atualizado.

* **A Segurança a Longo Prazo**: Agora, mesmo que você desligue o computador no fim de semana, na segunda-feira o Analista vai abrir a gaveta do Drive, ler o banco de dados e lembrar perfeitamente de tudo o que a diretoria pediu na semana anterior.
