### Autoren
- Tobias Bossert
- Jilin Elavathingal

### 1 Dokumentation

### 2 Einleitung

### 3 Speicher- und Zugriffsstrukturen
dbarc01 Index-Organized Tables mit einem zusätzlichen Sekundärindex

### 4 Aufgabe
#### 4.1 Studieren Sie die entsprechenden Abschnitte im Concepts Manual Kapitel 3.
#### 4.2 Konsultieren Sie zusätzliche Quellen (z.B. Wikipedia).
#### 4.3 Gehen Sie dabei folgenden Fragen nach:
##### 4.3.1 Wie „funktionieren“ die Strukturen bzw. was sind Index Scans und welche Arten
gibt es?
##### 4.3.2 Wo werden die Strukturen mit Vorteil eingesetzt bzw. werden die Index Scans
verwendet?
##### 4.3.3 Wann sind die Strukturen ungeeignet bzw. werden die Index Scans nicht eingesetzt?
##### 4.3.4 Wie werden die Strukturen physisch dargestellt?
#### 4.4 Beschreiben Sie ein typisches Beispiel und realisieren Sie es in Ihrer Datenbank.
#### 4.5 Zeigen Sie, mit welchen Abfragen die Strukturen bzw. die Index Scans durch den
Optimizer tatsächlich verwendet werden und wann nicht.

### 5 Hinweise
Ausführungspläne können Sie anzeigen und damit ermittlen, mit welchen Strukturen und
Scans Anfragen durchgeführt werden.
#### 5.1 Statistiken erheben z.B. für Tabelle emp im Schema scott mit
```sql
BEGIN
DBMS_STATS.GATHER_TABLE_STATS('scott','emp')
END;
```
#### 5.2 Ausführungsplans ermitteln für ein SELECT-Statement mit folgendem Muster
```sql
EXPLAIN PLAN FOR
SELECT *
FROM emp JOIN dept USING (deptno);
```
#### 5.3 Anzeigen des Ausführungsplans
```sql
SELECT plan_table_output
FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'typical'));
```