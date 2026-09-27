# MVP – Engenharia de Dados

## Análise dos ajuizamentos de execuções fiscais no Tribunal de Justiça do Estado de São Paulo

Este projeto foi desenvolvido como MVP da Sprint de Engenharia de Dados da PUC-Rio e tem como objetivo implementar um pipeline de dados em ambiente de nuvem utilizando a API Pública do DATAJUD, disponibilizada pelo Conselho Nacional de Justiça – CNJ.

O pipeline do projeto foi desenvolvido no Databricks e estruturado segundo a arquitetura Medalhão, com camadas Bronze, Silver e Gold. Os notebooks foram separados conforme as atividades desenvolvidas em cada etapa do pipeline.

Segue o link para os principais documentos do repositório:

**Repositório: https://github.com/filipegadelha/PUCRIO_MVP_ENGENHARIA_DE_DADOS**

**Notebook Bronze: https://github.com/filipegadelha/PUCRIO_MVP_ENGENHARIA_DE_DADOS/blob/main/MVP_Bronze_Ingestao.ipynb**

**Notebook Silver: https://github.com/filipegadelha/PUCRIO_MVP_ENGENHARIA_DE_DADOS/blob/main/MVP_Silver_Tratamento.ipynb**

**Notebook Gold: https://github.com/filipegadelha/PUCRIO_MVP_ENGENHARIA_DE_DADOS/blob/main/MVP_Gold_FatoDimens%C3%B5es.ipynb**

**Notebook Análises: https://github.com/filipegadelha/PUCRIO_MVP_ENGENHARIA_DE_DADOS/blob/main/MVP_An%C3%A1lises.ipynb**

**ReadME: https://github.com/filipegadelha/PUCRIO_MVP_ENGENHARIA_DE_DADOS/blob/main/README.md**

---

# 1. Contexto de Negócios e Perguntas

## 1.1 Contexto

A cobrança de créditos públicos inscritos em dívida ativa é realizada por meio do processo de execução fiscal, conforme previsão da Lei n.º 6.830/1980.

Porém, a demora inerente ao processo judicial somada com a utilização massificada e pouco seletiva do instituto terminou por resultar em um quadro de congestionamento do Poder Judiciário brasileiro. Nesse sentido, o estudo Justiça em Números, publicado anualmente pelo Conselho Nacional de Justiça, vem em diversas de suas edições apresentando as estatísticas que evidenciam que os processos de execução relacionam-se com esse relevante gargalo.

Com o propósito de contornar esse problema, o Supremo Tribunal Federal, ao julgar o Tema nº 1.184, estabeleceu importantes premissas para racionalizar a utilização da execução fiscal, reconhecendo a possibilidade de extinção de execuções de baixo valor e a necessidade de tentativa de cobrança ou regularização administrativa antes de ser utilizada a via judicial.

Acompanhando as conclusões do Supremo Tribunal Federal, o Conselho Nacional de Justiça editou, em 2024, a Resolução CNJ n.º 547, reproduzindo as mencionadas condicionantes em um ato normativo que orienta a atuação do Judiciário nacional.

Passados 2 anos de aplicação da normativa, mostra-se importante investigar se o propósito de redução do número de execuções fiscais foi atingido e como essa diminuição ocorreu nos diferentes órgãos do Poder Judiciário. 

Para os fins deste projeto, foram realizados dois recortes relacionados ao período de tempo e ao órgão judiciário envolvido. Quanto ao período, optou-se por fazer a análise compreendendo o período de janeiro de 2023 a junho de 2026. Já em relação ao órgão envolvido, considerando a necessidade de operacionalização da ingestão, concentrou-se a análise no Tribunal de Justiça do Estado de São Paulo - TJ/SP.


## 1.2 Perguntas de negócio

A partir do contexto descrito acima, pretende-se investigar, por meio deste MVP, de forma específica, os resultados da Resolução CNJ n.º 547 na Justiça Estadual de São Paulo.

A partir da coleta dos dados e de sua análise, objetiva-se responder os seguintes questionamentos:

**1) Após a edição da Resolução CNJ n.º 547, houve redução das execuções fiscais na Justiça Estadual de São Paulo?**

**2) Qual o perfil de distribuição das execuções fiscais entre os órgãos do Tribunal de Justiça do Estado de São Paulo?**

**3) Eventual redução decorrente da aplicação da Resolução CNJ n.º 547 ocorreu de maneira uniforme entre os diversos órgãos?**


## 1.3. Fonte dos dados

Para a obtenção dos dados que serão utilizados para a estruturação do banco de dados e posterior extração das informações qnecessárias para as análises pretendidas, foi utilizada a API Pública da Base Nacional de Dados do Poder Judiciário - DATAJUD, regulamentado pela Resolução CNJ n.º 33/2020 e no qual são armazenadas de forma centralizada as informações dos diversos órgãos que compõem o Poder Judiciário brasileiro (https://www.cnj.jus.br/sistemas/datajud/).

A plataforma disponibiliza uma API Pública para possibilitar a consulta pública aos dados de informações processuais. As orientações para utilização do serviço constam na Wiki da plataforma (https://datajud-wiki.cnj.jus.br/api-publica). Para este MVP, considerando o recorte de tema acima exposto, foi utilizado o _endpoint_ referente ao Tribunal de Justiça do Estado de São Paulo e a filtragem da classe processual correspondente às execuções fiscais:

```text
https://api-publica.datajud.cnj.jus.br/api_publica_tjsp/_search
```

```text
classe.codigo = 1116
```

## 1.4. Estrutura dos dados brutos

A resposta da API é fornecida em formato JSON e apresenta estrutura hierárquica que compreende os seguintes campos:


```text
id
tribunal
grau
numeroProcesso
dataAjuizamento
nivelSigilo
orgaoJulgador
classe
sistema
formato
dataHoraUltimaAtualizacao
@timestamp
movimentos
assuntos
```

A documentação técnica do serviço também contém glossário de dados descrevendo os referidos campos (https://datajud-wiki.cnj.jus.br/api-publica/glossario).

Alguns dos campos possuem estruturas aninhadas, como `classe` e `orgaoJulgador`, enquanto `movimentos` e `assuntos` são estruturas do tipo lista.

Considerando o objeto deste MVP, foram selecionados para o pipeline os atributos necessários à análise dos ajuizamentos:

| Campo | Descrição |
|---|---|
| `numero_processo` | Número identificador do processo |
| `tribunal` | Tribunal de origem |
| `data_ajuizamento` | Data e hora do ajuizamento |
| `grau` | Grau de jurisdição |
| `classe_codigo` | Código da classe processual |
| `classe_nome` | Nome da classe processual |
| `orgaoJulgador_nome` | Nome do órgão julgador |


Na camada Bronze foram adicionados também metadados próprios do processo de ingestão, como o período de referência e a data/hora da carga, conforme será descrito no tópico correspondente.


## 1.5. Condições de uso dos dados

Os dados disponibilizados por meio da API de Consulta Pública do DATAJUD são públicos e equivalem apenas aos metadados processuais, não englobando informações abrangidas por segredo de justiça ou funcional, tampouco dados pessoais protegidos pela Lei Geral de Proteção de Dados.

Por sua vez, os termos de uso da API constam no termo de uso encontrado na Wiki e que também foi juntada à pasta deste projeto.

Entre as condições de maior importância para as atividades deste trabalho, está a limitação de 120 requisições por minuto. Para atendimento desta condição, na etapa de ingestão dos dados, foram inseridas condições para garantir a observância do limite e o cumprimento às condições de uso da plataforma, em especial a inclusão do comando time.sleep(1.2) na execução das chamadas da API.

Assim, a fonte dos dados deve ser identificada como:

> Conselho Nacional de Justiça – CNJ / DATAJUD.


---

# 2. Carga dos Dados

A etapa de ingestão foi implementada no notebook:

[`MVP_Bronze_Ingestao.ipynb`](./MVP_Bronze_Ingestao.ipynb)

## 2.1 Paginação

Primeiramente, a API possui limite de até 10.000 registros por consulta. Para possibilitar a recuperação de um volume superior de dados, tornou-se necessária a utilização de paginação por meio do parâmetro search_after, que permite dar continuidade à consulta a partir do último registro retornado na página anterior:

```text
sort
+
search_after
```

A cada página, o valor de ordenação do último registro retornado é utilizado como referência para a próxima chamada.

O fluxo pode ser representado da seguinte forma:

```text
Requisição
   ↓
Página de resultados
   ↓
Último valor de sort
   ↓
search_after
   ↓
Próxima página
```

## 2.2 Ingestão mensal

Por outro lado, para evitar a execução de consultas excessivamente longas e reduzir o impacto de eventuais falhas durante o processo de extração, optou-se por realizar as pesquisas em períodos mensais. Dessa forma, cada mês passou a representar uma unidade independente de ingestão, permitindo que os dados fossem extraídos, validados e persistidos separadamente.

Exemplo:

```text
2023-01 → carga independente
2023-02 → carga independente
2023-03 → carga independente
...
2026-06 → carga independente
```

Essa estratégia permitiu persistir os períodos já concluídos e retomar posteriormente apenas aqueles que apresentassem falha.

## 2.3 Controle de ingestão

Considerando a estratégia de separar os períodos de ingestão, bem como a necessidade de controlar a consistência do recebimento do retorno da API, foi criada uma tabela denominada `ControleIngestao`, contendo os períodos mensais previstos para processamento.

Sua estrutura inclui:

| Campo | Finalidade |
|---|---|
| `periodo` | Mês de referência |
| `data_inicio` | Limite inicial da consulta |
| `data_fim` | Limite final exclusivo |
| `status` | Situação da ingestão |
| `quantidade_registros` | Quantidade de registros carregados |
| `data_ingestao` | Data/hora da conclusão |
| `mensagem_erro` | Informação sobre eventual falha |

Os principais estados utilizados foram:

```text
PENDENTE
→ PROCESSANDO
→ OK
```

ou:

```text
PENDENTE
→ PROCESSANDO
→ ERRO
```

## 2.4 Tratamento de falhas

Durante os testes executados com a API foram observados a ocorrência de erros que poderiam comprometer a consistência dos dados armazenados. Entre outros, os seguintes códigos HTTP:

```text
429 – Too Many Requests
504 – Gateway Timeout
```

Para evitar que a ingestão fosse comprometida pela ocorrência de um erro transitório, foram implementadas novas tentativas de requisição com intervalo de espera.

Por outro lado, para evitar a persistência parcial dos dados de um período, situação que poderia ensejar distorções na análise às perguntas de negócio, foi estabelecida validação para que a persistência de determinado período somente ocorra quando **todas as páginas daquele mês são obtidas com sucesso**. 

Caso a extração não seja integralmente concluída, os registros parciais permanecem apenas em memória e não são gravados na camada Bronze. Essa estratégia evita que uma carga incompleta seja posteriormente interpretada como um período integralmente processado. Nos casos de execução bem-sucedida, os registros são persistidos na tabela `Bronze_Processos` utilizando modo incremental de gravação (`append`).


---

# 3. Modelagem e Catálogo de Dados

O projeto foi estruturado segundo a arquitetura Medalhão, com a persistência dos dados em três camadas: Bronze, com a gravação dos dados ingeridos por meio da API do DATAJUD; Silver, no qual ocorreu o tratamento e normalização dos dados; e Gold, no qual estruturadas as tabelas que serão utilizadas para responder às questões propostas.

```text
DATAJUD
   ↓
Bronze_Processos
   ↓
ProcessosSilver
   ↓
Gold
├── Gold_Tempo
├── Gold_OrgaoJulgador
└── Gold_FatoAjuizamento
```

## 3.1 Camada Bronze

A tabela `Bronze_Processos` contém os registros obtidos da API e selecionados para o escopo do projeto, acrescidos dos metadados operacionais da ingestão.

Seu objetivo principal é preservar os dados necessários ao processamento posterior e garantir a rastreabilidade da carga.

### Catálogo – Bronze_Processos

| Campo | Tipo esperado | Descrição | Origem |
|---|---|---|---|
| `numero_processo` | STRING | Número do processo | DATAJUD |
| `tribunal` | STRING | Sigla do tribunal | DATAJUD |
| `data_ajuizamento` | STRING | Data original recebida da API | DATAJUD |
| `grau` | STRING | Grau de jurisdição | DATAJUD |
| `classe_codigo` | BIGINT | Código da classe | DATAJUD |
| `classe_nome` | STRING | Nome da classe | DATAJUD |
| `orgaoJulgador_nome` | STRING | Órgão julgador | DATAJUD |
| `periodo_ingestao` | STRING | Período mensal da extração | Pipeline |
| `data_ingestao` | TIMESTAMP | Data/hora da carga | Pipeline |

### Evidências - Bronze_Processos
<img width="707" height="495" alt="image" src="https://github.com/user-attachments/assets/ed37027e-cca5-40c2-afca-ee98efffb6a0" />

Oportuno também trazer a composição da tabela de Controle de Ingestão, utilizada na etapa de ingestão e preparatória para a persistência na Camada Bronze:

### Catálogo – ControleIngestao

| Campo | Tipo | Descrição | Domínio / Regra | Linhagem |
|---|---|---|---|---|
| `periodo` | STRING | Identificação do mês de ingestão | Formato `yyyy-MM` | Derivado de `data_inicio` na criação inicial da tabela |
| `data_inicio` | DATE | Data inicial inclusiva do período consultado | Primeiro dia de cada mês entre 01/2023 e 06/2026 | Gerada previamente pelo pipeline para controle dos períodos |
| `data_fim` | DATE | Limite final exclusivo da consulta | Primeiro dia do mês seguinte | Derivada de `data_inicio` por adição de um mês |
| `status` | STRING | Estado da execução do período | `PENDENTE`, `PROCESSANDO`, `OK` ou `ERRO` | Criado como `PENDENTE` e atualizado durante a execução do pipeline |
| `quantidade_registros` | BIGINT | Quantidade de registros obtidos na carga concluída | Inteiro não negativo ou `NULL` enquanto não concluída | Atualizado pelo pipeline após conclusão integral da extração |
| `data_ingestao` | TIMESTAMP | Data e hora de conclusão da ingestão | `NULL` enquanto não concluída | Gerada com timestamp no término bem-sucedido da carga |
| `mensagem_erro` | STRING | Informação sobre eventual erro de processamento | `NULL` em cargas sem erro | Preenchida pelo tratamento de exceções quando a ingestão não é concluída |


## 3.2 Camada Silver

A tabela `ProcessosSilver` contém os dados submetidos às transformações de limpeza, padronização e tipagem necessárias ao consumo analítico.

As principais transformações realizadas foram:

- conversão de `data_ajuizamento` para `TIMESTAMP`;
- tratamento de diferentes formatos temporais;
- criação dos atributos `ano`, `mes` e `ano_mes_ajuizamento`;
- validação de valores nulos;
- validação da unicidade de `numero_processo`;
- padronização das colunas utilizadas no modelo.

### Catálogo – Silver_Processos

| Campo | Tipo | Descrição | Domínio / Regra | Linhagem |
|---|---|---|---|---|
| `numero_processo` | STRING | Identificador do processo judicial | Único na camada Silver | Proveniente de `Bronze_Processos.numero_processo`; duplicidades são tratadas por `ROW_NUMBER()` particionado por número do processo, mantendo-se `rn = 1` |
| `tribunal` | STRING | Tribunal de origem | Para o escopo do MVP, `TJSP` | Proveniente diretamente de `Bronze_Processos.tribunal` |
| `data_ajuizamento` | DATE | Data do ajuizamento do processo, sem componente de horário | Datas entre janeiro/2023 e junho/2026 no recorte analisado | `Bronze_Processos.data_ajuizamento` é inicialmente convertido para timestamp por `TRY_TO_TIMESTAMP`/`TRY_CAST` e posteriormente convertido para `DATE` por `TO_DATE()` |
| `ano` | INT | Ano do ajuizamento | 2023 a 2026 | Derivado da data tratada por `YEAR(data_ajuizamento)` |
| `mes` | INT | Número do mês do ajuizamento | 1 a 12 | Derivado da data tratada por `MONTH(data_ajuizamento)` |
| `ano_mes_ajuizamento` | STRING | Ano e mês do ajuizamento | Formato `yyyy-MM` | Derivado da data tratada por `DATE_FORMAT(data_ajuizamento, 'yyyy-MM')` |
| `grau` | STRING | Grau de jurisdição | Valores existentes na fonte DATAJUD | Proveniente diretamente de `Bronze_Processos.grau` |
| `orgaoJulgador_nome` | STRING | Nome do órgão julgador | Órgãos julgadores existentes no TJSP no conjunto analisado | Proveniente diretamente de `Bronze_Processos.orgaoJulgador_nome` |

### Evidências - Silver_Processos
<img width="606" height="502" alt="image" src="https://github.com/user-attachments/assets/9fafbc1a-b4d1-433d-b8ac-2f8be2b75131" />


## 3.3 Camada Gold

A camada Gold foi estruturada utilizando um **modelo dimensional em esquema estrela**.

O evento central do modelo é o ajuizamento de uma execução fiscal.

A granularidade da tabela fato foi definida como:

> **Uma linha representa um processo de execução fiscal ajuizado.**

A estrutura do modelo é:

```text
             Gold_Tempo
                  |
                  |
      Gold_FatoAjuizamento
                  |
                  |
       Gold_OrgaoJulgador
```

### Gold_Tempo

Dimensão temporal com granularidade diária.

| Campo | Tipo | Descrição | Domínio / Regra | Linhagem |
|---|---|---|---|---|
| `id_data` | INT | Chave da dimensão temporal | Formato numérico `yyyyMMdd`; valor único para cada dia | Derivado de `Silver_Processos.data_ajuizamento` por `DATE_FORMAT(..., 'yyyyMMdd')` e conversão para `INT` |
| `ano_mes_ajuizamento` | STRING | Ano e mês do ajuizamento | Formato `yyyy-MM` | Proveniente de `Silver_Processos.ano_mes_ajuizamento` |
| `ano` | INT | Ano do ajuizamento | 2023 a 2026 | Proveniente de `Silver_Processos.ano` |
| `mes` | INT | Número do mês | 1 a 12 | Proveniente de `Silver_Processos.mes` |
| `data_ajuizamento` | DATE | Data de ajuizamento com granularidade diária | Uma ocorrência por data na dimensão | Derivada de `Silver_Processos.data_ajuizamento` por `TO_DATE()`; duplicidades removidas pelo `SELECT DISTINCT` |

### Evidências - Gold_Tempo
<img width="527" height="442" alt="image" src="https://github.com/user-attachments/assets/fd42631d-f47b-4a06-b102-95d07bdc51f3" />


### Gold_OrgaoJulgador

Dimensão contendo os órgãos julgadores existentes no conjunto de dados.

| Campo | Tipo | Descrição | Domínio / Regra | Linhagem |
|---|---|---|---|---|
| `id_orgao` | INT | Chave substituta da dimensão órgão julgador | Inteiro sequencial e único | Gerado por `ROW_NUMBER() OVER (ORDER BY orgaoJulgador_nome)` sobre a relação de órgãos distintos da Silver |
| `nome` | STRING | Nome do órgão julgador | Um registro por nome distinto de órgão | Proveniente de `Silver_Processos.orgaoJulgador_nome`, após aplicação de `SELECT DISTINCT` |

### Evidências - Gold_OrgaoJulgador
<img width="577" height="457" alt="image" src="https://github.com/user-attachments/assets/6b1fac4b-6634-4b92-a5d4-d6fc4e8c3610" />


### Gold_FatoAjuizamento

Tabela fato que representa o evento de ajuizamento.

| Campo | Tipo | Descrição | Domínio / Regra | Linhagem |
|---|---|---|---|---|
| `numero_processo` | STRING | Número identificador do processo ajuizado | Único conforme a granularidade definida para a fato | Proveniente de `Silver_Processos.numero_processo` |
| `id_data` | INT | Chave estrangeira lógica para `Gold_Tempo` | Deve possuir correspondência em `Gold_Tempo.id_data` | Obtido por `LEFT JOIN` entre `Silver_Processos.data_ajuizamento` e `Gold_Tempo.data_ajuizamento` |
| `id_orgao` | INT | Chave estrangeira lógica para `Gold_OrgaoJulgador` | Deve possuir correspondência em `Gold_OrgaoJulgador.id_orgao` | Obtido por `LEFT JOIN` entre `Silver_Processos.orgaoJulgador_nome` e `Gold_OrgaoJulgador.nome` |
| `grau` | STRING | Grau de jurisdição do processo | Valores oriundos do DATAJUD | Proveniente de `Silver_Processos.grau` |

### Evidências - Gold_FatoAjuizamento

<img width="647" height="451" alt="image" src="https://github.com/user-attachments/assets/dd077dc3-6b0e-4ff5-b313-f190900aa285" />


## 3.4. Linhagem dos dados

A partir do catálogo exposto para cada uma das camadas, pode-se resumir a linhagem dos dados por meio do seguinte fluxo:

> `API Pública DATAJUD` → `Bronze_Processos` → `Silver_Processos` → `Gold_Tempo` / `Gold_OrgaoJulgador` → `Gold_FatoAjuizamento`.


Na camada Bronze são persistidos os atributos selecionados da resposta da API e os metadados relativos à ingestão.

Na Silver são executadas as principais regras de qualidade, especialmente a deduplicação por número de processo e a padronização das datas.

Finalmente, na Gold, os dados tratados são reorganizados segundo modelo dimensional, com separação entre o evento de ajuizamento e suas dimensões temporal e organizacional.

---

# 4. Pipeline de Dados

Para facilitar a organização, manutenção e entendimento do pipeline, as diferentes etapas foram distribuídas em notebooks independentes.

A estrutura adotada foi:

```text
MVP_Bronze_Ingestao.ipynb
        ↓
MVP_Silver_Tratamento.ipynb
        ↓
MVP_Gold_FatoDimensões.ipynb
        ↓
MVP_Análises.ipynb
```

Cada notebook possui uma responsabilidade específica:

| Notebook | Responsabilidade |
|---|---|
| `MVP_Bronze_Ingestao.ipynb` | Consulta à API, paginação, tratamento de falhas, controle e persistência Bronze |
| `MVP_Silver_Tratamento.ipynb` | Qualidade, limpeza, tipagem e padronização |
| `MVP_Gold_FatoDimensões.ipynb` | Construção do modelo dimensional |
| `MVP_Análises.ipynb` | Consultas analíticas e visualizações |

O uso de tabelas persistidas no Databricks permite que cada notebook utilize como entrada o resultado materializado da etapa anterior, sem dependência de variáveis mantidas exclusivamente em memória.

### Evidências - Notebooks no Databricks

<img width="1072" height="582" alt="image" src="https://github.com/user-attachments/assets/5c56e5e8-1af0-419c-9c94-847239f3533d" />


---

# 5. Qualidade de Dados

A análise de qualidade foi realizada principalmente durante a transformação da camada Bronze para Silver.

Foram considerados principalmente os aspectos de:

- completude;
- unicidade;
- consistência;
- validade;
- coerência temporal.

## 5.1 Unicidade

O número do processo foi utilizado como identificador natural dos registros.

Foram comparados:

```sql
COUNT(*)
```

e:

```sql
COUNT(DISTINCT numero_processo)
```

para identificar eventual ocorrência de duplicidades.

Importante notar que, na camada Bronze, identificaram-se duplicidades decorrentes da configuração da paginação da API. Após análise dos registros, identificou-se que, em todos, havia identidade entre todos os campos envolvidos, com exceção dos metadados de controle da própria ingestão. O tratamento das duplicidades ocorreu na persistência para a camada Silver:

### Evidências tratamento de duplicidade

<img width="730" height="465" alt="image" src="https://github.com/user-attachments/assets/73a525bc-eac8-403c-8a7e-1e14a26338b9" />


<img width="700" height="487" alt="image" src="https://github.com/user-attachments/assets/68ad67cf-436f-4a2e-ac05-85dd2795106a" />


## 5.2 Valores nulos

Foram realizadas validações nas principais colunas utilizadas no modelo, especialmente:

```text
numero_processo
data_ajuizamento
tribunal
orgaoJulgador_nome
```

As verificações foram realizadas antes da persistência da camada Silver.

### Evidências - Contagem de numero_processo e data_ajuizamento nulo na Bronze

<img width="547" height="462" alt="image" src="https://github.com/user-attachments/assets/577f055c-def8-49ac-8596-c8f9166a19cf" />

<img width="542" height="477" alt="image" src="https://github.com/user-attachments/assets/6f276ad5-5522-480c-80c8-88ee7991d3e5" />



## 5.3 Inconsistência no formato das datas

O principal problema de consistência detectado foi a existência de mais de um formato para o campo `dataAjuizamento`.

Foram encontrados, por exemplo:

```text
20260102090439
```

e:

```text
2024-01-02T08:35:16.000Z
```
### Evidências - Formato das datas


<img width="1360" height="516" alt="MVP_Evidencia_AlteracaoFormatoData" src="https://github.com/user-attachments/assets/6a8f4a67-cc49-4aec-a3b0-64d27f38662d" />


Para evitar perda de registros durante a conversão, foi utilizado tratamento tolerante aos diferentes formatos.

Após a normalização, `data_ajuizamento` passou a ser armazenada como `TIMESTAMP`.

## 5.4 Granularidade da dimensão tempo

Durante a construção inicial da camada Gold verificou-se que o componente de horário estava sendo preservado na dimensão temporal.

Isso provocava múltiplas linhas para um mesmo dia e, consequentemente, multiplicação indevida dos registros durante o `JOIN` com a tabela fato.

A dimensão foi corrigida para granularidade diária, utilizando apenas a parte correspondente à data.

Após o ajuste foram verificadas as seguintes métricas:

```sql
COUNT(*)
COUNT(DISTINCT id_data)
COUNT(DISTINCT data)
```

de modo a garantir a unicidade da chave temporal.

Esse problema evidenciou a importância da correta definição de granularidade no modelo dimensional.

---

# 6. Análise de Dados

As análises foram realizadas no notebook:

[`MVP_Análises.ipynb`](./MVP_Análises.ipynb)

Retomando a proposta inicial, temos as seguintes perguntas a serem respondidas por meio da exploração dos dados:

> Após a edição da Resolução CNJ n.º 547, houve redução das execuções fiscais na Justiça Estadual de São Paulo?

> Qual o perfil de distribuição das execuções fiscais entre os órgãos do Tribunal de Justiça do Estado de São Paulo?

> Eventual redução decorrente da aplicação da Resolução CNJ n.º 547 ocorreu de maneira uniforme entre os diversos órgãos?

## 6.1 Evolução temporal dos ajuizamentos

> Após a edição da Resolução CNJ n.º 547, houve redução das execuções fiscais na Justiça Estadual de São Paulo?

A primeira pergunta envolve a compreensão do quantitativo de execuções fiscais ao longo do período e se após a edição da Resolução CNJ n.º 547/2024, publicada em 22 de fevereiro de 2024, houve alteração desse volume.

Para esse objetivo, será realizada consulta na camada Gold da quantidade de processos, agrupando-se o resultado pelo ano e pela combinação de ano e mês do ajuizamento. Seguem os gráficos gerados, no Databricks, a partir destas consultas:

<img width="1377" height="511" alt="image" src="https://github.com/user-attachments/assets/e687e70a-874f-4e76-bd0a-9ff12a522d09" />

<img width="1317" height="546" alt="image" src="https://github.com/user-attachments/assets/36c1197d-e9e7-4a23-a076-9b89f85d0953" />



O resultado da consulta demonstra que houve uma diminuição bastante significativa da quantidade de ajuizamentos na proximidade da publicação da Resolução CNJ n.º 547/2024 (22/02/2024). Esse comportamento é explicado pelo fato de que o Tema n.º 1184 do STF, que trazia as conclusões que estão na regulamentação, já havia sido julgado (julgamento em 19/12/2023), de forma que os entes públicos já tinham iniciado a adotar medidas para se adequar ao entendimento jurisprudencial acerca do ajuizamento.

Após a publicação, houve uma queda significativa do número de ajuizamentos, os quais permanecem atualmente em volume médio significativamente inferior àquele observado em 2023.

Interessante notar a existência de picos nos meses de dezembro ao longo dos anos, mesmo em 2023. Essa sazonalidade pode estar relacionada a algum tipo de fatores periódicos ou mesmo circunstâncias operacionais ou administrativas que repercutem na concentração dos ajuizamentos em determinados meses.


## 6.2 Distribuição por órgão julgador

> Qual o perfil de distribuição das execuções fiscais entre os órgãos do Tribunal de Justiça do Estado de São Paulo?

O segundo questionamento destina-se a compreender qual o perfil de distribuição das execuções fiscais entre os diversos órgãos judiciários do Tribunal de Justiça do Estado de São Paulo.

Para essa segunda análise, será realizada consulta na camada Gold da quantidade de processos, agrupando-se o resultado pelo órgão julgador. Tendo em vista a quantidade de órgãos julgadores (aproximadamente 400), optou-se pela realização de um recorte para análise dos 20 primeiros órgãos:

> <img width="1337" height="567" alt="image" src="https://github.com/user-attachments/assets/a45d20ed-954b-400f-a8ae-4637c19fe84d" />


Extraindo-se os 20 primeiros registros, observa-se que o volume de distribuições é significativo superior à média na Vara de Execuções Fiscais Municipais da Capital (217.786 execuções). A Vara de Execuções Fiscais Estaduais da Capital ocupa a 4ª posição, com um quantitativo significativamente inferior de execuções ajuizadas (59.243).

Um outro achado relevante na análise é que o quantitativo de distribuição não segue a proporcionalidade que se poderia esperar do tamanho dos municípios em que sediadas as unidades judiciárias. Com efeito, o Setor de Execuções Fiscais de Campinas, que é o 3º maior município de São Paulo segundo o IBGE, ocupa a 11ª posição, havendo varas em municípios de menor porte com maior volume de ajuizamentos.


## 6.3 Comportamento do volume de ajuizamento entre os órgãos após a Resolução CNJ nº 547/2024

> Eventual alteração decorrente da aplicação da Resolução CNJ n.º 547 ocorreu de maneira uniforme entre os diversos órgãos?

O terceiro questionamento destina-se a compreender como os efeitos da Resolução CNJ n.º 547/2024 repercutiram nos diversos órgãos judiciários do Tribunal de Justiça paulista.

Considerando o número de órgãos identificados, optou-se por realizar um recorte de um universo que possua maior representatividade estatística dentro do panorama analisado. Dessa maneira, a análise temporal tomou como base o grupo identificado na pergunta anterior, qual seja, os 20 órgãos judiciários com maior distribuição.

Para direcionar a consulta a este objetivo, foi utilizada uma CTE inicial na query, aproveitando o código utilizado na pergunta anterior, para possibilitar que o relacionamento com a tabela-fato Ajuizamento e a tabela-dimensão Tempo se limitassem ao grupo de órgãos que será analisado. Os gráficos do resultado por ano e por ano-mês seguem abaixo apresentados:

<img width="1342" height="567" alt="image" src="https://github.com/user-attachments/assets/8b02169b-9cb0-4cb1-987a-bf3e34db0ee6" />

<img width="1322" height="581" alt="image" src="https://github.com/user-attachments/assets/355bc850-270e-4679-854c-a188808e3ff7" />


O resultado da consulta traz uma demonstração que, no período imediatamente posterior à edição da Resolução CNJ n.º 547/2024, houve uma queda significiativa no número de ajuizamentos. Essa tendência foi seguida por uma posterior elevação, na maioria dos órgãos observados, seguido por uma estabilização do crescimento em momento posterior. Embora haja movimentos sazonais de alta dentro da série temporal, infere-se que, no geral, os patamares totais mantiveram-se inferiores àqueles observados antes da edição da normativa do Conselho Nacional de Justiça. 

Uma situação particular envolve a Vara de Execuções Fiscais Estaduais da Capital, conforme se percebe da série temporal a ela relacionada:

<img width="1342" height="572" alt="image" src="https://github.com/user-attachments/assets/edb8cc33-902d-4a99-9181-c20e8e06cf99" />

<img width="1332" height="555" alt="image" src="https://github.com/user-attachments/assets/9bd5677e-de6e-4364-abac-26b393d5347d" />

Essa distinção em relação às demais ocorreu porque, a partir de 07/01/2025, a Vara das Execuções Fiscais Estaduais da Fazenda Pública passou a ter competência para julgamento das execuções de todo o Estado de São Paulo, ressalvando-se apenas aquelas propostas no Núcleo Especializado de Justiça 4.0 - Execuções Fiscais e Estaduais do Interior e Litoral (Resolução n° 944/2024 do Órgão Especial do TJ/SP).

Em outras palavras, as execuções fiscais estaduais, antes propostas nos diversos órgãos judiciários do Estado de acordo com sua competência territorial, passaram a ser concentradas em apenas dois órgãos: a a Vara das Execuções Fiscais Estaduais da Fazenda Pública da Capital ou o Núcleo Especializado de Justiça 4.0 - Execuções Fiscais e Estaduais do Interior e Litoral. Isso implicou em uma ampliação significativa de sua competência e, consequentemente, a variação do volume de processos se comportou de forma diferente para esse órgão.

Um achado também relevante a ser descrito diz respeito à situação do Núcleo Especializado de Justiça 4.0 - Execuções Fiscais Estaduais. 

O referido Núcleo foi criado em 2025 estaria destinado ao processamento de ações de maior valor ou que tivessem alguma relevância estratégica para o Estado de São Paulo. As ações com data de ajuizamento anterior a sua criação e que estão vinculadas a ele referem-se aos processos que tramitavam nas Varas de origem e foram redistribuídos ao Nucleo.

Dentro desse contexto, a série temporal evidencia que, após criação e implantação do Núcleo, o número de ajuizamentos mantém-se em um patamar mais reduzido e a maior parte dos casos associados ao órgão são de processos redistribuídos e ajuizados antes de sua implantação:

<img width="1370" height="562" alt="image" src="https://github.com/user-attachments/assets/2a67b628-9b71-45e9-86a5-a3cea2aa07a5" />

<img width="1357" height="536" alt="image" src="https://github.com/user-attachments/assets/40072044-2c30-421a-b04e-a443bc1ddc5c" />

Logo, em conclusão, embora seja possível estabelecer uma associação temporal entre a diminuição do número de ajuizamentos de execuções fiscais após a vigência da Resolução CNJ n.º 547/2024, o estabelecimento de uma causalidade direta desse volume dentro dos diversos órgãos exigiria um maior detalhamento de fatores estruturais e administrativos, em especial as alterações envolvendo organização judiciária e a própria política de ajuizamento adotada por cada ente público na sua atividade de cobrança.

---

# 7. Autoavaliação

A experiência com este primeiro MVP foi bastante desafiadora mas igualmente enriquecedora. Por ser um aluno com formação jurídica e sem formação prévia em tecnologia da informação, foi o primeiro contato que tive com alguns conceitos e ferramentas, como é o caso do Databricks, e também uma oportunidade inicial de aplicar na prática muitos conceitos que eu havia estudado apenas teoricamente mas que, até então, não tive como implementar em algum projeto.

Por conta desse contexto de formação, optei por um projeto com uma delimitação bastante fechada de forma a possibilitar que eu dedicasse mais atenção às etapas do pipeline de dados e não houvesse ampliação do objeto que deslocasse a atenção para a etapa de análise e visualização dos dados.

A etapa de ingestão foi especialmente didática porque nela pude perceber algumas dificuldades que podem surgir na prática e os cuidados que um profissional de dados precisa ter para garantir a consistência dos resultados. Foi o caso da identificação dos erros HTTP durante a ingestão, que estava ocasionando a persistência parcial dos dados, e também a mudança do formato da data no curso da série temporal analisada, o que fez com que o código inicial da requisição ao DATAJUD retornasse com resultado incompleto em um primeiro momento. Esses erros exigiram que eu revisitasse todo o código que havia sido inicialmente construído e incluísse as validações necessárias para garantir uma melhor qualidade do resultado.

Em relação a pontos de melhoria, seria possível aprofundar a análise para envolver os 'movimentos' oferecidos por meio da API do DATAJUD. A inclusão dos movimentos na análise, porém, aumentaria bastante a complexidade da atividade, em especial por conta do volume envolvido, uma vez que o campo é uma estrutura de lista, com códigos próprios, e com uma cardinalidade de N:1 em relação os processos (um processo possui várias movimentações). Logo, seria necessária uma estruturação mais robusta das camadas e, também, uma análise mais detida dos diversos códigos de movimentos associados.

Também visualizo possibilidade de enriquecimento do trabalho por meio de scrapping de dados públicos na página do Tribunal de Justiça para possibilitar uma melhor identificação de outras informações processuais (ex: partes envolvidas e valor da causa).

Por fim, consegui direcionar o projeto para uma análise muito útil para a minha área de atuação atual e obtive respostas para questões que oferecerem insights úteis para o meu trabalho. Fiquei feliz de poder aplicar os conhecimentos que obtive na sprint com este projeto.



