# Projeto 2 - Banco de Dados 2: Modelagem Não-Relacional

## A1. Modelagem não-relacional de dados

### 1. Análise e Reorganização do Modelo Relacional para Não-Relacional

O modelo relacional fornecido é composto pelas seguintes entidades: `ITEM_PEDIDO`, `ESTADO`, `CIDADE`, `PEDIDO`, `CLIENTE`, `ENDERECO`, `PRODUTO`, `CATEGORIA`, `PESSOA_JURIDICA`, `PESSOA_FISICA` e `TELEFONE`.

Para a modelagem não-relacional (MongoDB), o objetivo principal é **desnormalizar** o esquema, agrupando dados que são frequentemente acessados juntos (princípio da **Projeção de Dados** e **Consultas**). A principal entidade de acesso e consulta neste domínio de e-commerce é o **Pedido** (`PEDIDO`).

Abaixo está a proposta de reorganização, focada em duas coleções principais: `pedidos` e `produtos`.

#### Coleção Principal: `pedidos`

Esta coleção será o centro do modelo, incorporando (embedding) a maioria dos dados relacionados para otimizar as consultas de histórico de pedidos, detalhes de entrega e informações do cliente.

| Campo | Tipo | Descrição | Justificativa |
| :--- | :--- | :--- | :--- |
| `_id` | ObjectId | ID único do pedido. | Padrão MongoDB. |
| `data_pedido` | Date | Data e hora do pedido. | Essencial para consultas temporais. |
| `data_remessa` | Date | Data de envio do pedido. | Essencial para rastreamento. |
| `data_entrega` | Date | Data de entrega do pedido. | Essencial para rastreamento. |
| `status` | String | Status atual do pedido (ex: "Em processamento", "Enviado", "Entregue"). | Campo de alta frequência de consulta. |
| `valor_total` | Double | Valor total do pedido (calculado). | Otimiza consultas de valor total. |
| `cliente` | Objeto | Informações do cliente que fez o pedido. | **Embedding** (Incorporação) - Otimiza a consulta de pedidos por cliente. |
| `cliente.id_cliente` | Number | ID original do cliente. | Referência para o caso de consultas que exijam dados mais detalhados do cliente. |
| `cliente.nome` | String | Nome/Razão Social do cliente. | |
| `cliente.tipo` | String | Tipo de cliente ("Física" ou "Jurídica"). | |
| `cliente.documento` | String | CPF/CNPJ do cliente. | |
| `cliente.telefone` | Array de Objetos | Telefones do cliente. | **Embedding** - Telefones são dados pequenos e específicos do cliente. |
| `endereco_entrega` | Objeto | Endereço completo de entrega. | **Embedding** - O endereço de entrega é estático após o pedido e crucial para a consulta do pedido. |
| `endereco_entrega.logradouro` | String | | |
| `endereco_entrega.numero` | String | | |
| `endereco_entrega.complemento` | String | | |
| `endereco_entrega.bairro` | String | | |
| `endereco_entrega.cidade` | String | | |
| `endereco_entrega.estado` | String | | |
| `itens` | Array de Objetos | Lista de produtos e suas quantidades no pedido. | **Embedding** - Os itens do pedido são a parte mais consultada junto com o pedido. |
| `itens.id_produto` | Number | ID original do produto. | **Referência** (para a coleção `produtos`). |
| `itens.nome_produto` | String | Nome do produto no momento da compra. | **Embedding** (Denormalização) - O nome do produto no pedido não deve mudar se o produto for renomeado. |
| `itens.quantidade` | Number | Quantidade do item. | |
| `itens.preco_unitario` | Double | Preço unitário no momento da compra. | **Embedding** (Denormalização) - O preço no pedido é histórico e não deve ser alterado. |

#### Coleção Secundária: `produtos`

Esta coleção armazena informações sobre os produtos e suas categorias.

| Campo | Tipo | Descrição | Justificativa |
| :--- | :--- | :--- | :--- |
| `_id` | ObjectId | ID único do produto. | Padrão MongoDB. |
| `id_produto_original` | Number | ID original do produto (PK do modelo relacional). | Para facilitar a migração e referência. |
| `nome` | String | Nome do produto. | |
| `descricao` | String | Descrição detalhada. | |
| `unidade_medida` | String | Unidade de medida (ex: "un", "kg"). | |
| `quantidade_estoque` | Number | Quantidade atual em estoque. | **Volátil** - Este campo será atualizado frequentemente, justificando a coleção separada. |
| `preco_unitario_atual` | Double | Preço de venda atual. | **Volátil** - Este campo será atualizado frequentemente. |
| `categoria` | Objeto | Informações da categoria. | **Embedding** - A categoria é um dado de baixa cardinalidade e raramente muda, otimizando consultas por categoria. |
| `categoria.id_categoria` | Number | ID original da categoria. | |
| `categoria.nome` | String | Nome da categoria. | |

### 2. Justificativa das Escolhas de Modelagem

As escolhas de modelagem foram guiadas pelo princípio da **otimização de leitura (read-heavy)**, que é comum em sistemas de e-commerce para histórico de pedidos, e pela necessidade de **isolamento de dados voláteis**.

1.  **Embedding de Cliente e Endereço em `pedidos`:**
    *   **Benefício:** A consulta mais comum é "Mostrar todos os detalhes de um pedido" (quem comprou, o que comprou, para onde vai). Ao incorporar (`embedding`) o cliente e o endereço, evitamos *joins* (operações de `$lookup` no MongoDB) e reduzimos o número de consultas ao banco de dados para obter todos os dados necessários.
    *   **Exceção:** O endereço de entrega é incorporado porque, uma vez feito o pedido, ele se torna um dado **histórico** e não deve ser alterado, mesmo que o cliente mude seu endereço de cadastro.
2.  **Embedding de Itens do Pedido em `pedidos`:**
    *   **Benefício:** Os itens são intrinsecamente ligados ao pedido. Incorporar a lista de itens, juntamente com o nome e preço unitário no momento da compra, garante a **integridade histórica** do pedido. Se o produto for renomeado ou tiver seu preço alterado, o registro do pedido permanece inalterado e correto.
3.  **Coleção Separada para `produtos`:**
    *   **Benefício:** O estoque (`quantidade_estoque`) e o preço atual (`preco_unitario_atual`) são dados **voláteis** e de alta frequência de atualização. Mantê-los em uma coleção separada evita a necessidade de atualizar documentos grandes na coleção `pedidos` (que poderiam ser milhões) a cada venda ou alteração de preço.
    *   **Referência:** Usamos o `id_produto` dentro do array `itens` na coleção `pedidos` como uma **referência** para a coleção `produtos`. Isso é um padrão conhecido como *denormalização mista* ou *referência manual* e é útil para consultas que precisam de dados atuais do produto (ex: "Qual o estoque atual do produto X que está neste pedido?").
4.  **Embedding de Categoria em `produtos`:**
    *   **Benefício:** A categoria é um dado de baixa cardinalidade e baixa frequência de alteração. Incorporá-la otimiza consultas como "Listar todos os produtos da categoria 'Eletrônicos'".

### 3. Construção do Esquema Não-Relacional (MongoDB Modeler / Hackolade)

A construção do esquema será realizada utilizando a sintaxe de *Schema Validation* do MongoDB, que é a forma mais robusta de garantir a estrutura dos documentos.

**Observação:** O esquema abaixo é uma representação em JSON Schema, que seria o resultado da ferramenta de modelagem.

#### Schema Validation para a Coleção `pedidos`

```json
{
  "$jsonSchema": {
    "bsonType": "object",
    "required": ["data_pedido", "status", "cliente", "endereco_entrega", "itens"],
    "properties": {
      "data_pedido": {
        "bsonType": "date",
        "description": "Data e hora da realização do pedido."
      },
      "data_remessa": {
        "bsonType": ["date", "null"],
        "description": "Data de envio do pedido."
      },
      "status": {
        "bsonType": "string",
        "enum": ["Em processamento", "Enviado", "Entregue", "Cancelado"],
        "description": "Status atual do pedido."
      },
      "valor_total": {
        "bsonType": "double",
        "minimum": 0,
        "description": "Valor total do pedido."
      },
      "cliente": {
        "bsonType": "object",
        "required": ["id_cliente", "nome", "tipo", "documento"],
        "properties": {
          "id_cliente": { "bsonType": "int" },
          "nome": { "bsonType": "string" },
          "tipo": { "bsonType": "string", "enum": ["Física", "Jurídica"] },
          "documento": { "bsonType": "string" },
          "telefone": {
            "bsonType": "array",
            "items": {
              "bsonType": "object",
              "required": ["numero"],
              "properties": {
                "numero": { "bsonType": "string" },
                "tipo": { "bsonType": "string", "enum": ["Celular", "Comercial", "Residencial"] }
              }
            }
          }
        }
      },
      "endereco_entrega": {
        "bsonType": "object",
        "required": ["logradouro", "numero", "cidade", "estado"],
        "properties": {
          "logradouro": { "bsonType": "string" },
          "numero": { "bsonType": "string" },
          "bairro": { "bsonType": "string" },
          "cidade": { "bsonType": "string" },
          "estado": { "bsonType": "string" }
        }
      },
      "itens": {
        "bsonType": "array",
        "minItems": 1,
        "items": {
          "bsonType": "object",
          "required": ["id_produto", "nome_produto", "quantidade", "preco_unitario"],
          "properties": {
            "id_produto": { "bsonType": "int" },
            "nome_produto": { "bsonType": "string" },
            "quantidade": { "bsonType": "int", "minimum": 1 },
            "preco_unitario": { "bsonType": "double", "minimum": 0 }
          }
        }
      }
    }
  }
}
```

#### Schema Validation para a Coleção `produtos`

```json
{
  "$jsonSchema": {
    "bsonType": "object",
    "required": ["id_produto_original", "nome", "quantidade_estoque", "preco_unitario_atual", "categoria"],
    "properties": {
      "id_produto_original": {
        "bsonType": "int",
        "description": "ID original do produto."
      },
      "nome": {
        "bsonType": "string",
        "description": "Nome do produto."
      },
      "descricao": {
        "bsonType": "string",
        "description": "Descrição detalhada do produto."
      },
      "quantidade_estoque": {
        "bsonType": "int",
        "minimum": 0,
        "description": "Quantidade atual em estoque."
      },
      "preco_unitario_atual": {
        "bsonType": "double",
        "minimum": 0,
        "description": "Preço de venda atual."
      },
      "categoria": {
        "bsonType": "object",
        "required": ["id_categoria", "nome"],
        "properties": {
          "id_categoria": { "bsonType": "int" },
          "nome": { "bsonType": "string" }
        }
      }
    }
  }
}
```
