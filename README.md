H/W for the lesson "Databases, their types" - Alexey Motorin

# Домашнее задание к занятию "`Кеширование Redis/Memcached`" - `Моторин Алексей`

## Задание 1. Кеширование

`Приведите примеры проблем, которые может решить кеширование.`

`Приведите ответ в свободной форме.`

---

**Использование кеширования позволяет значительно улучшить производительность приложений и снизить нагрузку на базы данных и улучшить отзывчивость систем, что особенно важно при высокой нагрузке и работе с большими объемами данных.**

**Memcached** и **Redis** решают разные классы задач благодаря своим особенностям.

## 1.1\. Memcached: простой и быстрый кеш

`Memcached¹ - Простая ключ-значение (key-value) база данных в оперативной памяти. Создана для ускорения веб-приложений за счёт кэширования. Особенностью является отсутствие persistence (данные теряются при перезагрузке).`

**Решаемые проблемы:**

### ① Высокая нагрузка на базу данных
* **Проблема**: Частые SQL-запросы к базе (например, топ товаров в интернет-магазине).
* **Решение**: Кешировать результаты в Memcached.

```python
# Пример (Python + Django)
from django.core.cache import cache

def get_top_products():
    cached_data = cache.get("top_products")
    if not cached_data:
        cached_data = db.query("SELECT * FROM products ORDER BY sales DESC LIMIT 10")
        cache.set("top_products", cached_data, timeout=3600)  # Кеш на 1 час
    return cached_data
```    

В примере при первом вызове функция сделает запрос к БД, следующие вызовы будут использовать данные из кэша, а через час данные обновятся автоматически.

### ② Сессии пользователей

* **Проблема**: Хранение сессий в файлах или БД медленное.
* **Решение**: Memcached хранит сессии в RAM (быстрый доступ).

```nginx
# Пример для PHP (php.ini)
session.save_handler = memcached
session.save_path = "127.0.0.1:11211"
```

Этот подход позволяет эффективно управлять сессиями в PHP-приложениях, особенно в высоконагруженных системах, где требуется быстрая обработка сессий и возможность горизонтального масштабирования.

### ③ Кеширование API-ответов

* **Проблема**: Внешние API имеют лимиты запросов (например, курсы валют).
* **Решение**: Сохранять ответ на 5–10 минут в Memcached.

```python
def get_exchange_rates():
    rates = cache.get("usd_to_eur")
    if not rates:
        rates = api.fetch("https://api.exchangerate.host/latest")
        cache.set("usd_to_eur", rates, timeout=300)  # 5 минут
    return rates
```

В примере на Python, без обработки ошибок и логирования, используется API https://api.exchangerate.host/latest, делается GET-запрос для получения актуальных курсов. Используется паттерн “cache-aside”, данные хранятся под ключом “usd_to_eur”, а время жизни кэша: 5 минут (300 секунд).

Преимущества такого подхода - оптимизация производительности путём минимизации обращений к API, быстрый доступ к данным из кэша.
Актуализация данных - курсы обновляются каждые 5 минут, обеспечивая баланс между актуальностью и нагрузкой.

### ④ Ускорение статического контента

* **Проблема:** Генерация HTML-страниц «на лету» (например, блоги).
* **Решение:** Кешировать готовый HTML.

```python
# Flask + Memcached
@app.route("/blog/<id>")
def blog_post(id):
    html = cache.get(f"post_{id}")
    if not html:
        post = db.get_blog_post(id)
        html = render_template("post.html", post=post)
        cache.set(f"post_{id}", html, timeout=86400)  # 1 день
    return html
```

При первом запросе генерируется HTML и сохраняется в кэш, следующие запросы используют кэшированный HTML. Через 24 часа данные обновляются автоматически.

¹ - `В IT-сообществе принято считать, что Memcached является устаревшей технологией. Но несмотря на то, что он был создан достаточно давно (первый выпуск состоялся 22 мая 2003 года), он продолжает активно развиваться и использоваться.` 👉[Подробнее, почему Memcached остается актуальным...](/memcached_its_ok.md)

Таким образом, Memcached остается актуальным и эффективным инструментом для решения задач кэширования данных в современных системах.

---

## 1.2\. Redis: расширенное кеширование + in-memory база данных

`Redis - Продвинутая in-memory база данных (не только кэш). Поддерживает разные структуры данных: строки, списки, хеши, множества. Есть persistence (можно сохранять данные на диск).`

**Решаемые проблемы:**

### ① Сложные структуры данных

* **Проблема**: Нужны рейтинги, очереди, графы.
* **Решение**: Redis поддерживает хеши, списки, множества.

```python
# Топ игроков (сортированный набор)
redis.zadd("leaderboard", {"Alice": 100, "Bob": 85, "Charlie": 120})

# Получить топ-3
top_players = redis.zrevrange("leaderboard", 0, 2)  # ['Charlie', 'Alice', 'Bob']
```

### ② Pub/Sub (уведомления в реальном времени)

* **Проблема**: Чат или live-обновления без опросов сервера.
* **Решение**: Redis Pub/Sub.

```python
# Канал "notifications"
redis.publish("notifications", "Новое сообщение!")

# Клиент подписывается
pubsub = redis.pubsub()
pubsub.subscribe("notifications")
```

### ③ Кеш с персистентностью

* **Проблема**: Memcached теряет данные при перезагрузке.
* **Решение**: Redis сохраняет данные на диск (**RDB**/**AOF**).

```bash
# В redis.conf
save 900 1       # Сохранять, если 1 изменение за 15 мин
appendonly yes   # Включить лог изменений
```

### ④ Распределённые блокировки

* **Проблема**: Конкурирующие запросы (например, оплата заказа).
* **Решение**: Redis-локи (**SETNX**).

```python
# Пример на python
def process_payment(order_id):
    lock = redis.setnx(f"lock:{order_id}", "1", ex=30)  # Блокировка на 30 сек
    if not lock:
        raise Exception("Операция уже выполняется!")
    try:
        make_payment(order_id)
    finally:
        redis.delete(f"lock:{order_id}")     
```

### ⑤ Геоданные

* **Проблема**: Поиск ближайших объектов (магазины, такси).
* **Решение**: Redis GEO API.

```python
# Добавить координаты
redis.geoadd("shops", -74.0059, 40.7128, "NYC Store")

# Найти магазины в радиусе 10 км
redis.georadius("shops", -74.0059, 40.7128, 10, unit="km")
```

## **Когда что использовать?**

| **Проблема** | **Memcached** | **Redis** |
| --- | --- | --- |
| Простое кеширование | ✅ Лучше | ⚠️ Можно |
| Сессии пользователей | ✅ | ✅ |
| Сложные структуры данных | ❌ Нет | ✅ |
| Очереди задач (Celery) | ❌ | ✅ |
| Pub/Sub (вебсокеты) | ❌ | ✅ |
| Персистентность | ❌ | ✅ |
| Глобальные блокировки | ❌ | ✅ |

+ **Memcached** — для простого, быстрого и временного кеша.
+ **Redis** — когда нужны сложные данные, персистентность и дополнительные функции.

Оба инструмента часто используются вместе и это даёт ряд 👉[преимуществ](/advantages_of_using_caching.md).

*   **Memcached** — для тяжёлых объектов (страницы, изображения).
*   **Redis** — для данных, требующих структуры (сессии, очереди, геоданные).

Оба решения популярны в высоконагруженных системах (**Facebook**, **Twitter**, **GitHub** используют оба).

**Redis** предоставляет более широкий функционал, лучшую производительность и более гибкую архитектуру, что делает его более привлекательным для современных проектов, но в однопоточная модели может уступать в пропускной способности **Memcached**.

👉&nbsp;[Сравнение особенностей Memcached и Redis](/Memcached&Redis_features.md).

## 1.3\. Типичные проблемы, которые решаются рассмотренными технологиями кэширования:

1.  **Медленные запросы к БД:** кеширование результатов запросов, предварительная агрегация данных.

2.  **Высокие пики нагрузки:** буферизация запросов, распределенное кеширование.

3.  **Ограниченная пропускная способность:** локальное хранение часто используемых данных, снижение количества обращений к медленным ресурсам.

4.  **Проблемы с масштабированием:** распределенное хранение данных, 👉&nbsp;[горизонтальное масштабирование](/horizontal_scaling.md).


## 1.4\. Примеры реальных проблем, решаемых в продакшен сервисах:

1.  **E-commerce платформы:**
	*   Кеширование карточек товаров
	*   Кеширование корзин покупок
	*   Хранение сессий пользователей

2.  **Социальные сети:**
    *   Кеширование новостной ленты
    *   Хранение данных профилей
    *   Буферизация сообщений

3.  **Игровые сервисы:**
    *   Хранение игровых сессий
    *   Кеширование рейтингов
    *   Управление инвентарем

---

## Задание 2 Memcached

`Установите и запустите memcached.`

`Приведите скриншот systemctl status memcached, где будет видно, что memcached запущен.`

---

![sudo systemctl status memcached](https://github.com/presdes/11-02-hw/blob/main/img/2025-04-01_13-50-40.png "sudo systemctl status memcached")

---

## Задание 3. Удаление по TTL в Memcached

`Запишите в memcached несколько ключей с любыми именами и значениями, для которых выставлен TTL 5.`

`Приведите скриншот, на котором видно, что спустя 5 секунд ключи удалились из базы.`

---

![sudo systemctl status memcached](https://github.com/presdes/11-02-hw/blob/main/img/2025-04-01_19-19-31.png "sudo systemctl status memcached")

`При получении значений 3-х ключей видим, что спустя 5 секунд "дожил" только ключ с ttl `

---

## Задание 4. Запись данных в Redis

`Запишите в Redis несколько ключей с любыми именами и значениями.`

`Через redis-cli достаньте все записанные ключи и значения из базы, приведите скриншот этой операции.`

---


---

## Задание 5*. Работа с числами

`Запишите в Redis ключ key5 со значением типа "int" равным числу 5. Увеличьте его на 5, чтобы в итоге в значении лежало число 10.`

`Приведите скриншот, где будут проделаны все операции и будет видно, что значение key5 стало равно 10.`

---
