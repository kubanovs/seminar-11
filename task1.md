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
   ```
   Bitmap Heap Scan on t_books  (cost=11.00..15.00 rows=1 width=33) (actual time=0.012..0.013 rows=0 loops=1)
   Recheck Cond: (category IS NULL)
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.011..0.011 rows=0 loops=1)
         Index Cond: (category IS NULL)
   Planning Time: 0.073 ms
   Execution Time: 0.029 ms
   ```
   
   *Объясните результат:*
   - используется индекс BitMap
   - запрос стоит от 11 до 15
   - время, которое планируется на запрос - 0.073 ms
   - фактическое время выполнения запроса - 0.029 ms

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
   ```
   Bitmap Heap Scan on t_books  (cost=13.00..16.00 rows=1 width=33) (actual time=14.893..14.895 rows=0 loops=1)
   Recheck Cond: ((category)::text = 'INDEX'::text)
   Rows Removed by Index Recheck: 150000
   Filter: ((author)::text = 'SYSTEM'::text)
   Heap Blocks: lossy=1225
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.044..0.045 rows=12250 loops=1)
         Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.141 ms
   Execution Time: 12.856 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   - используется индекс BitMap
   - Запрос стоит от 13 до 16
   - время, которое планируется на запрос - 0.141 ms
   - фактическое время выполнения запроса - 12.856 ms
   - сложность запроса увеличилась, поэтому затраченное время выше

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   Sort  (cost=3100.11..3100.12 rows=5 width=7) (actual time=29.564..29.567 rows=6 loops=1)
   Sort Key: category
   Sort Method: quicksort  Memory: 25kB
   ->  HashAggregate  (cost=3100.00..3100.05 rows=5 width=7) (actual time=29.541..29.544 rows=6 loops=1)
         Group Key: category
         Batches: 1  Memory Usage: 24kB
         ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.005..7.662 rows=150000 loops=1)
   Planning Time: 0.085 ms
   Execution Time: 30.573 ms
   ```
   
   *Объясните результат:*
   - для сортировки используется алгоритм быстрой сортировки
   - Запрос стоит от 3100.11 до 3100.12
   - время, которое планируется на запрос - 0.085 ms
   - фактическое время выполнения запроса - 30.573 ms

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3100.03..3100.04 rows=1 width=8) (actual time=13.027..13.028 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=13.023..13.023 rows=0 loops=1)
         Filter: ((author)::text ~~ 'S%'::text)
         Rows Removed by Filter: 150000
   Planning Time: 0.108 ms
   Execution Time: 13.057 ms
   ```
   
   *Объясните результат:*
   - в запросе выполняется агрегация
   - идет последовательное чтение строк таблицы
   - Запрос стоит от 3100.03 до 3100.04
   - время, которое планируется на запрос - 0.108 ms
   - фактическое время выполнения запроса - 13.057 ms

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
   ```
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=42.559..42.561 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=42.551..42.553 rows=1 loops=1)
         Filter: (lower((title)::text) ~~ 'o%'::text)
         Rows Removed by Filter: 149999
   Planning Time: 0.067 ms
   Execution Time: 42.583 ms
   ```
   
   *Объясните результат:*
   - в запросе выполняется агрегация
   - идет последовательное чтение строк таблицы
   - Запрос стоит от 3476.88 до 3476.89
   - время, которое планируется на запрос - 0.067 ms
   - фактическое время выполнения запроса - 42.583 ms

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
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=0.987..0.988 rows=0 loops=1)
   Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Rows Removed by Index Recheck: 8844
   Heap Blocks: lossy=73
   ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.021..0.022 rows=730 loops=1)
         Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.211 ms
   Execution Time: 1.018 ms
   ```
   
   *Объясните результат:*
   - используется индекс BitMap
   - Запрос стоит от 12 до 16.02
   - время, которое планируется на запрос - 0.211 ms
   - фактическое время выполнения запроса - 1.018 ms