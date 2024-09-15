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
2. Заранее рассчитать стоимоть и время поездки
3. Рейтнг и отзывы на водителей
4. Оформить заказ
5. Отслеживание местоположения такси
6. История заказов

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
| История поездок*    | `2,304 Кб`                             | `150**`                     | `350 Кб`                    |

*Расчет поездки: адреса , информация о водителе, стоимость, машина, дата - 1Кб+1Кб+0,004Кб+0,2Кб+0,1Кб = 2,304Кб<br>
** За 2023 было совершено 3,29 млрд поездок, яндекс такси занимает 80% рынка, тогда количество заказов через данный сервис составляет 2,5 млрд[^4] Получается в месяц 208 млн поездок, если MAU 49 млн, то в среднем совершается 4 поездки в месяц -> 50 поездок в год. Предположим, что средний пользоатель пользуется такси 3 года -> 150 поездок<br>
**Итого** - 550 Кб * 50 млн = 25 Тб

- Среднее количество действий пользователя по типам в день:


### Технические метрики


### Использованные источники
[^1]: [Презентация для инвесторов МКПАО "Яндекс" с данными за 2 квартал 2024](https://yastatic.net/s3/ir-docs/docs/2024/q2/57a1cu049ffbd144aeged36d47h173c2/IR_2Q2024_RUS_NEW.pdf)
[^2]: [Сколько пользователей пользуется Яндекс.Такси в день: статистика и актуальные данные](https://investim.guru/obzory/skolko-polzovateley-polzuetsya-yandeks-taksi-v-den-statistika-i-aktualnye-dannye)
[^3]: (https://yandex.ru/company/researches/2015/moscow/taxi)
[^4]: (https://transport.mos.ru/mostrans/all_news/115701)
https://marketing.rbc.ru/articles/15040/
