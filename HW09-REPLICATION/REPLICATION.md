1. Имеем два сервера с PostgreSQL 14 и БД demo на первом сервере.

2. Открываем файл pg_hba.conf и добавляем следующую строку:
host    replication     replica_user    192.168.88.254/24       md5

Где:
- host - Подключение по TCP/IP
- replication - Это не имя базы данных, а специальное ключевое слово для репликации
- replica_user - Имя пользователя, которому разрешено подключение
- 192.168.88.254/24 - IP-адрес, которому разрешён доступ
- md5 - Метод аутентификации — по паролю, захешированному через MD5

3. Создаем роль replica_user с парвами REPLICATION
CREATE ROLE replica_user WITH REPLICATION LOGIN PASSWORD 'secret';

4. Создаем слот репликации first_slot на первом сервере
SELECT pg_create_physical_replication_slot('first_slot');

5. Переходим на второй сервер и останавливаем службу PostgreSQL
systemctl stop postgrespro-std-14.service

6. Проверяем что служба остановилась
systemctl status postgrespro-std-14.service

7. Удаляем старый каталог data
rm -rf /var/lib/pgsql/14/data

8. Переключаемся на пользователя postgres и выполняем команду pg_basebackup, которая нам скопирует директорию data с первого сервера (мастера)
pg_basebackup -h 192.168.88.253 -D /var/lib/pgpro/std-14/data -U replica_user -Fp -Xs -P -R

где:
- pg_basebackup - Утилита для создания резервной копии базы с мастера для репликации
- -h 192.168.88.253 - IP-адрес мастера (primary-сервера), с которого забирается директория data
- -D /var/lib/pgpro/std-14/data - Каталог, куда на реплике будет записана копия данных (новый кластер)
- -U replica_user - Имя пользователя с правами REPLICATION
- -Fp - Формат plain (обычная директория с файлами данных)
- -Xs - Копировать WAL-файлы вместе с бэкапом (для консистентности), s = stream
- -P - Показывать прогресс выполнения
- -R - Автоматически создать файл standby.signal (включает режим реплики) и строку подключения к мастеру (primary_conninfo) в postgresql.auto.conf

9. Запустить службу PostgreSQL на втором сервере
systemctl start postgrespro-std-14.service

10. Проверяем, создалась ли реплика. Выполняем команду в консоли psql, которая должна нам вернуть значение true если реплика создалась:
SELECT pg_is_in_recovery();

11. На мастере то же выполняем команду для проверки, которая нам должна вернуть строку с активным слотом репликации
SELECT * FROM pg_replication_slots;

12. Добавим в БД demo таблицу test
CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    flight_no TEXT NOT NULL,
    departure TIMESTAMP NOT NULL,
    arrival TIMESTAMP NOT NULL,
    status TEXT,
    aircraft_code TEXT
);

13. Проверяем на реплике, что таблица появилась
\с demop
\d+
\d+ test

14. Теперь удалим таблицу test
DROP TABLE test;

15. Проверим на реплике что таблица удалилась
\с demop
\d+

16. Останавливаем службу PostgreSQL на первом сервере, имитируя тем самы недоступность мастера по каким либо причинам
systemctl stop postgrespro-std-14.service

17. Повышаем реплику до мастера при помощи promote на втором сервере выполниф команду под пользователем postgres
pg_ctl promote -D /var/lib/pgpro/std-14/data

18. Проверяем что реплика стала мастером при помощи запроса, который нам должен вернуть false
SELECT pg_is_in_recovery();

Репликация в PostgreSQL обеспечивает сохранность и доступность данных за счёт их передачи на резервный сервер в синхронном или асинхронном режиме. Это позволяет в случае отказа основного узла минимизировать простой системы, оперативно повысив реплику до роли мастера. Кроме того, на реплику можно перевести все запросы на чтение, что позволит нам распределить нагрузку и повысить отказоустойчивости инфраструктуры.