## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    ```text
    Bitmap Heap Scan on t_books  (cost=12.00..68.00 rows=1 width=176) (actual time=1.420..1.440 rows=1 loops=1)
      Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ to_tsquery('english'::regconfig, 'expert'::text))
      Heap Blocks: exact=1
      ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=1.390..1.390 rows=1 loops=1)
            Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ to_tsquery('english'::regconfig, 'expert'::text))
    Planning Time: 0.220 ms
    Execution Time: 1.520 ms
    ```
    *Объясните результат:*
    Используется GIN индекс `t_books_fts_idx`: сначала строится bitmap из индекса (`Bitmap Index Scan`), затем читаются соответствующие строки таблицы (`Bitmap Heap Scan`). Для полнотекстового поиска GIN подходит лучше всего, поэтому даже при маленьком числе совпадений план быстро находит нужные записи; `Recheck Cond` — стандартная часть bitmap-прохода.
6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```text
     Index Scan using t_lookup_pk on t_lookup  (cost=0.29..8.31 rows=1 width=116) (actual time=0.020..0.021 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.090 ms
     Execution Time: 0.040 ms
     ```
     *Объясните результат:*
     Поиск по первичному ключу выполняется через `Index Scan` по B-tree индексу `t_lookup_pk`. Это точечный доступ: находит одну строку по `item_key` и читает минимум страниц.
14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```text
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.29..8.31 rows=1 width=116) (actual time=0.018..0.019 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.080 ms
     Execution Time: 0.035 ms
     ```
     *Объясните результат:*
     План аналогичен шагу 13: также `Index Scan` по первичному ключу, но на кластеризованной таблице. Кластеризация не меняет факт использования PK-индекса; возможный выигрыш — чуть лучшая локальность данных на диске.
15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```text
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.29..8.31 rows=1 width=116) (actual time=0.030..0.030 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.070 ms
     Execution Time: 0.040 ms
     ```
     *Объясните результат:*
     После создания индекса по `item_value` запрос выполняется через `Index Scan` по `t_lookup_value_idx`. В наших данных `item_value` имеет вид `Value_...`, поэтому `'T_BOOKS'` не находится (rows=0), и индекс позволяет быстро убедиться в отсутствии результата без полного сканирования.
18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```text
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.29..8.31 rows=1 width=116) (actual time=0.028..0.028 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.070 ms
     Execution Time: 0.038 ms
     ```
     *Объясните результат:*
     Аналогично шагу 17: используется индекс `t_lookup_clustered_value_idx`. Кластеризация таблицы по первичному ключу почти не влияет на поиск по `item_value`, потому что физический порядок строк соответствует `item_key`, а не `item_value`.
19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     Поиск по `item_key` (шаги 13–14) в обеих таблицах выполняется через `Index Scan` по первичному ключу и имеет схожую стоимость. Кластеризация может дать небольшой выигрыш за счёт более последовательного чтения heap-страниц, но для точечного поиска эффект обычно минимальный.

     Поиск по `item_value` (шаги 17–18) в обеих таблицах использует отдельные B-tree индексы по `item_value`, поэтому планы и время тоже близки. Так как кластеризация сделана по PK (`item_key`), она почти не помогает запросам по `item_value`.
