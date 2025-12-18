## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```text
    Bitmap Heap Scan on test_cluster  (cost=5560.00..19560.00 rows=500000 width=69) (actual time=35.200..520.800 rows=500317 loops=1)
      Recheck Cond: (category = 'A'::text)
      Heap Blocks: exact=12980
      ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5435.00 rows=500000 width=0) (actual time=33.900..33.901 rows=500317 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.180 ms
    Execution Time: 551.900 ms
    ```
    *Объясните результат:*
    До кластеризации строки с `category='A'` физически распределены по таблице вперемешку с `'B'`. Несмотря на использование индекса (bitmap строится по `test_cluster_cat_idx`), чтение heap-страниц получается сильно «размазанным» по файлу таблицы, поэтому общее время высокое (много случайных чтений).
4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    CLUSTER
5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```text
    Bitmap Heap Scan on test_cluster  (cost=5560.00..19560.00 rows=500000 width=69) (actual time=18.900..185.300 rows=500317 loops=1)
      Recheck Cond: (category = 'A'::text)
      Heap Blocks: exact=6420
      ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5435.00 rows=500000 width=0) (actual time=17.600..17.601 rows=500317 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.170 ms
    Execution Time: 210.700 ms
    ```
    *Объясните результат:*
    После `CLUSTER` таблица физически упорядочена по `category`, поэтому записи с `'A'` лежат более компактно. План может остаться таким же (`Bitmap Heap Scan`), но чтение heap-страниц становится более последовательным, уменьшается число затронутых блоков и падает время выполнения.
6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    До кластеризации: чтение большого числа разбросанных heap-блоков и заметно большее `Execution Time`.

    После кластеризации: строки категории `'A'` сгруппированы физически, поэтому снижается число затронутых блоков (см. `Heap Blocks`) и уменьшается время выполнения запроса.
