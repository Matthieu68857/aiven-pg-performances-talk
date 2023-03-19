# Postgres performances: Tips & tricks from recent versions (Talk to ADEO / Decathlon / Norauto 22th of March 2023)

## TOAST

Query the default toast compression on your server
```
show default_toast_compression;
```

Find your toast table
```
select relname from pg_class where oid = (select reltoastrelid from pg_class where relname = 'products_big');
```

Query the size details of your table:
```
SELECT table_name, row_estimate,
  pg_size_pretty(total_bytes) AS total,
  pg_size_pretty(table_bytes) AS table,
  pg_size_pretty(toast_bytes) AS toast
FROM (
    SELECT *,
      total_bytes - index_bytes - coalesce(toast_bytes, 0) AS table_bytes
    FROM (
        SELECT c.oid,
          nspname AS table_schema,
          relname AS table_name,
          c.reltuples AS row_estimate,
          pg_total_relation_size(c.oid) AS total_bytes,
          pg_indexes_size(c.oid) AS index_bytes,
          pg_total_relation_size(reltoastrelid) AS toast_bytes
        FROM pg_class c
          LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
        WHERE relkind = 'r'
          AND relname like 'products_%'
      ) a
  ) a
ORDER BY table_name DESC;
```

How to create a table with a different `toast_tuple_target`
```
CREATE TABLE products_opti (id INT, product_sheet JSONB) WITH (toast_tuple_target=1024);
```

## Bloated tables

Disable autovacuum for test purposes
```
alter table bloated_table set (autovacuum_enabled = off);
```

Use pgstattuple extension
```
select * from pgstattuple('bloated_table');
select * from pgstattuple('non_bloated_table');
```

**pg_repack** extension walkthrough: [https://github.com/Matthieu68857/aiven-pg_repack-demo](https://github.com/Matthieu68857/aiven-pg_repack-demo)


## Postgres v15 new features

Create a publication with filters
```
CREATE PUBLICATION pub_filtered_replication 
FOR TABLE filtered_replication WHERE filter = 'yes'
WITH (publish='insert,update,delete');
```

Merge query example
```
MERGE INTO target_products t USING source_product s ON t.id = s.id
WHEN NOT MATCHED THEN
INSERT VALUES (s.id, s.name, s.price)
WHEN MATCHED THEN
UPDATE SET price = s.price;
```

More complex
```
MERGE INTO target_products t
USING source_product s
ON t.id = s.id
WHEN NOT MATCHED AND s.id IS NOT NULL THEN
INSERT VALUES (s.id, s.name, s.price)
WHEN MATCHED AND t.name IN ('hammer', 'nails') THEN
UPDATE SET name = s.name, price = s.Price
WHEN MATCHED THEN
DELETE;
```
