1. Сохраняем демо БД по авиа перелетам размер mediom (информация полетов за 3 иесяца) на сервере

2. Копируем файл в докер
docker cp /share/CACHEDEV1_DATA/Multimedia/Мои/demo-medium-20170815.sql anton-pg-17:/tmp

3. Проваливаемся внутрь докера
docker exec -it anton-pg-17 bash

4. Переключаемся на пользователя postgres
su postgres

5. Разворачиваем демо БД
psql -U postgres -d demo -f /tmp/demo-medium-20170815.sql

6. После восстановления заходим в консоль PostrgeSQL
psql

7. проверяем что демо БД появилась
\l

8. Выполняем несколько JOIN: INNER JOIN: Список пассажиров, у которых есть билеты на рейсы
SELECT
    t.ticket_no,
    t.passenger_name,
    f.flight_no,
    f.scheduled_departure
FROM
    tickets t
INNER JOIN ticket_flights tf ON t.ticket_no = tf.ticket_no
INNER JOIN flights f ON tf.flight_id = f.flight_id
ORDER BY
    f.scheduled_departure;

9. LEFT JOIN: Все бронирования, даже если нет связанных билетов
SELECT
    b.book_ref,
    b.book_date,
    t.ticket_no,
    t.passenger_name
FROM
    bookings b
LEFT JOIN tickets t ON b.book_ref = t.book_ref
ORDER BY
    b.book_date DESC;

10. RIGHT JOIN: Список всех посадочных талонов и связанных с ними билетов (если есть)
SELECT
    bp.ticket_no,
    tf.amount,
    bp.seat_no
FROM
    boarding_passes bp
RIGHT JOIN ticket_flights tf ON bp.ticket_no = tf.ticket_no AND bp.flight_id = tf.flight_id;

11. Создаем таблицу c JSONB на основе tickets + flights + airports + aircrafts
CREATE TABLE tickets_jsonb AS
SELECT
    t.ticket_no,
    jsonb_build_object(
        'passenger', jsonb_build_object(
            'name', t.passenger_name,
            'id', t.passenger_id,
            'contact', t.contact_data
        ),
        'booking_ref', t.book_ref,
        'flight', jsonb_build_object(
            'flight_no', f.flight_no,
            'departure', f.scheduled_departure,
            'arrival', f.scheduled_arrival,
            'status', f.status,
            'aircraft_code', f.aircraft_code,
            'aircraft_model', ad.model,
            'range_km', ad.range,
            'from', adp.airport_name,
            'to', aar.airport_name
        )
    ) AS data
FROM tickets t
JOIN ticket_flights tf ON t.ticket_no = tf.ticket_no
JOIN flights f ON tf.flight_id = f.flight_id
JOIN aircrafts_data ad ON f.aircraft_code = ad.aircraft_code
JOIN airports_data adp ON f.departure_airport = adp.airport_code
JOIN airports_data aar ON f.arrival_airport = aar.airport_code;

12. Выполним JSONB-запрос на поиск пассажиров летящих на самолёте с дальностью больше 10000 км
SELECT * FROM tickets_jsonb
WHERE (data -> 'flight' ->> 'range_km')::int > 10000;

13. Выполнил несколько запросов на времянные ряды. Среднее время в пути по дням
SELECT
    date_trunc('day', (data -> 'flight' ->> 'departure')::timestamp) AS day,
    avg(
        (data -> 'flight' ->> 'arrival')::timestamp -
        (data -> 'flight' ->> 'departure')::timestamp
    ) AS avg_flight_time
FROM tickets_jsonb
GROUP BY day
ORDER BY day;

14. Максимальная дальность перелёта по неделям
SELECT
    date_trunc('week', (data -> 'flight' ->> 'departure')::timestamp) AS week,
    max((data -> 'flight' ->> 'range_km')::int) AS max_range_km
FROM tickets_jsonb
GROUP BY week
ORDER BY week;