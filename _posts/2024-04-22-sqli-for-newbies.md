---
title: SQLi for newbies
date: 2024-04-22 19:11:00 +0300
categories: [anotacoes, web application security, sqli, exfiltration, out-of-band,blind-sql]
tags: [anotacoes, web application security, sqli, exfiltration, out-of-band,blind-sql]     # TAG names should always be lowercase
---


## O que é SQLi?

SQLi nada mais é do que uma técnica de invasão que manipula as consultas SQL feitas para um banco de dados afim de obter um resultado diferente daquele para o qual foi inicialmente feito, o objetivo é manipular a consulta SQL e obter resultados não previstos pelos devs, como acesso a colunas e tabelas não autorizadas, alteração de dados e em alguns casos rce(execução de código remoto).

## Como detectar vulnerabilidades SQLi?
- O caractere de aspas simples ' procura por erros ou outras anomalias.
- Condições booleanas como OR 1=1 e OR 1=2 e procuram diferenças nas respostas do aplicativo.
- Payloads projetadas para acionar atrasos quando executadas em uma consulta SQL e procurar diferenças no tempo necessário para responder.
- Payloads OAST projetadas para acionar uma interação de rede fora de banda quando executadas em uma consulta SQL e monitorar quaisquer interações resultantes.

## SQLi em diferentes partes da consulta:
A maioria das vulnerabilidades de injeção SQL ocorre na cláusula WHERE de uma consulta SELECT. Alguns outros locais comuns para injeção de SQL são:
- Em declarações UPDATE, nos valores atualizados ou na cláusula WHERE.
- Em declarações INSERT, nos valores inseridos.
- Em instruções SELECT, no nome da tabela ou coluna.
- Em declarações SELECT, na cláusula ORDER BY.

## 1 - Recuperando dados ocultos:
Imagine um aplicativo de compras que exibe produtos em diferentes categorias. Quando o usuário clica na categoria Presentes , o navegador solicita a URL:
```php
https://renadown.com/products?category=Gifts
```
Esta consulta SQL solicita que o banco de dados retorne:
- todos os detalhes ( *) da tabela **products** onde a **category** é **Gifts** e **released** é **1**.

A restrição **released=1** está sendo usada para ocultar os produtos que não são liberados. Então vamos assumir que para produtos não lançados, **released=0**.

O aplicativo não implementa nenhuma defesa contra ataques de SQLi. Então vamos construir o seguinte ataque:
```php
https://renadown.com/products?category=Gifts'--
```
Isso resulta na consulta SQL:
```sql
SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1
```
Em SQL, o -- significa que se inicia um comentário, então o restante da consulta SQL é interpretada como um comentário. Então todo o resto a frente como  **AND released = 1** não está incluído na consulta realizada. Com isso TODOS os produtos serão listados.

Agora imagine fazer com que o aplicativo exiba todos os produtos em qualquer categoria, incluindo categorias que eles não conhecem:
```php
https://renadown.com/products?category=Gifts'+OR+1=1--
```

Isso resulta na consulta SQL:
```sql
SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1
```
A consulta alterada busca retornar todos os itens em que a categoria seja "Gifts" ou onde a expressão 1 seja igual a 1. Como a expressão 1=1 é sempre verdadeira, a consulta retorna todos os itens.

## 2 - Subvertendo a lógica do aplicativo: 
Temos uma tela de login com usuário e senha. Se um usuário enviar o nome de usuário **zero** a senha **c00l**, o aplicativo verifica as credenciais executando a seguinte consulta SQL:
```sql
SELECT * FROM users WHERE username = 'zero' AND password = 'c00l'
```
Se a consulta retornar os detalhes de um usuário, o login foi bem-sucedido. Caso contrário, é rejeitado.

Nesse caso é possível fazer login apenas com o usuário usando a sequência de comentários SQL -- para remover a verificação de senha da cláusula WHERE. Exemplo **administrator'--**:
```sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = ''
```
Esta consulta retorna o usuário cujo usernamenome é administratore registra com êxito o invasor como esse usuário.

## 3 - Recuperando dados de outras tabelas de banco de dados

Usando a palavra-chave UNION, o invasor pode adicionar uma nova consulta SELECT à consulta original, obtendo assim resultados de outras tabelas. Por exemplo, na consulta abaixo:
```sql
SELECT name, description FROM products WHERE category = 'Gifts'
```
Ao injetar o código
```sql
 ' UNION SELECT username, password FROM users--
```
 um invasor pode obter nomes de usuário e senhas do banco de dados. Isso faz com que o aplicativo retorne todos os nomes de usuário e senhas junto com os nomes e descrições dos produtos.

## 4 - Vulnerabilidades de SQLi as cegas
As vulnerabilidades de injeção de SQL podem ser cegas, o que significa que o aplicativo não retorna resultados ou erros do banco de dados. Mesmo assim, é possível explorá-las para acessar dados não autorizados, embora isso seja mais complicado. Existem algumas técnicas para isso, como alterar a lógica da consulta para provocar uma resposta diferente do aplicativo, atrasar condicionalmente o processamento da consulta ou usar técnicas OAST para interações de rede fora de banda. Essas técnicas são mais complexas de executar.

Quando rola uma injeção de SQL e ela é do tipo "cega", significa que o app não entrega os resultados ou erros do banco de dados. Mas mesmo assim, dá pra se aproveitar disso pra pegar dados que não eram pra ser acessados, só que as técnicas são mais complexas e chatinhas de executar.

Tem umas manhas pra explorar esse tipo de brecha, dependendo de como tá o banco de dados:

Dá pra mexer na lógica da consulta pra ver se o app responde de um jeito diferente, dependendo de uma condição que você escolher. Isso pode ser enfiar uma condição nova numa lógica booleana ou fazer o app dar erro de propósito, como uma divisão por zero.
Também dá pra fazer o app atrasar o processamento da consulta de propósito. Aí dá pra saber se a condição tá certa ou não baseado no tempo que o app demora pra responder.
E tem uma técnica chamada OAST que usa interações de rede fora do app. Essa é bem poderosa e funciona quando as outras técnicas não dão certo. Muitas vezes, dá até pra pegar os dados diretamente pelo canal fora do app, usando uma pesquisa DNS pra um domínio que você controla.

## 5 - SQLi de segunda ordem
A injeção SQL de segunda ordem acontece quando um aplicativo recebe entrada do usuário, armazena esses dados para uso futuro e, mais tarde, ao recuperar esses dados armazenados para outra solicitação HTTP, os incorpora em uma consulta SQL de forma insegura. Isso ocorre mesmo quando os desenvolvedores cuidam da inserção inicial dos dados no banco de dados de forma segura. O problema surge quando esses dados são considerados seguros ao serem recuperados e processados posteriormente, o que não é verdadeiro, pois podem ser manipulados de forma insegura, causando a vulnerabilidade.

Vamos supor que um aplicativo de comércio eletrônico receba um comentário de um cliente e o armazene no banco de dados para exibição posterior. O desenvolvedor se preocupa em inserir esse comentário de forma segura no banco de dados, evitando assim uma injeção de SQL de primeira ordem.

No entanto, quando o aplicativo mais tarde recupera esse comentário armazenado para exibi-lo em uma página da web, ele incorpora o comentário na consulta SQL de forma insegura. Por exemplo, a consulta pode ser construída assim:
```sql
SELECT * FROM comentarios WHERE id = 123;
```
Se o comentário armazenado contiver dados maliciosos, como:
```sql
' OR 1=1 --
```
A consulta SQL final seria algo como:
```sql
SELECT * FROM comentarios WHERE id = 123' OR 1=1 --';
```
Isso faria com que a consulta retornasse todos os comentários, pois a condição 1=1 sempre é verdadeira.

## 6 - Examinando o banco de dados
A linguagem SQL é bem consistente em diferentes bancos de dados, então muitas técnicas para detectar e explorar vulnerabilidades de injeção SQL funcionam de maneira parecida em vários sistemas.

Mas cada banco de dados tem suas peculiaridades. Por exemplo, a forma de concatenar strings ou adicionar comentários pode variar entre eles. Pode ser no MySQL, PostgreSQL, Oracle, SQLite, MongoDB, etc.

Depois de achar uma vulnerabilidade de injeção de SQL, é válido pegar informações sobre o banco de dados. Isso pode ajudar a explorar a vulnerabilidade. Podemos descobrir a versão do banco de dados ou listar as tabelas existentes para ter uma ideia melhor do que está lidando.

## 7 - Injeção de SQL em diferentes contextos
As vulnerabilidades de injeção SQL podem ser exploradas usando qualquer entrada que o aplicativo processe como uma consulta SQL. Por exemplo, alguns sites recebem dados em JSON ou XML e os usam para consultar o banco de dados.

Esses formatos diferentes podem permitir que filtros de segurança sejam contornados, como WAFs, que bloqueiam ataques comuns de injeção SQL. Em implementações mais fracas, é possivel codificar ou escapar caracteres para contornar esses filtros. Por exemplo, você pode usar sequências de escape XML para codificar palavras-chave proibidas. Quando decodificado no servidor, isso se torna uma consulta SQL válida.

Suponhamos que um site use uma solicitação XML para verificar o estoque de um produto em um determinado local. A solicitação XML pode se parecer com isso:
```xml
<stockCheck>
    <productId>123</productId>
    <storeId>1</storeId>
</stockCheck>
```
Se o site não sanitizar corretamente a entrada, você pode usar uma injeção SQL no campo storeId para realizar uma consulta maliciosa. Por exemplo:
```xml
<stockCheck>
    <productId>123</productId>
    <storeId>1' OR '1'='1'--</storeId>
</stockCheck>
```
Neste exemplo, a parte 1' OR '1'='1'-- é inserida na consulta SQL como parte do XML. Isso pode fazer com que a consulta retorne resultados adicionais, ignorando a lógica original do aplicativo e potencialmente expondo dados sensíveis.

## 8 - Como evitar SQLi?
Não menos importante, como faremos para evitar SQLi? É recomendável usar consultas parametrizadas, também conhecidas como "declarações preparadas". Essas consultas ajudam a evitar a concatenação direta de strings na consulta, o que pode ser vulnerável a ataques.

Por exemplo, em vez de concatenar a entrada do usuário diretamente na consulta:
```javascript
String query = "SELECT * FROM products WHERE category = '"+ input + "'";
Statement statement = connection.createStatement();
ResultSet resultSet = statement.executeQuery(query);
```
Você pode usar uma consulta parametrizada para evitar a injeção de SQL:
```javascript
PreparedStatement statement = connection.prepareStatement("SELECT * FROM products WHERE category = ?");
statement.setString(1, input);
ResultSet resultSet = statement.executeQuery();
```
Consultas parametrizadas são eficazes para lidar com entradas não confiáveis em partes da consulta como a cláusula WHERE, os valores em instruções INSERT ou UPDATE. No entanto, elas não podem ser usadas para lidar com entradas não confiáveis em outras partes da consulta, como nomes de tabelas ou colunas, ou na cláusula ORDER BY. Para esses casos, é necessário adotar outras abordagens, como listas de permissões de valores de entrada permitidos ou lógica diferente para fornecer o comportamento necessário. Lembre-se sempre de não incluir dados variáveis em uma consulta parametrizada, para garantir a segurança contra injeção de SQL.

Por hoje é só, em breve quero trazer os laboratórios especificos de cada item que expliquei acima. Como trata-se de muito conteúdo vou segmentar por tipo como injeções UNION, depois Blind, e assim por diante... Obrigado pela visita e bons estudos.


- Referências: https://portswigger.net/web-security/sql-injection#what-is-sql-injection-sqli