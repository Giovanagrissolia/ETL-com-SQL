# ETL em SQL Server: do transacional ao Data Warehouse

Pipeline de ETL construído em T-SQL que leva dados de vendas de um banco transacional até um Data Warehouse em modelo estrela, pronto para consumo em ferramentas de BI.

---

## Pergunta de negócio

Como transformar os dados brutos do sistema transacional em tabelas limpas, padronizadas e rápidas de consultar para análise de vendas?

## Visão geral

O fluxo passa por três camadas:

```
 TRANSACIONAL              STAGE                    DATA WAREHOUSE            BI
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐    ┌─────────┐
│ PBS_PROCFIT_DADOS│ -> │ PBS_PROCFT_ST    │ -> │ PBS_PROCFIT_DW   │ -> │ Power BI│
│ (sistema origem) │    │ cópia + views    │    │ dimensões + fato │    │         │
│                  │    │ de tratamento    │    │                  │    │         │
└──────────────────┘    └──────────────────┘    └──────────────────┘    └─────────┘
```

| Camada | Banco | O que faz |
|---|---|---|
| Transacional | `PBS_PROCFIT_DADOS` | Origem dos dados (entidades, produtos, vendas analíticas etc.). Não é alterada. |
| Stage | `PBS_PROCFT_ST` | Recebe uma cópia dos dados da origem (tabelas `ST_*`) e aplica o tratamento por meio de views (`VW_*`). |
| Data Warehouse | `PBS_PROCFIT_DW` | Armazena as dimensões (`D_*`) e a tabela fato (`F_VENDAS`) a partir das views da stage. |

## Tecnologias

- SQL Server
- T-SQL (tabelas, views, stored procedures, funções, `MERGE`)

## Modelo dimensional

Modelo estrela com uma fato e quatro dimensões, mais uma dimensão de calendário gerada por função.

```
              D_CLIENTE
                  │
 D_EMPRESA ─── F_VENDAS ─── D_PRODUTO
                  │
              D_VENDEDOR

 D_CALENDARIO (gerada pela função FN_MONTAR_D_CALENDARIO)
```

| Tabela | Tipo | Conteúdo |
|---|---|---|
| `DIME_CLIENTE` | Dimensão | Nome, nome fantasia, classificação, cidade, UF e estado |
| `D_EMPRESA` | Dimensão | Código, nome e nome fantasia da empresa |
| `D_PRODUTO` | Dimensão | Descrição, família, seção, grupo, subgrupo e marca |
| `D_VENDEDOR` | Dimensão | Código e nome do vendedor |
| `F_VENDAS` | Fato | Quantidade, venda bruta, desconto total, venda líquida e número do documento, com chaves estrangeiras para as dimensões |
| `D_CALENDARIO` | Dimensão de tempo | Ano, trimestre, mês, semana, dia, ano fiscal e deslocamentos em relação à data atual |

## Como funciona

### 1. Stage (`PBS_PROCFT_ST`)

Para cada assunto existe uma procedure que cria as tabelas `ST_*` se não existirem, limpa com `TRUNCATE` e copia os dados da origem.

| Procedure | Tabelas carregadas |
|---|---|
| `USP_ST_CARGA_CLIENTE` | `ST_CLIENTES`, `ST_CLIENTES_CLASSIFICACOES`, `ST_CLIENTES_ENDERECOS`, `ST_CLIENTES_ESTADOS` |
| `USP_ST_CARGA_EMPRESA` | `ST_EMPRESAS` |
| `USP_ST_CARGA_PRODUTOS` | `ST_PRODUTOS` (já une família, seção, grupo, subgrupo e marca) |
| `USP_ST_CARGA_VENDEDOR` | `ST_VENDEDORES` |
| `USP_ST_CARGA_VENDAS (@INCREMENTAL)` | `ST_VENDAS` |

As procedures-mestre orquestram tudo: `USP_ST_CARGA` (full) e `USP_ST_CARGA_INCREMENTAL`. A `USP_LIMPAR_ST` zera todas as tabelas da stage.

### 2. Tratamento por views

O tratamento fica nas views da stage, e o DW apenas consome o resultado:

- **Registro "Não informado":** cada dimensão recebe uma linha com código `0` (`UNION ALL`) para que vendas sem cliente, produto, vendedor ou empresa não sejam perdidas nem quebrem a integridade referencial.
- **Valores nulos:** `ISNULL` troca nulos por "Não informado/Não informada" (ou `ND` para UF).
- **Padronização de texto:** a função `FN_CAMEL_CASE` converte textos em caixa alta para o formato "Nome Próprio".
- **Fato sem nulos nas chaves:** `VW_F_VENDAS` troca chaves nulas por `0`, apontando para o registro "Não informado".
- **Desconto total:** na carga de vendas, `DESCONTO + DESCONTO_NEGOCIADO` viram uma única coluna `DESCONTO_TOTAL`.

### 3. Data Warehouse (`PBS_PROCFIT_DW`)

- **Dimensões:** cada `USP_FULL_D_*` cria a tabela se não existir e recarrega a partir da view correspondente. Para clientes, empresas, produtos e vendedores foram desenvolvidas duas estratégias:
  - `TRUNCATE` + `INSERT`: simples, recarrega tudo.
  - `MERGE`: atualiza registros existentes, insere novos e (na versão de cliente) remove o que saiu da origem.
- **Fato:** `USP_CARGA_F_VENDAS (@INCREMENTAL)` carrega `F_VENDAS`.
- **Calendário:** `FN_MONTAR_D_CALENDARIO (@D_INI, @D_FIM, @ANO_FECHADO, @INICIO_ANO_FISCAL)` gera a tabela de datas com colunas de ano, trimestre, mês, semana, ano fiscal e offsets. Ela usa a tabela auxiliar `VEZES`, que guarda uma sequência de 1 a 99.999.

### Carga full x incremental

| | Full | Incremental |
|---|---|---|
| Procedure no DW | `USP_DW_CARGA_FULL` | `USP_DW_CARGA_INCREMENTAL` |
| Dimensões | Recarregadas | Não recarregadas |
| Fato | `USP_CARGA_F_VENDAS 0`: limpa e recarrega tudo | `USP_CARGA_F_VENDAS 1`: só insere o que é novo |
| Critério | Todas as vendas desde 1900 | `MOVIMENTO` maior que a maior data já existente em `F_VENDAS` |

## Como executar

1. Crie os bancos `PBS_PROCFT_ST` e `PBS_PROCFIT_DW` no mesmo servidor do banco de origem.
2. Na **stage**, crie a função `FN_CAMEL_CASE`, as tabelas, as procedures `USP_ST_CARGA_*` e as views `VW_*`.
3. No **DW**, crie a tabela `VEZES`, popule-a e crie a função `FN_MONTAR_D_CALENDARIO`.
4. Crie as dimensões e a fato, depois as procedures de carga.
5. Execute a carga completa:

```sql
-- Stage
USE PBS_PROCFT_ST;
EXEC USP_ST_CARGA;

-- Data Warehouse
USE PBS_PROCFIT_DW;
EXEC USP_DW_CARGA_FULL;
```

6. Nas cargas seguintes, use a versão incremental:

```sql
USE PBS_PROCFT_ST;
EXEC USP_ST_CARGA_INCREMENTAL;

USE PBS_PROCFIT_DW;
EXEC USP_DW_CARGA_INCREMENTAL;
```

7. Para gerar a dimensão calendário:

```sql
SELECT * FROM dbo.FN_MONTAR_D_CALENDARIO('2018-01-01', '2022-12-31', 1, 7);
```

## Decisões de projeto

- **Stage separada do DW:** isola o sistema de origem, evita consultas pesadas no transacional e concentra o tratamento em um só lugar.
- **Procedures idempotentes:** todas verificam se a tabela existe antes de criá-la, então podem ser executadas várias vezes.
- **Registro `0` nas dimensões:** garante que a fato sempre encontre uma chave válida.
- **Carga incremental só para a fato:** dimensões são pequenas e recarregadas em full; a fato é a tabela que cresce.

## Sobre o projeto

Projeto de portfólio para praticar modelagem dimensional e ETL em SQL Server, cobrindo as camadas transacional, stage e Data Warehouse.
