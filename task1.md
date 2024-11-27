# Задание 1. B-tree индексы в PostgreSQL

1. Запустите БД через docker compose в ./src/docker-compose.yml:

2. Выполните запрос для поиска книги с названием 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   "Seq Scan on t_books  (cost=0.00..3099.00 rows=1 width=33) (actual time=15.919..15.921 rows=1 loops=1)"
   "  Filter: ((title)::text = 'Oracle Core'::text)"
   "  Rows Removed by Filter: 149999"
   "Planning Time: 0.070 ms"

   "Execution Time: 15.943 ms"
   
   *Объясните результат:*
   поиск ведётся не по первичному ключу, поэтому происходит полное чтение таблицы, тк не заданы никакие индексы. в результате запросы выведена одна строка, время выполнения - 15.943 ms
3. Создайте B-tree индексы:
   ```sql
   CREATE INDEX t_books_title_idx ON t_books(title);
   CREATE INDEX t_books_active_idx ON t_books(is_active);
   ```
   
   *Результат:*
   CREATE INDEX
   Запрос завершён успешно, время выполнения: 1 secs 440 msec.

4. Проверьте информацию о созданных индексах:
   ```sql
   SELECT schemaname, tablename, indexname, indexdef
   FROM pg_catalog.pg_indexes
   WHERE tablename = 't_books';
   ```
   
   *Результат:*
   "public"	"t_books"	"t_books_id_pk"	"CREATE UNIQUE INDEX t_books_id_pk ON public.t_books USING btree (book_id)"
   "public"	"t_books"	"t_books_title_idx"	"CREATE INDEX t_books_title_idx ON public.t_books USING btree (title)"
   "public"	"t_books"	"t_books_active_idx"	"CREATE INDEX t_books_active_idx ON public.t_books USING btree (is_active)"
   
   *Объясните результат:*
   в результате команды в таблице созданы индексы вида Б-дерево для соответсвующих столбцов

5. Обновите статистику таблицы:
   ```sql
   ANALYZE t_books;
   ```
   
   *Результат:*
   ANALYZE
   Запрос завершён успешно, время выполнения: 546 msec.

6. Выполните запрос для поиска книги 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   "Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.103..0.105 rows=1 loops=1)"
   "  Index Cond: ((title)::text = 'Oracle Core'::text)"
   "Planning Time: 1.220 ms"
   "Execution Time: 0.138 ms"
   
   *Объясните результат:*
   планируется производить поиск с использованием созданного индекса, условие индекса - (title)::text = 'Oracle Core'::text 
   в результате применения индекса время выполнения запроса сократилось в сто раз до 0.138 ms вместо 15,9 

7. Выполните запрос для поиска книги по book_id и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE book_id = 18;
   ```
   
   *План выполнения:*
   "Index Scan using t_books_id_pk on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=2.741..2.744 rows=1 loops=1)"
   "  Index Cond: (book_id = 18)"
   "Planning Time: 0.085 ms"
   "Execution Time: 2.767 ms"
   
   *Объясните результат:*
   так как поиск ведётся по первичному ключу, то используется индекс, который postgre создал автоматически при назначении первичного ключа. благодаря индексу запрос выполняется быстро

8. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   "Seq Scan on t_books  (cost=0.00..2724.00 rows=74715 width=33) (actual time=0.014..17.160 rows=75123 loops=1)"
   "  Filter: is_active"
    "  Rows Removed by Filter: 74877"
    "Planning Time: 0.120 ms"
    "Execution Time: 19.788 ms"
   
   *Объясните результат:*
   в результате запроса выведено примерно половина записей таблицы. селективность запроса низкая, значит использовать индексы неэффективно. планировщик произвёл чтение всей таблицы. время выполнения запроса достаточно большое

9. Посчитайте количество строк и уникальных значений:
   ```sql
   SELECT 
       COUNT(*) as total_rows,
       COUNT(DISTINCT title) as unique_titles,
       COUNT(DISTINCT category) as unique_categories,
       COUNT(DISTINCT author) as unique_authors
   FROM t_books;
   ```
   
   *Результат:*
   "total_rows"	"unique_titles"	"unique_categories"	"unique_authors"
    150000	150000	6	1003

10. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:*
    DROP INDEX

    Запрос завершён успешно, время выполнения: 101 msec.

11. Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:
    a. `WHERE title = $1 AND category = $2`
    b. `WHERE title = $1`
    c. `WHERE category = $1 AND author = $2`
    d. `WHERE author = $1 AND book_id = $2`
    
    *Созданные индексы:*
    a. CREATE INDEX A ON t_books(title, category) 
    b. CREATE INDEX В ON t_books(title) 
    c. CREATE INDEX C ON t_books(category, author)
    d. CREATE INDEX D ON t_books(author)
    *Объясните ваше решение:*
    a. создаём индексы по искомым столбцам. так как различных категорий немного (6), то возможно создать индекс только по названию, в таком случае запрос будет также выполнятся достаточно быстро - 0.2мс, но медленнее, чем с составным индексом (0,089мс)
    b. можно было использвать индекс из прошлого пункта, тк поиск ведётся по первому столбцу из индекса А. если использовать индекс В, то запрос будет выполняться сканированием индекса В
    c. в данном случае в индексе также можно указать только столбец автор, однако для ускорения процесса (хоть и не намного) можно использовать индексирование по обеим столбцам
    d. так как ИД является первичным ключом, то включать его в индекс не нужно

12. Протестируйте созданные индексы.
    
    *Результаты тестов:*
    a. "QUERY PLAN"
        "Index Scan using a on t_books  (cost=0.42..8.21 rows=1 width=33) (actual time=9.765..9.766 rows=0 loops=1)"
        "  Index Cond: (((title)::text = 'aaa'::text) AND ((category)::text = 'aa'::text))"
        "Planning Time: 0.265 ms"
        "Execution Time: 9.786 ms"
    b. "QUERY PLAN"
        "Index Scan using ""В"" on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.025..0.025 rows=0 loops=1)"
        "  Index Cond: ((title)::text = 'aaa'::text)"
        "Planning Time: 0.076 ms"
        "Execution Time: 0.041 ms"
    с."QUERY PLAN"
        "Index Scan using c on t_books  (cost=0.29..8.23 rows=1 width=33) (actual time=0.851..0.852 rows=0 loops=1)"
        "  Index Cond: (((category)::text = 'aa'::text) AND ((author)::text = 'aaa'::text))"
        "Planning Time: 1.338 ms"
        "Execution Time: 0.879 ms"
    d."QUERY PLAN"
        "Index Scan using t_books_id_pk on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.030..0.030 rows=0 loops=1)"
        "  Index Cond: (book_id = 1)"
        "  Filter: ((author)::text = 'aaa'::text)"
        "  Rows Removed by Filter: 1"
        "Planning Time: 0.119 ms"
        "Execution Time: 0.053 ms"
    
    *Объясните результаты:*
    a.сканирование индекса А, условие индекса - совпадение двух строк с заданным значением, планируемое время выполнения 0,26мс
    b.используется индекс, условие - совпаение строк, время выполнения - 0,041
    c.используется индекс С, условие выполнения - совпадение двух строк, время выполнения - 0,87
    d.используется два индекса - созданный нами и индекс на первичный ключ, условие - совпадение значений, время выполнения - 0,053

13. Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

14. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    [Вставьте результат выполнения]

15. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

16. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

17. Попробуйте удалить все индексы:
    ```sql
    DO $$ 
    DECLARE
        r RECORD;
    BEGIN
        FOR r IN (SELECT indexname FROM pg_indexes 
                  WHERE tablename = 't_books' 
                  AND indexname != 'books_pkey')
        LOOP
            EXECUTE 'DROP INDEX ' || r.indexname;
        END LOOP;
    END $$;
    ```
    
    *Результат:*
    [Вставьте результат выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

18. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*
    [Вставьте планы выполнения для обоих вариантов]
    
    *Объясните результаты:*
    [Ваше объяснение]

19. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

20. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]

21. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    [Вставьте ваш запрос]
    
    *План выполнения:*
    [Вставьте план выполнения]
    
    *Объясните результат:*
    [Ваше объяснение]