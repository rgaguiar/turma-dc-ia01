# Aula Prática: N8N + Google Sheets — Agente que Registra Pedidos

**Módulo:** 5 — Automação de Fluxos de Trabalho  
**Aula:** 18 (Aula 2 do módulo de Automação)  
**Ambiente:** Docker Desktop + N8N + Google Cloud  
**Última revisão:** 2025-05  

---

Na aula passada, instalamos o N8N via Docker e construímos o primeiro AI Agent com Groq + memória + calculadora. Também deixamos a imagem do WAHA pronta nos computadores para a próxima semana.

Nesta aula, o agente ganha uma ferramenta real de negócio: ele conversa com o cliente, anota o pedido e **registra automaticamente numa planilha do Google Sheets** — sem intervenção humana.

O projeto é o **Marmitex Bot**: um atendente virtual de marmitaria que coleta nome, endereço, telefone e pedido, calcula o valor e salva tudo na planilha.

> 📌 **WAHA e WhatsApp:** vamos usar o chat interno do N8N hoje. O WAHA já está instalado nas máquinas — conectamos ao WhatsApp na próxima semana.

---

## 1. A Arquitetura da Aula

```
[When chat message received]  ← trigger interno do N8N
            |
            v
        [AI Agent]  ← Marmitex Bot (GPT-4.1 mini)
            |
     +------+------+----------+
     |             |          |
[OpenAI]    [Simple      [Calculator]
[Chat Model] Memory]
                             |
                    [Google Sheets Tool]
                    Registra o pedido
                    na planilha
```

### Os nós da aplicação

| No | Tipo | Funcao |
| --- | --- | --- |
| **When chat message received** | Trigger | Recebe a mensagem do chat interno do N8N |
| **AI Agent** | Agente | Orquestra o atendimento conforme o prompt |
| **OpenAI Chat Model** | LLM | GPT-4.1 mini — processa e gera as respostas |
| **Simple Memory** | Memoria | Guarda o historico da conversa (10 turnos) |
| **Calculator** | Tool | Calcula o valor total do pedido |
| **Google Sheets Tool** | Tool | Registra o pedido finalizado na planilha |

---

## 2. Configurando o Google Cloud Console

Antes de usar o Google Sheets no N8N, precisamos criar uma **Service Account** — uma conta de servico que permite ao N8N escrever na planilha sem precisar de login humano.

> 💡 **O que e uma Service Account?** E como criar um funcionario digital com uma chave propria. Voce da permissao para esse funcionario acessar a planilha, e o N8N usa a chave dele para escrever os dados automaticamente.

### Passo 1: Ativar a Drive API

1. Acessar [console.cloud.google.com](https://console.cloud.google.com)
2. Selecionar o projeto (ou criar um novo)
3. Menu lateral → **APIs & Services** → **Library**
4. Buscar `Google Drive API`
5. Clicar em **Enable**

### Passo 2: Ativar a Sheets API

1. Ainda em **APIs & Services** → **Library**
2. Buscar `Google Sheets API`
3. Clicar em **Enable**

> 📌 **Por que as duas APIs?** O N8N usa a Drive API para listar as planilhas disponiveis e a Sheets API para ler e escrever os dados.

### Passo 3: Criar a Service Account

1. Menu lateral → **APIs & Services** → **Credentials**
2. Clicar em **+ Create Credentials** → **Service Account**
3. Preencher:

| Campo | Valor |
| --- | --- |
| Service account name | `aula-n8n` |
| Service account ID | gerado automaticamente |
| Description | (opcional) |

4. Clicar em **Create and continue**
5. Em **Grant this service account access to project** → selecionar role: **Basic → Owner**
6. Clicar em **Continue** → **Done**

### Passo 4: Gerar a Chave JSON

1. Na lista de Service Accounts, clicar no email gerado (ex: `aula-n8n@seu-projeto.iam.gserviceaccount.com`)
2. Aba **Keys** → **Add Key** → **Create new key**
3. Selecionar tipo **JSON**
4. Clicar em **Create**
5. O arquivo `.json` sera baixado automaticamente — **guarde-o em local seguro**

> ⚠️ **O arquivo JSON e a senha da Service Account.** Nao compartilhe publicamente. Se vazar, qualquer pessoa pode escrever na sua planilha.

---

## 3. Criando a Planilha de Pedidos

### Passo 5: Criar a planilha no Google Sheets

1. Acessar [sheets.google.com](https://sheets.google.com)
2. Criar uma nova planilha
3. Renomear para: **pedidos-aula**
4. Criar as seguintes colunas na linha 1:

| A | B | C | D | E |
| --- | --- | --- | --- | --- |
| Nome | Telefone | Endereco | Pedido | Data |

### Passo 6: Compartilhar com a Service Account

1. Na planilha, clicar em **Compartilhar** (botao verde, canto superior direito)
2. No campo de email, colar o email da Service Account:

```
aula-n8n@seu-projeto.iam.gserviceaccount.com
```

3. Permissao: **Editor**
4. Desmarcar "Notificar pessoas"
5. Clicar em **Compartilhar**

> 📌 **Este passo e obrigatorio.** Sem ele, o N8N nao consegue escrever na planilha mesmo com a chave JSON correta — vai retornar erro de permissao negada.

---

## 4. Cadastrando as Credenciais no N8N

### Passo 7: Adicionar a credencial do Google Sheets

1. No N8N → menu lateral → **Settings** → **Credentials**
2. Clicar em **+ Add Credential**
3. Buscar e selecionar **Google Sheets**
4. Em **Authentication**, selecionar **Service Account**
5. Abrir o arquivo JSON baixado e copiar os campos:

| Campo no N8N | Campo no JSON |
| --- | --- |
| Service Account Email | `client_email` |
| Private Key | `private_key` |

6. Clicar em **Save**
7. O N8N vai testar a conexao — deve aparecer **Connection tested successfully**

---

## 5. Construindo o Workflow

### Passo 8: Criar o workflow no N8N

Crie um novo workflow chamado **Marmitex Bot**.

#### No 1: When chat message received

Trigger interno do N8N — abre o chat no proprio navegador para testar.

- Nao ha configuracao necessaria

#### No 2: AI Agent

- Clicar em **+ Add first step** → buscar **AI Agent**
- Em **Prompt type**: selecionar **Define below**
- No campo **Text**: colar o prompt completo (veja secao 6)

#### Sub-no: OpenAI Chat Model

Clicar no **+** abaixo do AI Agent → selecionar **OpenAI Chat Model**

| Campo | Valor |
| --- | --- |
| Model | `gpt-4.1-mini` |
| Credential | sua conta OpenAI |

#### Sub-no: Simple Memory

Clicar no **+** abaixo do AI Agent → selecionar **Simple Memory** (Window Buffer Memory)

| Campo | Valor |
| --- | --- |
| Context Window Length | `10` |

#### Sub-no: Calculator

Clicar no **+** abaixo do AI Agent → buscar **Calculator**

- Nao ha configuracao necessaria — o agente usa quando precisar calcular valores

#### Sub-no: Google Sheets Tool

Clicar no **+** abaixo do AI Agent → buscar **Google Sheets** → selecionar a versao **Tool**

| Campo | Valor |
| --- | --- |
| Authentication | Service Account |
| Credential | a credencial criada no Passo 7 |
| Operation | **Append row** |
| Document | `pedidos-aula` (selecionar na lista) |
| Sheet | `Sheet1` |

**Configurar as colunas usando IA:**

Para cada coluna, clicar em **Use AI to describe** e adicionar uma descricao:

| Coluna | Descricao para a IA |
| --- | --- |
| Nome | `nome do cliente` |
| Telefone | `contato de telefone do cliente` |
| Endereco | `endereco de entrega do pedido do cliente` |
| Pedido | `descricao do pedido do cliente e valor total` |
| Data | `={{ $now.format('dd-MM-yyyy HH:mm') }}` (expressao direta) |

> 💡 **Por que "Use AI to describe"?** Em vez de mapear campos fixos, voce descreve em linguagem natural o que cada coluna significa. O agente extrai os dados da conversa com base nessa descricao — muito mais flexivel do que mapeamento rigido.

---

## 6. O Prompt do Marmitex Bot

Este e o prompt completo do agente. Cole no campo **Text** do no AI Agent:

```
# Perfil e Objetivo

Voce e o Marmitex Bot, um assistente virtual simpatico e focado no atendimento
ao cliente para anotacao de pedidos de refeicoes.

Sua unica funcao e ajudar o cliente a escolher o almoco, registrar o pedido
corretamente, coletar os dados necessarios e confirmar o pedido.

# Cardapio do Dia

## Tamanhos e Valores

### Marmita Media — R$ 18,00
- Da direito a 1 mistura e acompanhamentos.

### Marmita Grande — R$ 22,00
- Da direito a ate 2 misturas e acompanhamentos.

## Misturas do Dia
- Frango Grelhado
- Feijoada Completa
- Carne de Panela
- Omelete de Queijo com Ervas (Vegetariana)

## Acompanhamentos (padrao — cliente pode alterar)
- Arroz Branco ou Integral
- Feijao Carioca ou Preto
- Macarrao Espaguete
- Farofa da Casa
- Salada Verde

## Bebidas e Extras
- Refrigerante 2LT (Coca-Cola ou Guarana) — R$ 5,00
- Suco Natural de Laranja (300ml) — R$ 6,00
- Ovo Frito Extra — R$ 2,00

# Regras de Atendimento
- Seja cordial, educado e objetivo.
- Faca apenas uma pergunta por mensagem.
- Nunca peca novamente uma informacao ja informada.
- Nunca invente produtos, precos ou informacoes do pedido.
- Sempre exiba precos ao apresentar opcoes do cardapio.

# Dados Obrigatorios
O atendimento somente podera ser encerrado quando coletados:
- nome
- endereco
- pedido
- telefone

# Fluxo da Conversa
1. Cumprimente o cliente.
2. Apresente as opcoes de marmita com precos.
3. Identifique a mistura desejada.
4. Confirme o tamanho e a quantidade.
5. Verifique alteracoes nos acompanhamentos.
6. Oferte bebidas e extras com precos.
7. Solicite os dados obrigatorios faltantes.
8. Confirme o resumo do pedido.
9. Registre na ferramenta Google Sheets.
10. Finalize o atendimento.

# IMPORTANTE
Apos coletar todos os dados obrigatorios (nome, endereco, pedido e telefone),
registre o pedido na ferramenta Google Sheets antes de finalizar o atendimento.

Somente apos o registro ser realizado com sucesso, informe que o pedido foi
enviado para a cozinha.

# Finalizacao

Quando todos os dados estiverem coletados e o pedido registrado:
1. Agradeca ao cliente.
2. Informe que o pedido foi enviado para a cozinha.
3. Informe o valor total.
4. Exiba o resumo no formato abaixo.

nome: [Nome do Cliente]
endereco: [Endereco Completo]
pedido: [Quantidade x Tamanho - Mistura(s) - Observacoes]
valor: [Valor Total]
telefone: [Telefone]

# Pedido do Cliente

{{ $json.chatInput }}
```

---

## 7. Testando o Sistema

Com o workflow ativo (toggle **Inactive → Active**), clicar em **Chat** no canto inferior esquerdo.

```
# TESTE 1 — Fluxo completo
Voce: "oi"
Esperado: saudacao + cardapio com precos

Voce: "quero uma marmita grande de frango grelhado"
Esperado: confirmacao do tamanho e pergunta sobre acompanhamentos

Voce: "sem feijao"
Esperado: anotado + oferta de bebidas

Voce: "nao quero bebida"
Esperado: solicita nome

Voce: "Rafael"
Esperado: solicita endereco

Voce: "Rua das Flores 123, Fortaleza"
Esperado: solicita telefone

Voce: "85 99999-9999"
Esperado: resumo do pedido + confirmacao

Voce: "confirmo"
Esperado: agente registra no Google Sheets + mensagem de confirmacao

# VERIFICAR NA PLANILHA
- Abrir pedidos-aula no Google Sheets
- Deve aparecer uma nova linha com: Nome, Telefone, Endereco, Pedido, Data

# TESTE 2 — Item fora do cardapio
Voce: "quero uma pizza"
Esperado: "esse item nao esta disponivel hoje"

# TESTE 3 — Calculo com Calculator
Voce: "quero 2 marmitas grandes e 1 suco"
Esperado: agente usa a Calculator e retorna R$ 50,00 (2x22 + 6)
```

---

## 8. Diferenca entre esta Aula e a Anterior

| Aula 17 (semana passada) | Aula 18 (hoje) |
| --- | --- |
| AI Agent com Groq + Calculator | AI Agent com OpenAI + Calculator + Google Sheets |
| Sem integracao externa | Integrado ao Google Sheets via Service Account |
| Agente generico | Agente especializado (Marmitex Bot) |
| Prompt simples | Prompt estruturado com regras de negocio |
| Sem registro de dados | Registro automatico na planilha |

> 💡 **O padrao se repete:** todo agente especializado tem um prompt estruturado, ferramentas especificas e um destino para os dados. Hoje foi Google Sheets — na proxima semana pode ser CRM, e-mail ou WhatsApp.

---

## 9. Atividade Extra

Agora que o bot esta funcionando, tente evoluir a aplicacao com o desafio abaixo antes da proxima aula.

### Desafio: adicionar o campo Valor na planilha

A planilha atual registra o pedido em texto, mas nao tem um campo numerico separado para o valor total. Isso dificulta calcular o faturamento do dia com formulas do Sheets.

**O que fazer:**

**1. Adicionar a coluna na planilha**

Na planilha `pedidos-aula`, adicione uma nova coluna `F` com o cabecalho:

```
F — Valor
```

**2. Atualizar a tool Google Sheets no N8N**

Abra o no `Google Sheets Tool` e adicione o novo campo:

| Coluna | Descricao para a IA |
| --- | --- |
| Valor | `valor total do pedido em reais, somente o numero` |

> 📌 Use a opcao **Use AI to describe** igual ao que fizemos nas outras colunas. O agente vai extrair o valor numerico do resumo do pedido.

**3. Testar**

Faca um novo pedido completo e verifique se a coluna Valor aparece preenchida na planilha — por exemplo `22.00` para uma marmita grande.

**4. Bonus: formula de faturamento**

Com o campo numerico disponivel, tente criar em uma aba separada da planilha uma formula de soma do faturamento do dia:

```
=SUMIF(E:E, TEXT(TODAY(),"dd-MM-yyyy")&"*", F:F)
```

Essa formula soma todos os valores da coluna F onde a data (coluna E) e a de hoje.

> 💡 **Por que isso importa?** Com o campo Valor separado, o dono da marmitaria consegue ver o faturamento do dia automaticamente — sem precisar ler o texto de cada pedido. Essa e a diferenca entre um dado util e um dado apenas registrado.

---

## 10. Proxima Semana — Parte 3

Tres melhorias para a proxima aula, em ordem de complexidade:

### Memoria mais robusta

O `Simple Memory` atual perde o historico quando o N8N reinicia. Para um bot de pedidos que roda o dia todo, isso e um problema — o cliente precisa repetir o que ja disse.

**O que vamos fazer:** substituir o `Simple Memory` pelo **Postgres Chat Memory** com um container PostgreSQL no Docker. O historico fica salvo em banco de dados e sobrevive a reinicializacoes.

```
Simple Memory (RAM)        →   Postgres Chat Memory (banco de dados)
Perde ao reiniciar         →   Persiste entre sessoes
Limite de contexto fixo    →   Historico consultavel e persistente
```

### RAG do Cardapio por Dia da Semana

O cardapio atual e fixo no prompt. Na pratica, marmitarias mudam o cardapio todo dia — e as vezes nao funcionam aos fins de semana.

**O que vamos fazer:** criar um documento com o cardapio por dia da semana e os horarios de funcionamento. O agente vai consultar esse documento via RAG antes de apresentar as opcoes.

```
Segunda: Frango + Carne de Panela
Terca: Peixe + Omelete
...
Sabado e Domingo: fechado

Se o cliente pedir num dia que nao ha funcionamento:
→ Agente informa educadamente e sugere o proximo dia util
```

**Ferramentas:** documento de cardapio indexado no ChromaDB ou Supabase + no `Vector Store Tool` conectado ao AI Agent.

### Conectar ao WhatsApp via WAHA

Com a memoria e o cardapio corretos, conectamos o Marmitex Bot ao WhatsApp. A imagem do WAHA ja esta baixada nas maquinas.

**O que muda no workflow:**

| Hoje (chat interno) | Proxima semana (WhatsApp) |
| --- | --- |
| `When chat message received` | `WAHA Trigger` (saida 1: message) |
| — | `IF` — filtra grupos (`@g.us`) |
| — | `Edit Fields` — extrai `from`, `body`, `session` |
| — | `Send Seen` — marca como lido |
| `AI Agent` (mesmo) | `AI Agent` (mesmo prompt) |
| — | `Send Text Message` — responde no WhatsApp |

O registro no Google Sheets continua igual — nenhuma mudanca na tool ou na planilha.

---

## 11. Problemas Comuns

| Problema | Causa | Solucao |
| --- | --- | --- |
| `Connection tested successfully` nao aparece | Credencial incorreta | Verifique `client_email` e `private_key` do JSON |
| `Permission denied` ao salvar na planilha | Planilha nao compartilhada | Compartilhe com o email da Service Account como Editor |
| Agente nao registra na planilha | Prompt nao instrui o agente a usar a tool | Verifique se a instrucao de registro esta no prompt |
| Planilha nao aparece na lista do N8N | Drive API nao ativada | Ative a Google Drive API no Google Cloud Console |
| Dados errados na planilha | Descricao dos campos pouco clara | Melhore a descricao no campo "Use AI to describe" |
| Agente pergunta dados ja informados | `contextWindowLength` muito baixo | Aumente para 15 ou 20 no Simple Memory |

---

## 12. Gabarito — Estrutura do Workflow JSON

O workflow completo esta disponivel no repositorio da turma para importar diretamente no N8N.

**Nos do workflow:**

```
When chat message received
    └── AI Agent (prompt do Marmitex Bot)
            ├── OpenAI Chat Model (gpt-4.1-mini)
            ├── Simple Memory (contextWindowLength: 10)
            ├── Calculator
            └── Google Sheets Tool
                    ├── Operation: Append row
                    ├── Document: pedidos-aula
                    └── Colunas: Nome, Telefone, Endereco, Pedido, Data
```

**Expressao da coluna Data:**

```
={{ $now.format('dd-MM-yyyy HH:mm') }}
```

**Permissoes necessarias no Google Cloud:**

```
APIs ativadas:
  - Google Drive API
  - Google Sheets API

Service Account:
  - Role: Basic > Owner
  - Chave: tipo JSON

Planilha compartilhada com:
  - email da Service Account
  - permissao: Editor
```
