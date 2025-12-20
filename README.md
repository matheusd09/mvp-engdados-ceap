# MVP Engenharia de Dados — CEAP

Este repositório contém o MVP da disciplina de **Engenharia de Dados**, desenvolvido na plataforma Databricks, com um pipeline de ponta a ponta (busca → coleta → modelagem → carga → análise), utilizando dados públicos da **Cota para o Exercício da Atividade Parlamentar (CEAP)**.

> **Tema:** Transparência e análise de gastos públicos (CEAP)  
> **Plataforma:** Databricks (Delta Lake + Spark + SQL + Dashboards)  
> **Modelo analítico:** Esquema Estrela (Star Schema)

---

## 1. Objetivo do MVP

Construir um **pipeline de dados em nuvem**, reprodutível e documentado, capaz de:
- Coletar os dados oficiais da CEAP,
- Persistir os dados em camadas **Bronze → Silver → Gold**,
- Modelar os dados em **Esquema Estrela** (fato + dimensões),
- Avaliar **qualidade de dados** por atributo,
- Responder perguntas analíticas via **SQL** (e/ou visualizações).

### 1.1 Problema a ser resolvido

Embora os dados da CEAP sejam públicos, eles são disponibilizados de forma **bruta** (arquivos CSV por ano, com campos tipados como texto e inconsistências esperadas). Para permitir análises confiáveis e rápidas, é necessário **organizar, padronizar e modelar** o conjunto de dados.

### 1.2 Perguntas que o MVP deve responder

**Padrões gerais de gasto**
- Qual é o valor total gasto pela CEAP no período analisado?
- Qual é o gasto médio mensal por parlamentar?
- Como os gastos se distribuem ao longo dos meses do ano?

**Análise por parlamentar**
- Quais parlamentares apresentam os maiores volumes de despesas?
- Existe grande variação de gastos entre parlamentares?

**Análise por tipo de despesa**
- Quais tipos de despesa concentram a maior parte dos gastos da CEAP?

**Análise por partido político**
- Qual é o gasto médio por partido político?
- A composição de gastos varia entre partidos?

**Análise geográfica (UF)**
- Como os gastos se distribuem entre as UFs?

**Análise temporal**
- Há meses com picos atípicos de despesas?

> As perguntas acima estão implementadas no notebook **06_Q&A**.

---

## 2. Fonte de dados

**Fontes oficiais (Câmara dos Deputados):**
- Swagger / Static Files: https://dadosabertos.camara.leg.br/swagger/api.html?tab=staticfile
- Documentação CEAP: https://dadosabertos.camara.leg.br/howtouse/2023-12-26-dados-ceap.html
- Dataset utilizado neste MVP (exemplo): https://www.camara.leg.br/cotas/Ano-2024.csv.zip

**Formato de ingestão:** CSV (dentro de .zip), separado por `;`.

---

## 3. Arquitetura do pipeline (Databricks)

O pipeline segue uma arquitetura em camadas:

- **Staging**: Download + extração + armazenamento do arquivo original (zip) e extraído (csv) em um Volume do Unity Catalog.
- **Bronze**: Ingestão dos dados **crus**, preservando a granularidade original.
- **Silver**: Limpeza, padronização textual, conversão de tipos (datas, decimais), criação de campos auxiliares e checagens de qualidade.
- **Gold**: Modelagem analítica em **Esquema Estrela** para consultas simples e consistentes.

---

## 4. Preparação do ambiente (Databricks)

### 4.1 Estrutura (Catalog e Schemas)

O notebook/arquivo `01_preparacao.sql` cria o **catalog** e os **schemas** do projeto:

- `staging`
- `layer_bronze`
- `layer_silver`
- `layer_gold`

> Observação: por simplicidade de reprodução, o script remove e recria o catálogo.

---

## 5. Etapas do pipeline

### 5.1 Staging — Download e extração do dataset
Notebook: `02_staging-ceap`

- Cria um **Volume** em `mvp_ed_ceap.staging.staging_data`
- Baixa o `.zip` diretamente da Câmara (`dbutils.fs.cp`)
- Lê o binário e extrai o `.csv`
- Persiste `.zip` e `.csv` no Volume para rastreabilidade

**Evidências sugeridas (screenshot):**
![](path)
- Volume criado
- Arquivos `.zip` e `.csv` no DBFS/Volume
- Amostra do CSV carregado


---

### 5.2 Bronze — Ingestão “as-is”
Notebook: `03_bronze-layer`

- Lê o CSV com `sep=';'`, `header=True`, sem inferência de schema
- Persiste em Delta como tabela `layer_bronze.bronze_ceap_despesas`

**Evidências sugeridas (screenshot):**
- `display(df_bronze.limit(10))`
- tabela Delta criada no catálogo

---

### 5.3 Silver — Tratamento, tipagem e qualidade
Notebook: `04_silver-layer`

Principais transformações:
- Renomeação de colunas (padronização semântica)
- Padronização textual (`trim`, `upper`)
- Conversão de datas ISO → `date` (`to_date`)
- Conversão de valores monetários → `decimal(12,2)`
- Conversão de chaves numéricas → `int`
- Normalização de documento CNPJ/CPF apenas dígitos (`regexp_replace`)
- Remoção de colunas `_raw` que foram utilizadas para validar as conversões

Saída: tabela `layer_silver.silver_ceap_despesas`

**Checagens de qualidade incluídas:**
- Completude (nulos em campos críticos e opcionais)
- Consistência de valores (negativos, liquido > documento)
- Estatísticas (min/mediana/média/máximo)
- Duplicidade lógica (chave: `id_documento`, `id_parlamentar`, `data_emissao`)
- Consistência temporal (`ano_ref` vs `year(data_emissao)`)
- Domínio categórico (tipos de despesa, UF)

> Importante: no MVP, outliers e duplicidades **são identificados**, mas fazem parte das análises e não necessariamente removidos (decisão documentada).

---

### 5.4 Gold — Modelo analítico (Esquema Estrela)
Notebook: `05_gold-layer`

Tabelas do modelo final:

**Fato**
- `layer_gold.fato_despesa_ceap`

**Dimensões**
- `layer_gold.dim_parlamentar`
- `layer_gold.dim_tempo`
- `layer_gold.dim_tipo_despesa`
- `layer_gold.dim_partido`
- `layer_gold.dim_uf`

A tabela fato inclui:
- `id_documento` (identificador do documento)
- FKs para dimensões (`id_parlamentar`, `id_tempo`, `id_tipo_despesa`, `id_partido`, `id_uf`)
- Métricas monetárias (`valor_documento`, `valor_glosa`, `valor_liquido`)

---

## 6. Modelo de dados (Diagrama)

O modelo segue um **Star Schema**, com uma tabela fato central e dimensões descritivas.

![Diagrama do Modelo](Diagrama.png)

---

## 7. Catálogo de Dados e Linhagem

Notebook: `00_catálogo-de-dados`

O catálogo documenta:
- Fonte e links oficiais
- Frequência de atualização
- Linhagem (Staging → Bronze → Silver → Gold)
- Descrição das tabelas (fato e dimensões)
- Tipos e domínios esperados

---

## 8. Análises (SQL) — Respostas às perguntas do MVP

Notebook: `06_Q&A`

Consultas implementadas:
- Total de gastos no período
- Gasto médio por parlamentar
- Distribuição mensal (ano/mês)
- Ranking de gastos por parlamentares
- Variação de gasto (min/max/média por parlamentar)
- Gastos por tipo de despesa
- Gasto médio por partido
- Composição de gasto por partido × tipo
- Gastos por UF
- Meses com maiores gastos (picos)

**Evidências**
- Resultado de cada query principal
- Databricks SQL Dashboard com filtros

---

## 9. Organização do repositório

Arquivos principais:
- `01_preparacao.sql` — Criação de catalog e schemas
- `02_staging-ceap.py` — Staging (download/extract)
- `03_bronze-layer.py` — Ingestão Bronze
- `04_silver-layer.py` — Transformação + qualidade
- `05_gold-layer.py` — Star schema (Gold)
- `06_Q&A.py` — Queries de negócio
- `00_catálogo-de-dados.py` — Catálogo + linhagem

---

## 10. Execução do MVP

1) Importar/abrir o repositório no Databricks  
2) Executar na ordem:

- `01_preparacao.sql`
- `02_staging-ceap`
- `03_bronze-layer`
- `04_silver-layer`
- `05_gold-layer`
- `06_Q&A`

3) Criar visualizações gráficas no **Databricks Dashboards** utilizando as queries do `06_Q&A`.

---

## 11. Autoavaliação

**O que foi atingido**
- Pipeline completo em nuvem com camadas Bronze/Silver/Gold
- Modelagem em Esquema Estrela
- Catálogo de dados e linhagem
- Análise de qualidade por atributo
- Respostas SQL para perguntas do objetivo

**Dificuldades**
- Padronização e tipagem dos dados (datas/decimais)
- Tratamento de possíveis duplicidades lógicas
- Definição de domínios/categorias para campos textuais

**Trabalhos futuros**
- Ingestão de múltiplos anos (ex.: 2019–2024) com orquestração
- Incremental load (merge) e particionamento por data/ano
- Enriquecimento com dados de deputados (API REST) e partidos oficiais (nome completo)
- Dashboards com filtros e KPIs (Top N, séries temporais, outliers)
- Testes automatizados de qualidade (expectations) e alertas

---

## 12. Licença e créditos

Dados: Câmara dos Deputados (Dados Abertos).  
Uso educacional e acadêmico.