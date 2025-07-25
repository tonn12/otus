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

9. Попробуем HASH индекс. Выполним сначало анализ третьего запроса без него.
EXPLAIN ANALYZE
SELECT * FROM tickets WHERE passenger_name = 'VLADIMIR SHEVCHENKO';

Вывод:
- Использовался Parallel Seq Scan.
- Execution Time: ~894 ms.
- Было просмотрено ~276 тыс. строк., из них только 236 подошли по фильтру.

10. Теперь создадим HASH INDEX
CREATE INDEX idx_passenger_name_hash ON tickets USING hash(passenger_name);

11. Запустим повторно анализ третиего запроса:
EXPLAIN ANALYZE
SELECT * FROM tickets WHERE passenger_name = 'VLADIMIR SHEVCHENKO';

Вывод:
- Использовался Bitmap Index Scan и Bitmap Heap Scan.
- Execution Time: ~0.699 ms.
- Было просмотрено 236 строк.

HASH INDEX намного быстрее, чем при B-Tree (~4.4 ms) и Parallel Seq Scan (~894 ms). Тем самым делаем вывод что HASH INDEX хорошо применять для поиска по точному совпадению ("=").

12. Теперь проанализируем BRIN индекс. Но сначало выполним анализ четвертого запроса без него:
EXPLAIN ANALYZE
SELECT * FROM flights
WHERE scheduled_departure BETWEEN '2024-01-01' AND '2024-01-15';

Вывод:
- Использовался Index Scan.
- Execution Time: ~31 ms
- Было просмотрено 4 строки.

13. Создаем BRIN индекс и на всякий случай отключим использование сканирование таблиц и индексов B-Tree, а сканирование bitmap на оборот включим.
CREATE INDEX idx_scheduled_departure_brin ON flights USING brin(scheduled_departure);
SET enable_indexscan = off;
SET enable_seqscan = off;
SET enable_bitmapscan = on;

14. Повторно запускаем анализ четвертого запроса:
EXPLAIN ANALYZE
SELECT * FROM flights
WHERE scheduled_departure BETWEEN '2024-01-01' AND '2024-01-15';

Вывод:
- Использовался Bitmap Heap Scan.
- Execution Time: 0.161 ms
- Было просмотрено 6 строки.

Видим что для поиска с фильтром по дате BRIN индекс справляется гораздо быстрее чем да же использование индексов B-Tree (~31 ms).

15. Проанализируем еще один индекс, GiST индекс. Выполним пятый запрос пока без него:
EXPLAIN ANALYZE 
SELECT * FROM airports_data
ORDER BY coordinates <-> point(37.62, 55.75)  -- Москва
LIMIT 5;

Вывод:
- Использовался: Seq Scan,Sort и Limit.
- Execution Time: 0.256 ms
- Было просмотрено 7 строк.

16. Создадим GiST индекс.
CREATE INDEX idx_coordinates_gist ON airports_data USING gist(coordinates);

17. Повторяем анализ пятого запроса:
EXPLAIN ANALYZE 
SELECT * FROM airports_data
ORDER BY coordinates <-> point(37.62, 55.75)  -- Москва
LIMIT 5;

Вывод:
- Использовался: GiST Index Scan, который мы создали.
- Execution Time: 0.123 ms
- Было просмотрено 5 строк.

Чуть быстрее чем Seq Scan (0.256 ms), но запрос при больших объемах покажет еще большую разницу в лучшую строну чем Seq Scan.

Вывод по проделанной работе:
- B-Tree индексы - универсальны и подходят практически ко всем данным и убыстряют выполнения запросов в отличии от полного сканирования таблиц.
- HASH индексы - хороши при поиске по точному совпадению ("="). Они гораздо быстрей чем B-Tree индексы и сканирование таблиц.
- BRIN индексы - приминяются про поиске данных с отбором по датам. Они гораздо быстрей в этом чем B-Tree индексы и сканирование таблиц.
- GiST индексы - приминяются при поиске данных с отбором по гео точкам. При малых объемах почти не делают прирост производительности, но на больших разница видно гораздо больше.