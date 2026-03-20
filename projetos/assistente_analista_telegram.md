# Desenvolvendo um Analista de Dados com Agno e Telegram

O projeto prático tem como objetivo integrar aplicações de IA Generativa com ferramentas de mensagens, podendo ser aplicadas ao Telegram, WhatsApp, Instagram, etc. 
A proposta é que o usuário possa criar assistentes usando frameworks de IA para realizar determinadas funções, como analisar dados, acessar e-mails, gerenciar agendas e cadastrar informações em bancos de dados.

## Pré-requisitos do Projeto

Antes de iniciarmos a construção do nosso Analista de Dados, você precisará preparar o seu ambiente de trabalho e ter em mãos algumas chaves de acesso essenciais. Pense nesta etapa como a separação dos ingredientes antes de começar a cozinhar.

### 1. Contas e Credenciais Necessárias

Você precisará criar contas gratuitas nas seguintes plataformas para gerar as suas chaves de acesso (Senhas). Copie essas chaves e guarde-as em um bloco de notas temporário:

* **Telegram (`TELEGRAM_TOKEN`)**: Necessário para criar o assistente no aplicativo. Você consegue essa chave conversando com o robô oficial chamado BotFather no próprio Telegram.
  
  1. **BotFather (Criação do Assistente e Senha de Acesso)**: O BotFather (o "Pai dos Bots") é o robô oficial do próprio Telegram responsável por criar novos assistentes virtuais. Você vai procurá-lo na barra de busca do aplicativo. É conversando com ele que você dará um nome e um perfil para o seu Analista de Dados. Ao final da criação, o BotFather te entregará o Token: uma senha única e longa que serve para conectar o nosso sistema de Inteligência Artificial ao aplicativo do Telegram.  
  <img width="1076" height="327" alt="image" src="https://github.com/user-attachments/assets/a5e73e28-5f95-4c6a-abfa-aab8e5bc9867" />

  2. **Get My ID Bot (Identificação de Segurança)**: O Get My ID é uma ferramenta de busca do Telegram que revela o seu número de identificação pessoal (o seu ID). Como o seu Analista de Dados lidará com planilhas corporativas e envios de e-mail, esse ID funciona como o seu "crachá de diretoria". Saber o seu próprio ID permite que, no futuro, nós possamos ensinar o sistema a obedecer e enviar relatórios exclusivamente para você, ignorando mensagens de pessoas desconhecidas.  
  <img width="1076" height="327" alt="image" src="https://github.com/user-attachments/assets/75df92c5-1982-4ca7-bbe3-66e425973dc6" />

* **OpenAI (`OPENAI_API_KEY`)**: A chave de acesso ao "cérebro" do ChatGPT. Gerada no painel de desenvolvedores da OpenAI.
* **Groq (`GROQ_API_KEY`)**: (Opcional) Uma chave alternativa de Inteligência Artificial, caso você prefira usar modelos de código aberto ultrarrápidos em vez da OpenAI.
* **Gmail corporativo ou pessoal**:
  * **`EMAIL_SENDER`**: O seu endereço de e-mail completo (ex: seu.nome@gmail.com).
  * **`EMAIL_PASSWORD`**: Atenção: Esta não é a senha normal que você usa para entrar no e-mail. É uma "Senha de Aplicativo" específica de 16 letras, gerada nas configurações de segurança da sua Conta Google, que permite que o nosso sistema envie mensagens com segurança.

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

### 3. Instalando os Pacotes de Software

Com as senhas guardadas no cofre, o primeiro passo no código é instalar os programas de Inteligência Artificial, leitura de dados e comunicação.

Adicione o seguinte código na sua primeira célula do Google Colab e execute:

```python
# Instala os pacotes de Inteligência Artificial, Telegram e manipulação de dados
!pip install -q agno openai pyTelegramBotAPI pandas
