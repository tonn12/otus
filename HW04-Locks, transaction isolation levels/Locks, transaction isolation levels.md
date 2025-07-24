1. Вносим в конфигурационный файл параметры. Включаем записо о блокировках в логи PostgreSQL привышающие 200 мс и перечитываем конфигурацию.
ALTER SYSTEM SET log_lock_waits = 'on';
ALTER SYSTEM SET deadlock_timeout = '200ms';
SELECT pg_reload_conf();

2. Производим первое подключение и выполняем запрос в транзакции на обновление строки со значением id=1 и меняем значение в столбце content на locked content. Транзакцию не закрываем (не выполняем команды COMMIT/ROLLBACK), она остается открытой
    - psql
    - \c test
    - BEGIN;
    - UPDATE test_table SET content = 'locked content' WHERE id = 1;

3. Производим второе подключение и выполняем тот же самый запрос в тразакции.
    - psql
    - \c test
    - BEGIN;
    - UPDATE test_table SET content = 'locked content' WHERE id = 1;

Получаем блокировку.

4. Открываем лог PostgreSQl и видим записи о нашей блокировки, которая привышает 200 мс.

5. Повторяем то же самое но уже с тремя соединениями. Первое соединение
    - psql
    - \c test
    - BEGIN;
    - UPDATE test_table SET content = 'locked content' WHERE id = 1;

6. Второе соединение
    - psql
    - \c test
    - BEGIN;
    - UPDATE test_table SET content = 'locked content' WHERE id = 1;

7. Третье соединение
    - psql
    - \c test
    - BEGIN;
    - UPDATE test_table SET content = 'locked content' WHERE id = 1;

8. Открываем новое соединение, не закрывая предыдущии 3.
psql

9. Выводим данные из pg_locks
select * from pg_locks;

10. Видим что у нас есть две блокировки у котороых есть время старта ожидания, с пидами 105 и 69. Так же видем что пид 105 ждет блокировку на объекты transactionid=169693 и tuple=55237, и у пида 69 блокировка на те же объекты.

11. Так же видим что пид 58 наложил RowExclusiveLock на теже объекты что пытаются изменить пид 69 и 105.

12. Из всего этого делаем вывод что изначально пид 58 наложил блокировку, далее пид 69 попытался изменить те же данные, но ушел в ожидание ожидая завершения изменений пида 58, и последжний пид 105 так же попытался изменить все эти же данные но то же ушел в ожидание ожиданя завершения изменений пида 69.

13. Теперь воспроизведем взаимоблокировки. Очищаем таблицу от старых данных
TRUNCATE TABLE test_table;

14. Проверяем что таблица пустая (должно выдать (0 rows) )
SELECT id FROM test_table ORDER BY id;

15. Создадим в нашей тестовой БД три строки с данными
INSERT INTO test_table (id, content)
VALUES (1, 'A'), (2, 'B'), (3, 'C')
ON CONFLICT (id) DO NOTHING;

16. Проверяем что данные добавились
SELECT * FROM test_table;

17. Выполняем запрос на изменения в первой сессии
BEGIN;
UPDATE test_table SET content = 'locked A' WHERE id = 1;

18. Выполняем запрос на изменение во второй сессии
BEGIN;
UPDATE test_table SET content = 'locked B' WHERE id = 2;

19. Выполняем запрос на изменение в третьей сессии
BEGIN;
UPDATE test_table SET content = 'locked C' WHERE id = 3;

20. Выполняем запрос на изменения в первой сессии
UPDATE test_table SET content = 'wait B' WHERE id = 2;

21. Выполняем запрос на изменение во второй сессии
UPDATE test_table SET content = 'wait C' WHERE id = 3;

22. Выполняем запрос на изменение в третьей сессии
UPDATE test_table SET content = 'wait A' WHERE id = 1;

23. PostgreSQL увидил взаимоблокировку и завершил одну из транзакций с ошибкой:
ERROR:  deadlock detected
DETAIL:  Process 105 waits for ShareLock on transaction 169698; blocked by process 58.
Process 58 waits for ShareLock on transaction 169699; blocked by process 69.
Process 69 waits for ShareLock on transaction 169700; blocked by process 105.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "test_table"

24. Можно разобраться в ситуации постфактум изучив лог PostgreSQL, при условии что произведены нужные настройки. В частности в логе можно увидеть как блокировки с долгим ожиданием, так и взаимоблокировки.