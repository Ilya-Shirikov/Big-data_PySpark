# Taxi_Chicago_PySpark

Раньше для вызова такси приходилось звонить на разные номера диспетчерских служб и ждать подачу машины полчаса или даже больше. Теперь сервисы такси хорошо автоматизированы, а среднее время подачи автомобиля Яндекс.Такси в Москве около 3-4 минут. Наверное самые важные вопросы это как научиться прогнозировать высокий спрос и дополнительно привлекать водителей, чтобы пользователи могли найти свободную машину в любое время, в этом проекте мы попробуем ответить на эти вопросы.

В нашем распоряжении исторические данные с [Чикагского информационного портала](https://data.cityofchicago.org) о заказах такси [2022](https://data.cityofchicago.org/Transportation/Taxi-Trips-2022/npd7-ywjz) и [2023](https://data.cityofchicago.org/Transportation/Taxi-Trips-2023/e55j-2ewb) года. Чтобы привлекать больше водителей в период пиковой нагрузки, нужно спрогнозировать количество заказов такси на следующий час. Мы попробуем построить модель для такого предсказания. Решать задачу мы будем с применением PySpark на локальном кластере из Docker контейнеров🐳.

## Основные шаги

Развернуть PySpark на локальном кластере из Docker контейнеров🐳

Провести:
 - EDA
 - предобработку и агрегацию данных,
 - генерацию и отбор признаков,
 - построение ML-моделей.


## **Метрики:**
Для бизнеса важна выгода а следовательно прибыль, поэтому правильное распределение (предсказание) заказов по районам в нужный час позволит максимизировать показатели компаний,  наша целевая метрика в данной  задаче - MAE.


## **Описание данных**

 - Trip ID - Уникальный идентификатор поездки.
 - Taxi ID - Уникальный идентификатор такси.
 - Trip Start Timestamp - Когда поездка началась, округлилась до ближайших 15 минут.
 - Trip End Timestamp - Когда поездка закончилась, округли до ближайших 15 минут.
 - Trip Seconds - Время поездки в секундах.
 - Trip Miles - Расстояние поездки в милях.
 - Pickup Census Tract - Переписной участок, где началась поездка. В отношении конфиденциальности этот переписной участок не показан для некоторых поездок. Эта колонка часто будет пустой для мест за пределами Чикаго.
 - Dropoff Census Tract - Переписной участок, где закончилась поездка. В отношении конфиденциальности этот переписной участок не показан для некоторых поездок. Эта колонка часто будет пустой для мест за пределами Чикаго.
 - Pickup Community Area - Общественная зона, где началась поездка. Эта колонка будет пустой для мест за пределами Чикаго.
 - Dropoff Community Area - Общественная зона, где закончилась поездка. Эта колонка будет пустой для мест за пределами Чикаго.
 - Fare - Стоимость поездки. В стоимость поездки.
 - Tips - Совет для поездки. Денежные чаевые, как правило, не регистрируются.
 - Tolls - Плата за поездку.
 - Extras - Дополнительная плата за поездку.
 - Trip Total - Общая стоимость поездки, общая сумма предыдущих столбцов.
 - Payment Type - Тип оплаты за поездку.
 - Company - Таксомоторная компания.
 - Pickup Centroid Latitude - Широта центра переписного участка или района общины, если переписной участок был скрыт для уединения. Эта колонка часто будет пустой для мест за пределами Чикаго.
 - Pickup Centroid Longitude - Долгота центра переписного участка или района общины, если переписной участок был скрыт для уединения. Эта колонка часто будет пустой для мест за пределами Чикаго.
 - Pickup Centroid Location - Расположение центра переписного участка или района общины, если переписной участок был скрыт для уединения. Эта колонка часто будет пустой для мест за пределами Чикаго.
 - Dropoff Centroid Latitude - Широта центра переписного участка или района общины, если переписной участок был скрыт для уединения. Эта колонка часто будет пустой для мест за пределами Чикаго.
 - Dropoff Centroid Longitude - Долгота центра переписного участка или района общины, если переписной участок был скрыт для уединения. Эта колонка часто будет пустой для мест за пределами Чикаго.
 - Dropoff Centroid Location - Расположение центра переписного участка или района общины, если переписной участок был скрыт для уединения. Эта колонка часто будет пустой для мест за пределами Чикаго.


### **Прогнозирование заказов такси и почему это важно**


В идеальном мире люди все делают заранее и всегда безошибочно планируют свое время. Но мы живем в реальном мире. Если человек опаздывает на работу или, хуже того, в аэропорт, ему важно понимать, успеет ли он вовремя выехать и добраться до места назначения. 
Решая, что заказать, будущий пассажир руководствуется в том числе временем ожидания. Оно может сильно отличаться и в разных приложениях для вызова такси, и в разных тарифах одного приложения. Чтобы пользователь не пожалел о выборе, очень важно показывать точное ЕТА.

Кажется, все просто. Придумать побольше признаков, обучить модель, например CatBoost, спрогнозировать время до прибытия машины — и можно закончить на этом. Но опыт показывает, что лучше не спешить и хорошенько подумать, а потом делать.

Сначала мы не сомневались, что нужно прогнозировать то время, через которое к пользователю фактически приедет водитель. Да, до заказа мы не знаем наверняка, какая именно машина будет назначена. Но мы можем предсказать ETA, используя данные не о конкретном водителе, а о водителях поблизости от заказа. Разумеется, прогноз должен быть достаточно честным, чтобы пользователь мог планировать время. 

Но что значит «честным»? Ведь любой алгоритм прогнозирования плох или хорош только статистически. Встречаются и удачные, и откровенно плохие результаты, но нужно «в среднем» не сильно отклоняться от правильных ответов. Здесь надо понимать, что «в среднем» бывает разное. Например, среднее — это как минимум три понятия из статистики: матожидание, медиана и мода.
