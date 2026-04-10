# Relatório de Execução — Projeto P01: Data Mart WWI

**Disciplina:** Decision Support Systems (SAD), 2025-26  
**Equipa 8:** Pedro (27960), Ricardo (27961), João (27964), Carolina (27983)

---

## 1. O Que o Professor Pediu

Segundo o enunciado (`DSS_P01_L08_PS.md`), o trabalho pede:

| Requisito | Descrição |
|---|---|
| **Fonte de dados** | Base de dados Wide World Importers (WWI) em `postgres2.ipca.pt` |
| **Arquitectura** | Pipeline ELT com **Medallion Architecture** (Bronze → Silver → Gold) |
| **Modelo dimensional** | Star schema com **2 tabelas de factos** (`fact_sales`, `fact_orders`) e **6 dimensões** conformes (`dim_date`, `dim_customer`, `dim_product`, `dim_employee`, `dim_geography`, `dim_delivery_method`) |
| **SCD** | Tipo 1 (overwrite) + Tipo 2 (histórico) — híbrido por dimensão |
| **Dimensão outrigger** | `dim_geography` ligada a `dim_customer` e `dim_employee` (não directamente aos factos) |
| **Medidas derivadas** | `line_total_excl_tax = quantity × unit_price`, `backordered_quantity = ordered_quantity − picked_quantity` |
| **Dimensões degeneradas** | `invoice_id` e `order_id` directamente nas fact tables |
| **Calendário** | `dim_date` gerada programaticamente com atributos (dia, mês, trimestre, ano, semana, dia da semana, fim-de-semana, feriado) |
| **fact_orders** | Duas date keys: `order_date_key` e `expected_delivery_date_key` (role-playing `dim_date`) |
| **Implementação** | 4 notebooks Jupyter (setup, bronze, silver, gold) + ficheiro `.env` de configuração |
| **Ferramenta** | Python + SQLAlchemy + pandas + PostgreSQL |

---

## 2. O Que Foi Feito (e Corrigido)

### 2.1 Adaptações à Base de Dados Real

A base de dados no servidor `postgres2.ipca.pt` difere do que o enunciado assume:

| Problema encontrado | Solução aplicada |
|---|---|
| Tabelas sem underscores nos nomes (ex: `customercategories` em vez de `customer_categories`) | Corrigidos todos os nomes em `01_bronze` e `02_silver` |
| Colunas sem underscores (ex: `customerid`, `customername`) | Bronze mantém nomes originais; Silver renomeia para snake_case |
| Todas as tabelas no schema `public` (não há schemas `sales`, `warehouse`, etc.) | `.env` configurado com `SRC_SCHEMA_*=public` |
| Tabelas `suppliers` e `stockitemstockgroups` **não existem** | `supplier_name` e `stock_group_name` removidos do modelo (colunas inexistentes na fonte) |
| Coluna `countryname` tem typo: `ountryname` | Preservado no Bronze, renomeado para `country_name` no Silver |
| Tabela `orders` não tem coluna `deliverymethodid` | Silver faz JOIN com `invoices` para obter `delivery_method_id` |
| Campos boolean são `SMALLINT` (0/1) na fonte | Convertidos para `BOOLEAN` no Silver |
| Tabela `people` não tem coluna `cityid` | `city_id = NULL` no Silver; `geography_key = NULL` no Gold |

### 2.2 Configuração (`.env`)

```
SRC_HOST=postgres2.ipca.pt
SRC_PORT=5432
SRC_DB=wwi
SRC_USER=dss
SRC_PASSWORD=dss
SRC_SCHEMA_SALES=public
SRC_SCHEMA_WAREHOUSE=public
SRC_SCHEMA_APPLICATION=public
SRC_SCHEMA_PURCHASING=public

DWH_HOST=localhost
DWH_PORT=5432
DWH_DB=wwi_dw
DWH_USER=postgres
DWH_PASSWORD=dss
```

### 2.3 Notebooks — Resultado Final

| Notebook | Estado | Descrição |
|---|---|---|
| `00_setup.ipynb` | ✅ Funcional | Cria schemas `bronze`, `silver`, `gold` e as tabelas gold (dimensões com suporte SCD2 + factos) |
| `01_bronze.ipynb` | ✅ Funcional | Extrai 16 tabelas da fonte para o schema `bronze` (snapshot diferencial + incremental por data) |
| `02_silver.ipynb` | ✅ Funcional | Limpa, transforma e renomeia colunas para snake_case; trata NULLs; converte tipos |
| `03_gold.ipynb` | ✅ Funcional | Popula dimensões (SCD1 + SCD2 híbrido) e factos com surrogate keys + medidas derivadas |

---

## 3. Como Executar

### 3.1 Pré-requisitos

1. **PostgreSQL** instalado localmente (versão 15+)
2. **Python 3.10+** com `pip`
3. Base de dados `wwi_dw` criada no PostgreSQL local:
   ```sql
   CREATE DATABASE wwi_dw;
   ```
4. Password do utilizador `postgres` definida como `dss`:
   ```sql
   ALTER USER postgres WITH PASSWORD 'dss';
   ```

### 3.2 Instalar dependências

```bash
pip install psycopg2-binary sqlalchemy pandas python-dotenv
```

### 3.3 Ordem de execução

Executar os notebooks **sequencialmente**, de cima a baixo, na seguinte ordem:

```
1.  notebooks/00_setup.ipynb      →  Cria schemas e tabelas gold
2.  notebooks/01_bronze.ipynb     →  Extrai dados da fonte para bronze
3.  notebooks/02_silver.ipynb     →  Transforma bronze → silver
4.  notebooks/03_gold.ipynb       →  Carrega dimensões e factos no gold
```

> **Importante:** Cada notebook deve ser executado por inteiro (Run All) antes de passar ao seguinte. O kernel deve ter acesso ao ficheiro `notebooks/.env`.

### 3.4 Tempo estimado de execução

| Notebook | Tempo aprox. |
|---|---|
| `00_setup` | ~5 segundos |
| `01_bronze` | ~70 segundos |
| `02_silver` | ~60 segundos |
| `03_gold` | ~100 segundos |
| **Total** | **~4 minutos** |

---

## 4. Contagens Finais do Data Mart

| Tabela | Tipo | Registos |
|---|---|---:|
| `gold.dim_date` | Dimensão (SCD1) | 1 248 |
| `gold.dim_geography` | Dimensão (SCD1) | 37 940 |
| `gold.dim_customer` | Dimensão (SCD2 híbrido) | 663 |
| `gold.dim_product` | Dimensão (SCD2 híbrido) | 227 |
| `gold.dim_employee` | Dimensão (SCD2 híbrido) | 1 111 |
| `gold.dim_delivery_method` | Dimensão (SCD1) | 10 |
| `gold.fact_sales` | Facto | 228 265 |
| `gold.fact_orders` | Facto | 231 412 |

---

## 5. Conformidade com o Enunciado

| Requisito do enunciado | Implementado? | Notas |
|---|:---:|---|
| Fonte WWI em `postgres2.ipca.pt` | ✅ | Ligação via SQLAlchemy |
| Medallion Architecture (Bronze/Silver/Gold) | ✅ | 3 schemas no PostgreSQL local |
| 2 tabelas de factos (`fact_sales`, `fact_orders`) | ✅ | Granularidade: 1 linha por invoice_line / order_line |
| 6 dimensões conformes | ✅ | `dim_date`, `dim_customer`, `dim_product`, `dim_employee`, `dim_geography`, `dim_delivery_method` |
| SCD Tipo 1 | ✅ | `dim_date`, `dim_geography`, `dim_delivery_method` — TRUNCATE + reload |
| SCD Tipo 2 (híbrido) | ✅ | `dim_product`, `dim_customer`, `dim_employee` — colunas de controlo: `start_date`, `end_date`, `is_active` |
| Surrogate keys (SERIAL) | ✅ | Geradas pelo PostgreSQL em todas as dimensões |
| `dim_date` gerada programaticamente | ✅ | 2017-01-01 a 2020-06-01, com nomes em PT, inclui `is_holiday` |
| Outrigger `dim_geography` | ✅ | Ligada a `dim_customer` e `dim_employee` via `geography_key` |
| Dimensões degeneradas (`invoice_id`, `order_id`) | ✅ | Colunas directas nas fact tables |
| `line_total_excl_tax = qty × price` | ✅ | Calculado no Silver |
| `backordered_quantity = ordered − picked` | ✅ | Calculado no Silver, com `.clip(lower=0)` |
| Role-playing dim (`dim_date`) | ✅ | `fact_orders` tem `order_date_key` + `expected_delivery_date_key` |
| `fact_sales.salesperson_employee_key` | ✅ | FK para `dim_employee` (renomeado de `employee_key`) |
| Integridade referencial (FK) | ✅ | Verificada: 0 orphans em `fact_sales` e `fact_orders` |
| Queries de smoke test | ✅ | Vendas por ano/trimestre, top 10 produtos, backorders por vendedor |

### Estratégia SCD Tipo 2

As seguintes colunas são rastreadas com SCD Tipo 2 (criação de nova versão quando o valor muda):

| Dimensão | Colunas SCD2 (histórico) | Colunas SCD1 (overwrite) |
|---|---|---|
| `dim_product` | `brand`, `size`, `tax_rate`, `lead_time_days` | `stock_item_name`, `color_name`, `unit_package_name`, `outer_package_name`, `quantity_per_outer`, `is_chiller_stock` |
| `dim_customer` | `customer_category_name`, `buying_group_name`, `is_on_credit_hold` | `customer_name`, `bill_to_customer_name`, `phone_number`, `geography_key` |
| `dim_employee` | `is_salesperson` | `full_name`, `preferred_name`, `geography_key` |

**Colunas de controlo:** `start_date` (DATE), `end_date` (DATE, default `9999-12-31`), `is_active` (BOOLEAN).  
**Índices parciais:** `CREATE UNIQUE INDEX ... ON (business_key) WHERE is_active = TRUE` — garante exactamente 1 registo activo por business key.  
**Factos:** apontam sempre para a surrogate key do registo corrente (`is_active = TRUE`).

### Desvios justificados

| Desvio | Justificação |
|---|---|
| `supplier_name` removido do modelo | Tabela `suppliers` não existe na fonte; coluna deixou de constar no modelo dimensional |
| `dim_geography` sem `city_id` e `sales_territory` | Removidos do modelo final (usados apenas internamente no ETL) |
| `dim_customer` sem `primary_contact_name`, `website_url`, `delivery_city_name` | Colunas removidas no modelo actualizado |
| `dim_product` sem `unit_price` e `stock_group_name` | `unit_price` movido para factos; `stock_group_name` removido (tabela fonte inexistente) |
| `dim_date` cobre 2017-01-01 a 2020-06-01 (não 2013–2016) | Dados reais na fonte começam em 2017 |
| 1 111 employees com `geography_key = NULL` | Tabela `people` não tem `city_id` na fonte |
| `DimDate` tem ~1 248 registos | Amplitude real dos dados: 2017-01-01 a 2020-06-01 |
| `fact_orders` sem `delivery_method_key` | Encomendas não têm método de entrega directo na fonte |
| SCD Tipo 2 em 3 dimensões (enunciado pedia apenas SCD1) | Implementação mais completa: campos de negócio sensíveis rastreados com histórico; restantes colunas mantêm SCD1 (overwrite) |

---

## 6. Estrutura do Projecto

```
projeto_1/
├── notebooks/
│   ├── .env                  ← Configuração de ligação
│   ├── 00_setup.ipynb        ← Criação de schemas e tabelas
│   ├── 01_bronze.ipynb       ← Extracção bronze
│   ├── 02_silver.ipynb       ← Transformação silver
│   └── 03_gold.ipynb         ← Carregamento gold (data mart)
├── DSS_P01_L08_PS.md         ← Enunciado do trabalho
└── RELATORIO_EXECUCAO.md     ← Este documento
```
