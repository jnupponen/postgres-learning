# Testing Postgresql JSON functions

## Start Postgresql

```
docker run -e POSTGRES_PASSWORD=secret -e POSTGRES_USER=postgres -e POSTGRES_DB=mydb -it --rm -p 5432:5432 --name postgres postgres:10.5-alpine
```

## Connect to Postgresql with psql

```
docker run -it --rm --link postgres:postgres postgres:10.5-alpine psql -h postgres -U postgres -d mydb
```

Type in password `secret` when prompted.

## Create table with data

```sql
CREATE TABLE purchases (id integer PRIMARY KEY, document json);


INSERT INTO purchases (id, document)
    VALUES (1, '{ "customer": "Jill Doe", "items": [{"product": "Beer","qty": 2}, {"product": "Soda","qty": 3}]}'),
           (2, '{ "customer": "Jane Doe", "items": [{"product": "Wine","qty": 1}, {"product": "Beer","qty": 2}]}'),
           (3, '{ "customer": "Joan Doe", "items": [{"product": "Soda","qty": 4}, {"product": "Soda","qty": 2}]}');

SELECT * from purchases;
```

which will output

```
 id |                                             document                                             
----+--------------------------------------------------------------------------------------------------
  1 | { "customer": "Jill Doe", "items": [{"product": "Beer","qty": 2}, {"product": "Soda","qty": 3}]}
  2 | { "customer": "Jane Doe", "items": [{"product": "Wine","qty": 1}, {"product": "Beer","qty": 2}]}
  3 | { "customer": "Joan Doe", "items": [{"product": "Soda","qty": 4}, {"product": "Soda","qty": 2}]}
(3 rows)
```

## Create view with most popular products

```sql
CREATE OR REPLACE VIEW popular_products as(
    WITH x AS
      (SELECT json_array_elements(document -> 'items') AS items, document ->>'customer' AS customer
       FROM purchases)
        SELECT sum(cast(items ->>'qty' AS INTEGER)) AS qty, items ->> 'product' AS product, array_agg(customer) AS customers
        FROM x
        GROUP BY product
        ORDER BY qty DESC
);

SELECT *
FROM popular_products;
```

which will output

```
 qty | product |             customers              
-----+---------+------------------------------------
   9 | Soda    | {"Jill Doe","Joan Doe","Joan Doe"}
   4 | Beer    | {"Jill Doe","Jane Doe"}
   1 | Wine    | {"Jane Doe"}
(3 rows)
```

## Get output as JSON

```sql
WITH data(qty, product, customers) as
  (SELECT qty, product, customers
   FROM popular_products)
SELECT row_to_json(data) as popular_products_as_json
FROM data;
```

which will output

```
                         popular_products_as_json                          
---------------------------------------------------------------------------
 {"qty":9,"product":"Soda","customers":["Jill Doe","Joan Doe","Joan Doe"]}
 {"qty":4,"product":"Beer","customers":["Jill Doe","Jane Doe"]}
 {"qty":1,"product":"Wine","customers":["Jane Doe"]}
(3 rows)
```
