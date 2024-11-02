# Проектирование высоконагруженного приложения для сервиса Яндекс Такси

Работа в рамках предмета "Проектирование высоконагруженных приложений" программы "Веб-разработка" образовательного центра VK в МГТУ им. Баумана

Выполнила **Валова София**

## Содержание:
1. [Тема и целевая аудитория](#1)
2. [Расчет нагрузки](#2)
3. [Глобальная балансировка нагрузки](#3)
4. [Локальная балансировка нагрузки](#4)
5. [Логическая схема БД](#5)
6. [Физическая схема БД](#6)
7. [Алгоритмы](#7)
8. [Технологии](#8)
9. [Обеспечение надежности](#8)

## 1. Тема и целевая аудитория БД <a name="1"></a>

Темой курсовой работы выбран агрегатор такси Яндекс.Такси

**Аудитория**

Согласно отчетам за второй квартал 2024 год аудитория сервиса составляет 49 млн пользователей в месяц, с увеличением аудитории на 16% от года к году [^1]

Основная аудитория это Россия, но так же сервис предоставляет услуги еще в 17 странах

**Функционал**

1. Указать адрес
2. Динамический расчет стоимости и времени поездок
3. "Маркетплейс" водителей и пользователей
4. Выйти на линию
5. Принять заказ
6. Рейтинг и отзывы на водителей
7. Оформить заказ
8. Отслеживание местоположения такси
9. Просмотр информации о заказе
10. Написать сообщение в чат
11. История заказов

**Продуктовые решения**
1. Предложить точку останова, если это сэкономит время и деньги пользователю
2. Совместные поездки
3. Кэшбэк баллами

## 2. Расчет нагрузки БД <a name="2"></a>

### Продуктовые метрики
- MAU - 49 млн
- DAU - 4 млн (в 2020 году [^2]), предположим, что сейчас 5 млн
- Средний размер хранилища пользователя:
  
| Хранимые данные   | Размер единицы                           | Количество на пользователя  | Размер на пользователя      | 
|-------------------|------------------------------------------|-----------------------------|-----------------------------|
| Персональные данные (ФИО, почта, пароль и т.д.) | `1 КБ`     | `1`                         | `1 Кб`                      |
| Аватар              | `200 Кб`                               | `1`                         | `200 Кб`                    |
| Сохраненные адреса  | `0,5 КБ`                               | `6`                         | `3 Кб`                      |
| История поездок*    | `60 Кб`                                | `150**`                     | `8800 Кб`                   |

*Расчет поездки: адреса, информация о водителе, стоимость, машина, дата, статус поездки, метаданные, маршрут и другое = 60 Кб<br>
** За 2023 было совершено 3,29 млрд поездок, яндекс такси занимает 80% рынка, тогда количество заказов через данный сервис составляет 2,5 млрд[^4] Получается в месяц 208 млн поездок, если MAU 49 млн, то в среднем совершается 4 поездки в месяц -> 50 поездок в год. Предположим, что средний пользователь пользуется такси 3 года -> 150 поездок<br>
**Итого** - 9 Мб * 50 млн = 430 Пб

- Среднее количество действий пользователя, водителя в день:<br>
**Оформление заказа:** `4 заказа/мес / 30 сут/мес = 0,13`<br>
**Посмотреть стоимость:** `0,13 * 3 = 0,4` предположим пользователь 2 раза проверит стоимость такси, на третий раз закажет<br>
**Выйти на линию:** `25`<br>
**Принять заказ:** `25`<br>
**Отслеживание местоположения такси:** `3*4 / 30 = 0,4` пользователя проверит местоположение 3 раза - после оформления, когда пройдет половина от времени, за которое такси должно приехать, и за 5 минут до приезда<br>
**Просмотр информации о поездке:** `2*4 / 30 = 0,27 ` пользователь совершит действие два раза: первый - чтоб ознакомитсья с машиной, второй - уточнить модель и номер машины

### Технические метрики
1. Размер хранения в разбивке по типам данных

| Тип                                 | Размер |
|-------------------------------------|--------|
| Фото                                | 9 Тб   |
| Данные о поездках                   | 140 Тб | 
| Данные о пользователях и таксистах  | 47 Гб  |

3. Сетевой трафик<br>
- Пиковое потребление в теченнии суток (в Гбит/с) - разбивка по существенным типам трафика (для выбора сетевых интерфейсов)
- Суммарный суточный (Гбайт/сутки) - разбивка по существенным типам трафика (опционально, для подсчета стоимости хостинга)

| Тип               | Суммарный                                                         | Пиковый                              |
|-------------------|-------------------------------------------------------------------|--------------------------------------|
| API               |`50к запросов * 86400 с * 2 Кб / (1024 * 1024) = 8200 Гбайт/сутки` |`82000 * 8 бит * 2 = 100к Гбит/с`|
| Статические файлы |`80 запросов * 86400 с * 200 Кб / (1024 * 1024) = 1300 Гбайт/сутки`|`1300 * 8 бит * 2 = 20к Гбит/с`       |

3. RPS в разбивке по типам запросов (запросов в секунду) - для основных запросов

| Тип                                    | Среднее | Пиковое |
|----------------------------------------|---------|---------|
| Оформление заказа***                   | 80      | 160     |
| Запрос на расчет стоимости поездок     | 240     | 480     |
| Выйти на линию                         | 80      | 160     |
| Отказаться от поездки                  | 7       | 10      |
| Отправка местоположения водителя ****  | 40к     | 80к     |

***7 млн поездок в день - 80 в секунду, в 2015 году пиковое значение было в 2,1 раза больше среднего [^3] <br>
****7 млн поездок в день, 25 поездок в день делает водитель => 280 000 водителей. Из наблюдения за экраном телефона таксиста во время поездки, задержка по отображению перемещения 2-3 секунды. Тогда пусть данные о местоположении отображаются каждые 2-3 секунды => в секунду ~100к запросов, будем также считать, что в пик работает 80% водителей (rps 80к пик, среднее в 2 раза меньше)

## 3. Глобальная балансировка нагрузки БД <a name="3"></a>

### Функциональное разбиение по доменам

taxi.yandex.ru - основной домен яндекс такси<br>
ride.taxi.yandex.ru - запросы, связанные с поездками: расчет цены, "маркетплейс" водителей и пользователей<br>
pro.taxi.yandex.ru  - запросы из Яндекс Про (приложение для водителей яндекс такси) <br>
user.taxi.yandex.ru - запросы от пользователей<br>

### Расположение ДЦ

Для обслуживания запросов по Россиии ДЦ расположим в Москве, Новосибирске, из средней азии запросы так же идут в Новосибирск, для европейских, африканских стран и ОАЭ - в Стамбуле.

### Расчет распределения запросов из секции "Расчет нагрузки" по типам запросов по датацентрам

66% пользователей Яндекс Такси из России, значит на Россию приходится 4,62 млн поездок/день и 2,64 млн пользователей. На остальные страны приходится 2,38 млн поездок/день и 1,36 млн пользователей/день.<br>
Количество поездок в центральной России составляет 60%.<br>
ДЦ в Москве обслуживает все запросы центральной России и является основным ДЦ.<br>
Локальные ДЦ принимаеют запросы от ride.taxi.yandex.ru и user.taxi.yandex.ru, то есть запросы на расчет цены поездок, составление маршрутов и запросы от водителей. Запросы от юзеров идут в Московский ДЦ.

| ДЦ                     | Млн пользователей/день | Млн поездок/день | RPS            | Запросы                                                          |
|------------------------|------------------------|------------------|----------------|------------------------------------------------------------------|
| Москва                 | 1,6                    | 2,8              | 20к            | Все для центральной России и действия юзеров                     |
| Новосибирск            |`1 + 0,68 = 1,68`      |`1,8 + 1,1 = 2,9`|`13 + 8к = 21к`| Расчет цены поездок, составление маршрутов, запросы от водителей |
| Стамбул                |  0,68                  |  1,1             | 8к             | Расчет цены поездок, составление маршрутов, запросы от водителей |

RPS по ДЦ и запросам

| ДЦ\Запрос              | Расчет стоимости поездки| Выйти на линию | Оформить заказ| Отправка местоположения водителя |
|------------------------|-------------------------|----------------|---------------|----------------------------------|
| Москва                 | 96                      | 32             | 80            | 16к                              |
| Новосибирск            |`40 + 64 = 106`          |`13 + 21 = 34  `| -             |`6,6к + 10к = 16,6к               |
| Стамбул                | 40                      | 13             | -             |6,6к                              |

### Схема DNS балансировки

Для балансировки использовать Geo-based DNS, так как расположение серверов обусловлено географией

## 4. Локальная балансировка нагрузки БД <a name="4"></a>
### Схема балансировки для входящих и межсерверных запросов

В качестве L7-балансировщика работает Envoy, алгоритмы балансировки: Least Connections для сервисов расчета цен и "маркетплейса водителей и пользователей" (так как это затратные по времени операции), для остальных - Round-Robin.

### Схема отказоустойчивости

Протокол VRRP позволяет нескольким хостам работать как единое целое, предоставляя один виртуальный IP-адрес. В случае отказа главного резервный автоматически становится главным

### Нагрузка по терминации SSL

SSL-терминация будет происходить на балансировщике. У ДЦ в Москве и Новосибирске ~20к rps, в Стамбуле 6к, рассчитаем для 20к: время на установление соединения - 300мс -> для обработки запросов потребуется 6000 с 

## 5. Логическая схема БД <a name="5"></a>
![Пример изображения](bd5.png)

| Таблица             | Размер строки, байт | Кол-во записей | Объем | Чтение, Кбит/с | Запись, Кбит/с |
|---------------------|---------------------------|----------------|-------|--------|--------|
| ride                | 530  | 8 млрд  | 4 Тб   |  255  |  425  |
| driver              | 870  | 1 млн   | 830 Мб |  135  |  -  |
| passanger           | 370  | 300 млн | 100 Гб |  135  |  -  |
| car                 | 116  | 3 млн   | 330 Мб |  70  |  -  |
| garage              | 130  | 40 млн  | 4,8 ГБ |  82  |    |
| type                | 64   | 100     | 6 Кб   |  -  |  -  |
| statistics          | 86   | 8 млрд  | 640 Гб |  -  |  53  |
| rating              | 14   | 80 млн  | 1 Гб   |  9  |  3  |
| driver rating values| 106  | 2,5 мдрд| 240 Гб |  -  |  10  |
| passanger rating values|106| 2,5 млрд| 240 Гб |  -  |  10  |
| geoposition         | 36   | 7 трлн  | 229 Тб |  -  |  10000  |
| current geoposition | 76   | 280 тыс | 20 Мб  |  25  |  20000  |
| demand              | 148  | 7 млн   | 1 Гб   | 115  |  115  |
| proposal            | 128  | 7 млн   | 950 Мб | 66   |  82  |

## 6. Физическая схема БД <a name="6"></a>

### Запросы
| Таблица             | Запросы                |
|---------------------|------------------------|
| ride                |`INSERT INTO ride(passanger_id, start_address, end_address, price, status) ..`-создание заказа<br> `UPDATE ride SET garage_id=$1, points=$2, waiting_duration=$3, waiting_price=$4, status=$5, started_at=$6 WHERE id=$7`- начало поездки<br> `UPDATE ride SET price=$1, fee=$2, duration=$3, status=$4 WHERE id=$5`-завершение поездки<br> `SELECT id, garage_id, route_points, start_address, end_address, price, started_at, duration, status WHERE id=$1` - данные об определенной поездке<br> `SELECT id, garage_id, route_points, start_address, end_address, price, started_at, duration, status WHERE passanger_id=$1 ORDER BY started_at`-история поездок пользователя|
| driver/passanger rating |`UPDATE .. SET raiting=$1, count=$2 WHERE .._id=$3` - каждый раз, когда пользователя оценили<br> `SELECT rating FROM .. WHERE .._id=$1`|
| driver/passanger    | `SELECT .. FROM driver/passanger WHERE id=$1` - данные о водителях/пользователях<br> Запись происходит редко|
| current geoposition | `UPDATE сurrent_geoposition SET point=$1, status=$2, speed=$3, at=$4 WHERE driver_id=$5` - каждые 3 секунды<br> `SELECT driver_id, point, status, speed, at WHERE id=$1`<br> `SELECT driver_id, point, status, speed, at WHERE at=$1 AND status=$2` |
| geoposition | `INSERT INTO сurrent_geoposition(driver_id, point, at, ride_id) ..` - каждые 3 секунды<br> `SELECT driver_id, point, at WHERE ride_id=$1 ORDER BY at`<br> `SELECT driver_id, point, at WHERE at<$1 AND at>$2` |
| car                 | `SELECT .. FROM car WHERE id=$1` - данные о машине по id|
| garage              | `SELECT driver_id, car_id, class, options FROM garage WHERE id=$1`|
| demand              | `INSERT INTO demand(start_address, ride_id, class, options, hegaxon_id) ..`<br> `SELECT id, start_address, ride_id, class, options, hegaxon_id FROM demand WHERE hegaxon_id IN ($1, $2..)`|
| proposal            | `INSERT INTO proposal(driver_geo, class, options, hegaxon_id, garage_id) ..`<br> `SELECT id, driver_geo, class, options, hegaxon_id, garage_id FROM proposal WHERE hegaxon_id IN ($1, $2..)`|

### Денормализация
- таблица type хранит марки и модели машин, когда водитель берет новую машину ему предлагается список марок и моделей из этой таблицы. И в таблицу car сохраняется не id вида машины, а model и brand, избавляемся от join таким образом

### Выбор БД
1. Основной БД для храненения данных является PostgreSQl
2. Сессии пользователей будут храниться в Redis
3. В таблице statistics хранятся данные по поездкам для аналитики, поэтому так же будем использовать Clickhouse
4. Таблица current-geoposition, proposal, demand должны быть быстро доступны, так как на основе их подбираются водители и пассажиры и рассчитываются цены. Так что эти таблицы будем хранить в Redis, так как она работает в оперативной памяти и обеспечивает быстрый доступ к данным.
5. Logbroker для динамического ценообразования и расчета спроса, данные так же кешируются в Redis [^5]
6. S3-хранилище для хранения сгенерированных карт спроса и аватарок [^5]

### Индексы
Будем использовать BTree индекс, он хорошо подходит для столбцов , являющихся ключами
- **Ride:** по passanger_id, started_at
- **Current geoposition:** по driver_id, at, status
- **Geoposition:** по ride_id, at
- **Driver Rating:** по driver_id
- **Passanger Rating:** по passanger_id
- **Demand:** по hegaxon_id
- **Proposal:** по hegaxon_id

### Клиентские библиотеки
- Postgres: https://github.com/jackc/pgx
- Redis: https://github.com/redis/go-redis
- Kafka: https://github.com/segmentio/kafka-go
- S3: https://github.com/aws/aws-sdk-go

### Партицирование
Таблица Ride довольна велика, Geoposition особенно велика, произведем партицирование(вертикальный шардинг) по полю started_at и at соответственно, по истечению определенного времени старые партиции удаляются.

### Репликация
- Postgres: 1 мастер хост и 2 ведущих
- Clickhouse: 1 мастер хост и 2 ведущих

## 6. Алгоритмы <a name="7"></a>

### Подбор водителей
Водитель отправляет свою геолокацию каждые 3-4 секунды. Информация только о широте и долготе не эффективна при подборе водителей для заказов. Поэтому необходимо разделить пространство на ячейки, внутри которых будет осуществляться поиск водителей, одним из алгоритмов позволяющих это делать является [H3 от Uber](https://newsletter.systemdesign.one/p/how-does-uber-find-nearby-drivers). Вся планета разбита на шестиугольники(гексагоны), у каждого свой 64-битный идентификатор. На основе отправленной геопозиции водителя расчитывается в какой ячейке он находится, и эта информация сохраняется. В итоге подбор водителей и пассажиров происходит не в необъятном пространстве, а в ограниченной области.

### Распределение водителей по пассажирам
Когда пассажир оформляет заказ, добавляется запись о заказе в талицу Demand, начинается поиск водителей по дорожному графу в пределах гексагона(возможность поиска в ближайших гексагонах так же возможна), запись о водителе сохраняется в таблицу Proposal. Таким образом накапливается список водителей и пассажиров, в котором надо эффективно распределить пары. Для каждого гексагона выбираются водители и пассажиры, для каждой пары рассчитывается время подачи. Задача системы сделать так, чтобы общее время подачи было минимальным. Иногда бывает, что на какой-то конкретный заказ подача не так мала, как могла бы быть, но суммарное время все равно минимальное
![Пример изображения](dispatch.png)

### Динамическое ценообразование
Самая главная задача динамического ценообразования – предоставлять возможность заказать такси всегда. Достигается она с помощью коэффициента surge pricing coefficient, на который умножается базовая цена. Если выставить сурдж слишком большим – снизится спрос слишком сильно, будет избыток свободных машин. Если выставить слишком низким – пользователи будут видеть «нет свободных машин».[^5] Коэффициент расчитывается, как отношение заказов к водителям, в каждой геозоне. 

## 8. Технологии <a name="1"></a>
| Технология          | Область применения        | Мотивационная часть |
|---------------------|---------------------------|---------------------|
| Go     | Backend | Высокая скорость разработки, асинхронность, стандартная библиотека включает много всего|
| Swift  | iOS | Основной язык разработки приложений под iOS, разработан Apple |
| PostreSQL | Основная SQL БД | Реляционная база данных с открытым исходным кодом, обеспечивающая высокую надежность и целостность данных|
| ClickHouse | Хранение статистики поездок | Разработанная Яндексом колончатая OLAP-БД, отлично подходит для аналитики|
| Redis  | Хранение таблиц с большой нагрузкой| In-memory БД обеспечивает быстрый доступ к данным| 
| Kafka  | Асинхронный стриминговый сервис | Обработки асинхронных задач и обмен сообщениями между сервисами|
| AWS S3 | Хранилище картинок | Объектно-ориентированное хранилище, распространенный способ хранения статики |
| Envoy  | Прокси-сервер | L7-балансировщик |
| Vault  | Система управления секретами| Шифрование, гибкие политики доступа |
| React+TypeScript| Frontend| Библиотека для создания интерфейсов на основе компонентов, позволяет управлять их состоянием|
| VictoriaMetrics | Хранилище метрик| |
| Grafana | Даiборды| Популярная платформа для создание дашбордов, может подключаться к множеству исnочников данных |

## 9. Обеспечение надежности <a name="9"></a>

### Использованные источники
[^1]: [Презентация для инвесторов МКПАО "Яндекс" с данными за 2 квартал 2024](https://yastatic.net/s3/ir-docs/docs/2024/q2/57a1cu049ffbd144aeged36d47h173c2/IR_2Q2024_RUS_NEW.pdf)
[^2]: [Сколько пользователей пользуется Яндекс.Такси в день: статистика и актуальные данные](https://investim.guru/obzory/skolko-polzovateley-polzuetsya-yandeks-taksi-v-den-statistika-i-aktualnye-dannye)
[^3]: [Исследование Яндекс Такси в Москве](https://yandex.ru/company/researches/2015/moscow/taxi)
[^4]: [Численность поездок в такси в России в 2023](https://marketing.rbc.ru/articles/15040/)
[^5]: [Как устроена платформа динамиче­ского ценообразо­вания Яндекс Такси](https://dev.go.yandex/blog/dynamic-pricing-platform-2024-06-15)
[^6]: [System Design: Uber](https://dev.to/karanpratapsingh/system-design-uber-56b1)
[^7]: [Инструменты надежности Яндекс Такси](https://dev.go.yandex/blog/yandex-taxi-reliability-2024-05-30)
[^8]: [Доклад Яндекса об использовании С++ в Яндекс.Такси](https://habr.com/ru/companies/yandex/articles/571408/)
