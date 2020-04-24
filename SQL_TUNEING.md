# 2. Vorbereitung
## 2.1 Manual Links
- Abbildung 4-2: Optimizer Components
- [Kapitel 8](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/optimizer-access-paths.html): Optimizer Access Paths, Introduction to Access Paths
- [Kapitel 6](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/generating-and-displaying-execution-plans.html): Explaining and Displaying Execution Plans
- [Kapitel 19](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/influencing-the-optimizer.html): Influencing the Optimizer, Influencing the Optimizer with Hints
## 2.2 TPC-H Schema
![](img_tuning/schema.png)
## 2.3 Einrichten der Datenbasis
```sql
DROP TABLE regions;
DROP TABLE nations;
DROP TABLE parts;
DROP TABLE customers;
DROP TABLE suppliers;
DROP TABLE orders;
DROP TABLE partsupps;
DROP TABLE lineitems;

CREATE TABLE regions
AS SELECT *
FROM dbarc00.regions;

CREATE TABLE nations
AS SELECT *
FROM dbarc00.nations;

CREATE TABLE parts
AS SELECT *
FROM dbarc00.parts;

CREATE TABLE customers
AS SELECT *
FROM dbarc00.customers;

CREATE TABLE suppliers
AS SELECT *
FROM dbarc00.suppliers;

CREATE TABLE orders
AS SELECT *
FROM dbarc00.orders;

CREATE TABLE partsupps
AS SELECT *
FROM dbarc00.partsupps;

CREATE TABLE lineitems
AS SELECT *
FROM dbarc00.lineitems;
```
## 2.4 Statistiken erheben
Anzahl Zeilen (a) und Anzahl Blocks:
```sql
SELECT table_name, blocks, num_rows FROM all_tables WHERE owner='DBARC01';
```
![](img_tuning/3ee44549.png)
Anzahl extents (d) und Bytes (b)
```sql
SELECT segment_name, extents, bytes FROM DBA_SEGMENTS WHERE owner='DBARC01';
```
![](img_tuning/54620291.png)

# 3. Ausführungsplan
# 4. Versuche ohne Index
## 4.1 Projektionen
```sql
EXPLAIN PLAN FOR
    SELECT * FROM ORDERS;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));

-- output
/*
Plan hash value: 1275100350
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |  1500K|   158M|  6599   (1)| 00:00:01 |
|   1 |  TABLE ACCESS FULL| ORDERS |  1500K|   158M|  6599   (1)| 00:00:01 |
----------------------------------------------------------------------------
*/
```
```sql
EXPLAIN PLAN FOR
    SELECT o_clerk FROM ORDERS;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
/*
Plan hash value: 1275100350
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |  1500K|    22M|  6597   (1)| 00:00:01 |
|   1 |  TABLE ACCESS FULL| ORDERS |  1500K|    22M|  6597   (1)| 00:00:01 |
----------------------------------------------------------------------------
*/
```
Hier ist interessant anzumerken, dass der selbe Ausführungsplan verwendet wird wie wenn alle Spalten
ausgewählt werden. Der einzige Unterschied ist der Memory-Footprint. Ferner sind auch die Kosten minimal tiefer.

```sql
EXPLAIN PLAN FOR
    SELECT DISTINCT o_clerk FROM ORDERS;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
/*
Plan hash value: 3394267636
 
-----------------------------------------------------------------------------
| Id  | Operation          | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |        |  1000 | 16000 |  6640   (1)| 00:00:01 |
|   1 |  HASH UNIQUE       |        |  1000 | 16000 |  6640   (1)| 00:00:01 |
|   2 |   TABLE ACCESS FULL| ORDERS |  1500K|    22M|  6597   (1)| 00:00:01 |
-----------------------------------------------------------------------------
*/
```
Hier starten wir wieder mit einem full access scan über die Tabelle orders. Durch `HASH UNIQUE` reduziert
sich die Anzahl rows auf 1000 was bedeutet es gibt genau 1000 unique `o_clerk`. Auch hier trägt der 
full access scan den grössten Teil der _Cost_ bei.

## 4.2 Selektion
### 4.2.1 Exact point query
```sql
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_orderkey = 44480;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
/*
Plan hash value: 1275100350
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |     1 |   111 |  6594   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS |     1 |   111 |  6594   (1)| 00:00:01 |
----------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("O_ORDERKEY"=44480)
*/
```
Wie bei den Projektionen auch muss hier jeweils ein full access scan gemacht werden um anschliessend die Filter anwenden zu können. Das ist bei jedem der 
Beispiele in diesem Kapitel der Fall.

### 4.2.2 Partial point query (OR)
```sql
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_custkey = 97303 OR o_clerk = 'Clerk#000000860';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
/*
Plan hash value: 1275100350
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |  1515 |   164K|  6611   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS |  1515 |   164K|  6611   (1)| 00:00:01 |
----------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("O_CLERK"='Clerk#000000860' OR "O_CUSTKEY"=97303)
*/
```
Als wesentlichen Unterschied zum _exact point query_ kann hier die Grösse in Bytes genannt werden. Was auch Sinn ergibt da durch die `OR`
operation die Schnittmenge erhöht wird. Auch die Kosten sind etwas höher durch den komplexeren Filter erklärbar ist.

### 4.2.3 Partial point query (AND)
```sql
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_custkey = 97303 AND o_clerk = 'Clerk#000000860';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
/*
Plan hash value: 1275100350
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |     1 |   111 |  6599   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS |     1 |   111 |  6599   (1)| 00:00:01 |
----------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("O_CUSTKEY"=97303 AND "O_CLERK"='Clerk#000000860')
*/
```
Hier haben wir eigentlich wieder ein _exact point query_ (Sichtbar an den Rows welche jeweils 1 sind). Ist aber wieder teurer als in [4.3.1](#4.3.1-exact-point-query)
da der Filter komplexer ist.
### 4.2.4 Partial point query (weighed)
```sql
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_custkey*3 = 194606 AND o_clerk = 'Clerk#000000286';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
/*
Plan hash value: 1275100350
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |    15 |  1665 |  6601   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS |    15 |  1665 |  6601   (1)| 00:00:01 |
----------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("O_CUSTKEY"*2=194606 AND "O_CLERK"='Clerk#000000286')
*/
```
--> Frage was hat das gewichten hier für einen Effekt?

### 4.2.5 Range query
```sql
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_orderkey BETWEEN 111111 AND 222222;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
--output
/*
Plan hash value: 1275100350
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        | 27780 |  3011K|  6594   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS | 27780 |  3011K|  6594   (1)| 00:00:01 |
----------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("O_ORDERKEY"<=222222 AND "O_ORDERKEY">=111111)
*/
```
Die Range hat sehr wohl einen Effekt - allerdings nur auf den Speicher:
```sql
-- select larger range
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_orderkey BETWEEN 0 AND 222222;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
/*
Plan hash value: 1275100350
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        | 55556 |  6022K|  6594   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS | 55556 |  6022K|  6594   (1)| 00:00:01 |
----------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("O_ORDERKEY"<=222222 AND "O_ORDERKEY">=0)
*/
-- select smaller range
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_orderkey BETWEEN 222220 AND 222222;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
/*
Plan hash value: 1275100350
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |     3 |   333 |  6594   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS |     3 |   333 |  6594   (1)| 00:00:01 |
----------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("O_ORDERKEY"<=222222 AND "O_ORDERKEY">=222220)
*/
```

### 4.2.6 Partial range query
```sql
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_orderkey BETWEEN 44444 AND 55555
    AND o_clerk BETWEEN 'Clerk#000000130' AND 'Clerk#000000139';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
--output
/*
Plan hash value: 1275100350
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |     6 |   666 |  6599   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS |     6 |   666 |  6599   (1)| 00:00:01 |
----------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("O_ORDERKEY"<=55555 AND "O_CLERK"<='Clerk#000000139' AND 
              "O_ORDERKEY">=44444 AND "O_CLERK">='Clerk#000000130')
*/
```

## 4.3 Join
### 4.3.1 Natural join
### 4.3.2 Mit zusätzlicher Selektion

# 5. Versuche mit Index
Indizies erstellen:
```sql
CREATE INDEX o_orderkey_ix ON orders(o_orderkey);
CREATE INDEX o_clerk_ix ON orders(o_clerk);
CREATE INDEX o_custkey_ix ON orders(o_custkey);
```
Wie gross sind die Indizies in Bytes?
```sql
select segment_name,bytes from user_segments where segment_name IN('o_orderkey_ix','o_clerk_ix','o_custkey_ix', 'ORDERS');
```

## 5.1 Projektion
```sql
EXPLAIN PLAN FOR
    SELECT DISTINCT o_clerk FROM ORDERS;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
```
## 5.2 Selektion
### 5.2.1 Exact point query
```sql
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_orderkey = 44480;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
```
Mit erzwungenem _full table scan_:
```sql
EXPLAIN PLAN FOR
    SELECT /*+ FULL(orders) */ *FROM orders WHERE o_orderkey = 44480;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
```
### 5.2.2 Partial point query (OR)
```sql
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_custkey = 97303 OR o_clerk = 'Clerk#000000860';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
```
### 5.2.3 Partial point query (AND)
```sql
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_custkey = 97303 AND o_clerk = 'Clerk#000000860';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
```

### 5.2.4 Partial point query (weighed)
```sql
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_custkey*3 = 194606 AND o_clerk = 'Clerk#000000286';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
```

### 5.2.5 Range query
```sql
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_orderkey BETWEEN 111111 AND 222222;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
--output
```
```sql
-- select larger range
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_orderkey BETWEEN 0 AND 222222;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output

-- select smaller range
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_orderkey BETWEEN 222220 AND 222222;
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
-- output
```
### 5.2.6 Partial range query
```sql
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE o_orderkey BETWEEN 44444 AND 55555
    AND o_clerk BETWEEN 'Clerk#000000130' AND 'Clerk#000000139';
SELECT plan_table_output FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
--output
```


## 5.3 Join
### 5.3.1 Natural join
### 5.3.2 Mit zusätzlicher Selektion

# 6. Quiz
# 7. Deep Left Join?
# 8. Eigene SQL Anfragen
