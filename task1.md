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
    "Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=33) (actual time=304.167..304.168 rows=0 loops=1)"
    "  Filter: ((title)::text ~~* 'Relational%'::text)"
    "  Rows Removed by Filter: 150000"
    "Planning Time: 7.717 ms"
    "Execution Time: 304.188 ms"
    
    *Объясните результат:*
    происходит последовательное чтение, для каждой строки происходит регистронезависимое сравнение с заданным выражением, время выполнения запроса 304мс, в результате отфильтровано 150000 строк тк книги со схожим названием нет в таблице

14. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    CREATE INDEX

    Запрос завершён успешно, время выполнения: 869 msec.

15. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    "Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=33) (actual time=72.332..72.333 rows=0 loops=1)"
    "  Filter: (upper((title)::text) ~~ 'RELATIONAL%'::text)"
    "  Rows Removed by Filter: 150000"
    "Planning Time: 0.398 ms"
    "Execution Time: 72.359 ms"
    
    *Объясните результат:*
    планировщик не использовал индекс для выполнения запроса, так как при регулярных выражениях нельзя установить точное соответсвие, то есть переход по Б-дереву не ускоряет процесс, а наоборот замедляет (не знаем, в какую сторону двигаться)

16. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    "Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=33) (actual time=124.855..124.862 rows=1 loops=1)"
    "  Filter: ((title)::text ~~* '%Core%'::text)"
    "  Rows Removed by Filter: 149999"
    "Planning Time: 0.186 ms"
    "Execution Time: 124.885 ms"
    
    *Объясните результат:*
    последовательное чтение всего файла и сравнение с использованием регулярного выражения, время выполнения - 124,9 мс

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
    ERROR:  cannot drop index t_books_id_pk because constraint t_books_id_pk on table t_books requires it
    HINT:  You can drop constraint t_books_id_pk on table t_books instead.
    CONTEXT:  SQL statement "DROP INDEX t_books_id_pk"
    PL/pgSQL function inline_code_block line 9 at EXECUTE 
    
    *Объясните результат:*
    при попытке удалить индекс для первичного ключа выдаёт ошибку, тк индекс для пк необходим по умолчанию. это происходит тк в данном коде название индексп для пк указано неверно, если заменить название индекса на t_books_id_pk (индекс первичного ключа), то запрос выолняется, те удаляются все индексы кроме первичного ключа 
    результат: 
    DO

    Запрос завершён успешно, время выполнения: 114 msec.

18. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*
    Вариант 1: 
    "Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=33) (actual time=150.445..150.446 rows=0 loops=1)"
    "  Filter: ((title)::text ~~* 'Relational%'::text)"
    "  Rows Removed by Filter: 150000"
    "Planning Time: 12.419 ms"
    "Execution Time: 150.483 ms"


    Вариант 2: 
    "Bitmap Heap Scan on t_books  (cost=107.89..111.90 rows=1 width=33) (actual time=0.153..0.155 rows=0 loops=1)"
    "  Recheck Cond: ((title)::text = 'Relational%'::text)"
    "  Rows Removed by Index Recheck: 1"
    "  Heap Blocks: exact=1"
    "  ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..107.89 rows=1 width=0) (actual time=0.144..0.144 rows=1 loops=1)"
    "        Index Cond: ((title)::text = 'Relational%'::text)"
    "Planning Time: 0.760 ms"
    "Execution Time: 0.396 ms"
    
    *Объясните результаты:*
    pg_trgm более эффективен при поиске по подстроке, reverse не применяется, тк при сравнении с подстрокой всё равно необходимо читать всю таблицу (невозможно однознячно сравнить с регулярным выражением). в случае триграмм строки уже отсортированы по триграммам (те частям строк), поэтому применяется битмап и поиск происходит намного быстрее

19. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
    "Bitmap Heap Scan on t_books  (cost=116.57..120.58 rows=1 width=33) (actual time=0.046..0.048 rows=1 loops=1)"
    "  Recheck Cond: ((title)::text = 'Oracle Core'::text)"
    "  Heap Blocks: exact=1"
    "  ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..116.57 rows=1 width=0) (actual time=0.036..0.036 rows=1 loops=1)"
    "        Index Cond: ((title)::text = 'Oracle Core'::text)"
    "Planning Time: 0.120 ms"
    "Execution Time: 0.081 ms"
    
    *Объясните результат:*
    используется созданный ранее битмап индекс с триграммами, время выполнения очень маленькое - 0,081, тк индексный поиск эффективен при поиске точного совпадения

20. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    "Bitmap Heap Scan on t_books  (cost=95.15..150.36 rows=15 width=33) (actual time=0.040..0.041 rows=0 loops=1)"
    "  Recheck Cond: ((title)::text ~~* 'Relational%'::text)"
    "  Rows Removed by Index Recheck: 1"
    "  Heap Blocks: exact=1"
    "  ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..95.15 rows=15 width=0) (actual time=0.030..0.030 rows=1 loops=1)"
    "        Index Cond: ((title)::text ~~* 'Relational%'::text)"
    "Planning Time: 0.232 ms"
    "Execution Time: 0.067 ms"
    
    *Объясните результат:*
    триграммный индекс эффективен при поиске по подстроке

21. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
 explain analyze
	 select * from t_books order by title desc limit 10 
    
    *План выполнения:*
    "Limit  (cost=0.42..1.03 rows=10 width=33) (actual time=1.598..1.606 rows=10 loops=1)"
    "  ->  Index Scan using t_books_desc_idx on t_books  (cost=0.42..9084.34 rows=150000 width=33) (actual time=1.597..1.602 rows=10 loops=1)"
    "Planning Time: 0.420 ms"
    "Execution Time: 1.629 ms"
    
    *Объясните результат:*
    так как в индексе записи отсортированы так же как в запросе, отдельная сортировка не происходит, выводятся первые 10 строк и они уже отсортированны по названию