# Notebook: Write-up

Автор: [@javach](https://github.com/javach)

Сервис __n0t3b00k__ предоставляет нам невероятно безопасный способ хранить записи. Создаются заметки 
с помощью `POST`-запроса с параметрами названия, текста заметки и ключа. Чтобы вывести заметку в 
её первоначальном виде, необходимо знать комбинацию название-ключ, иначе заметка будет выведена 
в зашифрованном виде.  

## Баг 1 — RCE

Даже не самые внимательные наверняка заметили довольно интересную строчку в `app.py`: 

```python
text = os.popen('cat "notes/%s"' % name).read()
```

`os.popen("command")` выполнит команду `command` в консоли Linux. 
Но, как ни странно, эта уязвимость мало чем нам поможет, потому что для того, чтобы
получить расшифрованные флаги, мы должны знать ключи, а ключи хранятся в базе данных. Через
консоль возможно, но сложнее вывести содержимое базы. 

### Как пофиксить?

Заменить на Pythonic-way чтение из файла

```python
if re.search(r"[./]", name):
    # nice AFR try
    text = ""
else:
    with open("notes/%s" % name) as f:
        text = f.read()
```

## Баг 2 — База данных

Какая же удача, что база данных хранится 
в статических файлах и любой желающий может скачать её как любой статический файл (прописав после 
адреса сервиса `/static/db.sqlite`). А мы можем использовать это в написании эксплойта.  
Для начала скачаем базу и поиграемся с ней, чтобы узнать, какова её структура. Для работы с базой
будем использовать модуль [peewee](http://docs.peewee-orm.com/en/latest/). Для начала, узнаем, 
какие таблицы содержатся в базе данных:
```python
db = SqliteDatabase('db.sqlite')
print(db.get_tables)
```
По выводу понимаем, что таблица всего одна и имя ей — `note`. Теперь мы можем узнать названия столбцов:
```python
print(db.get_columns('note'))
```
Столбцы называются __id__, __name__ и __key__. Два последних нам и нужны.  
Теперь у нас есть всё, чтобы написать работающий эксплойт. Алгоритм эксплойта довольно простой: мы скачиваем базу
данных, подключаемся к ней, получаем все пары «_название-ключ_», а затем с помощью них получаем расшифрованные 
флаги. Для запросов и скачивания базы данных можно использовать модуль 
[requests](http://docs.python-requests.org/en/master/).

[Код эксплойта](javach_notebook_opendb.py)

### Как пофиксить

1. Не хранить базу данных в директории `static/`

2. Не сервить директорию `static/` с помощью `uwsgi` — убрать из `/home/notebook/uwsgi.ini` строку

```
static-map=/static=/home/notebook/static
```

На самом деле, Flask при запуске через `app.run()` или `flask run` продолжит сервить директорию `static/`, но при запуске через `uwsgi` этого не происходит, так как Flask не предназначен для отдачи статических файлов.