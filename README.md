# Tratamento dos Dados - Projeto Relative Risk

Repositório com os detalhes da limpeza de dados do Projeto do Risco Relativo

    
→ As etapas de limpeza de dados desse projeto foram feitas no Big Query. 

→ Nome do projeto no Big Query: **dados_proj3**

## Biblioteca de Dados

  Para esse projeto, tivemos a disposição quatro diferentes tabelas com dados sobre os empréstimos concedidos a um grupo de clientes no banco:
**user_info:** dados do usuário/cliente

**loans_outstanding:** dados do tipo empréstimo

**loans_detail:** informações do comportamento de pagamento desses empréstimos

**default:** informação dos clientes já identificados como inadimplentes.

 Descrição das variáveis que compõem as tabelas deste conjunto de dados:

| Arquivo | Variável | Descrição |
| --- | --- | --- |
| user_info | user id | Número de identificação do cliente (único para cada cliente) |
|  | age | Idade do cliente |
|  | sex | Gênero do cliente |
|  | last month salary | Último salário mensal que o cliente informou ao banco |
|  | number dependents | Número de dependentes |
| loans_outstanding | loan id | Número de identificação do empréstimo (único para cada empréstimo) |
|  | user id | Número de identificação do cliente |
|  | loan type | Tipo de empréstimo (real state = imóveis, others= outros) |
| loans_detail | user id | Número de identificação do cliente |
|  | more 90 days overdue | Número de vezes que o cliente apresentou atraso superior a 90 dias |
|  | using lines not secured personal assets | Quanto o cliente está utilizando em relação ao seu limite de crédito, em linhas que não são garantidas por bens pessoais, como imóveis e automóveis |
|  | number times delayed payment loan 30 59 days | Número de vezes que o cliente atrasou o pagamento de um empréstimo (entre 30 e 59 dias) |
|  | debt ratio | Relação entre dívidas e ativos do cliente. Taxa de endividamento = Dívidas / Patrimonio |
|  | number times delayed payment loan 60 89 days | Número de vezes que o cliente atrasou o pagamento de um empréstimo (entre 60 e 89 dias) |
| default | user id | Número de identificação do cliente |
|  | default flag | Classificação dos clientes inadimplentes (1 para clientes já registrados alguma vez como inadimplentes, 0 para clientes sem histórico de inadimplência) |

## Tratamento dos Dados
### Valores nulos

Primeiramente, foi identificado os valores nulos no banco de dados. 

Com esse propósito, foi feita a seguinte consulta no *BigQuery*:

```sql
SELECT
  COUNT(*)
FROM `tabela X
WHERE variável is null
```

Ex.:

```sql
-- Consulta dos valores nulos tabela default
SELECT 
  count(*)
FROM `expanded-flame-423403-n5.dados_proj3.default`
WHERE default_flag IS NULL

```

A tabela abaixo mostra quais variáveis possuem valores nulos e a quantidade.

| Arquivo | Variável | Qtd de nulos |
| --- | --- | --- |
| user_info | user id | 0 |
|  | age | 0 |
|  | sex | 0 |
|  | last month salary | 7199 |
|  | number dependents | 943 |
| loans_outstanding | loan id | 0 |
|  | user id | 0 |
|  | loan type | 0 |
| loans_detail | user id | 0 |
|  | more 90 days overdue | 0 |
|  | using lines not secured personal assets | 0 |
|  | number times delayed payment loan 30 59 days | 0 |
|  | debt ratio | 0 |
|  | number times delayed payment loan 60 89 days | 0 |
| default | user id | 0 |
|  | default flag | 0 |
|  |  |  |

Foi calculado o valor da média e mediana para as variáveis **last_month_salary e number_dependents** para decidir qual métrica será utilizada para substituir os valores nulos dessas variáveis**:**

```sql
SELECT
  AVG(last_month_salary) OVER() as media_last_mont,
  PERCENTILE_CONT(last_month_salary, 0.5) OVER() AS mediana_last_month,
  AVG(number_dependents) OVER() as media_dependents,
  PERCENTILE_CONT(number_dependents, 0.5) OVER() AS mediana_number_dependents
FROM `dados_proj3.user_info`
limit 1 
```

Foi encontrado os seguintes resultados:

| Variável | média | mediana |
| --- | --- | --- |
| last month salary | 6675.0520468039313 | 5400.0 |
| number dependents | 0.7580796987762789 | 0.0 |

Como os valores da média e mediana estão distantes e a média é sensível a outliers, foi escolhida a mediana para fazer substituição dos valores nulos. O seguinte código foi utilizado para substituir os valores nulos:

```sql
SELECT *,
  COALESCE(last_month_salary, mediana_last_month) AS last_month_salary_mediana,
  COALESCE(number_dependents, mediana_number_dependents) AS number_dependents_mediana
  FROM
  `dados_proj3.user_info`,
  (
  SELECT
  PERCENTILE_CONT(last_month_salary, 0.5) OVER() AS mediana_last_month,
  PERCENTILE_CONT(number_dependents, 0.5) OVER() AS mediana_number_dependents
  FROM `dados_proj3.user_info`
  limit 1
  )

```
### Valores duplicados
As variáveis no qual foi investigado valores duplicados , foram as referentes ao id, pois são variáveis, que nessas tabelas precisam ter valores únicos. A consulta para detecção dos valores duplicados foi a seguinte:

```sql
-- Detecção duplicados - tabela default

SELECT  
  COUNT(*) AS CONT
FROM `expanded-flame-423403-n5.dados_proj3.default`
GROUP BY user_id
HAVING CONT >1

-- Detecção duplicados - tabela loans details
SELECT  
  COUNT(*) AS CONT
FROM `dados_proj3.loans_details`
GROUP BY user_id
HAVING CONT >1

-- Detecção duplicados - tabela loans outstanding
SELECT  
  COUNT(*) AS CONT
FROM `dados_proj3.loans_outstanding`
GROUP BY user_id
HAVING CONT >1

-- Detecção duplicados - tabela user_info
SELECT  
  user_id,
  COUNT(*) AS CONT
FROM `dados_proj3.user_info`
GROUP BY user_id
HAVING CONT >1
```

Quantidade de duplicados encontrados:

| Arquivo | Variável | Descrição |
| --- | --- | --- |
| user_info | user id | Não foi identificado valores duplicados |
| loans_outstanding | loan id | Não foi identificado valores duplicados |
|  | user id | Vários valores duplicados, pois essa tabela, apresenta um id de impréstimo (loan_id) diferente para cada empréstimo que uma mesma pessoa faz. |
| loans_detail | user id | Não foi identificado valores duplicados |
| default | user id | Não foi identificado valores duplicados |

Como pode-se ver acima, somente a tabela **loans_outstanding**, possui valores duplicados, pois nessa tabela contém as informações de cada empréstimo que o banco fez, sendo que cada cliente pode ter feito mais de um empréstimo.

Essas informações foram condensadas para que a tabela apresentasse valores distintos de *user_id* com a informação de quantos empréstimos cada cliente fez. Veja com mais detalhes na seção [**Criando novas variáveis**](#criando-novas-variáveis)



### Variáveis fora do escopo

### Multicolinearidade das variáves

 Primeiro, investiguei a multicolinearidade das variáveis. Como ainda estou para investigar quais variáveis são independentes e quais não são, verifiquei a correlação de todas as variáveis da tabela loan details. Script no Big Query:

```sql
-- Correlação variável "more_90_days_overdue"
SELECT  
  CORR(more_90_days_overdue, using_lines_not_secured_personal_assets) AS corrl_90_usinglin,
  CORR(more_90_days_overdue, number_times_delayed_payment_loan_30_59_days) AS corrl_90_number30,
  CORR(more_90_days_overdue, debt_ratio) AS corrl_90_debt,
  CORR(more_90_days_overdue, number_times_delayed_payment_loan_60_89_days) AS corrl_90_number60,
-- Correlação variável "using_lines_not_secured_personal_assets"

  CORR(using_lines_not_secured_personal_assets, number_times_delayed_payment_loan_30_59_days) AS corrl_usinglin_number30,
  CORR(using_lines_not_secured_personal_assets, debt_ratio) AS corrl_usinglin_debt,
  CORR(using_lines_not_secured_personal_assets, number_times_delayed_payment_loan_60_89_days) AS corrl_usinglin_number60,

-- Correlação variável "number_times_delayed_payment_loan_30_59_days"
  CORR(number_times_delayed_payment_loan_30_59_days, debt_ratio) AS corrl_number30_debt,
  CORR(number_times_delayed_payment_loan_30_59_days, number_times_delayed_payment_loan_60_89_days) AS corrl_number30_number60,

-- Correlação variável "debt_ratio"
  CORR(debt_ratio, number_times_delayed_payment_loan_60_89_days) AS corrl_debt_number60,
FROM `expanded-flame-423403-n5.dados_proj3.loans_details` 
```

Tabela com os valores:

|  | more 90 days overdue | using lines not secured personal assets | number times delayed payment loan 30 59 days | debt ratio | number times delayed payment loan 60 89 days |
| --- | --- | --- | --- | --- | --- |
| more 90 days overdue |  | -0.0013609406054606965 | 0.98291680661459857 | 0.0082076486652037147 | 0.99217552634075257 |
| using lines not secured personal assets |  |  | -0.0012496300975538808 | 0.015012138743289461 | -0.000772448431582398 |
| number times delayed payment loan 30 59 days |  |  |  | -0.0052226124100982953 | 0.98655364549866986 |
| debt ratio |  |  |  |  | -0.0074023325612315215 |
| number times delayed payment loan 60 89 days |  |  |  |  |  |

Variáveis com alta correlação:

| Variável 1 | Variável 2 | R-pearson |
| --- | --- | --- |
| more 90 days overdue | number times delayed payment loan 30 59 days | 0.9829 |
| more 90 days overdue | number times delayed payment loan 60 89 days | 0.9921 |
| number times delayed payment loan 30 59 days | number times delayed payment loan 60 89 days | 0.9865 |

A multicolinearidade se refere à alta correlação entre variáveis independentes em um modelo de regressão linear. Quando isso ocorre, pode ser necessário remover uma ou mais dessas variáveis para tornar o modelo mais claro e confiável, pois a multicolinearidade dificulta a compreensão da importância relativa de cada variável na predição da variável dependente.

Assim, vou investigar quais variáveis devo retirar antes de modelar meus dados.

**more_90_days_overdue:** Número de vezes que o cliente apresentou atraso superior a 90 dias.

**number_times_delayed_payment_loan_30_59_days:** Número de vezes que o cliente atrasou o pagamento de um empréstimo (entre 30 e 59 dias).

**number_times_delayed_payment_loan_60_89_days:** Número de vezes que o cliente atrasou o pagamento de um empréstimo (entre 60 e 89 dias).

Todas as variáveis acima se referem ao atraso de pagamento do empréstimo. Elas apresentam alta correlação entre si, indicando multicolinearidade, e são redundantes em um modelo de regressão linear. Portanto, as variáveis que serão descartadas devido à multicolinearidade são **number_times_delayed_payment_loan_30_59_days** e **number_times_delayed_payment_loan_60_89_days**.

A escolha pela variável **more_90_days_overdue** se deve ao fato de ela representar o maior período 

de atraso no pagamento do empréstimo, sugerindo que ela pode fornecer uma melhor indicação do risco de inadimplência. Pois atrasos mais longos geralmente têm um maior impacto no risco de crédito.

### Valores Discrepantes
**Variáveis STRING**

Para identificação de valores discrepantes das variáveis do tipo STRING, foi utilizado o seguinte código:

```sql
-- Dados discrepantes - Identificação
SELECT 
  *
FROM `dados_proj3.user_info`
WHERE sex LIKE'%[^A-Za-z0-9, ]%'
```

As variáveis que foram testadas são:

| Arquivo | Variável | Valores Discrepantes |
| --- | --- | --- |
|  | user id | Não foi encontrado |
| user_info | sex | Não foi encontrado |
| loans_outstanding | loan type | Não foi encontrado |

**Variáveis Numéricas:**

Para identificação de valores discrepantes das variáveis do tipo INTEGER, foi utilizado o seguinte código:

```sql
SELECT 
  *
FROM `dados_proj3.user_info`
WHERE REGEXP_CONTAINS(CAST(user_id AS STRING), r'[^0-9]')
```

REGEXP_CONTAINS (string, pattern): Retorna verdadeiro se a string contiver um padrão que corresponda à expressão regular fornecida. A expressão regular

r'[^0-9]': especifica qualquer caractere que não seja um dígito.

CAST: converte a variável user_id de INT para STRING para poder usar o comando REGEXP_CONTAINS, pois este só pode ser usado se for uma variável do tipo STRING.

WHERE: trás as linhas que o regexp_contains identificou alguma letra ou caractere especial

Para variável using_lines_not_secured_personal_assets, foi usada  ocódigo a seguir, pois ela contém casas decimais que as vezes é representada por “.” ou por notação científica “-e”:

```sql
SELECT 
  *
FROM `dados_proj3.loans_details`
WHERE REGEXP_CONTAINS(CAST(using_lines_not_secured_personal_assets AS STRING), r'[^0-9a-z. -]')
```

| Arquivo | Valor discrepante |
| --- | --- |
| user_info | Não foi identificado valores discrepantes |
| loans_outstanding | Não foi identificado valores discrepantes |
|  | Não foi identificado valores discrepantes |
| loans_detail | Não foi identificado valores discrepantes |
| default | Não foi identificado valores discrepantes |

Por não ter sido encontrado nenhum valor discrepante, nenhuma limpeza desse tipo foi feita.

A variável loan type tem valores com nome maiúsculo e outras com nome minúsculo, assim foi utilizado um código para colocar todas em maiúsculo:

```sql
SELECT 
  loan_id,
  user_id,
  UPPER(loan_type)
FROM `expanded-flame-423403-n5.dados_proj3.loans_outstanding`
```

### Valores outliers

Box plots foram gerados para investigar a distribuição dos valores das variáveis e seus respectivos outliers, utilizando o Google Colab.

O script utilizado com esse propósito foi o seguinte:

```python
import plotly.express as px

fig = px.box(df, x="age")
fig.show()

fig2 = px.box(df, x="last_month")
fig2.show()

fig3 = px.box(df, x="number_dependents")
fig3.show()

fig4 = px.box(df, x="total_loan")
fig4.show()

fig5 = px.box(df, x="total_90_days_overdue")
fig5.show()

fig6 = px.box(df, x="total_lines")
fig6.show()

fig7 = px.box(df, x="debt_ratio")
fig7.show()
```

Os gráficos obtidos foram os seguintes:

![newplot (1)](https://github.com/annesantos1990/relative_risk_project/assets/166059836/9665d07b-83bb-4fcc-ba80-a17ea8080fcf)
![newplot (2)](https://github.com/annesantos1990/relative_risk_project/assets/166059836/1881b8a7-e0c2-4683-9392-f6308bcb7736)
![newplot (3)](https://github.com/annesantos1990/relative_risk_project/assets/166059836/58734183-9e24-4456-9394-8af47f8630b0)
![newplot (4)](https://github.com/annesantos1990/relative_risk_project/assets/166059836/d6970b32-a62e-4e0e-9ca8-258b5018d48a)
![newplot (5)](https://github.com/annesantos1990/relative_risk_project/assets/166059836/3f947b0f-95d3-4cfa-b3f0-8f0e96ea87a8)
![newplot (6)](https://github.com/annesantos1990/relative_risk_project/assets/166059836/3ca1f74c-b985-425e-9e8d-a417e82c1a65)
![newplot (7)](https://github.com/annesantos1990/relative_risk_project/assets/166059836/d2c5707f-f731-49fa-90c1-abfc203e3fd2)

Nesse projeto, foram mantidos os outliers em todas as análises.

### **Novas variáveis**

Foi criada a variável *total_loan* na tabela loans_outstanding:

```sql
  SELECT
    user_id,
    CASE
      WHEN UPPER(COALESCE(loan_type, '')) LIKE '%REAL ESTATE%' THEN 'REAL STATE'
      WHEN UPPER(COALESCE(loan_type, '')) LIKE 'REAL STATE' THEN 'REAL STATE'
      WHEN UPPER(COALESCE(loan_type, '')) LIKE '%OTHER%' THEN 'OTHER'
      ELSE 'UNKNOWN'
    END AS loan_type,
    COUNT(loan_id) AS quantidade_emprestimo
  FROM `expanded-flame-423403-n5.dados_proj3.loans_outstanding`
  GROUP BY 
    user_id, loan_type
```

Nessa consulta, a variável **total_loan** contém a informação da quantidade de empréstimo feito por cada cliente.

Além disso, foi utilizado o comando CASE para uniformizar os valores da variável loan_type para que todas ficassem com valores maiúsculos e sem o ‘s’ no final de ‘OTHERS’.

### Código Completo e união das tabelas

Para união das tabelas, foi utilizado uma Common Table Express (WITH), no qual foi adicionada em cada uma delas as limpezas e tratamentos feitos em cada uma das tabelas abaixo:

**user_info:**

Tabelas com vários valores nulos das variáveis **last_month_salary e number_dependents**, no qual foram trocados pelas respectivas medianas dessas variáveis.

**loans_outstanding:**

Nessa tabela foi agrupados os **user_ids**  e criada uma nova variável que contabiliza o número de empréstimos feitos por cada cliente.

Foi uniformizado os valores da variável **loan_type**.

E foi criada uma outra tabela para que ao unir as quatro tabelas, os user_ids não ficassem duplicados:

```sql
aggregated_loans AS (
  SELECT
    user_id,
    STRING_AGG(loan_type) AS loan_types,
    SUM(quantidade_emprestimo) AS total_emprestimos
  FROM loans_outstanding_clean
  GROUP BY user_id
),
```

Nessa tabela auxiliar, foi utilizado :

STRING_AGG(loan_type): teve o propósito de unificar os valores do loan_type para o mesmo user_id. Por isso, no final, foi realizado um GROUP BY user_id.

Assim, os clientes que contraíram dois tipos diferentes de empréstimos terão esses empréstimos agrupados na mesma célula. Por exemplo: REAL STATE, OTHER.

SUM(quantidade_emprestimo): Isso somou a quantidade total de empréstimos para cada user_id obtida na tabela auxiliar *loans_outstanding_clean*.

**loans_details:**

Nessa tabela, foi retirada da análise as variáveis **number_times_delayed_payment_loan_30_59_days** e **number_times_delayed_payment_loan_60_89_days**, por serem variáveis redundantes. Mas não houve a necessidade de criar uma Common Table (CTE), pois essas variáveis não foram incluídas na tabela unificada.

**default:**

Nenhum tratamento foi realizado nessa tabela.

Código:

```sql
WITH user_info_clean AS (
  SELECT *,
    COALESCE(last_month_salary, mediana_last_month) AS last_month_salary_mediana,
    COALESCE(number_dependents, mediana_number_dependents) AS number_dependents_mediana
  FROM
    `dados_proj3.user_info`,
    (
      SELECT
        PERCENTILE_CONT(last_month_salary, 0.5) OVER() AS mediana_last_month,
        PERCENTILE_CONT(number_dependents, 0.5) OVER() AS mediana_number_dependents
      FROM `dados_proj3.user_info`
      LIMIT 1
    )
),

loans_outstanding_clean AS (
  SELECT
    user_id,
    CASE
      WHEN UPPER(COALESCE(loan_type, '')) LIKE '%REAL ESTATE%' THEN 'REAL STATE'
      WHEN UPPER(COALESCE(loan_type, '')) LIKE 'REAL STATE' THEN 'REAL STATE'
      WHEN UPPER(COALESCE(loan_type, '')) LIKE '%OTHER%' THEN 'OTHER'
      ELSE 'UNKNOWN'
    END AS loan_type,
    COUNT(loan_id) AS quantidade_emprestimo
  FROM `expanded-flame-423403-n5.dados_proj3.loans_outstanding`
  GROUP BY 
    user_id, loan_type
),

aggregated_loans AS (
  SELECT
    user_id,
    STRING_AGG(loan_type) AS loan_types,
    SUM(quantidade_emprestimo) AS total_emprestimos
  FROM loans_outstanding_clean
  GROUP BY user_id
),

aggregated_loans_details AS (
  SELECT
    user_id,
    SUM(more_90_days_overdue) AS total_90_days_overdue,
    SUM(using_lines_not_secured_personal_assets) AS total_using_lines_not_secured_personal_assets,
    AVG(debt_ratio) AS avg_debt_ratio
  FROM `dados_proj3.loans_details`
  GROUP BY user_id
)

SELECT 
  default_table.user_id,
  age,
  sex,
  last_month_salary_mediana,
  number_dependents_mediana,
  loan_types,
  total_emprestimos,
  default_flag,
  total_90_days_overdue,
  total_using_lines_not_secured_personal_assets,
  avg_debt_ratio
FROM `dados_proj3.default` AS default_table
JOIN user_info_clean ON default_table.user_id = user_info_clean.user_id
JOIN aggregated_loans ON default_table.user_id = aggregated_loans.user_id
JOIN aggregated_loans_details ON default_table.user_id = aggregated_loans_details.user_id
```

Ao unificar as tabelas, a tabela resultante apresentou 35.575 linhas, ou seja, 425 linhas a menos. Isso ocorreu porque a tabela loan_outstanding, que contém as informações sobre quantos empréstimos cada cliente fez e o tipo de empréstimo, não inclui todos os clientes (user_id) das outras tabelas.



</details>
<details> <summary> <h2> Análise Exploratória dos Dados</h2>  - Clique em ▶ para ver os detalhes </summary> 

### Medidas de Tendência Central


Os detalhes da análise exploratória, poderá ser visto na Seção **Resultados e Discussão**.


