# 

## Documento consolidado de definição arquitetural e analítica

**Status do documento:** versão consolidada com os quatro schemas conhecidos no momento

**Escopo atual:** arquitetura lógica, camadas, stack tecnológica, esteira ELT, modelagem dimensional inicial, domínio fiscal principal, domínio de folha/pessoal, domínio institucional auxiliar, uso do Qdrant no RAG e papel de cada tecnologia entre as camadas

**Observação:** este documento já é suficientemente detalhado para orientar a implementação inicial e deverá ser revisado posteriormente à medida que novos schemas, regras de negócio e convenções de integração forem confirmados.

---

# 1. Objetivo deste documento

Este documento consolida, em um único lugar, a definição da arquitetura de dados e IA para o ecossistema analítico do Portal da Transparência de Sergipe.

O objetivo não é apenas dizer “qual ferramenta usar”, mas explicar:

- por que a stack foi escolhida;
- por que as camadas existem;
- que tipo de dado vive em cada camada;
- o que acontece entre uma camada e outra;
- o que será feito em Python, o que será feito em SQL e o que será orquestrado;
- como os quatro bancos-fonte passam a influenciar a modelagem;
- qual é a modelagem dimensional recomendada agora que todos os schemas principais foram apresentados;
- como essa base servirá, ao mesmo tempo, para dashboard e para um agente de IA especialista;
- como separar dado numérico auditável de dado documental/conceitual.

A meta final desta arquitetura é suportar dois grandes produtos:

1. um **banco analítico corporativo** para o Portal da Transparência;
2. um **agente de IA especialista**, focado principalmente em **arrecadação pública e execução orçamentária**, com possibilidade de expansão futura para outros domínios.

---

# 2. Contexto de negócio e princípio orientador

O foco principal do agente e do analítico **não** é a ouvidoria em si.

O foco principal é:

- arrecadação pública;
- receita prevista, atualizada e realizada;
- execução da despesa;
- empenho, liquidação e pagamento;
- contratos, convênios, licitações e favorecidos;
- consultas numéricas, analíticas e explicativas sobre finanças públicas.

Isso muda a forma de modelar.

Mesmo que o ecossistema tenha outras fontes, como ouvidoria/e-SIC e folha/pessoal, a arquitetura não deve tratar tudo como um bloco homogêneo. O certo é organizar o warehouse por **domínios de negócio**, com uma espinha dorsal fiscal e domínios complementares acoplados por dimensões conformadas.

Em termos práticos, este documento assume o seguinte princípio:

> O domínio fiscal/arrecadatório é o núcleo do DW e do agente; os demais domínios entram como expansão controlada, não como centro da modelagem.
> 

---

# 3. Inventário das fontes e papel de cada uma

Agora que os quatro schemas foram analisados, já dá para definir claramente o papel de cada fonte no desenho final.

## 3.1 `consumer_sefaz`

É a **fonte principal do domínio fiscal**.

Ela já entrega tabelas muito próximas do consumo analítico do portal, inclusive com vários objetos que parecem ter sido preparados para consumo, conciliação ou exploração gerencial.

Principais objetos relevantes:

- `previsao_realizacao_receita`
- `despesa_detalhada`
- `empenho`
- `liquidacao`
- `pagamento`
- `contrato`
- `contrato_empenho`
- `convenio_despesa`
- `receita`
- `dados_orcamentarios`
- `unidade_gestora`
- `totalizadores_execucao`
- `ordem_fornecimento`
- `restos_a_pagar`
- `base_despesa_licitacao`
- `base_despesa_credor`

### Papel na arquitetura

- backbone do DW fiscal;
- principal fonte de serving para dashboard e IA fiscal;
- fonte mais “portalizada” e próxima do consumo final;
- melhor candidata para fatos mensais e parte dos fatos detalhados.

### Conclusão

Se fosse preciso escolher uma única fonte para iniciar o DW fiscal, seria o `consumer_sefaz`.

---

## 3.2 Oracle CGE / IGESP

O schema do IGESP mostrou algo muito importante: ele **não** é apenas uma fonte complementar qualquer. Ele traz tanto tabelas transacionais de execução quanto tabelas-dicionário de classificação fiscal e administrativa.

Principais objetos relevantes:

### Tabelas transacionais

- `TB_EMP_EMPENHO_IGESP`
- `TB_LIQ_LIQUIDACAO_IGESP`
- `TB_PIG_PAGAMENTO_IGESP`

### Tabelas de referência / dicionário

- `TB_ORI_ORGAO_IGESP`
- `TB_UGI_UNIDADE_GOV_IGESP`
- `TB_UOR_UNIDADE_ORG_IGESP`
- `TB_FAI_FAVORECIDO_IGESP`
- `TB_FIG_FUNCAO_IGESP`
- `TB_SFI_SUBFUNCAO_IGESP`
- `TB_FRI_FONTE_RECURSO_IGESP`
- `TB_CDI_CATEGORIA_DESPESA_IGESP`
- `TB_GDI_GRUPO_DESPESA_IGESP`
- `TB_EDI_ELEMENTO_DESPESA_IGESP`
- `TB_TDI_TIPO_DESPESA_IGESP`
- `TB_PGI_PROGRAMA_GOVERNO_IGESP`
- `TB_API_ACAO_PROJETO_ATVD_IGESP`
- `TB_SOI_SITUACAO_OB_IGESP`

### Papel na arquitetura

- fonte de **conformance fiscal**;
- fonte de **dicionário oficial de classificações**;
- fonte de reconciliação para empenho, liquidação e pagamento;
- fonte de enriquecimento institucional e cadastral.

### Conclusão

O IGESP muda a modelagem porque ele permite sair de uma modelagem apenas consumidora de tabelas prontas e passar para um desenho com **dimensões fiscais canônicas**, o que é muito melhor para explicar dados, fazer drill-down e dar suporte a um agente de IA com semântica estável.

---

## 3.3 Oracle SEAD

O schema SEAD trouxe um novo domínio, muito claro: **folha/pessoal**.

Views relevantes:

- `VW_TRANSPARENCIA`
- `VW_TRANSPARENCIA_RUBRICA`
- `VW_VINCULO_VAGA`
- `VW_ORGAO`

### Papel na arquitetura

- fonte principal do domínio de folha e pessoal;
- base para futuros dashboards de remuneração/servidores;
- fonte separada do núcleo inicial do agente de arrecadação.

### Conclusão

O SEAD não muda o foco principal do agente agora, mas muda a arquitetura do DW como um todo: o warehouse deixa de ser somente fiscal e passa a ter ao menos um segundo domínio de primeira classe.

---

## 3.4 `esic`

O `esic` continua **fora do centro do agente**, mas não deixa de ser útil.

### Objetos relevantes

- `entidades`
- `responsavel`
- `competencias`
- `municipios`
- `estados_brasil`
- `TB_REPASSE_IPVA`

### Papel na arquitetura

- cadastro institucional auxiliar;
- enriquecimento de nomes, siglas, telefone, site e estrutura institucional;
- eventual apoio a consultas específicas;
- observação especial para `TB_REPASSE_IPVA`, que é um dado fiscal isolado e pode virar um mart auxiliar específico de repasse.

### Conclusão

O `esic` não deve dirigir o núcleo do DW fiscal, mas não deve ser descartado integralmente. Ele entra como **domínio auxiliar**, especialmente em dimensões institucionais e, se fizer sentido, em algum mart específico de repasse municipal.

---

---

# 4. Stack tecnológica definida

A stack recomendada e considerada definida neste momento é:

## Banco analítico / dimensional

**PostgreSQL**

### Por que

- robusto;
- maduro;
- excelente para modelagem analítica relacional;
- suporta materialized views;
- integra muito bem com dbt;
- excelente para marts e camadas semânticas;
- viável on-prem, coerente com a infraestrutura local.

---

## Orquestrador

**Apache Airflow**

### Por que

- ideal para pipelines batch com dependências;
- ótimo para múltiplas fontes heterogêneas;
- DAGs em Python;
- retries, logs, SLA, observabilidade e reprocessamento;
- adequado para extração, transformação, testes e indexação documental.

---

## Extração / ingestão

**Python**

### Por que

- conecta com PostgreSQL, Oracle e MySQL/MariaDB com facilidade;
- melhor para lidar com incremental heterogêneo;
- melhor para snapshot, hash diff e controle de lotes;
- melhor para integração com APIs e serviços externos se necessário.

---

## Transformação analítica

**dbt Core + SQL**

### Por que

- separa transformação de ingestão;
- excelente para staging, core, dimensional e marts;
- versiona lógica de negócio em SQL;
- gera lineage e documentação;
- permite testes;
- favorece governança e auditabilidade;
- é a abordagem ideal para ELT dentro do PostgreSQL.

---

## Camada vetorial / documental

**Qdrant**

### Por que

- especializado em busca vetorial;
- adequado para RAG com filtros e payloads;
- ótimo para conteúdo documental, não para fatos numéricos;
- funciona bem como base de legislação, FAQ, glossário, textos metodológicos e páginas explicativas do portal.

---

## Modelo LLM

**Qwen 3.5 9B ou superior**

### Por que

- compatível com a estratégia local/on-prem;
- adequado ao ecossistema do agente com LangChain;
- possibilidade de operar em CPU, aceitando latências de alguns segundos em troca de maior qualidade.

---

## Orquestração do agente

**LangChain**

### Por que

- integra ferramentas;
- permite separar SQL e RAG;
- adequado para construir um agente especialista orientado a workflow;
- permite isolar consulta estruturada da parte documental.

---

# 5. O que é dbt e por que ele entra nesta arquitetura

## 6.1 Explicação simples

O dbt é a ferramenta que organiza a transformação de dados **dentro do warehouse**.

Ele não existe para extrair dados de origem. Ele existe para transformar, documentar, testar e versionar a lógica analítica depois que os dados já chegaram ao PostgreSQL.

## 6.2 Em termos práticos

Com dbt, você escreve modelos SQL como:

- `stg_consumer_sefaz__empenho.sql`
- `core_execucao_despesa.sql`
- `dim_unidade_gestora.sql`
- `fato_receita_realizacao_mensal.sql`
- `dm_ai__receita_classificacao.sql`

E o dbt cuida de:

- compilar;
- resolver dependências;
- materializar como tabela/view/incremental;
- rodar testes;
- gerar lineage;
- gerar documentação.

## 6.3 Por que não fazer tudo em Python

Porque Python é excelente para ingestão e automação, mas ruim como padrão principal de transformação analítica em grande escala quando comparado a SQL bem modelado dentro do warehouse.

Se toda regra de negócio for parar em Python, você tende a ter:

- menos transparência;
- menos auditabilidade;
- mais código procedural do que o necessário;
- mais dificuldade para que outras pessoas entendam as transformações.

## 6.4 Por que não fazer tudo só em SQL solto

Porque sem dbt você perde:

- organização dos modelos;
- lineage;
- documentação centralizada;
- testes padronizados;
- disciplina de dependência entre camadas.

## 6.5 Conclusão

Nesta arquitetura, o dbt é a **espinha dorsal da transformação entre raw, staging, core, DW e marts**.

---

# 6. ETL ou ELT?

Tecnicamente, o desenho recomendado é **ELT**, não ETL clássico.

## 7.1 ETL clássico

- extrai da origem;
- transforma fora do warehouse;
- carrega resultado final.

## 7.2 ELT recomendado aqui

- extrai da origem;
- carrega dado bruto no PostgreSQL analítico;
- transforma dentro do PostgreSQL com dbt/SQL.

## 7.3 Por que ELT é melhor aqui

- preserva dado bruto para auditoria;
- facilita replay e reprocessamento;
- centraliza a lógica analítica no warehouse;
- aproveita o banco analítico como engine de transformação;
- reduz código procedural desnecessário.

## 7.4 Regra prática do projeto

### Python

- extrai;
- carrega raw;
- controla incrementalidade na origem;
- faz indexação documental no Qdrant.

### dbt/SQL

- transforma da raw até o consumo analítico.

---

# 7. Camadas lógicas da arquitetura

A seguir está a explicação completa, camada por camada, com:

- finalidade;
- tipo de dado presente;
- exemplos de colunas;
- tecnologia usada antes/depois;
- por que a camada existe.

---

## 8.1 Camada de fontes

### O que é

É o conjunto dos bancos operacionais e bases originais de onde o ecossistema analítico puxará dados.

### Fontes nesta arquitetura

- `consumer_sefaz`
- `Oracle CGE / IGESP`
- `Oracle SEAD`
- `esic`

### Tipo de dado nesta camada

Dados operacionais, administrativos, fiscais e institucionais como foram modelados pelos sistemas de origem.

### Exemplos de colunas reais

### `consumer_sefaz.previsao_realizacao_receita`

- `cd_categoria_economica`
- `cd_origem`
- `cd_especie`
- `cd_desdobramento`
- `cd_tipo`
- `cd_unidade_gestora`
- `dt_ano_exercicio_ctb`
- `nu_mes`
- `vl_previsto`
- `vl_atualizado`
- `vl_realizado`

### `consumer_sefaz.despesa_detalhada`

- `cd_natureza_despesa`
- `cd_orgao`
- `cd_programa_governo`
- `cd_funcao`
- `cd_sub_funcao`
- `cd_unidade_gestora`
- `dt_ano_exercicio_ctb`
- `nu_mes`
- `vl_dotacao_inicial`
- `vl_dotacao_atualizada`
- `vl_total_empenhado`
- `vl_total_liquidado`
- `vl_total_pago`

### `IGESP.TB_EMP_EMPENHO_IGESP`

- `EMP_NRANO`
- `EMP_NRMES`
- `EMP_CDUG`
- `EMP_CDGESTAO`
- `EMP_NREMPENHO`
- `EMP_CDFUNCAO`
- `EMP_CDSUBFUNCAO`
- `EMP_CDPROGRAMA`
- `EMP_CDFONTERECURSO`
- `EMP_CDELEMENTODESP`
- `EMP_CDFAVORECIDO`
- `EMP_VLORIGINAL`
- `EMP_VLREFORCADO`
- `EMP_VLANULADO`
- `EMP_VLEMPENHADO`

### `SEAD.VW_TRANSPARENCIA`

- `COD_SEQ_ORGAO`
- `ANO_FOLHA`
- `MES_FOLHA`
- `NOM_SERVIDOR`
- `DES_CARGO`
- `DES_LOCAL`
- `TIP_FOLHA`
- `VLR_BRUTO`
- `COD_VINCULO`
- `DTA_ADMISSAO`
- `NOM_ORGAO`
- `SGL_ORGAO`
- `REG_JURIDICO`

### `esic.entidades`

- `idEntidades`
- `idOrgaos`
- `nome`
- `sigla`
- `site`
- `telefone`
- `endereco`
- `idEntidadePai`
- `subEntidade`

### Por que essa camada existe

Porque ela é a **origem da verdade**. O DW não substitui a origem; ele a organiza para consumo analítico.

Sem esta camada claramente identificada, você perde rastreabilidade conceitual e reconciliação.

### O que acontece depois dela

Sai da fonte e entra um processo de **ingestão em Python** para a camada raw.

---

## 8.2 Camada raw / landing

### O que é

É a cópia quase fiel dos dados de origem dentro do PostgreSQL analítico.

### O que tem nela

Tabelas espelho por fonte, por exemplo:

- `raw_consumer_sefaz.previsao_realizacao_receita`
- `raw_consumer_sefaz.despesa_detalhada`
- `raw_consumer_sefaz.empenho`
- `raw_oracle_cge.tb_emp_empenho_igesp`
- `raw_oracle_cge.tb_pig_pagamento_igesp`
- `raw_oracle_sead.vw_transparencia`
- `raw_esic.entidades`

### Colunas presentes

As colunas originais **mais** colunas técnicas de ingestão.

### Exemplo

`raw_consumer_sefaz.previsao_realizacao_receita`

- `id`
- `cd_categoria_economica`
- `cd_desdobramento`
- `cd_especie`
- `cd_origem`
- `cd_tipo`
- `cd_unidade_gestora`
- `dt_ano_exercicio_ctb`
- `nm_categoria_economica`
- `nm_desdobramento`
- `nm_especie`
- `nm_origem`
- `nm_tipo`
- `vl_previsto`
- `vl_atualizado`
- `vl_realizado`
- `nu_mes`
- `source_system`
- `source_object`
- `extracted_at`
- `batch_id`
- `row_hash`

### Por que essa camada existe

- preserva o dado bruto;
- permite replay;
- facilita auditoria;
- evita reacessar a fonte em todo reprocessamento;
- permite comparar cargas entre si.

### Tecnologia entre fonte e raw

- **Python**
- conectores específicos por banco
- carga via batch/incremental/snapshot

### O que **não** fazer aqui

- não aplicar regra de negócio complexa;
- não renomear sem necessidade;
- não consolidar diferentes fontes em uma mesma tabela raw;
- não perder colunas da origem.

---

## 8.3 Camada staging

### O que é

É a camada em que o dado bruto é padronizado tecnicamente.

### O que tem nela

Modelos SQL/dbt que limpam e alinham formatos.

Exemplos:

- `stg_consumer_sefaz__previsao_realizacao_receita`
- `stg_consumer_sefaz__despesa_detalhada`
- `stg_consumer_sefaz__empenho`
- `stg_oracle_cge__tb_emp_empenho_igesp`
- `stg_oracle_cge__tb_pig_pagamento_igesp`
- `stg_oracle_sead__vw_transparencia`
- `stg_esic__entidades`

### Tipo de transformação aqui

- cast de tipos;
- parsing de datas;
- padronização de nomes de colunas;
- remoção de duplicidade técnica;
- limpeza de whitespace;
- normalização de nulos;
- padronização de códigos textuais vs numéricos.

### Exemplo 1: `stg_consumer_sefaz__empenho`

A origem `empenho` mistura colunas muito úteis, mas com nomes de origem ainda acoplados ao sistema. Na staging, elas seriam trazidas para uma convenção consistente.

### Origem relevante

- `sq_empenho`
- `cd_unidade_gestora`
- `cd_gestao`
- `dt_ano_exercicio_ctb`
- `nm_razao_social_pessoa`
- `nu_documento`
- `vl_original_empenho`
- `vl_total_liquidado_empenho`
- `vl_total_pago_empenho`
- `vl_total_anulado_empenho`
- `cd_natureza_despesa_completa`

### Saída staging sugerida

- `empenho_numero`
- `unidade_gestora_codigo`
- `gestao_codigo`
- `ano_exercicio`
- `credor_nome`
- `credor_documento`
- `valor_original_empenho`
- `valor_total_liquidado`
- `valor_total_pago`
- `valor_total_anulado`
- `natureza_despesa_codigo`

### Exemplo 2: `stg_oracle_cge__tb_emp_empenho_igesp`

### Origem relevante

- `EMP_NRANO`
- `EMP_NRMES`
- `EMP_CDUG`
- `EMP_CDGESTAO`
- `EMP_NREMPENHO`
- `EMP_DTEMPENHO`
- `EMP_CDFUNCAO`
- `EMP_CDSUBFUNCAO`
- `EMP_CDPROGRAMA`
- `EMP_CDFONTERECURSO`
- `EMP_CDELEMENTODESP`
- `EMP_CDFAVORECIDO`
- `EMP_VLORIGINAL`
- `EMP_VLREFORCADO`
- `EMP_VLANULADO`
- `EMP_VLEMPENHADO`

### Saída staging sugerida

- `ano_exercicio`
- `mes_exercicio`
- `unidade_gestora_codigo`
- `gestao_codigo`
- `empenho_numero`
- `data_empenho`
- `funcao_codigo`
- `subfuncao_codigo`
- `programa_codigo`
- `fonte_recurso_codigo`
- `elemento_despesa_codigo`
- `favorecido_codigo`
- `valor_original`
- `valor_reforcado`
- `valor_anulado`
- `valor_empenhado`

### Por que essa camada existe

Porque ela separa **problema técnico** de **problema semântico**.

A staging ainda não é negócio consolidado; ela é dado limpo e coerente.

### Tecnologia entre raw e staging

- **dbt + SQL**

---

## 8.4 Camada core / integração semântica

### O que é

É a camada em que múltiplas tabelas limpas passam a formar **entidades de negócio**.

Aqui surgem objetos como:

- receita arrecadatória;
- execução da despesa;
- empenho consolidado;
- liquidação consolidada;
- pagamento consolidado;
- cadastro conformado de órgão;
- cadastro conformado de unidade gestora;
- classificação fiscal conformada.

### O que tem nela

Exemplos:

- `core_receita_arrecadacao`
- `core_execucao_despesa`
- `core_empenho`
- `core_liquidacao`
- `core_pagamento`
- `core_orgao`
- `core_unidade_gestora`
- `core_classificacao_despesa`
- `core_favorecido`
- `core_contrato`
- `core_convenio`
- `core_folha_mensal`

### Exemplo: `core_receita_arrecadacao`

### Fontes

- `stg_consumer_sefaz__previsao_realizacao_receita`

### Colunas de negócio

- `ano_exercicio`
- `mes`
- `unidade_gestora_codigo`
- `categoria_economica_codigo`
- `categoria_economica_nome`
- `origem_codigo`
- `origem_nome`
- `especie_codigo`
- `especie_nome`
- `desdobramento_codigo`
- `desdobramento_nome`
- `tipo_codigo`
- `tipo_nome`
- `valor_previsto`
- `valor_atualizado`
- `valor_realizado`

### Por que ela existe

Porque a tabela de origem já é forte, mas ainda não está formalizada como entidade corporativa. A core assume explicitamente que esse conjunto representa o conceito de **realização da receita**.

---

### Exemplo: `core_execucao_despesa`

### Fontes

- `stg_consumer_sefaz__despesa_detalhada`
- eventualmente enriquecimento com `stg_oracle_cge__dicionarios`

### Colunas de negócio

- `ano_exercicio`
- `mes`
- `unidade_gestora_codigo`
- `orgao_codigo`
- `programa_codigo`
- `programa_nome`
- `funcao_codigo`
- `funcao_nome`
- `subfuncao_codigo`
- `subfuncao_nome`
- `natureza_despesa_codigo`
- `natureza_despesa_nome`
- `acao_codigo`
- `subacao_codigo`
- `dotacao_inicial`
- `dotacao_atualizada`
- `credito_adicional`
- `valor_empenhado`
- `valor_liquidado`
- `valor_pago`

### Por que ela existe

Porque a execução da despesa é um conceito transversal e precisa de identidade semântica própria. A core a transforma em objeto de negócio corporativo.

---

### Exemplo: `core_orgao`

### Fontes

- `stg_oracle_cge__tb_ori_orgao_igesp`
- `stg_oracle_sead__vw_orgao`
- `stg_esic__entidades`

### Colunas sugeridas

- `orgao_codigo_cge`
- `orgao_codigo_sead`
- `orgao_seq_sead`
- `orgao_nome`
- `orgao_sigla`
- `orgao_cnpj`
- `tipo_administracao`
- `natureza_orgao`
- `orgao_supervisor_codigo`
- `inicio_vigencia`
- `fim_vigencia`
- `site`
- `telefone`
- `endereco`

### Por que ela existe

Porque é na core que você resolve o problema de múltiplos cadastros institucionais convergindo para um mesmo conceito de órgão.

### Tecnologia entre staging e core

- **dbt + SQL**

---

## 8.5 Camada dimensional `dw`

### O que é

É a camada analítica clássica, com fatos e dimensões.

Aqui os conceitos de negócio da core são reorganizados para análise e consumo em estrela.

### O que tem nela

### Dimensões

- `dim_tempo`
- `dim_orgao`
- `dim_unidade_gestora`
- `dim_classificacao_receita`
- `dim_classificacao_despesa`
- `dim_favorecido`
- `dim_contrato`
- `dim_convenio`
- `dim_vinculo_funcional`
- `dim_rubrica_folha`

### Fatos

- `fato_receita_realizacao_mensal`
- `fato_despesa_execucao_mensal`
- `fato_empenho`
- `fato_liquidacao`
- `fato_pagamento_ob`
- `fato_contrato`
- `fato_contrato_empenho`
- `fato_convenio`
- `fato_folha_mensal`
- `fato_folha_rubrica`

### Por que essa camada existe

Porque ela transforma o negócio em estrutura ideal para:

- BI;
- agregações;
- drill-down;
- métricas consistentes;
- semântica estável para dashboards e agentes.

### Tecnologia entre core e DW

- **dbt + SQL**

---

## 8.6 Camada de data marts

### O que é

É a camada de consumo simplificado.

### Subdivisão recomendada

### `dm_dashboard`

Voltado para visualização e dashboards.

### `dm_ai`

Voltado para consumo pelo agente.

### Por que dividir

Porque dashboard e agente têm necessidades diferentes.

### Dashboard quer

- rapidez visual;
- agregações prontas;
- menos joins;
- filtros simples.

### Agente quer

- semântica clara;
- menos ambiguidade;
- menos risco de SQL mal formulado;
- tabelas com nomes e colunas orientados à linguagem de negócio.

### Tecnologia entre DW e marts

- **dbt + SQL**

---

## 8.7 Camada documental / vetorial (Qdrant)

### O que é

É a camada de RAG documental.

### O que vive nela

- legislação;
- FAQ do portal;
- notas metodológicas;
- manuais;
- glossário de termos fiscais;
- páginas explicativas;
- documentação de classificação de receita e despesa;
- texto institucional que ajude o agente a explicar conceitos.

### O que **não** deve ser o núcleo dessa camada

- fatos numéricos principais;
- consultas oficiais de arrecadação;
- valores de execução como fonte primária.

Esses devem vir do PostgreSQL analítico.

### Por que essa camada existe

Porque o agente precisa responder dois tipos de pergunta:

1. **numérica/estruturada** → PostgreSQL
2. **conceitual/documental** → Qdrant

### Tecnologia

- Python para chunking e carga
- embeddings
- Qdrant para armazenamento vetorial

---

## 8.8 Camada de serving / consumo final

### O que é

É a camada por onde dashboard, APIs e agente efetivamente acessam o ecossistema analítico.

### O que pode existir aqui

- views finais;
- APIs internas;
- endpoints de consulta controlados;
- tabelas `dm_dashboard`;
- tabelas `dm_ai`;
- conectores do BI;
- serviços do agente.

### Por que ela existe

Porque você não quer cada consumidor atacando diretamente a estrutura interna do warehouse.

Ela fornece:

- governança;
- segurança;
- estabilidade semântica;
- desacoplamento entre consumo e estrutura interna.

---

# 8. O que usar entre as camadas

Agora de forma direta e objetiva.

## Fonte → Raw

**Python**

### Motivo

- conectividade heterogênea;
- controle de incrementalidade;
- snapshot;
- logging de extração;
- maior flexibilidade operacional.

---

## Raw → Staging

**dbt + SQL**

### Motivo

- limpeza técnica declarativa;
- tipagem e padronização;
- versionamento do processo.

---

## Staging → Core

**dbt + SQL**

### Motivo

- semântica de negócio;
- integração entre fontes;
- legibilidade e auditabilidade.

---

## Core → DW

**dbt + SQL**

### Motivo

- construção de dimensões e fatos;
- incrementalidade;
- facilidade para testes e documentação.

---

## DW → Marts

**dbt + SQL**

### Motivo

- entrega de produtos de dados;
- simplificação de consumo;
- pré-agregação e semântica final.

---

## DW / Marts documentais → Qdrant

**Python**

### Motivo

- embeddings;
- chunking;
- carga vetorial;
- atualização de coleções.

---

## Orquestração de tudo

**Apache Airflow**

### Motivo

- coordena dependências;
- agenda e monitora;
- roda Python e dbt;
- integra testes e publish.

---

# 9. Modelagem por domínio após os quatro schemas

Agora entra a parte de modelagem em si.

---

## 10.1 Domínio fiscal/arrecadatório (principal)

Este é o núcleo do projeto.

## Fontes principais

- `consumer_sefaz`
- `Oracle CGE / IGESP`

## Objetivo

Responder perguntas como:

- qual foi a arrecadação por período?
- qual a diferença entre previsto, atualizado e realizado?
- qual órgão executou mais despesa?
- quanto foi empenhado, liquidado e pago?
- quais contratos tiveram maior valor pago?
- quais convênios estão vigentes?
- qual credor recebeu mais em determinado recorte?

---

### Dimensões do domínio fiscal

## `dw.dim_tempo`

### Colunas sugeridas

- `sk_tempo`
- `data`
- `dia`
- `mes`
- `nome_mes`
- `trimestre`
- `semestre`
- `ano`
- `inicio_mes`
- `fim_mes`
- `flag_fim_mes`

### Por que existe

Toda análise fiscal depende de granularidade temporal consistente.

---

## `dw.dim_orgao`

### Fontes

- `TB_ORI_ORGAO_IGESP`
- `VW_ORGAO`
- `esic.entidades`

### Colunas sugeridas

- `sk_orgao`
- `nk_orgao_cge`
- `nk_orgao_sead`
- `cd_orgao_cge`
- `cd_orgao_sead`
- `seq_orgao_sead`
- `nm_orgao`
- `sgl_orgao`
- `nr_cnpj`
- `tp_administracao`
- `ds_tipo_administracao`
- `cd_natureza_orgao`
- `ds_natureza_orgao`
- `cd_orgao_supervisor`
- `site`
- `telefone`
- `endereco`
- `dt_inicio_vigencia`
- `dt_fim_vigencia`
- `fl_ativo`

### Por que existe

Unifica a identidade institucional e evita depender de nomes textuais espalhados.

---

## `dw.dim_unidade_gestora`

### Fontes

- `consumer_sefaz.unidade_gestora`
- `TB_UGI_UNIDADE_GOV_IGESP`
- `TB_UOR_UNIDADE_ORG_IGESP`

### Colunas sugeridas

- `sk_unidade_gestora`
- `cd_unidade_gestora`
- `nm_unidade_gestora`
- `sg_unidade_gestora`
- `tp_unidade`
- `cd_orgao`
- `nm_unidade_organizacional`
- `sg_unidade_organizacional`
- `dt_inicio_atividade`
- `fl_ativo`

### Por que existe

É uma dimensão central de qualquer recorte fiscal do portal.

---

## `dw.dim_classificacao_receita`

### Fonte

- `consumer_sefaz.previsao_realizacao_receita`

### Colunas sugeridas

- `sk_classificacao_receita`
- `cd_categoria_economica`
- `nm_categoria_economica`
- `cd_origem`
- `nm_origem`
- `cd_especie`
- `nm_especie`
- `cd_desdobramento`
- `nm_desdobramento`
- `cd_tipo`
- `nm_tipo`

### Por que existe

Materializa a hierarquia da receita para análise, drill-down e resposta semântica pelo agente.

---

## `dw.dim_classificacao_despesa`

### Fontes principais

- `TB_CDI_CATEGORIA_DESPESA_IGESP`
- `TB_GDI_GRUPO_DESPESA_IGESP`
- `TB_EDI_ELEMENTO_DESPESA_IGESP`
- `TB_TDI_TIPO_DESPESA_IGESP`
- `TB_FRI_FONTE_RECURSO_IGESP`
- `TB_FIG_FUNCAO_IGESP`
- `TB_SFI_SUBFUNCAO_IGESP`
- `TB_PGI_PROGRAMA_GOVERNO_IGESP`
- `TB_API_ACAO_PROJETO_ATVD_IGESP`
- complemento de `consumer_sefaz.dados_orcamentarios`

### Colunas sugeridas

- `sk_classificacao_despesa`
- `cd_categoria_despesa`
- `ds_categoria_despesa`
- `cd_grupo_despesa`
- `ds_grupo_despesa`
- `cd_elemento_despesa`
- `ds_elemento_despesa`
- `cd_tipo_despesa`
- `ds_tipo_despesa`
- `cd_fonte_recurso`
- `ds_fonte_recurso`
- `cd_funcao`
- `ds_funcao`
- `cd_subfuncao`
- `ds_subfuncao`
- `cd_programa`
- `ds_programa`
- `cd_acao`
- `ds_acao`

### Por que existe

Centraliza a classificação fiscal canônica do projeto.

---

## `dw.dim_favorecido`

### Fontes

- `TB_FAI_FAVORECIDO_IGESP`
- complemento com `consumer_sefaz.empenho`, `liquidacao`, `pagamento`

### Colunas sugeridas

- `sk_favorecido`
- `cd_favorecido`
- `tp_favorecido`
- `nm_favorecido`
- `municipio`
- `uf`
- `documento`

### Por que existe

Permite análise por credor/favorecido e melhora a resposta do agente quando a pergunta envolve recebedores de recursos.

---

## `dw.dim_contrato`

### Fonte

- `consumer_sefaz.contrato`

### Colunas sugeridas

- `sk_contrato`
- `cd_contrato`
- `cd_aditivo`
- `cd_unidade_gestora`
- `ano_exercicio`
- `dt_inicio_vigencia`
- `dt_fim_vigencia`
- `nm_fornecedor`
- `nu_documento_fornecedor`
- `tp_contrato`
- `nm_categoria`
- `vl_contrato`
- `ds_objeto_contrato`

---

## `dw.dim_convenio`

### Fonte

- `consumer_sefaz.convenio_despesa`

### Colunas sugeridas

- `sk_convenio`
- `cd_convenio`
- `cd_unidade_gestora`
- `cd_gestao`
- `cd_convenio_situacao`
- `cd_area_atuacao`
- `nm_concedente`
- `nm_beneficiario`
- `nm_convenio`
- `dt_celebracao`
- `dt_inicio_vigencia`
- `dt_fim_vigencia`
- `dt_publicacao`
- `dt_prazo_prestacao_contas`

---

### Fatos do domínio fiscal

## `dw.fato_receita_realizacao_mensal`

### Grão

1 linha por `ano + mês + unidade gestora + classificação da receita`

### Fonte

- `consumer_sefaz.previsao_realizacao_receita`

### Chaves

- `sk_tempo`
- `sk_unidade_gestora`
- `sk_classificacao_receita`

### Medidas

- `valor_previsto`
- `valor_atualizado`
- `valor_realizado`
- `qt_linhas_origem`

### Por que existe

É o fato principal de arrecadação do portal.

---

## `dw.fato_despesa_execucao_mensal`

### Grão

1 linha por `ano + mês + UG + órgão + classificação da despesa`

### Fonte

- `consumer_sefaz.despesa_detalhada`

### Chaves

- `sk_tempo`
- `sk_unidade_gestora`
- `sk_orgao`
- `sk_classificacao_despesa`

### Medidas

- `vl_dotacao_inicial`
- `vl_dotacao_atualizada`
- `vl_credito_adicional`
- `vl_total_empenhado`
- `vl_total_liquidado`
- `vl_total_pago`

### Por que existe

É o fato executivo principal para análise orçamentária consolidada.

---

## `dw.fato_empenho`

### Grão

1 linha por empenho

### Fontes

- `consumer_sefaz.empenho`
- reconciliação com `TB_EMP_EMPENHO_IGESP`

### Chaves

- `sk_tempo`
- `sk_unidade_gestora`
- `sk_orgao`
- `sk_classificacao_despesa`
- `sk_favorecido`

### Identificadores naturais importantes

- `ano_exercicio`
- `gestao_codigo`
- `unidade_gestora_codigo`
- `empenho_numero`

### Medidas

- `vl_original`
- `vl_reforcado`
- `vl_anulado`
- `vl_empenhado`
- `vl_total_liquidado`
- `vl_total_pago`

### Por que existe

É o fato detalhado central para responder perguntas específicas sobre despesa comprometida.

---

## `dw.fato_liquidacao`

### Grão

1 linha por liquidação

### Fontes

- `consumer_sefaz.liquidacao`
- reconciliação com `TB_LIQ_LIQUIDACAO_IGESP`

### Identificadores naturais importantes

- `ano_exercicio`
- `gestao_codigo`
- `unidade_gestora_codigo`
- `empenho_numero`
- `liquidacao_numero`

### Medidas

- `vl_bruto`
- `vl_estorno`
- `vl_pago_relacionado`

### Por que existe

Permite separar claramente compromisso, liquidação e desembolso.

---

## `dw.fato_pagamento_ob`

### Grão

1 linha por ordem bancária / pagamento

### Fontes

- `consumer_sefaz.pagamento`
- reconciliação com `TB_PIG_PAGAMENTO_IGESP`

### Identificadores naturais importantes

- `ano_exercicio`
- `gestao_codigo`
- `unidade_gestora_codigo`
- `empenho_numero`
- `ordem_bancaria_numero`

### Medidas

- `vl_bruto_pd`
- `vl_ob`
- `vl_retido_pd`
- `vl_pago`

### Por que existe

Representa a despesa efetivamente desembolsada.

---

## `dw.fato_contrato`

### Grão

1 linha por contrato

### Fonte

- `consumer_sefaz.contrato`

### Medidas principais

- `vl_contrato`

### Atributos úteis

- vigência;
- fornecedor;
- tipo;
- categoria;
- objeto.

---

## `dw.fato_contrato_empenho`

### Grão

1 linha por vínculo contrato x empenho

### Fonte

- `consumer_sefaz.contrato_empenho`

### Medidas

- `vl_contrato`
- `vl_original_ne`
- `vltotal_liquidado_ne`
- `vl_total_pago_ne`

### Por que existe

É a ponte analítica entre o mundo contratual e a execução efetiva da despesa.

---

## `dw.fato_convenio`

### Grão

1 linha por convênio

### Fonte

- `consumer_sefaz.convenio_despesa`

### Medidas

- `vl_concedente_convenio`
- `vl_contrapartida_convenio`

### Por que existe

Permite análise de convênios, vigência, concedente/beneficiário e recursos envolvidos.

---

## Opcional: `dw.fato_repasse_ipva_municipal`

### Grão

1 linha por `ano + mês + município`

### Fonte

- `esic.TB_REPASSE_IPVA`

### Medidas

- `NO_MES`
- `ATE_MES`

### Por que considerar

Embora o `esic` não seja o centro do projeto, essa tabela específica é fiscal e pode render um mart útil de repasse municipal.

---

## 10.2 Domínio de folha/pessoal

Este domínio deve ser modelado desde já, mas o consumo inicial do agente pode ficar fora dele.

### Dimensões sugeridas

- `dim_orgao` (compartilhada)
- `dim_tempo` (compartilhada)
- `dim_vinculo_funcional`
- `dim_rubrica_folha`
- `dim_cargo`
- `dim_regime_juridico`

### Fatos sugeridos

- `fato_folha_mensal`
- `fato_folha_rubrica`

---

## `dw.dim_vinculo_funcional`

### Fonte

- `VW_VINCULO_VAGA`

### Colunas sugeridas

- `sk_vinculo_funcional`
- `cod_vinculo`
- `cod_vaga`
- `seq_vinculo_vaga`
- `cod_grupo`
- `cod_cargo`
- `num_categoria`
- `num_classe`
- `num_padrao`
- `num_referencia_nivel`
- `cod_forma_ingresso`
- `dta_ini_exercicio`
- `dta_nomeacao`
- `dta_posse`
- `dta_exclusao`
- `cod_atividade`
- `sta_ativo`

### Observação

Essa dimensão deve ser tratada com cuidado por causa de possíveis implicações funcionais e históricas.

---

## `dw.dim_rubrica_folha`

### Fonte

- `VW_TRANSPARENCIA_RUBRICA`

### Colunas sugeridas

- `sk_rubrica`
- `cod_rubrica`
- `nom_abreviatura`
- `tip_rubrica_ficha_mes`

---

## `dw.fato_folha_mensal`

### Grão

1 linha por `ano + mês + vínculo + órgão + folha`

### Fonte

- `VW_TRANSPARENCIA`

### Medidas

- `vlr_bruto`

### Atributos importantes

- cargo;
- local;
- jornada;
- órgão;
- regime jurídico;
- data de admissão;
- tipo de folha.

### Observação de governança

Essa tabela deve ter controle mais rigoroso de acesso, já que contém `NOM_SERVIDOR`.

---

## `dw.fato_folha_rubrica`

### Grão

1 linha por `rubrica + folha + vínculo + mês`

### Fonte

- `VW_TRANSPARENCIA_RUBRICA`

### Medidas

- `vlr_rubrica_ficha_mes`

### Por que existe

Permite decompor remuneração em rubricas e separar proventos e descontos.

---

## 10.3 Domínio institucional auxiliar

Este domínio não deve dominar o agente, mas pode enriquecer análises.

### Fontes

- `esic.entidades`
- `esic.competencias`
- `esic.responsavel`
- `estados_brasil`
- `municipios`

### Uso recomendado

- enriquecer `dim_orgao`;
- enriquecer `dim_localidade`;
- apoiar navegação institucional do portal;
- apoiar consultas específicas que dependam de cadastro institucional.

---

# 10. Integração entre `consumer_sefaz` e `IGESP`

Essa é uma parte crítica da modelagem.

O `consumer_sefaz` deve continuar como fonte principal de consumo fiscal, mas o IGESP deve ser usado para conformance e reconciliação.

## 11.1 Chaves naturais prováveis

### Empenho

- `consumer.dt_ano_exercicio_ctb` ↔︎ `EMP_NRANO`
- `consumer.cd_unidade_gestora` ↔︎ `EMP_CDUG`
- `consumer.cd_gestao` ↔︎ `EMP_CDGESTAO`
- `consumer.sq_empenho` ↔︎ `EMP_NREMPENHO`

### Liquidação

- `consumer.dt_ano_exercicio_ctb` ↔︎ `LIQ_NRANO`
- `consumer.cd_unidade_gestora` ↔︎ `LIQ_CDUG`
- `consumer.cd_gestao` ↔︎ `LIQ_CDGESTAO`
- `consumer.sq_empenho` ↔︎ `LIQ_NREMPENHO`
- `consumer.sq_liquidacao` ↔︎ `LIQ_NRLIQUIDACAO`

### Pagamento / OB

- `consumer.dt_ano_exercicio_ctb` ↔︎ `PIG_NRANO`
- `consumer.cd_unidade_gestora` ↔︎ `PIG_CDUG`
- `consumer.cd_gestao` ↔︎ `PIG_CDGESTAO`
- `consumer.sq_empenho` ↔︎ `PIG_NREMPENHO`
- `consumer.sq_ob` ↔︎ `PIG_NRORDEMBANCARIA`

## 11.2 Estratégia recomendada

Não usar o IGESP para substituir o `consumer_sefaz` no consumo. Usá-lo para:

- validar integridade;
- preencher dicionários oficiais;
- enriquecer classificações;
- detectar divergências;
- aumentar confiança semântica.

---

# 11. Data marts recomendados

## 12.1 `dm_dashboard`

### Fiscal

- `receita_mensal`
- `receita_por_classificacao`
- `execucao_despesa_mensal`
- `execucao_por_orgao`
- `empenho_resumo`
- `liquidacao_resumo`
- `pagamento_resumo`
- `contrato_resumo`
- `convenio_resumo`
- `favorecido_resumo`

### Pessoal

- `folha_mensal_orgao`
- `folha_rubrica_mensal`
- `folha_cargo_orgao`

### Auxiliar

- `repasse_ipva_municipal`

---

## 12.2 `dm_ai`

### AI fiscal

- `ai_receita_classificacao`
- `ai_receita_mensal_ug`
- `ai_execucao_despesa`
- `ai_empenho_resumo`
- `ai_liquidacao_resumo`
- `ai_pagamento_resumo`
- `ai_contrato_resumo`
- `ai_convenio_resumo`
- `ai_favorecido_execucao`

### AI pessoal (fase posterior)

- `ai_folha_mensal`
- `ai_rubrica_folha`
- `ai_vinculo_funcional`

## Por que esses marts existem

Porque o agente não deve, de início, consultar o DW inteiro. Ele deve consultar um subconjunto de tabelas com nomes semânticos, menor ambiguidade e menos risco de SQL incorreto.

---

# 12. Como o agente de IA usará essa arquitetura

## 13.1 Perguntas estruturadas

Exemplos:
- quanto foi arrecadado em determinado mês?
- qual UG realizou mais receita?
- quanto foi pago para certo favorecido?
- qual contrato teve maior valor pago?

### Fonte de resposta

- PostgreSQL analítico (`dm_ai` / `dw` controlado)

---

## 13.2 Perguntas conceituais

Exemplos:
- o que é receita atualizada?
- qual a diferença entre empenho e pagamento?
- o que significa função e subfunção?

### Fonte de resposta

- Qdrant (RAG documental)
- eventualmente com apoio do DW para dar exemplo numérico

---

## 13.3 Perguntas mistas

Exemplos:
- quanto foi arrecadado por espécie e o que essa espécie significa?
- qual a diferença entre liquidado e pago e quais foram os valores do período?

### Fonte de resposta

- SQL no PostgreSQL + RAG no Qdrant

---

# 13. Orquestração: DAGs recomendadas

## DAG 1 — ingestão fiscal principal

### Passos

1. extrair `consumer_sefaz`
2. carregar `raw_consumer_sefaz`
3. registrar batch e metadados
4. validar contagem básica

---

## DAG 2 — ingestão de conformance fiscal

### Passos

1. extrair `Oracle CGE / IGESP`
2. carregar `raw_oracle_cge`
3. validar tabelas de referência e transacionais

---

## DAG 3 — ingestão de folha/pessoal

### Passos

1. extrair `Oracle SEAD`
2. carregar `raw_oracle_sead`
3. validar views

---

## DAG 4 — ingestão institucional auxiliar

### Passos

1. extrair `esic`
2. carregar `raw_esic`
3. validar entidades, localidades e tabelas auxiliares

---

## DAG 5 — transformação analítica

### Passos

1. `dbt run` staging
2. `dbt run` core
3. `dbt run` DW
4. `dbt run` marts
5. `dbt test`

---

## DAG 6 — indexação documental do RAG

### Passos

1. coletar documentos
2. chunking
3. embeddings
4. upsert no Qdrant
5. validação amostral

---

## DAG 7 — publicação e checks finais

### Passos

1. atualizar materialized views
2. validar contagens por domínio
3. validar somatórios críticos
4. publicar marts para consumo

---

# 14. Estratégia incremental por fonte

## 15.1 `consumer_sefaz`

### Estratégia preferencial

- incremental por `updated_at` quando existir;
- senão por chave natural + merge/upsert;
- em fatos transacionais, usar combinações de ano/UG/gestão/número do documento.

### Exemplos

- `previsao_realizacao_receita`: refresh incremental por exercício/mês ou reprocesso particionado;
- `despesa_detalhada`: incremental por exercício/mês;
- `empenho`, `liquidacao`, `pagamento`: incremental por datas de evento e chaves naturais.

---

## 15.2 `IGESP`

### Estratégia recomendada

- snapshot inicial completo;
- depois incremental por ano/mês/documento onde aplicável;
- dicionários carregados integralmente ou por diff;
- fatos por partição temporal e chaves naturais.

---

## 15.3 `SEAD`

### Estratégia recomendada

- carga por `ANO_FOLHA` + `MES_FOLHA`;
- rubricas por competência;
- vínculo/vaga com snapshot e histórico.

---

## 15.4 `esic`

### Estratégia recomendada

- foco em entidades e auxiliares institucionais;
- incremental por atualização quando existir;
- `TB_REPASSE_IPVA` por ano/mês.

---

# 15. Qualidade de dados e governança

## 16.1 Testes mínimos no dbt

- not null em chaves críticas;
- unique em chaves naturais relevantes;
- relationships entre fatos e dimensões;
- accepted values em domínios pequenos;
- testes de completude temporal por competência.

## 16.2 Reconciliações importantes

- soma de receita no DW vs origem `previsao_realizacao_receita`;
- soma de empenho/pagamento no DW vs origem `consumer_sefaz`;
- amostras comparativas `consumer_sefaz` vs `IGESP`;
- folha mensal vs rubricas agregadas no SEAD.

## 16.3 Governança de acesso

### Fiscal

Pode ter acesso mais amplo, conforme política do portal.

### Pessoal

Acesso mais controlado por conter dados de pessoas.

### e-SIC

Muito cuidado com dados pessoais do cidadão. O `esic` não deve ser replicado sem critério no domínio analítico geral.

---

# 16. Segurança, LGPD e sensibilidade

## `esic`

Evitar levar para marts gerais:
- CPF;
- e-mail;
- RG;
- telefone;
- endereço;
- conteúdo textual sensível.

## `SEAD`

Controlar acesso a:
- `NOM_SERVIDOR`;
- detalhes individualizados de folha;
- combinações que permitam perfilamento indevido.

## Regra geral

O DW corporativo deve conter o necessário para transparência pública e analytics, mas com atenção a:

- minimização de dados sensíveis;
- segregação de acesso;
- mascaramento quando necessário;
- pseudonimização quando o caso exigir.

---

# 17. Convenções recomendadas de nomenclatura

## Schemas

- `raw_consumer_sefaz`
- `raw_oracle_cge`
- `raw_oracle_sead`
- `raw_esic`
- `stg`
- `core`
- `dw`
- `dm_dashboard`
- `dm_ai`

## Prefixos de modelos dbt

- `stg_`
- `core_`
- `dim_`
- `fato_`
- `dm_`

## Nome de tabelas

Sempre orientado ao negócio, não à estrutura de origem.

Exemplo:
- melhor: `fato_receita_realizacao_mensal`
- pior: `tb_prev_real_rec_portal`

---

# 18. Roadmap de implementação recomendado

## Fase 1 — fiscal mínimo viável

- raw `consumer_sefaz`
- raw `IGESP`
- staging fiscal
- core fiscal
- dimensões básicas
- fatos principais: receita, despesa, empenho, liquidação, pagamento
- marts fiscais para dashboard e IA
- agente fiscal inicial

## Fase 2 — contratos e convênios

- contrato
- contrato_empenho
- convenio_despesa
- marts específicos
- expansão do agente

## Fase 3 — folha/pessoal

- ingestão SEAD
- dimensões e fatos de folha
- marts separados
- eventual expansão controlada do agente

## Fase 4 — institucional e auxiliares

- integração melhor do `esic.entidades`
- repasse IPVA municipal
- ajustes semânticos interdomínio

---

# 19. Conclusão final

Com os quatro schemas em mãos, a arquitetura correta é esta:

## Estrutura lógica

- **PostgreSQL** como banco analítico e dimensional corporativo;
- **Airflow** como orquestrador;
- **Python** para extração, carga raw e indexação vetorial;
- **dbt + SQL** para transformar entre raw, staging, core, DW e marts;
- **Qdrant** para o RAG documental;
- **Qwen 3.5 9B ou superior** como modelo local do agente;
- **LangChain** para orquestrar consulta estruturada + RAG.

## Estrutura semântica

- `consumer_sefaz` = backbone fiscal e principal fonte de consumo;
- `IGESP` = conformance, reconciliação e dicionário fiscal;
- `SEAD` = domínio separado de folha/pessoal;
- `esic` = domínio institucional auxiliar, com eventual mart específico de repasse IPVA.

## Estrutura em camadas

- fontes
- raw
- staging
- core
- DW dimensional
- data marts
- RAG documental
- serving controlado

## Estratégia de dados

- ELT, não ETL clássico;
- ingestão em Python;
- transformação em dbt/SQL;
- fatos e dimensões por domínio;
- marts próprios para dashboard e IA;
- separação clara entre dado numérico auditável e conteúdo documental.

## Direção estratégica

O agente inicial deve ser claramente fiscal/arrecadatório. O warehouse, porém, já deve nascer com estrutura capaz de acomodar o domínio de folha e o domínio institucional, sem misturar tudo de forma confusa.

Esse é o desenho mais sólido, mais governável e mais coerente com o que os quatro schemas revelaram até aqui.

---

# 20. Próximos passos sugeridos

1. validar este documento como definição-base;
2. transformar este desenho em modelo físico inicial de tabelas;
3. definir DAGs e frequência de carga por fonte;
4. detalhar regras de integração `consumer_sefaz` ↔︎ `IGESP`;
5. definir o primeiro conjunto de marts do agente (`dm_ai` fiscal);
6. consolidar documentação final em versão ainda mais detalhada conforme novas regras e ajustes surgirem.