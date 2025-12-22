#📝 Autoavaliação

O desenvolvimento deste MVP permitiu a aplicação prática dos principais conceitos da **Sprint: Engenharia de Dados** e as disciplinas estudadas: **Banco de Dados, Data Warehouse e Data Lake, Gestão e Governança de Dados**.  

A arquitetura adotada, baseada no padrão Medallion (Staging, Bronze, Silver e Gold), mostrou-se adequada para garantir rastreabilidade, organização e separação clara de responsabilidades entre as camadas. A utilização da plataforma Databricks facilitou a implementação do pipeline, bem como a execução de análises exploratórias e consultas analíticas em SQL.  

Durante o desenvolvimento, alguns desafios técnicos foram enfrentados, especialmente no entendimento detalhado da documentação do conjunto de dados da CEAP e na definição correta da granularidade e tipagem das tabelas. Esses desafios contribuíram para um maior cuidado na construção da camada Bronze, assegurando a preservação fiel dos dados originais, e no tratamento técnico aplicado na camada Silver.  

Em relação à análise de dados, o MVP conseguiu responder às principais perguntas de negócio propostas, fornecendo uma visão consolidada dos gastos da CEAP no período analisado.  
A análise de qualidade dos dados evidenciou que, apesar da existência de valores nulos, zerados ou atípicos, tais registros refletem características legítimas do conjunto de dados e, portanto, foram preservados para manter a integridade da informação.  

Como oportunidades de melhoria, destaca-se a possibilidade de ampliar o escopo temporal do projeto para incluir múltiplos anos, bem como a implementação de mecanismos de atualização incremental e automação do pipeline.  
Além disso, a construção de dimensões adicionais e a aplicação de técnicas mais avançadas de análise poderiam enriquecer ainda mais as análises realizadas.  

De forma geral, o MVP atingiu seus objetivos propostos, consolidando o aprendizado dos conceitos abordados ao longo da disciplina e demonstrando a viabilidade da construção de pipelines de dados em nuvem com foco analítico e governança.  

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