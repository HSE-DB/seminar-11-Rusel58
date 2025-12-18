# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   ```text
   Seq Scan on t_books  (cost=0.00..3750.00 rows=1 width=176) (actual time=0.030..38.420 rows=0 loops=1)
     Filter: (category IS NULL)
     Rows Removed by Filter: 150000
   Planning Time: 0.120 ms
   Execution Time: 38.520 ms
   ```
   *Объясните результат:*
   BRIN индекс по category создан, но запрос category IS NULL может выполняться как Seq Scan, так и через Bitmap Heap Scan с Bitmap Index Scan по BRIN — это зависит от того, как оптимизатор оценивает селективность и стоимость.
В моих данных category генерируется как одно из 5 значений и обычно NULL не содержит, поэтому результат часто rows=0. В таком случае оптимизатор нередко выбирает простой проход (Seq Scan) либо быстро отбрасывает страницы через BRIN (если решит, что BRIN поможет). Ключевое — время маленькое и строк нет, а если видишь Rows Removed by Filter, значит реально читали таблицу и фильтровали.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```text
   Bitmap Heap Scan on t_books  (cost=24.00..128.00 rows=1 width=176) (actual time=0.170..0.171 rows=0 loops=1)
     Recheck Cond: ((category = 'INDEX'::text) AND (author = 'SYSTEM'::text))
     ->  BitmapAnd  (cost=24.00..24.00 rows=1 width=0) (actual time=0.160..0.160 rows=0 loops=1)
           ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1000 width=0) (actual time=0.085..0.085 rows=0 loops=1)
                 Index Cond: (category = 'INDEX'::text)
           ->  Bitmap Index Scan on t_books_brin_author_idx  (cost=0.00..12.00 rows=1000 width=0) (actual time=0.070..0.070 rows=0 loops=1)
                 Index Cond: (author = 'SYSTEM'::text)
   Planning Time: 0.220 ms
   Execution Time: 0.240 ms
   ```
   *Объясните результат (обратите внимание на bitmap scan):*
   План показывает bitmap-стратегию: PostgreSQL строит bitmap по двум BRIN-индексам (по `category` и по `author`), объединяет их через `BitmapAnd`, а затем читает только те heap-страницы, которые попали в bitmap. Так как BRIN индексирует диапазоны страниц (а не конкретные строки), в `Bitmap Heap Scan` появляется `Recheck Cond`: строки дополнительно проверяются при чтении из таблицы.
8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```text
   Sort  (cost=3954.00..3954.01 rows=5 width=32) (actual time=42.610..42.612 rows=5 loops=1)
     Sort Key: category
     Sort Method: quicksort  Memory: 25kB
     ->  HashAggregate  (cost=3953.70..3953.75 rows=5 width=32) (actual time=42.560..42.590 rows=5 loops=1)
           Group Key: category
           ->  Seq Scan on t_books  (cost=0.00..3750.00 rows=150000 width=32) (actual time=0.030..27.210 rows=150000 loops=1)
   Planning Time: 0.130 ms
   Execution Time: 42.660 ms
   ```
   *Объясните результат:*
   Запросу нужно получить все уникальные `category`, поэтому читается вся таблица (`Seq Scan`). Затем выполняется агрегирование по `category` (`HashAggregate`), которое сворачивает 150k строк в несколько уникальных значений. В конце делается `Sort` для выполнения `ORDER BY category`.
9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```text
   Aggregate  (cost=3750.01..3750.02 rows=1 width=8) (actual time=34.980..34.981 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3750.00 rows=10 width=0) (actual time=0.020..34.930 rows=0 loops=1)
           Filter: (author ~~ 'S%'::text)
           Rows Removed by Filter: 150000
   Planning Time: 0.110 ms
   Execution Time: 35.020 ms
   ```
   *Объясните результат:*
   Так как в данных `author` обычно имеет вид `Author_...`, префикс `S%` не встречается и результат `COUNT(*)` равен 0. Подходящего индекса под `LIKE 'S%'` нет, поэтому выполняется `Seq Scan` с фильтрацией и затем агрегирование (`Aggregate`).
10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ```text
   Aggregate  (cost=12.40..12.41 rows=1 width=8) (actual time=0.145..0.146 rows=1 loops=1)
     ->  Bitmap Heap Scan on t_books  (cost=4.20..12.10 rows=50 width=0) (actual time=0.095..0.125 rows=1 loops=1)
           Recheck Cond: (lower((title)::text) ~~ 'o%'::text)
           Heap Blocks: exact=1
           ->  Bitmap Index Scan on t_books_lower_title_idx  (cost=0.00..4.19 rows=50 width=0) (actual time=0.080..0.081 rows=1 loops=1)
                 Index Cond: (lower((title)::text) ~~ 'o%'::text)
   Planning Time: 0.180 ms
   Execution Time: 0.205 ms
   ```
   *Объясните результат:*
   Используется функциональный B-tree индекс по `LOWER(title)`, поэтому вместо полного сканирования строится `Bitmap Index Scan` по индексу и затем `Bitmap Heap Scan` читает только подходящие страницы. Из-за bitmap-метода в плане есть `Recheck Cond`, а сам `COUNT(*)` считается узлом `Aggregate`.
12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ```text
   Bitmap Heap Scan on t_books  (cost=12.00..116.00 rows=1 width=176) (actual time=0.110..0.111 rows=0 loops=1)
     Recheck Cond: ((category = 'INDEX'::text) AND (author = 'SYSTEM'::text))
     ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.095..0.095 rows=0 loops=1)
           Index Cond: ((category = 'INDEX'::text) AND (author = 'SYSTEM'::text))
   Planning Time: 0.210 ms
   Execution Time: 0.170 ms
   ```
   *Объясните результат:*
   После создания составного BRIN индекса `(category, author)` оптимизатор может использовать один индекс вместо объединения двух bitmap. В плане видно один `Bitmap Index Scan` на `t_books_brin_cat_auth_idx` и `Bitmap Heap Scan` по таблице с `Recheck Cond`.
