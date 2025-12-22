# Catálogo de Dados 
## MVP Transparência e análise de gastos públicos (CEAP)

Este catálogo provê documentação técnica para os conjuntos de dados utilizados no **Projeto de MVP** com dados públicos da **Cota para o Exercício da Atividade Parlamentar (CEAP)**, disponibilizados pela **Câmara dos Deputados**.

É utilizada a arquitetura Medallion em camadas (**Bronze → Silver → Gold**), com **modelagem analítica em Esquema Estrela (Star Schema)** para permitir análises de gastos públicos com confiabilidade.

---

### Identificação do Conjunto de Dados

- **Nome do conjunto de dados:** Cota para o Exercício da Atividade Parlamentar (CEAP)
- **Instituição responsável:** Câmara dos Deputados
- **Fonte oficial:** Portal de Dados Abertos da Câmara dos Deputados
- **Sistema de origem:** Dados Abertos – CEAP
- **Formato original:** CSV compactado em arquivo ZIP
- **Periodicidade de atualização:** Mensal, com revisões eventuais
- **Recorte temporal utilizado no MVP:** Ano de 2024
- **Data da última extração:** 20/12/2025
- **Separador:** `;`  
- **Codificação:** UTF-8  

**Referências oficiais:**  
- [Portal de Dados Abertos da Câmara dos Deputados](https://dadosabertos.camara.leg.br)
- [Arquivos Disponíveis](https://dadosabertos.camara.leg.br/swagger/api.html?tab=staticfile)
- [Documentação dos dados CEAP](https://dadosabertos.camara.leg.br/howtouse/2023-12-26-dados-ceap.html)
- [Fonte do Dataset](https://www.camara.leg.br/cotas/Ano-2024.csv.zip)
- [Dataset utilizado](https://github.com/matheusd09/mvp-engdados-ceap/blob/main/Referencias/Ano-2024.csv)

---

## Arquitetura e Linhagem dos Dados

- O dataset de dados utilizado neste trabalho foi obtidos a partir da base pública da Câmara dos Deputados, disponibilizada em diversos formatos.  
- Este projeto utiliza uma área de staging temporária para ingestão inicial dos arquivos, armazenamento transitório dos dados antes da persistência na camada Bronze. Os dados são coletados no formato CSV.ZIP, em seguida extraídos e armazenados tanto o arquivo original .zip, quanto o .csv no volume `staging_data`.
- A **camada Bronze** preserva a estrutura original da fonte, sem alterações de schema, garantindo reprodutibilidade e rastreabilidade.  
- Na **camada Silver**, os dados passam por processos de limpeza, padronização de tipos (datas, valores numéricos e campos categóricos), tratamento de valores nulos e registros de inconsistências.  
- Na **camada Gold**, os dados foram reorganizados em modelo analítico no Esquema Estrela, composto por uma tabela fato e tabelas dimensão, visando facilitar análises e consultas analíticas.  

> ```
Fonte Oficial (CSV/ZIP)
        ↓
Staging (Volume / DBFS)
        ↓
Bronze (Delta Lake – dados crus)
        ↓
Silver (Delta Lake – tipagem, limpeza, qualidade)
        ↓
Gold (Delta Lake – modelo estrela analítico)
        ↓
Visualização (Dashboards)        
> ```

---

## Camada Bronze
#### Catálogo técnico (Origem, Ingestão e Rastreabilidade)

**Tabela:** `mvp_ed_ceap.layer_bronze.bronze_ceap_despesas`  

**Descrição:** Tabela contendo os dados brutos CEAP, sem qualquer transformação ou limpeza, com os mesmos campos e formatos disponibilizados no arquivo CSV original.  
**Origem:** Portal de Dados Abertos da Câmara dos Deputados.  
**Camada:** Bronze.  
**Granularidade:** Uma linha por registro de despesa.  
**Chave primária lógica:** Nenhum campo único garantido na própria fonte (usar combinação de campos ou outra técnica na camada Silver, se necessário).  

> *Observação: Todos os campos permanecem como string nesta camada, conforme boas práticas de ingestão as-is.*

| Campo                     | Tipo (raw) | Descrição                                                        |
| ------------------------- | ---------- | ---------------------------------------------------------------- |
| txNomeParlamentar         | string     | Nome adotado pelo parlamentar                                    |
| cpf                       | string     | CPF do parlamentar (pode estar vazio ou parcialmente preenchido) |
| ideCadastro               | string     | Identificador numérico do parlamentar                            |
| nuCarteiraParlamentar     | string     | Número de identificação da carteira parlamentar                  |
| nuLegislatura             | string     | Ano de início da legislatura                                     |
| sgUF                      | string     | Sigla da unidade federativa do parlamentar                       |
| sgPartido                 | string     | Sigla do partido do parlamentar                                  |
| codLegislatura            | string     | Identificador da legislatura                                     |
| numSubCota                | string     | Código numérico da subcota (categoria de despesa)                |
| txtDescricao              | string     | Descrição da categoria de despesa                                |
| numEspecificacaoSubcota   | string     | Código da especificação da subcota                               |
| txtDescricaoEspecificacao | string     | Descrição da especificação da despesa                            |
| txtFornecedor             | string     | Nome do fornecedor do serviço/produto                            |
| txtCNPJCPF                | string     | CPF/CNPJ do fornecedor                                           |
| txtNumero                 | string     | Número do documento fiscal                                       |
| indTipoDocumento          | string     | Indicador do tipo de documento fiscal                            |
| datEmissao                | string     | Data de emissão do documento (ISO 8601)                          |
| vlrDocumento              | string     | Valor bruto do documento fiscal                                  |
| vlrGlosa                  | string     | Valor glosado (não coberto pela CEAP)                            |
| vlrLiquido                | string     | Valor líquido debitado da cota                                   |
| numMes                    | string     | Mês de competência da despesa                                    |
| numAno                    | string     | Ano de competência da despesa                                    |
| numParcela                | string     | Número da parcela da despesa                                     |
| txtPassageiro             | string     | Nome do passageiro (quando aplicável)                            |
| txtTrecho                 | string     | Trecho da viagem (quando aplicável)                              |
| numLote                   | string     | Número do lote de documentos                                     |
| numRessarcimento          | string     | Indicador de ressarcimento                                       |
| datPagamentoRestituicao   | string     | Data/hora do pagamento de restituição                            |
| vlrRestituicao            | string     | Valor devolvido pelo parlamentar                                 |
| nuDeputadoId              | string     | Identificador alternativo do parlamentar                         |
| ideDocumento              | string     | Identificador do documento fiscal                                |
| urlDocumento              | string     | Link para o comprovante digital                                  |


---
## Camada Silver
#### Catálogo técnico (Transformações e padronização)

**Tabela:** `mvp_ed_ceap.layer_silver.silver_ceap_despesas`  

**Descrição:** Tabela com dados tratados, tipados, padronizados e avaliados quanto à qualidade, servindo de base para a modelagem analítica.  
**Origem:** Camada Bronze (`layer_bronze.bronze_ceap_despesas`).  
**Camada:** Silver.  
**Granularidade:** Uma linha por documento de despesa.  
**Chave primária lógica:** ideDocumento.  

**Visão geral do mapeamento**

| Ação       | Bronze    | Silver                   |
| ---------- | --------- | ------------------------ |
| Estrutura  | CSV cru   | Tabela Spark estruturada |
| Tipos      | String    | Tipos corretos           |
| Datas      | Texto ISO | Date                     |
| Valores    | String    | Decimal                  |
| Duplicatas | Mantidas  | Avaliadas                |
| Outliers   | Mantidos  | Identificados            |
| Semântica  | Original  | Preservada               |

**Campos e Tipagem**  
Mapeamento campo a campo.

**Parlamentar / Identificação pessoal**
| Campo Bronze            | Campo Silver               | Tipo Silver | Regra / Observação                                     |
| ----------------------- | -------------------------- | ----------- | ------------------------------------------------------ |
| `txNomeParlamentar`     | `nome_parlamentar`         | string      | `trim()`                                               |
| `cpf`                   | `cpf`                      | string      | manter como string e somente dígitos                   |
| `ideCadastro`           | `id_cadastro`              | int         | `cast(int)`                                            |
| `nuCarteiraParlamentar` | `num_carteira_parlamentar` | int         | `cast(int)`                                            |
| `nuLegislatura`         | `num_legislatura`          | int         | `cast(int)`                                            |
| `codLegislatura`        | `cod_legislatura`          | int         | `cast(int)`                                            |
| `sgUF`                  | `sigla_uf_parlamentar`     | string      | `trim()+upper()`                                       |
| `sgPartido`             | `sigla_partido`            | string      | `trim()+upper()`                                       |
| `nuDeputadoId`          | `id_parlamentar`           | int         | `cast(int)` (**FK futura para dimensão**)              |

**Identificação do documento / despesa**
| Campo Bronze (CSV) | Campo Silver       | Tipo Silver | Regra / Observação                   |
| ------------------ | ------------------ | ----------- | ------------------------------------ |
| `ideDocumento`     | `id_documento`     | string      | `trim()`                             |
| `txtNumero`        | `numero_documento` | string      | `trim()` (**não é único**)           |
| `indTipoDocumento` | `tipo_documento`   | string/int  | manter                               |
| `urlDocumento`     | `url_documento`    | string      | manter                               |

**Classificação da despesa (subcota / tipo)**
| Campo Bronze (CSV)          | Campo Silver                   | Tipo Silver | Regra / Observação                       |
| --------------------------- | ------------------------------ | ----------- | ---------------------------------------- |
| `numSubCota`                | `id_tipo_despesa`              | int         | `cast(int)` (equivale ao “tipo”/subcota) |
| `txtDescricao`              | `descricao_tipo_despesa`       | string      | `trim()`                                 |
| `numEspecificacaoSubCota`   | `id_especificacao_tipo`        | int         | `cast(int)`                              |
| `txtDescricaoEspecificacao` | `descricao_especificacao_tipo` | string      | `trim()`                                 |

**Fornecedor**
| Campo Bronze (CSV) | Campo Silver         | Tipo Silver | Regra / Observação  |
| ------------------ | -------------------- | ----------- | ------------------- |
| `txtFornecedor`    | `nome_fornecedor`    | string      | `trim()`            |
| `txtCNPJCPF`       | `cnpj_cpf_formatado` | string      | `trim()`            |

**Datas e Valores**
| Campo Bronze (CSV) | Campo Silver      | Tipo Silver   | Regra / Observação      |
| ------------------ | ----------------- | ------------- | ----------------------- |
| `datEmissao`       | `data_emissao`    | date          | `to_date()` (parse ISO) |
| `vlrDocumento`     | `valor_documento` | decimal(12,2) | `cast(decimal)`         |
| `vlrGlosa`         | `valor_glosa`     | decimal(12,2) | `cast(decimal)`         |
| `vlrLiquido`       | `valor_liquido`   | decimal(12,2) | `cast(decimal)`         |
| `numMes`           | `mes_ref`         | int           | `cast(int)` (auxiliar)  |
| `numAno`           | `ano_ref`         | int           | `cast(int)` (auxiliar)  |

**Parcelamento / Lotes / Ressarcimento**
| Campo Bronze (CSV)        | Campo Silver                 | Tipo Silver   | Regra / Observação                   |
| ------------------------- | ---------------------------- | ------------- | ------------------------------------ |
| `numParcela`              | `num_parcela`                | int           | `cast(int)`                          |
| `numLote`                 | `num_lote`                   | int           | `cast(int)`                          |
| `numRessarcimento`        | `num_ressarcimento`          | string/int    | manter (alguns datasets têm alfanum) |
| `datPagamentoRestituicao` | `data_pagamento_restituicao` | date          | `to_date()`                          |
| `vlrRestituicao`          | `valor_restituicao`          | decimal(12,2) | `cast(decimal)`                      |

**Itens de Passagem (quando aplicável)**
| Campo Bronze (CSV) | Campo Silver | Tipo Silver | Regra / Observação |
| ------------------ | ------------ | ----------- | ------------------ |
| `txtPassageiro`    | `passageiro` | string      | `trim()`           |
| `txtTrecho`        | `trecho`     | string      | `trim()`           |  


**Identificação lógica da despesa:** (id_documento, id_parlamentar, data_emissao, valor_documento).


**Regras de Qualidade Avaliadas**
- Presença de valores nulos
- Valores monetários negativos ou zerados
- Regra: valor_liquido ≤ valor_documento
- Duplicidades lógicas (id_documento, id_parlamentar, data_emissao)
- Consistência temporal (ano da data vs. ano de referência)
- Identificação de outliers monetários  

> *Observação: Registros problemáticos são identificados e documentados, mas não removidos automaticamente.*

---
## Camada Gold
#### Catálogo Analítico (Esquema Estrela, Fato e Dimensões)

**Descrição:** A camada Gold concentra os dados prontos para análise, com semântica de negócio definida, métricas consolidadas e estrutura adequada para consultas SQL analíticas e dashboards.  
**Origem:** Camada Silver (`layer_silver.silver_ceap_despesas`).  
**Camada:** Gold.  
**Granularidade:** Uma linha por documento de despesa.  

### FATO
**Tabela Fato:** `mvp_ed_ceap.layer_gold.fato_despesa_ceap`  

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
**Tabela Dimensão Parlamentar:**  
`mvp_ed_ceap.layer_gold.dim_parlamentar`
| Campo            | Tipo   |
| ---------------- | ------ |
| id_parlamentar   | int    |
| nome_parlamentar | string |
| sigla_partido    | string |
| sigla_uf         | string |
| num_legislatura  | int    |
    
**Tabela Dimensão Tempo:**  
`mvp_ed_ceap.layer_gold.dim_tempo`
| Campo     | Tipo   |
| --------- | ------ |
| id_tempo  | int    |
| data      | date   |
| ano       | int    |
| mes       | int    |
| trimestre | int    |
| nome_mes  | string |

**Tabela Dimensão Tipo de Despesa**  
`mvp_ed_ceap.layer_gold.dim_tipo_despesa`
| Campo                        | Tipo   |
| ---------------------------- | ------ |
| id_tipo_despesa              | int    |
| descricao_tipo_despesa       | string |
| id_especificacao_tipo        | int    |
| descricao_especificacao_tipo | string |

**Tabela Dimensão Partido**  
 `mvp_ed_ceap.layer_gold.dim_partido`
| Campo         | Tipo   | Observação                               |
| ------------- | ------ | ---------------------------------------- |
| id_partido    | string | Sigla do partido (PK)                    |
| sigla_partido | string | Redundância proposital para legibilidade |

**Tabela Dimensão Unidade Federativa:**  
`mvp_ed_ceap.layer_gold.dim_uf`
| Campo    | Tipo   |
| -------- | ------ |
| id_uf    | string |
| sigla_uf | string |

---

## Observações de Consistência

- `id_partido` e `id_uf` definidos corretamente como **string**
- Tipagem monetária padronizada em decimal(12,2)
- Consistência entre Catálogo, Diagrama e Notebooks

---

