1. Выполним несколько запросов и проанализируем их при помощи EXPLAIN ANALYZE. Первый запрос - поиск по имени пассажира
SELECT * FROM tickets
WHERE passenger_name = 'VLADIMIR SHEVCHENKO';

2. Анализ первого запроса
EXPLAIN ANALYZE
SELECT * FROM tickets
WHERE passenger_name = 'VLADIMIR SHEVCHENKO';

Вывод:
- Использовался Parallel Seq Scan.
- Execution Time: ~208 ms.
- Было просмотрено ~276 тыс. строк.

3. Второй запрос - поиск по коду аэропорта:
SELECT * FROM airports_data
WHERE airport_code = 'SVO';

4. Анализ второго запроса
EXPLAIN ANALYZE
SELECT * FROM airports_data
WHERE airport_code = 'SVO';

Вывод:
- Использовался Parallel Seq Scan.
- Execution Time: ~0.089ms.
- Было просмотрено 103 строки.

5. Создаем индекс по имени пассажира
CREATE INDEX idx_passenger_name ON tickets(passenger_name);

6. Создаем индекс по коду аэропорта
CREATE INDEX idx_airport_code ON airports_data(airport_code);

7. Анализ запроса поиск по имени пассажира:
EXPLAIN ANALYZE
SELECT * FROM tickets
WHERE passenger_name = 'VLADIMIR SHEVCHENKO';

Вывод:
- Используется Bitmap Index Scan + Bitmap Heap Scan.
- Execution Time: ~4.4 ms — в 47 раз быстрее!
- План уже не обрабатывает всё подряд, а эффективно выбирает нужное.

8. Анализ запроса поиска по коду аэропорта:
EXPLAIN ANALYZE
SELECT * FROM airports_data
WHERE airport_code = 'SVO';

Вывод:
Не смотря на созданный индекс PostgreSQL все равно использует Seq Scan. Потому что таблица очень маленькая (всего 100 строк), и PostgreSQL считает, что прочитать всё быстрее, чем идти по индексу. Это ожидаемое поведение.