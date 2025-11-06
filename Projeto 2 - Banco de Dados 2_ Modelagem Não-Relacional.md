# Projeto 2 - Banco de Dados 2: Modelagem Não-Relacional

**Aluno:** [Seu Nome Completo]
**Matrícula:** [Sua Matrícula]
**Disciplina:** Banco de Dados 2
**Professor(a):** Dra. Aline de Campos
**Data:** 06 de Novembro de 2025

---

## A1. Modelagem não-relacional de dados

### 1. Análise e Reorganização do Modelo Relacional para Não-Relacional

O modelo relacional fornecido (diagrama Entidade-Relacionamento) foi analisado com o objetivo de transformá-lo em um esquema não-relacional (MongoDB), seguindo o princípio da **desnormalização** e da **otimização de consultas (read-heavy)**, que é o padrão para sistemas de e-commerce.

A principal entidade de acesso e consulta no domínio é o **Pedido** (`PEDIDO`). A estratégia adotada foi centralizar a informação em torno de duas coleções principais: `pedidos` e `produtos`.

#### 1.1. Esquema Não-Relacional Proposto

O esquema foi projetado para minimizar a necessidade de *joins* (operações de `$lookup` no MongoDB) nas consultas mais frequentes, como a visualização de um pedido completo.

**Coleções:**

1.  **`pedidos`**: Contém o histórico completo do pedido, incorporando (embedding) dados do cliente, endereço de entrega e a lista de itens comprados.
2.  **`produtos`**: Contém informações atuais e voláteis do produto, como estoque e preço atual, além de incorporar a categoria.

#### 1.2. Imagem do Esquema (Representação Conceitual)

Abaixo, a representação conceitual do esquema não-relacional, que seria o resultado da ferramenta de modelagem (como MongoDB Modeler ou Hackolade), ilustrando as relações de *Embedding* e *Referência*.

![Diagrama Conceitual do Modelo Não-Relacional](diagrama_conceitual_mongodb.png)

*Observação: Esta imagem é uma representação conceitual do esquema. A implementação real utiliza o JSON Schema para validação.*

### 2. Justificativa das Escolhas de Modelagem

As decisões de modelagem foram tomadas com base na frequência de acesso aos dados e na necessidade de manter a integridade histórica dos pedidos.

| Escolha de Modelagem | Coleção | Justificativa Técnica |
| :--- | :--- | :--- |
| **Embedding** de Cliente e Endereço | `pedidos` | Otimiza a consulta de um pedido completo, evitando *joins*. O endereço de entrega e os dados do cliente no momento da compra são **históricos** e não devem ser alterados se o cliente atualizar seu cadastro. |
| **Embedding** de Itens do Pedido | `pedidos` | Garante a **integridade histórica** do pedido. O preço unitário e o nome do produto no momento da compra são fixos, mesmo que o produto seja renomeado ou tenha seu preço alterado posteriormente. |
| **Referência** para Produto | `pedidos` (dentro de `itens`) | O campo `id_produto` é mantido como referência para a coleção `produtos`. Isso permite que consultas que necessitem de dados **atuais** do produto (como estoque ou preço atual) possam ser realizadas, mantendo a flexibilidade. |
| **Coleção Separada** para `produtos` | `produtos` | O estoque (`quantidade_estoque`) e o preço atual (`preco_unitario_atual`) são dados **voláteis** e de alta frequência de atualização. Mantê-los separados evita a atualização constante de documentos grandes na coleção `pedidos`, que pode conter milhões de registros. |
| **Embedding** de Categoria | `produtos` | A categoria é um dado de baixa cardinalidade e baixa frequência de alteração. Incorporá-la otimiza consultas por categoria, como "Listar todos os produtos da categoria X". |

### 3. Construção do Esquema Não-Relacional (JSON Schema)

Abaixo estão os esquemas de validação (Schema Validation) para as coleções, que garantem a estrutura e os tipos de dados, conforme solicitado em A1.c.

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
