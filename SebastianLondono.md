TABLA CONVERSION_1

conectarse a la bd y realziar import
Usted deberá crear un pipeline completo, ingestando datos de una base de datos y archivos de access log de Apache. La idea es relacionar la información de navegación de la página web con las transacciones de ventas.

Ingestar datos de las transacciones de ventas:
IP: 34.205.65.241
Puerto: 3306
Usuario: bootcamp
Contraseña: bootcamp
Base de datos: ecommerce
Tabla: product_transaction
Motor: MySQL

sqoop import \
 --connect jdbc:mysql://34.205.65.241:3306/ecommerce_cloudera \
 --username bootcamp \
 --password bootcamp \
 --table product_transaction \
 --target-dir /user/sebas/desafio \
 --null-non-string '\\N' \
 --compression-codec=snappy \
 --hive-import \
 --hive-table product_transaction

#Validacion de extraccion de la data
#Consulta en HIVE
SELECT * FROM product_transaction; 

2. Logs de acceso de Apache Web Server: http://34.205.65.241/access.log

wget http://34.205.65.241/access.log

#Copio el log al HDFS
hdfs dfs -put /home/ec2-user/access.log /user/sebas/desafio

#Primero intenté crear una tabla de una columna con todo el string para luego divirlo pero no funcionó
create table x (z string)
location '/user/sebas/cdr/'

#Despues de algunos fallos de sintaxis me funcionó
CREATE TABLE logs_products2 (ip STRING
                            , time_local STRING
                            , method STRING
                            , uri STRING 
                            , protocol STRING
                            , status STRING
                            , bytes_sent STRING
                            , referer STRING
                            , useragent STRING
                            )
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES('input.regex' = '^(\\S+) \\S+ \\S+ \\[([^\\[]+)\\] "(\\w+) (\\S+) (\\S+)" (\\d+) (\\d+) "([^"]+)" "([^"]+)".*')

STORED AS TEXTFILE LOCATION '/user/sebas/cdr/';

4. Calcular las ventas por producto: 

create table ventas_por_producto as select pt.product_id
        ,Sum(pt.product_cantity) as Cantidad_total 
From product_transaction pt 
group by product_id

5. calcular las visualizaciones por producto
create table clicks_por_producto as Select  split(uri, '=')[1] as product_id
        , count(*)         as clicks 
from logs_products2
GROUP BY split(uri, '=')[1]

6. Se unen las tablas
Create table Conversion as SELECT v.product_id, v.cantidad_total, c.clicks FROM ventas_por_producto v
JOIN clicks_por_producto c On c.product_id = v.product_id

7. se opera el resultado y se almacena en tabla conversion1
create table conversion1 as SELECT product_id, cantidad_total/clicks as conversion from conversion


8.#Exporte a sqoop
 sqoop export \
--connect jdbc:mysql://34.205.65.241:3306/ecommerce_cloudera \
--username bootcamp \
--password bootcamp \
--table conversion_1 \
--hcatalog-table conversion_1

Notas
ALTER TABLE conversion1 RENAME TO conversion_1; //para cambiar el nombre de la tabla al esperado en la BD
- Tengo este error al hacer el export:
ERROR tool.ExportTool: Encountered IOException running export job:
java.io.IOException: Caught Exception checking database column sku in  hcatalog table.

