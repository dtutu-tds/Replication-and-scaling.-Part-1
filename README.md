# Домашнее задание: Репликация MySQL

## Задание 1

На лекции рассматривались режимы репликации master-slave, master-master, опишите их различия.

### Ответ:

## Master-Slave репликация
- Один главный сервер (master) и несколько подчиненных (slave)
- Данные идут только от master к slave
- Запись только на master, чтение можно с slave
- При падении master нужно вручную переключаться
- Хорошо для приложений где много читают

## Master-Master репликация  
- Несколько серверов, каждый может быть и master и slave
- Данные синхронизируются между всеми серверами
- Запись и чтение на любом сервере
- Автоматическое переключение при сбоях
- Сложнее настроить, могут быть конфликты данных

## Главные отличия
- Master-Slave: одна точка записи, проще настроить
- Master-Master: несколько точек записи, лучше отказоустойчивость
- Master-Master сложнее в управлении но лучше для высокой доступности
---

## Задание 2

Выполните конфигурацию master-slave репликации, примером можно пользоваться из лекции.

### Конфигурация Master-Slave репликации

#### Docker Compose конфигурация:
```yaml
version: '3.8'

services:
  mysql-master:
    image: mysql:8.0
    container_name: mysql-master
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: testdb
      MYSQL_USER: repl_user
      MYSQL_PASSWORD: repl_password
    ports:
      - "3306:3306"
    volumes:
      - ./master.cnf:/etc/mysql/conf.d/mysql.cnf
      - master_data:/var/lib/mysql
    command: --server-id=1 --log-bin=mysql-bin --binlog-do-db=testdb
    networks:
      - mysql-network

  mysql-slave:
    image: mysql:8.0
    container_name: mysql-slave
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: testdb
    ports:
      - "3307:3306"
    volumes:
      - ./slave.cnf:/etc/mysql/conf.d/mysql.cnf
      - slave_data:/var/lib/mysql
    command: --server-id=2 --relay-log=mysql-relay-bin
    networks:
      - mysql-network
    depends_on:
      - mysql-master

volumes:
  master_data:
  slave_data:

networks:
  mysql-network:
    driver: bridge
```

#### Команды для проверки состояния серверов:

**Состояние Master сервера:**
```bash
docker exec -it mysql-master mysql -u root -prootpassword -e "SHOW MASTER STATUS"
```

**Состояние Slave сервера:**
```bash
docker exec -it mysql-slave mysql -u root -prootpassword -e "SHOW SLAVE STATUS\G" | grep -E "(Slave_IO_Running|Slave_SQL_Running|Master_Host|Seconds_Behind_Master)"
```

### Скриншоты выполнения работы:

![Скриншот 2 - Master-Slave репликация](скрин2.png)

*Скриншот показывает состояние и режимы работы серверов в конфигурации master-slave*

---

## Дополнительные задания (со звёздочкой*)

### Задание 3*

Выполните конфигурацию master-master репликации. Произведите проверку.

### Конфигурация Master-Master репликации

#### Docker Compose конфигурация:
```yaml
version: '3.8'

services:
  mysql-master1:
    image: mysql:8.0
    container_name: mysql-master1
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: testdb
      MYSQL_USER: repl_user
      MYSQL_PASSWORD: repl_password
    ports:
      - "3307:3306"
    volumes:
      - ./master1.cnf:/etc/mysql/conf.d/mysql.cnf
      - master1_data:/var/lib/mysql
    command: --server-id=1 --log-bin=mysql-bin --binlog-do-db=testdb
    networks:
      - mysql-network

  mysql-master2:
    image: mysql:8.0
    container_name: mysql-master2
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: testdb
      MYSQL_USER: repl_user
      MYSQL_PASSWORD: repl_password
    ports:
      - "3308:3306"
    volumes:
      - ./master2.cnf:/etc/mysql/conf.d/mysql.cnf
      - master2_data:/var/lib/mysql
    command: --server-id=2 --log-bin=mysql-bin --binlog-do-db=testdb
    networks:
      - mysql-network

volumes:
  master1_data:
  master2_data:

networks:
  mysql-network:
    driver: bridge
```

#### Команды для проверки состояния серверов:

**Состояние первого сервера (mysql-master1):**
```bash
docker exec -it mysql-master1 mysql -u root -prootpassword -e "SHOW MASTER STATUS; SHOW SLAVE STATUS" | grep -E "(File|Position|Slave_IO_Running|Slave_SQL_Running|Master_Host)"
```

**Состояние второго сервера (mysql-master2):**
```bash
docker exec -it mysql-master2 mysql -u root -prootpassword -e "SHOW MASTER STATUS; SHOW SLAVE STATUS" | grep -E "(File|Position|Slave_IO_Running|Slave_SQL_Running|Master_Host)"
```

### Скриншоты выполнения работы:

![Скриншот 3.1 - Состояние первого сервера Master-Master](скрин3.1.png)

*Скриншот показывает состояние mysql-master1 в конфигурации master-master*

![Скриншот 3.2 - Состояние второго сервера Master-Master](скрин3.2.png)

*Скриншот показывает состояние mysql-master2 в конфигурации master-master*

#### Проверка работы репликации:

**Тест записи на первом сервере:**
```sql
-- На mysql-master1
USE testdb;
CREATE TABLE test_replication (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(100),
    server VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO test_replication (message, server) VALUES ('Запись с первого сервера', 'master1');
```

**Проверка на втором сервере:**
```sql
-- На mysql-master2
USE testdb;
SELECT * FROM test_replication;
```

**Тест обратной репликации:**
```sql
-- На mysql-master2
INSERT INTO test_replication (message, server) VALUES ('Запись со второго сервера', 'master2');
```

**Проверка на первом сервере:**
```sql
-- На mysql-master1
SELECT * FROM test_replication;
```

### Результаты выполнения:

✅ **Master-Slave репликация**: Настроена и работает корректно  
✅ **Master-Master репликация**: Настроена двусторонняя синхронизация  
✅ **Тестирование**: Данные успешно реплицируются в обоих направлениях  

---

## Заключение

В ходе выполнения домашнего задания были изучены и практически реализованы два основных типа репликации MySQL:

1. **Master-Slave репликация** - обеспечивает масштабирование чтения и базовую отказоустойчивость
2. **Master-Master репликация** - обеспечивает высокую доступность и возможность записи на несколько серверов

Оба типа репликации имеют свои преимущества и области применения, выбор зависит от конкретных требований приложения к производительности, доступности и консистентности данных.
