# CineData Analytics — Agente Text-to-SQL
 
Projeto da atividade **GenAI — RocketLab 2026.2**. Um agente que permite a pessoas não técnicas fazerem perguntas em português sobre o catálogo de filmes da CineData Analytics e receberem respostas baseadas em consultas de leitura (SQL) feitas diretamente na camada Gold.

## Sumário
 
- [Visão geral](#visão-geral)
- [Stack](#stack)
- [Como executar](#como-executar)
- [Arquitetura e decisões técnicas](#arquitetura-e-decisões-técnicas)
- [Guardrails de segurança](#guardrails-de-segurança)
- [Resiliência](#resiliência)
- [Interface de chat](#interface-de-chat)
- [Avaliação](#avaliação)
- [Limitações conhecidas](#limitações-conhecidas)
- [Estrutura do repositório](#estrutura-do-repositório)
---
 
## Visão geral
 
Fluxo de uma pergunta:
 
```
Pergunta em português
        │
        ▼
LLM (OpenRouter, modelo gratuito com tool calling)
        │  decide chamar a ferramenta executar_sql(query)
        ▼
Guardrails (apenas SELECT) ──► SQLite (cinerocket.db, somente leitura)
        │
        ▼
Resultado volta ao LLM ──► Resposta final em português
```
 
O agente foi implementado **sem framework de agentes**: o loop de *tool calling* é escrito à mão com a biblioteca `openai` (compatível com a API da OpenRouter). Isso dá controle total sobre quantas requisições cada pergunta consome, o que importa porque a conta gratuita da OpenRouter tem limite de 50 requisições por dia em modelos gratuitos, e cada pergunta usa no mínimo 2 (uma para o modelo gerar o SQL e outra para formular a resposta).
 
## Stack
 
- **Linguagem:** Python 3.11+
- **Banco:** SQLite (`cinerocket.db`, camada Gold com o modelo dimensional)
- **LLM:** modelos gratuitos (`:free`) via OpenRouter, com suporte a tool calling
- **Cliente:** biblioteca `openai` apontando para `https://openrouter.ai/api/v1`
- **Interface (opcional):** Gradio
- **Entregável:** Jupyter Notebook
## Como executar
 
### Pré-requisitos
 
1. Python 3.11 ou superior.
2. Uma conta na [OpenRouter](https://openrouter.ai) e uma API key (*Settings → Keys → Create Key*).
3. O arquivo `cinerocket.db`, disponibilizado na pasta compartilhada da atividade.
### Passo a passo (execução local)
 
```bash
# 1. Clone o repositório
git clone https://github.com/lorenascarv/Rocketlab-GenIa.git
cd Rocketlab-GenIa
 
# 2. Crie e ative um ambiente virtual
python -m venv venv
venv\Scripts\activate          # Windows
# source venv/bin/activate     # Linux/macOS
 
# 3. Instale as dependências
pip install -r requirements.txt
```
 
4. Coloque o arquivo cinerocket-db.zip, sem extrair, na raiz do projeto (mesma pasta do notebook). Ao ser executado, o notebook extrai o zip para a pasta cinerocket_extracted/ e localiza o arquivo .db dentro dela automaticamente. Se não houver zip, o notebook procura um arquivo .db solto na mesma pasta; se não encontrar nada, interrompe com uma mensagem indicando o que falta.
5. Crie um arquivo `.env` na raiz com a sua chave:
```
OPENROUTER_API_KEY=sua_chave_aqui
```
 
6. Abra o notebook e execute as células **em ordem, de cima para baixo**:
```bash
jupyter notebook agente_cinedata.ipynb
```
 
### Execução no Google Colab
 
1. Faça upload do notebook e do `cinerocket-db.zip` (sem extrair) para o Colab, pelo painel de arquivos à esquerda. Os uploads ficam em `/content`, a pasta de trabalho do notebook, e o notebook extrai o zip e localiza o `.db` sozinho.
2. Em *Secrets* (ícone de chave), crie `OPENROUTER_API_KEY` com a sua chave e ative o acesso ao notebook.
3. Execute as células em ordem.
> **Observação:** ambientes de Jupyter que rodam no navegador via WebAssembly (como o JupyterLite) não funcionam, porque a biblioteca `openai` depende de pacotes binários incompatíveis com esse ambiente.
 
### Modelos
 
Os nomes dos modelos ficam em variáveis no topo do notebook (`MODELO` e `MODELOS_FALLBACK`). Modelos gratuitos da OpenRouter mudam com frequência; se algum sair do ar, consulte openrouter.ai/models (filtro Free + tools) e atualize essas variáveis
 
## Arquitetura e decisões técnicas
 
### Schema no prompt
 
O *system prompt* contém a descrição completa do modelo dimensional (tabelas, colunas, chaves e relacionamentos) mais as regras de negócio. Os nomes de colunas e os valores de domínio (`tipo_pessoa`, `status_filme`, nomes de gêneros) foram **conferidos diretamente no banco** antes de entrarem no prompt, para evitar que o modelo invente valores. Pontos que o prompt trata explicitamente:
 
- **Sinônimos de negócio:** receita, faturamento e bilheteria são a mesma métrica (`receita_usd`/`receita_brl`).
- **Moeda:** colunas `*_brl` quando o usuário pede "em R$"; `*_usd` caso contrário.
- **Nulos:** filtros `IS NOT NULL` explícitos quando a pergunta fala em "com receita informada" etc.; médias e somas ignoram nulos.
- **Margem de lucro:** `lucro_usd / orcamento_usd * 100`, apenas com orçamento maior que zero.
- **Recortes temporais:** "últimos N anos" é relativo à **maior data de lançamento da base** (ignorando datas futuras), não à data de hoje.
- **Gêneros:** a base usa nomes em inglês; o prompt traz a lista exata e instrui a traduzir o que o usuário disser (ex.: "ação" → `Action`).
- **Contagens:** `COUNT(DISTINCT sk_movie_id)` para não duplicar filmes ao passar pelas tabelas ponte.
### Few-shot
 
O prompt inclui exemplos de SQL correto para as consultas em que modelos costumam errar: dupla ator–diretor (join duplo na mesma tabela ponte), recorte de "últimos N anos" e agregação por gênero. Exemplos no prompt custam tokens, mas não custam requisições adicionais.
 
### Loop de tool calling
 
1. O modelo recebe a pergunta (e o histórico recente, na interface de chat) e decide chamar `executar_sql`.
2. O código valida e executa a query, e devolve o resultado ao modelo como mensagem de ferramenta.
3. O modelo pode chamar a ferramenta novamente (por exemplo, para corrigir uma query com erro), em **até 3 rodadas**. Quando tem os resultados de que precisa, formula a resposta final em português.
4. Se as rodadas acabarem, ou se o modelo devolver texto vazio, o código faz uma última chamada com `tool_choice="none"` pedindo a resposta final com base nos resultados já obtidos. No pior caso são 5 requisições por pergunta (o caso comum usa 2).
 
## Guardrails de segurança
 
O enunciado pede apenas consultas de leitura. Há duas camadas independentes:
 
1. **Validação da query** (`executar_sql`): aceita somente um único comando que comece com `SELECT`; bloqueia múltiplos comandos (`;`) e palavras de escrita ou administração (`DROP`, `DELETE`, `INSERT`, `UPDATE`, `ALTER`, `ATTACH`, `PRAGMA`, `CREATE`, `REPLACE`, `VACUUM`) usando busca por **palavra inteira**, para não bloquear por engano consultas legítimas como um título de filme contendo "Drop".
2. **Conexão somente leitura:** o banco é aberto com `mode=ro`, de modo que o SQLite recusa qualquer escrita mesmo que algo escape da validação. A conexão é aberta e fechada a cada consulta, porque a interface Gradio executa o agente em threads separadas e o SQLite não permite usar uma conexão criada em outra thread. Esse problema fazia o agente funcionar no notebook e falhar no chat durante o desenvolvimento.
Além disso, o resultado de cada consulta é limitado a 50 linhas antes de ser devolvido ao modelo, para não estourar o contexto.
 
## Resiliência
 
Modelos gratuitos têm disponibilidade instável (já observei erro 503 de sobrecarga do provedor durante os testes). Por isso:
 
- **Fallback entre modelos:** a requisição envia uma lista de modelos reserva (`models` no corpo da requisição) e a OpenRouter tenta o próximo se o principal falhar. O modelo efetivamente usado em cada chamada pode variar e é exibido no modo debug.
- **Retry com espera progressiva:** erros temporários (429, 502, 503, 504) são repetidos até 3 vezes, com espera de 2s e 4s. Erros definitivos falham imediatamente.
- **Erros legíveis:** a OpenRouter pode devolver um erro no corpo da resposta, sem levantar exceção na biblioteca. O código detecta esse caso e mostra a mensagem real do provedor.
- **Mensagens amigáveis na interface:** falhas não derrubam o chat, o usuário recebe uma mensagem pedindo para tentar novamente.
## Interface de chat
- **Respostas vazias:** modelos podem devolver texto vazio (por exemplo, ao chamar a ferramenta de novo em vez de responder). O loop de rodadas e a resposta final forçada tratam esse caso.
 
Há uma interface opcional em Gradio, com **memória de conversa**: as últimas 6 mensagens são enviadas ao modelo, o que permite perguntas de acompanhamento ("e nos últimos 2 anos?"). A memória consome tokens, mas não requisições adicionais.
 
Atenção ao limite diário: cada mensagem no chat consome pelo menos 2 das 50 requisições gratuitas.
 
## Avaliação
 
O conjunto de avaliação foi construído em duas etapas:
 
1. **Gabarito:** para cada uma das 14 perguntas de exemplo do enunciado foi escrito um SQL de referência, executado diretamente no banco (sem uso de LLM e sem consumir a cota de requisições).
2. **Agente:** as perguntas são feitas ao agente, e o SQL gerado e a resposta são comparados com o gabarito.
| # | Categoria | Pergunta | Resultado do agente |
|---|-----------|----------|---------------------|
| 1 | Bilheteria e finanças | Top 10 filmes com maior receita em R$ | ✅ |
| 2 | Bilheteria e finanças | Lucro médio por gênero (apenas filmes com receita informada) | ✅ |
| 3 | Bilheteria e finanças | Filmes com maior margem de lucro | ✅ |
| 4 | Popularidade e engajamento | Os 5 filmes mais populares | ✅ |
| 5 | Popularidade e engajamento | Maior divergência entre nota TMDB e IMDb | ✅ (Interessante que o agente entendeu que as notas 0.0 eram nulas e deu dois resultados, um com as notas 0.0 e outro mais realista) |
| 6 | Popularidade e engajamento | Nota média IMDb por ano de lançamento | ❌ Não testado |
| 7 | Elenco e equipe | Ator com mais participações nos últimos 5 anos | ✅ |
| 8 | Elenco e equipe | Diretores com maior nota média (mínimo de 5 filmes) | ✅ |
| 9 | Elenco e equipe | Dupla ator–diretor que mais trabalhou junta | ✅ |
| 10 | Gêneros e produtoras | Quantidade de filmes por gênero | ❌ Não testado |
| 11 | Gêneros e produtoras | Produtora com maior lucro total | ✅ |
| 12 | Gêneros e produtoras | Gênero com maior margem de lucro média | ✅ |
| 13 | Avaliações de usuários | Filmes mais avaliados pelos usuários | ✅ |
| 14 | Avaliações de usuários | Maior divergência entre nota dos usuários e nota IMDb | ❌ Não testado |
 
Evidências do chat:
<img width="706" height="396" alt="image" src="https://github.com/user-attachments/assets/7e9996c4-7f7a-4c6d-9959-74c0aa061420" />
<img width="1366" height="622" alt="image" src="https://github.com/user-attachments/assets/9ae6f713-f9e5-45f0-aa51-d6dbaa27665a" />
<img width="1365" height="598" alt="image" src="https://github.com/user-attachments/assets/c60a9af4-813d-461b-aca4-9f598e40d9f3" />
<img width="1366" height="618" alt="image" src="https://github.com/user-attachments/assets/5c777831-709b-4c73-bdb5-6b8e9d4971cf" />


 
Por causa do limite de 50 requisições diárias, a avaliação do agente foi distribuída por categoria em vez de rodar todas as perguntas repetidamente.
 
## Limitações conhecidas
 
- **Qualidade dos dados de origem.** A coluna `popularidade` contém valores como `2018`, `2019` e `2020`, que parecem ser o ano de lançamento vazado para a coluna errada (problema de deslocamento de colunas já presente na base bruta). O agente apresenta esses valores como se fossem popularidade real, porque a camada Gold os considera válidos.
- **Margens de lucro extremas.** Há orçamentos de poucos dólares na base, o que gera margens percentuais exageradas nas perguntas de margem. Não foi aplicado corte de plausibilidade, pois isso não é uma regra definida no enunciado.
- **Correção do SQL não é garantida.** O modelo pode gerar uma consulta que executa sem erro mas não responde exatamente à pergunta. Por isso existe o conjunto de avaliação com gabarito, e o SQL gerado pode ser exibido no modo debug para auditoria.
- **Modelos gratuitos.** Têm limites de requisições, podem sair do ar ou deixar de ser gratuitos sem aviso, e os endpoints gratuitos podem registrar entradas e saídas para treino pelo provedor. Como o banco é um catálogo público de filmes, isso não envolve dados sensíveis, mas não inclua dados pessoais nas perguntas.
- **Escopo.** O agente só consulta o SQLite fornecido. Não usa busca semântica sobre sinopses nem se conecta ao Databricks.
## Estrutura do repositório
 
```
.
├── agente_cinedata.ipynb   # notebook principal (agente, guardrails, gabarito, interface)
├── cinerocket-db.zip       # zip com o banco SQLite da camada Gold (fornecido na atividade)
├── cinerocket_extracted/   # criada pelo notebook ao extrair o zip (gerada automaticamente)
├── requirements.txt        # dependências
└── README.md
```
