1. Включить pg_stat_statements (анализ запросов):
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

2. Проверяем что расширение установилось:
\dx

3. Добавляем в postgres.auto.conf
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';

4. Выходим из psql
\q

5. Выходим из докера
exit

6. Перезагружаем докер для принятия изменения в postgres.auto.conf
docker restart anton-pg-17

7. Заходим обратно в докер и в консоль psql
docker exec -it anton-pg-17 bash

8. Запускаем первый тяжелый зпрос:
SELECT date_trunc('day', book_date) AS day, COUNT(*), SUM(total_amount)
FROM bookings
GROUP BY day
ORDER BY day DESC
LIMIT 1000;

9. Запускаем второй тяжелый запрос:
SELECT COUNT(*)
FROM tickets t, bookings b
WHERE t.book_ref = b.book_ref AND b.total_amount > 100;

10. Запускаем третий тяжелый запрос:
SELECT f.aircraft_code, COUNT(*)
FROM flights f
JOIN ticket_flights tf ON tf.flight_id = f.flight_id
GROUP BY f.aircraft_code;

11. Выводм ТОП 5 тяжелых запросов:
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 5;

Вывод: при помощи pg_stat_statements можно посмотреть тяжелые запросы. Вдальнейшем их проанализировать и исправить сами запросы или добавить недостающии индексы в нужные таблицы для ускорения выполнения самих запросов.