# Поиск координат по фотографии
https://www.catalyzex.com/paper/arxiv:2307.05845

https://arxiv.org/abs/2307.05845

1. Составить перечень координат Барнаула
2. Загрузить панорамы Барнаула
3. Разрезать фото на более маленькие, отрезать небо и дорогу
4. Найти фото на которых площадь площадь автомобилей более 50%, изъять эти изображения
5. Подготовить структуру геозон, геоячеек
6. Сформировать структуру датасета (фото -> классы геозон)
7. Подготовить обучить модель


# Общий подход к созданию датасета

Датасет будет построен на базе панорамных снимков улиц GOOGLE MAPS

Датасет создается для определенной географической области. Первичный датасет задан для Барнаула. 
Если получиться планирую расширить до 4 городов (2 в России и 2 не в России) для формования иерархической классификации Страна -> Регион/Город -> geocell

Изначально задается географическая область - прямоугольник. Нижний левый угол - начало и верхний правый угол - конец.
Далее формируется матрица координат этого прямоугольника. Шаг между ячейками задается отдельно для долготы и широты в метрах.
Переход от градусов к метрам для первичного датасета задан в виде жестких коэффициентов. Эти коэффициенты не применимы для иных географически областей в связи со смещением широты. Для разных широт в одно градусе содержится различное кол-во км.


Первоначально шаг задан в 20 метров. Это значение подбирается в ручную. 

Для получения снимков использован сервис "streetview" https://github.com/robolyst/streetview. 
Загрузка фотографий происходит в 2 этапа. 
Первично для заданной координаты из google maps извлекается список уникальных названий панорамных снимков смежных с указанной координатой. Если задать маленький шаг между координатами будут получены списки с часто повторяющимися снимками. Если шаг будет слишком большой часть снимков могут быть упущены.
Далее по уникальному имени панорам происходит загрузка изображения.

Постобработка полученных изображений
- для расширения датасета возможно попробовать использую автоэнкодеры создать зимние, весенние, осенние снимки


# 1. Подготовка списка координат Барнаул
https://colab.research.google.com/drive/1BKJvkXlxelojyEPH_ZjippBw4HlrViX6?usp=sharing

В качестве первичного расположения был выбран Барнаул. Для этого города был задан прямоугольник в пределах
нижний левый угол
start_lat = 53.291315
start_lon = 83.590640
верхний правый угол
stop_lat = 53.399023
stop_lon = 83.805205

Шаг был выбран в 20 метров - примерно на таком расстоянии список панорамных снимков не пересекается и не пропускается

Далее простым циклом перебора координат с заданным шагом для локации Барнаул было получено 433922 координат. 

Список координат составлен. Теперь возможно для каждой координаты получить лейблы с данными о помаранных снимках.
Пример списка лейблов:
```
[Panorama(pano_id='suorKTZrkJTaGhUUlgkwIg', lat=53.32822141869757, lon=83.68271865044099, heading=280.7651977539062, pitch=90.88703918457031, roll=359.2576599121094, date='2012-07'),
 Panorama(pano_id='4sFaZfPw9WekRt7CLTFpEw', lat=53.32811947079741, lon=83.6827456451473, heading=101.4230499267578, pitch=89.6850357055664, roll=0.5733432173728943, date=None)]
```
Эта процедура занимает много времени. Google colab выдает среднюю скорость 4-5 координат в секунду, общее время выполнения может достигать 30 часов.
Для ускорения процесса выполнения одновременно было запущено несколько ноутбуков. Полученные данные объединены и очищены от дублирующих значений.

На данном этапе есть потенциал ускорения работы сервиса (распараллеливать процессы, найти иной сервис)

# 2. Загрузка панорам


По Барнаулу всего порядка 76000 панорам
По полученным уникальным лейблам необходимо загрузить панорамные снимки. Достаточно длительный процесс.
На данный момент реализован простой python скрипт для загрузки панорам высокого разрешения
https://colab.research.google.com/drive/1KAsBBn4Bf4ZxjmjQniubULUELyC1Rqeo?usp=sharing

Среднее время загрузки одного файла - 1-3 мин. Что делается для ускорения:
- запуск в терминале сразу нескольких скриптов одновременно
- запуск одновременно нескольких colab notebooks

Для загрузки панорам низкого разрешения (быстро порядка 7-8 часов)

https://colab.research.google.com/drive/1yG5Ja4PBmQVnT1ppAaOs0SxEO3aLBk9p?usp=sharing



Загрузка в процессе https://drive.google.com/drive/folders/1Yru5IrrlYxgW8JVRkLVXEQGhAvbBR5yp?usp=sharing


# 3. Обрезка полученных панорамных снимков

- возможно срезать верхнюю и нижнюю часть снимка (небо, земля). Исходное изображение делим на 6 частей, 
- полученное изображение разрезать на 4 части, и далее со смещением еще на 3. смещение необходимо чтобы граница была на изображении полностью

# 4. Найти фото на которых площадь площадь автомобилей более 50%, изъять эти изображения

На некоторых изображениях есть большие автомобили - автобусы, грузовики, которые могут занимать значительную площадь фото. Гипотеза такая - сегментировать данные объекты и если их площадь более допустим 50 % - такие фото удалять

# 5. Подготовить структуру геозон, геоячеек

Иерархия геозон будет иметь следующую структуру Страна -> Регион/город -> геоячейка.
Согласно https://arxiv.org/abs/2307.05845 два первых уровня оптимальнее разделить в границах административных границ.
Что касается 3 - минимального уровня иерархи, то в статье предложено извлекать кластеры, используя алгоритм кластеризации OPTICS (Ankerst et al., 1999) https://scikit-learn.org/stable/modules/generated/sklearn.cluster.OPTICS.html. Далее присваиваем все еще неназначенные точки данных их
ближайшим кластерам и используем тесселяцию Вороного для определения смежных геоячеек для каждого извлеченного кластера. https://ru.wikipedia.org/wiki/Диаграмма_Вороного. Элементами множетсва, вокруг которых образуются подмножества предложено использовать координаты Достопримеча́тельностей. 
В Барнауле - немного достопримечтаельностей - и здесь нужно подумать, что брать за основу, можно:
- перекрестки - пока наиболее перспективный. В структуре загруженных лейблов есть координаты. По изменению координат вероятно будет возможно понять где будут перекрестки. Получить их координаты - это будут центры подмножеств
- торговые центры
- супермаркеты
- все-таки найти достопремичательности)))
Этот прием используется для равномерно распределения классов и признаков.


# 6. Сформировать структуру датасета

....
 
# 7. Подготовить обучить модель
Чтобы генерировать визуальные представления для последующего проецирования на наши георешетки, наша архитектура использует модель OpenAI CLIP ViT-L/14 336 в качестве основы, которая представляет собой мультимодальную модель, предварительно обученную на наборе данных из 400 миллионов изображений и подписей
https://huggingface.co/openai/clip-vit-large-patch14-336

В наших экспериментах мы добавили линейный слой поверх CLIP's vision encoder для прогнозирования георешеток. Для версий модели с несколькими входными изображениями (например, панорама с четырьмя изображениями для PIGEON) мы усредняем вложения всех изображений. Усреднение встраиваний привело к более высокой производительности по сравнению с объединением нескольких встраиваний с помощью многоголовочного внимания или дополнительных слоев трансформатора.

С этой целью мы дополняем наши обучающие наборы вспомогательными данными по географии, климату и направлению. Эти данные используются для создания синтетических подписей к каждому изображению путем выборки компонентов подписей из разных шаблонов категорий и
их объединения. Для PIGEOTTO мы используем компоненты подписи, основанные на местоположении, климате и направлении движения. Между тем, для PIGEON метаданные Street View позволяют нам дополнительно определять направление по компасу и время года.
Примеры компонентов подписи, выводимых из метаданных изображения, включают:

- Местоположение: “Фотография, которую я сделал в регионе Гаутенг в Южной Африке".
- Климат: “В этом месте умеренный океанический климат".
- Направление по компасу: “Эта фотография направлена на север".
- Сезон (месяц): “Это фото было сделано в декабре".
- Дорожное движение: “В этом месте люди ездят по левой стороне дороги".

Все вышеперечисленные компоненты подписи содержат информацию, относящуюся к геолокации изображения. Следовательно, наше непрерывное контрастивное предварительное обучение создает неявную многозадачную настройку и гарантирует, что модель изучает богатые представления данных, одновременно изучая функции, которые имеют отношение к задаче геолокации изображения.

Чтобы еще больше уточнить предположения нашей модели в пределах георешетки и улучшить производительность на уровне улиц и городов, вместо простого прогнозирования средней широты и долготы всех точек в пределах георешетки (Pramanick et al., 2022), мы выполняем уточнение внутри георешетки. С этой целью мы разрабатываем иерархический механизм поиска по кластерам местоположений, похожий на прототипные сети (Snell et al., 2017) с фиксированными параметрами. Мы снова используем алгоритм оптической кластеризации (Ankerst et al., 1999) для кластеризации всех точек внутри георешетки g и, таким образом, предлагаем кластеры местоположений Cg, представление которых является средним значением всех соответствующих вложений изображений. Чтобы вычислить все вложения изображений, мы используем нашу предварительно обученную CLIP модель f(·), описанную в разделе 3.3, сопоставляя каждое изображение l в кластере c его вложению f(l).

Во время логического вывода мы предсказываем кластер местоположения c∗ входного изображения x, выбирая кластер с минимальным евклидовым расстоянием встраивания изображения до встраивания входного изображения f(x). Как только кластер c определен, мы дополнительно
уточняем наше предположение, выбирая единственное наилучшее местоположение внутри кластера, опять же путем минимизации евклидова расстояния встраивания. Поиск по кластерам местоположений и уточнение внутри кластера добавляют в нашу систему два дополнительных уровня иерархии прогнозирования, при этом количество уникальных потенциальных предположений равно размеру обучающего набора данных.
Хотя иерархическое уточнение с помощью поиска само по себе является новой идеей, наша работа идет на шаг дальше. Вместо уточнения прогноза геолокации в пределах одной ячейки, наш механизм оптимизируется для нескольких ячеек, что еще больше повышает производительность. Во время логического вывода наша модель классификации георешеток выводит верхние K прогнозируемых георешеток (5 для PIGEON, 40 для PIGEOTTO), а также связанные с моделью вероятности для этих ячеек. Затем модель уточнения выбирает наиболее вероятное местоположение в пределах каждой из предложенных геоячейков topK, после чего вычисляется softmax для евклидовых расстояний встраивания изображений topK. Мы используем temperature softmax с температурой, которая тщательно откалибрована на основе наборов данных проверки, чтобы сбалансировать вероятности для разных георешеток. Наконец, эти вероятности уточнения умножаются на начальные вероятности геоячейки topK для определения конечного кластера местоположений, и выполняется уточнение внутри кластера


## 7.1. Функция потерь


....
