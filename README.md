Налаштування Реплікації MySQL Master-Slave
==========================================

Цей README документує кроки по налаштуванню реплікації MySQL master-slave на Windows, включаючи встановлення MySQL, налаштування бази даних, управління реплікацією та обробку відмов серверів.

Передумови
----------

-   Windows система (можуть бути віртуальні машини, в мене раняться 2 sql сервери з одної)
-   Chocolatey для встановлення програмного забезпечення

Встановлення MySQL
------------------

Використовуючи Chocolatey, встановила MySQL:

```bash
choco install mysql
```

Налаштування
------------

### Налаштування сервера майстра

1.  Файл конфігурації MySQL:

    -   Знайшла файл конфігурації MySQL (`my.ini`) та в едітері дописала:
        -   `server-id=1`
        -   `log-bin=mysql-bin`
2.  Створення користувача для реплікації:

    -   Створила користувача для реплікації (використається потім slave сервером) :
      
        ```sql
        CREATE USER 'replicator'@'%' IDENTIFIED BY 'replicator_password';
        GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
        ```
    -   Налаштування користувача реплікації для використання `mysql_native_password`:

        У деяких налаштуваннях MySQL, особливо у версіях 8.0 і вище, стандартний метод автентифікації може викликати проблеми з реплікацією. Щоб забезпечити сумісність, налаштувала          користувача реплікації для використання `mysql_native_password`:
        
        ```sql
        ALTER USER 'replicator'@'%' IDENTIFIED WITH 'mysql_native_password' BY 'replicator_password';
        FLUSH PRIVILEGES;
        ```
        

### Налаштування сервера слейва

1.  Дублювання та зміна файлу конфігурації:
    -   Скопіювала файл `my.ini` і створила `my_slave.ini` в тій же папці.
    -   Змінила `my_slave.ini`:
        -   Унікальний `server-id` (просто 2 в моєму випадку)
        -   Інший порт (так як на одній машині з майстром)
        -   Окремий каталог даних (коли запускала сервер, використала команду ініціалізації)
          
          ```bash
          mysqld --defaults-file="C:\\tools\\mysql\\current\\my_slave.ini" --initialize
          ```


Налаштування бази даних на майстрі
----------------------------------

1.  Створення бази даних та таблиць:

    ```sql

    CREATE DATABASE mydb;
    USE mydb;
    CREATE TABLE Clients (client_id INT, name VARCHAR(255), PRIMARY KEY(client_id));
    CREATE TABLE Orders (order_id INT, order_date DATE, client_id INT, PRIMARY KEY(order_id), FOREIGN KEY(client_id) REFERENCES Clients(client_id));
    CREATE TABLE Products (product_id INT, product_name VARCHAR(255), price DECIMAL, PRIMARY KEY(product_id));
    ```

2.  Вставка даних:

    ```sql

    INSERT INTO Clients VALUES (1, 'John Doe'), (2, 'Jane Doe');
    INSERT INTO Orders VALUES (1, '2021-01-01', 1), (2, '2021-01-02', 2);
    INSERT INTO Products VALUES (1, 'Laptop', 1200), (2, 'Phone', 300);
    ```

Створення дампу бази даних з майстра
------------------------------------

На майстер-сервері створила дамп бази даних з позицією бінарного лога:

1.  Експорт бази даних `mydb` з даними майстра:

    -   Використала утиліту `mysqldump` для створення резервної копії бази даних `mydb`, включаючи позицію бінарного лога. Опція `--master-data=2` додає коментар з командою `CHANGE MASTER TO` у файл дампу, яка включає поточне ім'я файлу бінарного лога та позицію на майстер-сервері.

        ```bash
        mysqldump -u root -p --databases mydb --master-data=2 > mydb_dump.sql
        ```

    -   Ця команда створює дамп бази даних `mydb` на майстер-сервері та включає позицію майстер бінарного лога, яка є важливою для правильного налаштування реплікації на сервері слейва.
2.  Перенесення файлу дампу:

    -   Так як в мене обидва сервіси на одній машині, то робити нічого не трабе. Якщо слейв-сервер знаходиться на іншій машині, то треба передати файл `mydb_dump.sql` на цю машину використовуючи один з методів (наприклад, SCP, FTP, файловий обмін).

Налаштування сервера слейва
---------------------------

1.  Імпорт дампу бази даних:

    Імпортувала `mydb_dump.sql` у інстанц MySQL на слейв-сервері.     
      
    ```bash
    mysql -u root -p --port=3307 < path_to_mydb_dump.sql
    ```


3.  Запуск слейв-сервера:

    ```sql
    CHANGE MASTER TO
    MASTER_HOST='ip_адреса_майстра (localhost в моєму випадку)',
    MASTER_USER='replicator',
    MASTER_PASSWORD='replicator_password',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=123 (можна знайти через команду STATUS на мастері);
    START SLAVE;
    ```

Тестування відмови сервера
--------------------------

1.  Симуляція відмови майстер-сервера:

    -   Зупинила MySQL на майстер-сервері.
    -   Спостерігала за реакцією слейва: `SHOW SLAVE STATUS\G`. Отримала failed to connect to server :)

2.  Перезапуск майстер-сервера:

    -   Перезапустила MySQL на майстрі.
    -   Перевірила статус слейва для відновлення реплікації - все знову було добре.

Тест реплікації даних
---------------------

1.  Вставка даних на майстрі:

    ```sql
    INSERT INTO mydb.clients (client_id, name) VALUES (3, 'Alice Wonderland');
    ```

2.  Перевірка даних на слейві:

    Перевірила таблицю `clients` на слейв-сервері, все було там.
      
      ```sql
      SELECT * FROM mydb.clients;
      ```
