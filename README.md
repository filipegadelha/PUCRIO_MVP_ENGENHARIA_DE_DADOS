# MVP – Engenharia de Dados

## Análise dos ajuizamentos de execuções fiscais no Tribunal de Justiça do Estado de São Paulo

Este projeto foi desenvolvido como MVP da Sprint de Engenharia de Dados da PUC-Rio e tem como objetivo implementar um pipeline de dados em ambiente de nuvem utilizando a API Pública do DATAJUD, disponibilizada pelo Conselho Nacional de Justiça – CNJ.

O pipeline do projeto foi desenvolvido no Databricks e estruturado segundo a arquitetura Medalhão, com camadas Bronze, Silver e Gold. Os notebooks foram separados conforme as atividades desenvolvidas em cada etapa do pipeline.


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

> **Inserir screenshot da tabela `ControleIngestao`.**

> **Inserir screenshot da tabela `Bronze_Processos` persistida no Databricks.**

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

| Campo | Tipo | Descrição |
|---|---|---|
| `numero_processo` | STRING | Identificador do processo |
| `tribunal` | STRING | Tribunal |
| `data_ajuizamento` | TIMESTAMP | Data/hora de ajuizamento padronizada |
| `ano` | INT | Ano do ajuizamento |
| `mes` | INT | Mês do ajuizamento |
| `ano_mes_ajuizamento` | STRING | Ano e mês no padrão `yyyy-MM` |
| `grau` | STRING | Grau de jurisdição |
| `orgaoJulgador_nome` | STRING | Nome do órgão julgador |

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

| Campo | Descrição |
|---|---|
| `id_data` | Chave lógica da dimensão no padrão `yyyyMMdd` |
| `data` | Data sem componente de horário |
| `ano` | Ano |
| `mes` | Mês |
| `ano_mes_ajuizamento` | Ano e mês |

### Evidências - Gold_Tempo
<img width="527" height="442" alt="image" src="https://github.com/user-attachments/assets/fd42631d-f47b-4a06-b102-95d07bdc51f3" />


### Gold_OrgaoJulgador

Dimensão contendo os órgãos julgadores existentes no conjunto de dados.

| Campo | Descrição |
|---|---|
| `id_orgao` | Chave substituta do órgão |
| `nome` | Nome do órgão julgador |

### Evidências - Gold_OrgaoJulgador
<img width="577" height="457" alt="image" src="https://github.com/user-attachments/assets/6b1fac4b-6634-4b92-a5d4-d6fc4e8c3610" />


### Gold_FatoAjuizamento

Tabela fato que representa o evento de ajuizamento.

| Campo | Descrição |
|---|---|
| `numero_processo` | Número do processo |
| `id_data` | Chave estrangeira lógica para `Gold_Tempo` |
| `id_orgao` | Chave estrangeira lógica para `Gold_OrgaoJulgador` |
| `grau` | Grau de jurisdição |
| `quantidade` | Medida unitária do ajuizamento, quando utilizada |

### Evidências - Gold_FatoAjuizamento

<img width="647" height="451" alt="image" src="https://github.com/user-attachments/assets/dd077dc3-6b0e-4ff5-b313-f190900aa285" />


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

> **Evidências - Notebooks no Databricks

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

## 5.2 Valores nulos

Foram realizadas validações nas principais colunas utilizadas no modelo, especialmente:

```text
numero_processo
data_ajuizamento
tribunal
orgaoJulgador_nome
```

As verificações foram realizadas antes da persistência da camada Silver.

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

## 6.1 Evolução temporal dos ajuizamentos

A análise mensal revelou elevada oscilação no número de ajuizamentos, com picos relevantes em determinados meses.

Em 2023 foram observados volumes mais elevados e maior volatilidade.

A partir de 2024 ocorreu redução relevante no volume mensal de ajuizamentos, embora permanecessem picos pontuais em determinados períodos.

O comportamento observado sugere a existência de fatores sazonais, administrativos ou operacionais capazes de concentrar os ajuizamentos em determinados meses.

> **Inserir gráfico da evolução mensal dos ajuizamentos.**

## 6.2 Distribuição por órgão julgador

Também foram calculados os volumes de ajuizamento por órgão julgador, permitindo identificar as unidades que concentraram maior quantidade de execuções fiscais no período analisado.

Para determinadas análises foram selecionadas as unidades com maior volume total de ajuizamentos, evitando que órgãos com número reduzido de processos produzissem variações percentuais pouco representativas.

> **Inserir gráfico dos principais órgãos julgadores.**

## 6.3 Comportamento após a Resolução CNJ nº 547/2024

Foi analisada a evolução mensal dos principais órgãos julgadores antes e após o início de 2024.

Os dados indicam redução expressiva dos ajuizamentos em diversas unidades após esse período.

Entretanto, a intensidade e a persistência dessa redução não ocorreram de forma uniforme entre todos os órgãos julgadores.

Algumas unidades apresentaram redução acentuada e manutenção de um patamar inferior, enquanto outras registraram recuperação posterior do volume de ajuizamentos.

Dessa forma, os resultados permitem identificar uma **associação temporal** entre o período posterior à Resolução CNJ nº 547/2024 e a redução dos ajuizamentos, mas não permitem atribuir causalmente essa redução exclusivamente à norma.

> **Inserir gráfico da evolução das maiores unidades.**

---

# 7. Autoavaliação

A experiência com este primeiro MVP foi bastante desafiadora mas igualmente enriquecedora. Por ser um aluno com formação jurídica e sem formação prévia em tecnologia da informação, foi o primeiro contato que tive com alguns conceitos e ferramentas, como é o caso do Databricks, e também uma oportunidade inicial de aplicar na prática muitos conceitos que eu havia estudado apenas teoricamente mas que, até então, não tive como implementar em algum projeto.

Por conta desse contexto de formação, optei por um projeto com uma delimitação bastante fechada de forma a possibilitar que eu dedicasse mais atenção às etapas do pipeline de dados e não houvesse ampliação do objeto que deslocasse a atenção para a etapa de análise e visualização dos dados.

A etapa de ingestão foi especialmente didática porque nela pude perceber algumas dificuldades que podem surgir na prática e os cuidados que um profissional de dados precisa ter para garantir a consistência dos resultados. Foi o caso da identificação dos erros HTTP durante a ingestão, que estava ocasionando a persistência parcial dos dados, e também a mudança do formato da data no curso da série temporal analisada, o que fez com que o código inicial da requisição ao DATAJUD retornasse com resultado incompleto em um primeiro momento. Esses erros exigiram que eu revisitasse todo o código que havia sido inicialmente construído e incluísse as validações necessárias para garantir uma melhor qualidade do resultado.

Em relação a pontos de melhoria, seria possível aprofundar a análise para envolver os 'movimentos' oferecidos por meio da API do DATAJUD. A inclusão dos movimentos na análise, porém, aumentaria bastante a complexidade da atividade, em especial por conta do volume envolvido, uma vez que o campo é uma estrutura de lista, com códigos próprios, e com uma cardinalidade de N:1 em relação os processos (um processo possui várias movimentações). Logo, seria necessária uma estruturação mais robusta das camadas e, também, uma análise mais detida dos diversos códigos de movimentos associados.

Também visualizo possibilidade de enriquecimento do trabalho por meio de scrapping de dados públicos na página do Tribunal de Justiça para possibilitar uma melhor identificação de outras informações processuais (ex: partes envolvidas e valor da causa).

Por fim, consegui direcionar o projeto para uma análise muito útil para a minha área de atuação atual e obtive respostas para questões que oferecerem insights úteis para o meu trabalho. Fiquei feliz de poder aplicar os conhecimentos que obtive na sprint com este projeto.



