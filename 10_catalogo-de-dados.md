# Catálogo de Dados  
## MVP Engenharia de Dados – CEAP (Câmara dos Deputados)

---

## 1. Visão Geral

Este catálogo documenta os conjuntos de dados utilizados no **MVP de Engenharia de Dados**, desenvolvido na plataforma **Databricks**, com dados públicos da **Cota para o Exercício da Atividade Parlamentar (CEAP)**, disponibilizados pela **Câmara dos Deputados**.

O projeto segue uma arquitetura em camadas (**Staging → Bronze → Silver → Gold**) e utiliza **modelagem analítica em Esquema Estrela (Star Schema)** para permitir análises de gastos públicos com confiabilidade, rastreabilidade e desempenho.

---

## 2. Fonte dos Dados

- **Origem oficial:** Câmara dos Deputados – Dados Abertos  
- **Dataset:** Cota para o Exercício da Atividade Parlamentar (CEAP)  
- **Formato original:** CSV compactado em arquivo ZIP  
- **Separador:** `;`  
- **Codificação:** UTF-8  

### Links oficiais
- https://dadosabertos.camara.leg.br/swagger/api.html?tab=staticfile  
- https://dadosabertos.camara.leg.br/howtouse/2023-12-26-dados-ceap.html  

---

## 3. Arquitetura e Linhagem dos Dados

```text
Fonte Oficial (CSV/ZIP)
        ↓
Staging (Volume / DBFS)
        ↓
Bronze (Delta Lake – dados crus)
        ↓
Silver (Delta Lake – tipagem, limpeza, qualidade)
        ↓
Gold (Delta Lake – modelo estrela analítico)
```

---

## 4. Camada Staging

>`staging_data` (Volume)

**Descrição:**  
Armazenamento dos arquivos originais (.zip e .csv) baixados diretamente da fonte oficial, garantindo rastreabilidade e possibilidade de reprocessamento.
- **Transformações:** Nenhuma
- **Objetivo:** Auditoria, versionamento da fonte e reprodutibilidade

## 4. Camada Staging

### staging_data (Volume)

**Descrição:**  
Armazenamento dos arquivos originais (.zip e .csv) baixados diretamente da fonte oficial, garantindo rastreabilidade e possibilidade de reprocessamento.

- **Transformações:** Nenhuma  
- **Objetivo:** Auditoria, versionamento da fonte e reprodutibilidade  

---



---

## 5. Camada Bronze
>`layer_bronze.bronze_ceap_despesas`

**Descrição:**
Tabela contendo os dados da CEAP exatamente como fornecidos pela fonte, sem conversão de tipos ou tratamento semântico.

| Campo                   | Tipo   | Descrição                             |
| ----------------------- | ------ | ------------------------------------- |
| id_documento            | string | Identificador do documento de despesa |
| data_emissao            | string | Data de emissão (ISO string)          |
| ano_ref                 | string | Ano de referência                     |
| mes_ref                 | string | Mês de referência                     |
| id_parlamentar          | string | Identificador do parlamentar          |
| nome_parlamentar        | string | Nome do parlamentar                   |
| sigla_partido           | string | Sigla do partido                      |
| sigla_uf                | string | Unidade Federativa                    |
| descricao_tipo_despesa  | string | Tipo de despesa                       |
| descricao_especificacao | string | Especificação do tipo                 |
| valor_documento         | string | Valor bruto                           |
| valor_glosa             | string | Valor glosado                         |
| valor_liquido           | string | Valor líquido                         |
| *_raw                   | string | Campos auxiliares do staging          |


**Observação:** Todos os campos permanecem como string nesta camada, conforme boas práticas de ingestão as-is.

---

## 6. Camada Silver
>`layer_silver.silver_ceap_despesas`

**Descrição:**
Tabela com dados tratados, tipados, padronizados e avaliados quanto à qualidade, servindo de base para a modelagem analítica.

**Campos e Tipagem**
| Campo                   | Tipo          | Domínio / Observações          |
| ----------------------- | ------------- | ------------------------------ |
| id_documento            | string        | Não nulo                       |
| id_parlamentar          | int           | Identificador numérico         |
| nome_parlamentar        | string        | Texto padronizado (UPPER/TRIM) |
| sigla_partido           | string        | Ex: PT, PL, PSDB               |
| sigla_uf                | string        | Ex: SP, RJ, MG                 |
| data_emissao            | date          | Data válida                    |
| ano                     | int           | Extraído de `data_emissao`     |
| mes                     | int           | 1–12                           |
| trimestre               | int           | 1–4                            |
| nome_mes                | string        | Janeiro…Dezembro               |
| descricao_tipo_despesa  | string        | Categoria principal            |
| descricao_especificacao | string        | Subcategoria                   |
| valor_documento         | decimal(12,2) | ≥ 0                            |
| valor_glosa             | decimal(12,2) | ≥ 0                            |
| valor_liquido           | decimal(12,2) | ≥ 0                            |

**Regras de Qualidade Avaliadas**
- Presença de valores nulos
- Valores monetários negativos ou zerados
- Regra: valor_liquido ≤ valor_documento
- Duplicidades lógicas (id_documento, id_parlamentar, data_emissao)
- Consistência temporal (ano da data vs. ano de referência)
- Identificação de outliers monetários
>⚠️ Registros problemáticos são identificados e documentados, mas não removidos automaticamente.

---

## 7. Camada Gold – Modelo Analítico (Star Schema)
### Tabela Fato
#### `layer_gold.fato_despesa_ceap`

| Campo           | Tipo          | Descrição                  |
| --------------- | ------------- | -------------------------- |
| id_documento    | string        | Identificador do documento |
| id_parlamentar  | int           | FK → `dim_parlamentar`     |
| id_tempo        | int           | FK → `dim_tempo`           |
| id_tipo_despesa | int           | FK → `dim_tipo_despesa`    |
| id_partido      | string        | FK → `dim_partido`         |
| id_uf           | string        | FK → `dim_uf`              |
| valor_documento | decimal(12,2) | Valor bruto                |
| valor_glosa     | decimal(12,2) | Valor glosado              |
| valor_liquido   | decimal(12,2) | Valor líquido              |


### Dimensões
#### `dim_parlamentar`
| Campo            | Tipo   |
| ---------------- | ------ |
| id_parlamentar   | int    |
| nome_parlamentar | string |
| sigla_partido    | string |
| sigla_uf         | string |
| num_legislatura  | int    |
    
#### `dim_tempo`
| Campo     | Tipo   |
| --------- | ------ |
| id_tempo  | int    |
| data      | date   |
| ano       | int    |
| mes       | int    |
| trimestre | int    |
| nome_mes  | string |

#### `dim_tipo_despesa`
| Campo                        | Tipo   |
| ---------------------------- | ------ |
| id_tipo_despesa              | int    |
| descricao_tipo_despesa       | string |
| id_especificacao_tipo        | int    |
| descricao_especificacao_tipo | string |

#### `dim_partido`
| Campo         | Tipo   | Observação                               |
| ------------- | ------ | ---------------------------------------- |
| id_partido    | string | Sigla do partido (PK)                    |
| sigla_partido | string | Redundância proposital para legibilidade |

#### `dim_uf`
| Campo    | Tipo   |
| -------- | ------ |
| id_uf    | string |
| sigla_uf | string |

---

## 8. Observações de Consistência

- `id_partido` e `id_uf` definidos corretamente como **string**
- Tipagem monetária padronizada em decimal(12,2)
- Campos do catálogo alinhados 100% com o modelo Gold
- Consistência entre Catálogo, Diagrama e Notebooks

---

## 9. Conclusão

Este catálogo garante:
- Clareza semântica
- Linhagem completa dos dados
- Reprodutibilidade do pipeline
- Aderência às boas práticas de Engenharia de Dados

