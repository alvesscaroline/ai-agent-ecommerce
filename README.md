Agente IA de E-commerce com Memória Persistente

## Visão Geral

Este projeto implementa um Agente de Inteligência Artificial (IA) para um e-commerce, desenvolvido com n8n, capaz de interagir com uma base de dados de produtos hospedada no PostgreSQL (via Supabase) e manter memória persistente das interações com o usuário. O agente foi projetado para interpretar requisições em linguagem natural, buscar dados relevantes e fornecer respostas claras e concisas.

## Pré-requisitos

Para configurar e executar este projeto, você precisará dos seguintes itens:

*   **n8n:** Uma instância funcional do n8n (https://n8n.io/). Pode ser self-hosted ou via serviço cloud.
*   **Supabase:** Uma conta Supabase (https://supabase.com/) com um projeto criado. Este projeto será usado para hospedar o banco de dados PostgreSQL.
*   **Claude API Key:** Uma chave de API válida da Anthropic para o modelo de linguagem (LLM) usado pelo `AI Agent`.
*   **Dados de Produto:** Devem ser gerados dados dos produtos a serem importados para o Supabase.

## Configuração

Siga os passos abaixo para configurar o ambiente e o workflow no n8n.

### 1. Configuração do Banco de Dados PostgreSQL (Supabase)

1.  **Crie a Tabela de Produtos (`produtos`):**
    *   No seu projeto Supabase, crie uma tabela chamada `produtos` com as seguintes colunas:
        *   `id` (UUID, Primary Key, com `default gen_random_uuid()`)
        *   `nome` (Text ou Varchar)
        *   `descricao` (Text)
        *   `preco` (Numeric ou Float)
        *   `quantidade_estoque` (Integer)
        *   `categoria` (Text ou Varchar)
        *   `url_imagem` (Text)
        *   `created_at` (Timestamp with Time Zone)
        *   `updated_at` (Timestamp with Time Zone)
    *   **Popule a tabela:** Importe os dados do arquivo `produtos_rows.csv` para esta tabela no Supabase.

2.  **Crie a Tabela de Histórico de Conversas (`historico_conversas`):**
    *   Esta tabela é usada pelo nó `Postgres Chat Memory` para persistir o histórico da conversa.
    *   Crie a tabela com o seguinte esquema SQL:

    ```sql
    create table public.historico_conversas (
      id uuid not null default gen_random_uuid (),
      user_id character varying(255) not null,
      timestamp timestamp with time zone null default now(),
      remetente character varying(50) not null,
      message text not null,
      session_id text null,
      constraint historico_conversas_pkey primary key (id)
    ) TABLESPACE pg_default;

    create index IF not exists idx_historico_conversas_user_id_timestamp on public.historico_conversas using btree (user_id, "timestamp") TABLESPACE pg_default;
    create index IF not exists idx_historico_conversas_user_session_timestamp on public.historico_conversas using btree (user_id, session_id, "timestamp") TABLESPACE pg_default;
    ```

### 2. Configuração do n8n Workflow

1.  **Importe o Workflow:**
    *   Importe o arquivo `.json` do workflow (`Agente IA e-commerce.json`) para sua instância n8n.

2.  **Configure o Nó `Webhook`:**
    *   Este é o ponto de entrada do workflow. Após ativar o workflow, o n8n fornecerá um URL para este webhook.

3.  **Configure o Nó `Edit Fields`:**
    *   Este nó é responsável por extrair as variáveis necessárias (`id_conta`, `id_mensagem`, `message`, `messageType`, `timestamp`, `id_conversa`, `telefone`) do corpo da requisição do webhook.
    *   A variável `telefone` (que será usada como `user_id` para a memória persistente) é extraída de `={{ $json.body.sender.phone_number }}`.

4.  **Configure o Nó `Anthropic Chat Model`:**
    *   Crie ou selecione uma credencial para a Anthropic API.
    *   Certifique-se de que o `Model` esteja definido para `claude-sonnet-4-20250514` (ou a versão mais recente do Claude, conforme o desafio).

5.  **Configure o Nó `Postgres Chat Memory`:**
    *   Este nó é crucial para a memória persistente.
    *   `Credential`: Crie ou selecione uma credencial `Postgres` que aponte para o seu banco de dados Supabase (forneça a URL de conexão e a API Key).
    *   `Session ID Type`: Selecione `Custom Key`.
    *   `Session Key`: Defina como `={{ $json.telefone }}`. Isso garante que o histórico de conversa seja associado ao número de telefone do usuário.
    *   `Table Name`: Defina como `historico_conversas`.

6.  **Configure o Nó `buscar_produto_especifico` (Supabase Tool):**
    *   `Credential`: Use a mesma credencial Supabase configurada para a memória.
    *   `Operation`: `getAll`.
    *   `Table ID`: `produtos`.
    *   `Filters`: Configure os filtros para as colunas `nome`, `descricao`, `preco` e `categoria`. Use a condição `eq` (ou `ilike` para buscas parciais) para cada filtro. O `keyValue` deve ser mapeado para os parâmetros que o `AI Agent` fornecerá. **Exemplo:** Para o filtro `nome`, o `keyValue` deve ser `={{ $fromAI('product_name', '', 'string') }}`. 
Faça o mesmo para `description`, `price` e `category`.
    *   **Importante:** Verifique se o nó Supabase permita que esses filtros sejam aplicados condicionalmente (ou seja, apenas se o valor fornecido pelo `AI Agent` não for vazio). Caso contrário, a abordagem de duas ferramentas separadas (como a `listar_todos_produtos`) é fundamental.

7.  **Configure o Nó `listar_todos_produtos` (Supabase Tool):**
    *   `Credential`: Use a mesma credencial Supabase.
    *   `Operation`: `getAll`.
    *   `Table ID`: `produtos`.
    *   **Filters: Deixe esta seção completamente VAZIA.** Este nó foi projetado para retornar todos os produtos sem qualquer filtragem.

8.  **Configure o Nó `AI Agent`:**
    *   Certifique-se de que as entradas `ai_languageModel` (do `Anthropic Chat Model`), `ai_memory` (do `Postgres Chat Memory`) e `ai_tool` (de `buscar_produto_especifico` e `listar_todos_produtos`) estejam corretamente conectadas.
    *   `Chat ID`: Defina como `={{ $json.telefone }}`. Isso garante que o `AI Agent` passe o ID de sessão correto para o nó de memória.
    *   `System Message`: Cole o seguinte conteúdo (este é o prompt atualizado do agente, que o instrui a usar as duas ferramentas de forma adequada):

    ```text
    HOJE É: {{ $now.format('FFFF') }}
    TELEFONE DO CONTATO: {{ $json.telefone }}
    ID DA CONVERSA: {{ $json.id_conversa }}

    I. DEFINIÇÃO DE PAPEL E MISSÃO

    Você é um Agente de Dados de Produtos, um assistente de IA especializado, meticulosamente engenheirado para interagir com uma base de dados de produtos. Sua missão primária é:

    Interpretar requisições de usuário em linguagem natural relacionadas a produtos.
    Orquestrar a recuperação de dados relevantes da base de dados via ferramentas designadas.
    Sintetizar as informações recuperadas em respostas claras, concisas e orientadas a dados.

    II. REPERTÓRIO DE FERRAMENTAS EXTERNAS

    Você está equipado com ferramentas externas precisamente definidas para interagir com o inventário de produtos:

    Nome da Ferramenta: buscar_produto_especifico
    Descrição da Funcionalidade: Esta ferramenta permite acesso direto e eficiente à base de dados de produtos para buscar detalhes abrangentes sobre UM PRODUTO ESPECÍFICO ou FILTRAR produtos por critérios. Ela pode recuperar informações como nome, descrição, preço, e níveis de estoque.
    Parâmetros de Invocação:
      product_name (Tipo: string, Opcional): Um identificador (nome completo ou parcial) para o produto específico que o usuário deseja consultar. EX: "Notebook Gamer Elite", "Smart TV".
      description (Tipo: string, Opcional): Palavras-chave ou trechos da descrição do produto. EX: "4K HDR", "RGB".
      price (Tipo: number, Opcional): O preço exato ou aproximado do produto. EX: 2800, 350.
      category (Tipo: string, Opcional): A categoria do produto. EX: "Eletrônicos", "Periféricos", "Acessórios", "Livros".
    Regra de Uso:
      - OBRIGATÓRIO fornecer ao menos um parâmetro (product_name, description, price, ou category) se a requisição do usuário se refere a um produto nomeado explicitamente, um tipo de produto, ou para aplicar filtros.
      - OMITIR TODOS os parâmetros se a intenção do usuário for uma listagem geral de produtos ou uma consulta que não especifica um critério de busca.
      - Use esta ferramenta quando o usuário busca por algo específico ou com filtros.

    Nome da Ferramenta: listar_todos_produtos
    Descrição da Funcionalidade: Esta ferramenta permite obter uma visão geral e completa de TODOS os produtos disponíveis no catálogo. Ela retorna a lista de todos os produtos sem aplicar nenhum filtro.
    Parâmetros de Invocação: Nenhum. Esta ferramenta não aceita parâmetros.
    Regra de Uso:
      - Use esta ferramenta EXCLUSIVAMENTE quando a intenção do usuário for uma listagem geral de produtos (e.g., "Quais produtos vocês têm?", "Liste o catálogo completo", "Quero ver todos os produtos disponíveis").

    Gatilhos de Invocação (Quando usar as ferramentas):
    - Se o usuário solicita informações detalhadas sobre um produto específico, com ou sem critérios de busca (nome, descrição, preço, categoria), use a ferramenta `buscar_produto_especifico`.
    - Se o usuário pede uma lista de produtos disponíveis ou uma visão geral do inventário SEM NENHUM CRITÉRIO DE FILTRO, use a ferramenta `listar_todos_produtos`.

    III. PROTOCOLOS DE COMPORTAMENTO OPERACIONAL

    Para garantir performance e aderência ao escopo:

    Prioridade de Ferramenta: A escolha da ferramenta deve ser baseada na intenção mais clara do usuário.
      - Se a requisição do usuário contém um produto nomeado, categoria, preço ou descrição para filtro, a ferramenta `buscar_produto_especifico` tem prioridade.
      - Se a requisição é explicitamente para "listar todos" ou "ver o catálogo completo" sem filtros, a ferramenta `listar_todos_produtos` tem prioridade.
    Síntese de Dados: Após a execução de qualquer ferramenta e o recebimento de sua saída, sintetize a informação de forma clara e factual. Evite apresentar a saída bruta da ferramenta. Converta dados em formato de lista ou tabela em linguagem natural compreensível.
    Adesão ao Escopo: Seu domínio de conhecimento é estritamente limitado aos dados recuperáveis via `buscar_produto_especifico` ou `listar_todos_produtos`. Requisições fora deste escopo (e.g., opiniões, informações não relacionadas a produtos, dados externos não acessíveis, cálculos complexos não baseados em dados brutos) devem ser educadamente recusadas, com uma breve declaração de sua limitação de escopo.
    Abstração de Banco de Dados: Você está proibido de tentar gerar código SQL, inferir esquemas de banco de dados, ou interagir com o banco de dados de qualquer forma que não seja pela invocação formal das ferramentas `buscar_produto_especifico` ou `listar_todos_produtos`.
    Tratamento de Resultados Nulos: Se uma ferramenta retornar nenhum dado para uma consulta específica (e.g., produto não encontrado, ou nenhuma correspondência para os filtros), informe o usuário de forma clara e cortês sobre a ausência de resultados.
    Gerenciamento de Listas: Se uma ferramenta retornar uma lista extensa de produtos, sumarize os resultados ou destaque os itens mais relevantes conforme a aparente intenção do usuário, evitando respostas excessivamente longas.
    ```

### 3. Variáveis de Ambiente (Opcional)

Para maior segurança e facilidade de gerenciamento, você pode armazenar chaves de API e credenciais sensíveis como variáveis de ambiente no n8n. Consulte a documentação do n8n para configurar variáveis de ambiente e referenciá-las nos seus nós.

## Execução

1.  **Ative o Workflow:** No n8n, ative o workflow para que ele comece a escutar por requisições no endpoint do `Webhook`.
2.  **Envie uma Requisição POST:** Envie uma requisição HTTP POST para o URL do seu webhook n8n. O corpo da requisição deve seguir a estrutura esperada, conforme o exemplo abaixo (simulando uma mensagem do WhatsApp):

    ```json
    {
      "event": "message_created",
      "id": "2094_TESTE",
      "content": "Quais produtos vocês têm?",
      "message_type": "incoming",
      "conversation": {
        "id": 104
      },
      "sender": {
        "id": 1,
        "name": "Ana",
        "phone_number": "+551194581xxxx",
        "type": "contact"
      },
      "account": {
        "id": 1,
        "name": "Go"
      },
      "created_at": "2025-08-16T18:44:30.900Z"
    }
    ```
    *   **`content`**: O texto da mensagem do usuário.
    *   **`phone_number`**: O número de telefone do usuário (essencial para a memória persistente).
    *   Adapte os demais campos (`id`, `conversation.id`, `account.id`, `created_at`) conforme necessário para testes ou integração.

## Roteiro de Testes

Este roteiro guiará você pelos principais cenários de teste para verificar a funcionalidade do agente e a memória persistente.

### Testes Funcionais do Agente de Produtos

1.  **Listar Todos os Produtos:**
    *   **Mensagem:** "Quais produtos vocês têm?" ou "Liste o catálogo completo."
    *   **Resultado Esperado:** O agente deve retornar uma lista de todos os produtos presentes na tabela `produtos` do Supabase.
2.  **Buscar Produto Específico por Nome:**
    *   **Mensagem:** "Qual o preço do Smartphone Pro Max?" (substitua por um nome de produto existente na tabela de produtos).
    *   **Resultado Esperado:** O agente deve retornar o preço correto e/ou outros detalhes do "Smartphone Pro Max".
3.  **Buscar Produto por Descrição/Palavra-chave:**
    *   **Mensagem:** "Vocês têm algum monitor com tela 4K HDR?"
    *   **Resultado Esperado:** O agente deve identificar e listar produtos cuja descrição contenha "4K HDR" (ex: "Smart TV 55" 4K HDR").
4.  **Buscar Produto por Categoria:**
    *   **Mensagem:** "Quais produtos vocês têm na categoria Eletrônicos?"
    *   **Resultado Esperado:** O agente deve listar apenas os produtos da categoria "Eletrônicos".
5.  **Produto Não Encontrado:**
    *   **Mensagem:** "Qual o preço do Carro Voador XYZ?" (use um produto que não exista na sua base de dados).
    *   **Resultado Esperado:** O agente deve informar de forma clara e cortês que o produto não foi encontrado.
6.  **Combinação de Filtros (se aplicável ao seu `buscar_produto_especifico`):**
    *   **Mensagem:** "Quero ver notebooks na faixa de 3000 reais."
    *   **Resultado Esperado:** O agente deve buscar por notebooks que correspondam à faixa de preço.

### Testes de Memória Persistente

A memória persistente garante que o agente "lembre" do contexto da conversa com um usuário específico através das interações.

1.  **Primeira Interação e Estabelecimento de Contexto:**
    *   **Mensagem (1):** "Olá, meu nome é Ana. Você consegue me ajudar a encontrar um mouse gamer?"
    *   **Resultado Esperado:** O agente deve responder à pergunta sobre o mouse e "registrar" o nome "Ana" (embora não explicitamente usado no system message, o contexto da conversa é armazenado).
2.  **Continuação da Conversa:**
    *   **Mensagem (2 - na mesma sessão/telefone):** "Ele precisa ter iluminação RGB."
    *   **Resultado Esperado:** O agente deve continuar a conversa sobre o mouse, aplicando o filtro de "RGB" ao contexto anterior da busca por "mouse gamer". Ele não deve perguntar novamente "Qual tipo de produto você procura?".
3.  **Verificação de Contexto Após Inatividade (Simulada):**
    *   Após algumas interações (e alguns minutos, se você tiver um timeout de sessão no n8n), envie uma nova mensagem na **mesma sessão/telefone**.
    *   **Mensagem (3 - na mesma sessão/telefone, após um tempo):** "Você lembra qual era o último produto que eu estava procurando?"
    *   **Resultado Esperado:** O agente deve ser capaz de relembrar o contexto da última conversa, indicando que a memória persistente funcionou.

### Observações Importantes

*   Adapte os exemplos de mensagens nos testes com base nos dados reais que você importou para a sua tabela `produtos`.
*   Monitore os logs do n8n durante a execução para observar quais ferramentas o `AI Agent` está escolhendo (`buscar_produto_especifico` ou `listar_todos_produtos`) e quais dados estão sendo processados. Isso é fundamental para depuração.
*   Para testar a memória persistente com múltiplos usuários, altere o `phone_number` no JSON de requisição POST para simular diferentes usuários. Cada `phone_number` deve ter seu próprio histórico de conversa salvo.
