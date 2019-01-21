### Supprimer les doublons : 

column1, 2, x représentent les colonnes que l'on veut dédoubler ...

```sql
DELETE FROM tablename
WHERE id IN (SELECT id
              FROM (SELECT id,
                             ROW_NUMBER() OVER (partition BY column1, column2, column3 ORDER BY id) AS rnum
                     FROM tablename) t
              WHERE t.rnum > 1);
```
