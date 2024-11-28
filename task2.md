# Задание 2: Специальные случаи использования индексов

# Партиционирование и специальные случаи использования индексов

1. Удалите прошлый инстанс PostgreSQL - `docker-compose down` в папке `src` и запустите новый: `docker-compose up -d`.

2. Создайте партиционированную таблицу и заполните её данными:

    ```sql
    -- Создание партиционированной таблицы
    CREATE TABLE t_books_part (
        book_id     INTEGER      NOT NULL,
        title       VARCHAR(100) NOT NULL,
        category    VARCHAR(30),
        author      VARCHAR(100) NOT NULL,
        is_active   BOOLEAN      NOT NULL
    ) PARTITION BY RANGE (book_id);

    -- Создание партиций
    CREATE TABLE t_books_part_1 PARTITION OF t_books_part
        FOR VALUES FROM (MINVALUE) TO (50000);

    CREATE TABLE t_books_part_2 PARTITION OF t_books_part
        FOR VALUES FROM (50000) TO (100000);

    CREATE TABLE t_books_part_3 PARTITION OF t_books_part
        FOR VALUES FROM (100000) TO (MAXVALUE);

    -- Копирование данных из t_books
    INSERT INTO t_books_part 
    SELECT * FROM t_books;
    ```

3. Обновите статистику таблиц:
   ```sql
   ANALYZE t_books;
   ANALYZE t_books_part;
   ```
   
   *Результат:*
   ANALYZE

    Запрос завершён успешно, время выполнения: 1 secs 620 msec.

4. Выполните запрос для поиска книги с id = 18:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part WHERE book_id = 18;
   ```
   
   *План выполнения:*
   "Seq Scan on t_books_part_1 t_books_part  (cost=0.00..1032.99 rows=1 width=32) (actual time=0.018..5.630 rows=1 loops=1)"
    "  Filter: (book_id = 18)"
    "  Rows Removed by Filter: 49998"
    "Planning Time: 0.296 ms"
    "Execution Time: 5.832 ms"
   
   *Объясните результат:*
   планировщик выбрал нужную партицию по ключу партицирования (ИД). поскольку в партицированной таблице не установлен первичный ключ на ИД, поиск совпадения ведётся с помощью последовательного чтения партиции

5. Выполните поиск по названию книги:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   "Append  (cost=0.00..3101.01 rows=3 width=33) (actual time=3.934..12.094 rows=1 loops=1)"
   "  ->  Seq Scan on t_books_part_1  (cost=0.00..1032.99 rows=1 width=32) (actual time=3.934..3.935 rows=1 loops=1)"
   "        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "        Rows Removed by Filter: 49998"
   "  ->  Seq Scan on t_books_part_2  (cost=0.00..1034.00 rows=1 width=33) (actual time=3.760..3.761 rows=0 loops=1)"
   "        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "        Rows Removed by Filter: 50000"
   "  ->  Seq Scan on t_books_part_3  (cost=0.00..1034.01 rows=1 width=34) (actual time=4.393..4.393 rows=0 loops=1)"
   "        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "        Rows Removed by Filter: 50001"
   "Planning Time: 0.363 ms"
   "Execution Time: 12.267 ms"
   
   *Объясните результат:*
   в каждой партиции происходило последовательно чтение строк и поиск соответсвия (тк не установлен индекс на название книги), после чего произошла агрегация результатов и вывод

6. Создайте партиционированный индекс:
   ```sql
   CREATE INDEX ON t_books_part(title);
   ```
   
   *Результат:*
   CREATE INDEX

   Запрос завершён успешно, время выполнения: 601 msec.

7. Повторите запрос из шага 5:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   "Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.042..0.102 rows=1 loops=1)"
"  ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.041..0.042 rows=1 loops=1)"
"        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
"  ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.028..0.028 rows=0 loops=1)"
"        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
"  ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.030..0.030 rows=0 loops=1)"
"        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
"Planning Time: 0.698 ms"
"Execution Time: 0.137 ms"
   
   *Объясните результат:*
   так как теперь на название книги установлен индекс, запрос выполнялся через индекс скан, так же отдельно в каждой партиции и объединение результата в конце

8. Удалите созданный индекс:
   ```sql
  DROP INDEX t_books_part_title_idx;
   ```
   
   *Результат:*
   DROP INDEX

   Запрос завершён успешно, время выполнения: 141 msec.

9. Создайте индекс для каждой партиции:
   ```sql
   CREATE INDEX ON t_books_part_1(title);
   CREATE INDEX ON t_books_part_2(title);
   CREATE INDEX ON t_books_part_3(title);
   ```
   
   *Результат:*
   CREATE INDEX

   Запрос завершён успешно, время выполнения: 582 msec.

10. Повторите запрос из шага 5:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part 
    WHERE title = 'Expert PostgreSQL Architecture';
    ```
    
    *План выполнения:*
    "Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.028..0.059 rows=1 loops=1)"
   "  ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.027..0.028 rows=1 loops=1)"
   "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "  ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.014..0.014 rows=0 loops=1)"
   "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "  ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.015..0.015 rows=0 loops=1)"
   "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
   "Planning Time: 0.617 ms"
   "Execution Time: 0.092 ms"
    
    *Объясните результат:*
    Как и в случае с общим индексом для всей таблицы, индекс скан происходяит по партициям и после результат объединяется. однако при создании индекса для отдельных партиций время выполнения сокращается

11. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_part_1_title_idx;
    DROP INDEX t_books_part_2_title_idx;
    DROP INDEX t_books_part_3_title_idx;
    ```
    
    *Результат:*
    DROP INDEX

   Запрос завершён успешно, время выполнения: 146 msec.

12. Создайте обычный индекс по book_id:
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id);
    ```
    
    *Результат:*
    CREATE INDEX

   Запрос завершён успешно, время выполнения: 204 msec.
13. Выполните поиск по book_id:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part WHERE book_id = 11011;
    ```
    
    *План выполнения:*
    "Index Scan using t_books_part_1_book_id_idx on t_books_part_1 t_books_part  (cost=0.29..8.31 rows=1 width=32) (actual time=0.020..0.021 rows=1 loops=1)"
    "  Index Cond: (book_id = 11011)"
    "Planning Time: 0.463 ms"
    "Execution Time: 0.041 ms"
    
    *Объясните результат:*
    с помощью ключа партиции была выбрана нужная партиция и индекс скан производился только в ней, остальные партиции не рассматривались. благодаря индексу запрос выполнился очень быстро

14. Создайте индекс по полю is_active:
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active);
    ```
    
    *Результат:*
    CREATE INDEX

    Запрос завершён успешно, время выполнения: 259 msec.

15. Выполните поиск активных книг с отключенным последовательным сканированием:
    ```sql
    SET enable_seqscan = off;
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE is_active = true;
    SET enable_seqscan = on;
    ```
    
    *План выполнения:*
    "Bitmap Heap Scan on t_books  (cost=843.25..2819.45 rows=75220 width=33) (actual time=2.324..13.119 rows=74967 loops=1)"
    "  Recheck Cond: is_active"
    "  Heap Blocks: exact=1224"
    "  ->  Bitmap Index Scan on t_books_active_idx  (cost=0.00..824.44 rows=75220 width=0) (actual time=2.182..2.183 rows=74967 loops=1)"
    "        Index Cond: (is_active = true)"
    "Planning Time: 0.086 ms"
    "Execution Time: 16.017 ms"
    
    *Объясните результат:*
    тк последовательно чтение недоступно, используется битмап, однако время выполнения все равно большое тк селективность запроса низкая (вывело половину строк)

16. Создайте составной индекс:
    ```sql
    CREATE INDEX t_books_author_title_index ON t_books(author, title);
    ```
    
    *Результат:*
    CREATE INDEX

    Запрос завершён успешно, время выполнения: 1 secs 57 msec.
17. Найдите максимальное название для каждого автора:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, MAX(title) 
    FROM t_books 
    GROUP BY author;
    ```
    
    *План выполнения:*
    "HashAggregate  (cost=3474.00..3484.01 rows=1001 width=42) (actual time=112.592..112.836 rows=1003 loops=1)"
    "  Group Key: author"
    "  Batches: 1  Memory Usage: 193kB"
    "  ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=21) (actual time=0.038..12.777 rows=150000 loops=1)"
    "Planning Time: 0.957 ms"
    "Execution Time: 113.614 ms"
    
    *Объясните результат:*
    низкая селективность запроса (сравниваем все данные) поэтому используется последовательное чтение. после - агрегация

18. Выберите первых 10 авторов:
    ```sql
    EXPLAIN ANALYZE
    SELECT DISTINCT author 
    FROM t_books 
    ORDER BY author 
    LIMIT 10;
    ```
    
    *План выполнения:*
    "Limit  (cost=0.42..56.57 rows=10 width=10) (actual time=0.252..1.053 rows=10 loops=1)"
    "  ->  Result  (cost=0.42..5621.42 rows=1001 width=10) (actual time=0.250..1.049 rows=10 loops=1)"
    "        ->  Unique  (cost=0.42..5621.42 rows=1001 width=10) (actual time=0.249..1.046 rows=10 loops=1)"
    "              ->  Index Only Scan using t_books_author_title_index on t_books  (cost=0.42..5246.42 rows=150000 width=10) (actual time=0.247..0.935 rows=1388 loops=1)"
    "                    Heap Fetches: 1"
    "Planning Time: 0.333 ms"
    "Execution Time: 12.365 ms"
    
    *Объясните результат:*
    есть покрывающий индекс, поэтому сканируется только он, после чего отбираются уникальные значения и вывыводятся первые 10 строк. сортировку не проводят тк в индексе уже записи упорядочены по автору

19. Выполните поиск и сортировку:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE author LIKE 'T%'
    ORDER BY author, title;
    ```
    
    *План выполнения:*
    "Sort  (cost=3099.27..3099.30 rows=14 width=21) (actual time=19.784..19.785 rows=1 loops=1)"
    "  Sort Key: author, title"
    "  Sort Method: quicksort  Memory: 25kB"
    "  ->  Seq Scan on t_books  (cost=0.00..3099.00 rows=14 width=21) (actual time=19.439..19.440 rows=1 loops=1)"
    "        Filter: ((author)::text ~~ 'T%'::text)"
    "        Rows Removed by Filter: 149999"
    "Planning Time: 6.895 ms"
    "Execution Time: 20.179 ms"
    
    *Объясните результат:*
    с регулярным выражением поиск по индексу неэффективен, поэтому используется последовательное чтение. долго выполняется запрос

20. Добавьте новую книгу:
    ```sql
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true);
    COMMIT;
    ```
    
    *Результат:*
    WARNING:  there is no transaction in progress
    COMMIT

    Запрос завершён успешно, время выполнения: 129 msec.

21. Создайте индекс по категории:
    ```sql
    CREATE INDEX t_books_cat_idx ON t_books(category);
    ```
    
    *Результат:*
    CREATE INDEX

    Запрос завершён успешно, время выполнения: 394 msec.

22. Найдите книги без категории:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    "Index Scan using t_books_cat_idx on t_books  (cost=0.29..8.14 rows=1 width=21) (actual time=0.070..0.072 rows=1 loops=1)"
    "  Index Cond: (category IS NULL)"
    "Planning Time: 1.461 ms"
    "Execution Time: 0.095 ms"
    
    *Объясните результат:*
    тк в постргесе нулевые значения включаются в индекс, происходит индекс скан по категории, условие - category is null

23. Создайте частичные индексы:
    ```sql
    DROP INDEX t_books_cat_idx;
    CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL;
    ```
    
    *Результат:*
    CREATE INDEX

    Запрос завершён успешно, время выполнения: 184 msec.

24. Повторите запрос из шага 22:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ["Index Scan using t_books_cat_null_idx on t_books  (cost=0.12..7.97 rows=1 width=21) (actual time=0.021..0.022 rows=1 loops=1)"
    "Planning Time: 0.319 ms"
    "Execution Time: 0.044 ms"]
    
    *Объясните результат:*
    [вывелись все строки по которым был создан индекс, выполнилось быстрее, чем при индексировании всей таблицы]

25. Создайте частичный уникальный индекс:
    ```sql
    CREATE UNIQUE INDEX t_books_selective_unique_idx 
    ON t_books(title) 
    WHERE category = 'Science';
    
    -- Протестируйте его
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150002, 'Unique Science Book', 'Author 1', 'Science', true);
    
    -- Попробуйте вставить дубликат
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150003, 'Unique Science Book', 'Author 2', 'Science', true);
    
    -- Но можно вставить такое же название для другой категории
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150004, 'Unique Science Book', 'Author 3', 'History', true);
    ```
    
    *Результат:*
    [CREATE INDEX

    Запрос завершён успешно, время выполнения: 223 msec.

    INSERT 0 1

    Запрос завершён успешно, время выполнения: 101 msec.

    ERROR:  Key (title)=(Unique Science Book) already exists.duplicate key value violates unique constraint "t_books_selective_unique_idx" 

    ERROR:  duplicate key value violates unique constraint "t_books_selective_unique_idx"
    SQL-состояние: 23505
    Подробности: Key (title)=(Unique Science Book) already exists.
    
    INSERT 0 1

    Запрос завершён успешно, время выполнения: 97 msec.]
    
    *Объясните результат:*
    [уникальный индекс запрещает вставку двух записей, в которых совпадают пары название-категория, однако если одна из записей поменяется, нарушения целостности не произойдет]