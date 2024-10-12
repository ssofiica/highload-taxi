# Проектирование высоконагруженного приложения для сервиса Яндекс Такси

Работа в рамках предмета "Проектирование высоконагруженных приложений" программы "Веб-разработка" образовательного центра VK в МГТУ им. Баумана

Выполнила **Валова София**

## 1. Тема и целевая аудитория

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

## 2. Расчет нагрузки

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

## 3. Глобальная балансировка нагрузки

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

## 4. Локальная балансировка нагрузки
### Схема балансировки для входящих и межсерверных запросов

В качестве L7-балансировщика работает Envoy, алгоритмы балансировки: Least Connections для сервисов расчета цен и "маркетплейса водителей и пользователей" (так как это затратные по времени операции), для остальных - Round-Robin.

### Схема отказоустойчивости

Протокол VRRP позволяет нескольким хостам работать как единое целое, предоставляя один виртуальный IP-адрес. В случае отказа главного резервный автоматически становится главным

### Нагрузка по терминации SSL

SSL-терминация будет происходить на балансировщике. У ДЦ в Москве и Новосибирске ~20к rps, в Стамбуле 6к, рассчитаем для 20к: время на установление соединения - 300мс -> для обработки запросов потребуется 6000 с 

## 5. Схема БД
![Пример изображения](bd3.jpg)

| Таблица             | Размер одной строки, б | Кол-во записей | Объем |
|---------------------|------------------------|----------------|-------|
| ride                | 530  | 8 млрд  | 4 Тб   |
| driver              | 870  | 2 млн   | 1,6 Гб |
| passanger           | 370  |  |  |
| car                 | 60   | 5 млн   | 300 Мб |
| garage              | 130  | 40 млн  | 4,8 ГБ |
| type                | 64   |  |  |
| statistics          | 86   | 8 млрд  | 640 Гб |
| rating              | 14   | 80 млн  | 1 Гб  |
| geoposition         | 32   | 7 трлн  | 203 Тб |
| current geoposition | 64   | 280 тыс | 17 Мб |

### Использованные источники
[^1]: [Презентация для инвесторов МКПАО "Яндекс" с данными за 2 квартал 2024](https://yastatic.net/s3/ir-docs/docs/2024/q2/57a1cu049ffbd144aeged36d47h173c2/IR_2Q2024_RUS_NEW.pdf)
[^2]: [Сколько пользователей пользуется Яндекс.Такси в день: статистика и актуальные данные](https://investim.guru/obzory/skolko-polzovateley-polzuetsya-yandeks-taksi-v-den-statistika-i-aktualnye-dannye)
[^3]: [Исследование Яндекс Такси в Москве](https://yandex.ru/company/researches/2015/moscow/taxi)
[^4]: [Численность поездок в такси в России в 2023](https://marketing.rbc.ru/articles/15040/)
[^5]: [Как устроена платформа динамиче­ского ценообразо­вания Яндекс Такси](https://dev.go.yandex/blog/dynamic-pricing-platform-2024-06-15)
[^6]: [System Design: Uber](https://dev.to/karanpratapsingh/system-design-uber-56b1)
[^7]: [Инструменты надежности Яндекс Такси](https://dev.go.yandex/blog/yandex-taxi-reliability-2024-05-30)
