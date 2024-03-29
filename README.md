# **Реализация нейросети PIGEON для определения местоположение объектов на фотографиях**

<img src="./images/pigeon.png" width="800" />


Аспиранты Стэнфорда разработали нейросеть Predicting Image Geolocations (или сокращенно PIGEON), которая способна определять местоположение по фотографии. Точность работы PIGEON составляет 40 км, и она правильно называет страну в 95% случаев.

Попробуем повторить успех коллег. 

Основные источники данных

https://habr.com/ru/news/783090/

https://www.catalyzex.com/paper/arxiv:2307.05845

https://arxiv.org/abs/2307.05845

# 1. Формирование датасета

#### Общий подход к созданию датасета

Изображения будут загружаться из сервиса Google maps. РАбота будет разделена на несколько шагов:
- формирования списка координат для каждой геозоны (города)
- получение описательных данных для каждой координаты
- загрузка на основе описательных данных изображений понорам
- нарезка панорамы на 4 отдельных изображения

Далее изображения кластеризуем по геоячекам с помощью алгоритма OPTICS

Улучшаем кластеризацию с помощью тесселяции Вороного


## 1.1. Загрузка изображений

### 1.1.1 Получение координат и лейблов изображений

Вероятно быстрый способ загрузить понорами из Google maps отсутствует. Google предоставляет API для загрузки фото, но только по уникальному наименованию, допустим "MI6AdiEx00Ses3qzyKiXeg" для координат lat 53.29146481360133; lon 83.63833204994756.

Поэтому изначально мы составляем список координат для выбранной геозоны. Было реализовано два подхода:
- получение координат с определенным шагом в пределах заданного прямоугольника
- получение координат с определенным шагом в пределах заданного ромба

Прямоугольник позволяет нам извлечь большее количество координат, ромб позволяет извлечь основные координаты прилегающие к центру геозоны (города). Лучшим решением окажется ромб, так как плотность понорам в гул очень высока, и даже с ромбом их количество окажется избыточным.

Когда координаты получены возможно приступить к получению описаний (лейблов) изображений. Полученные данные будут записываться в csv файлы для дальнейшего объединения.

Использовать будем вот этот сервис - "streetview" https://github.com/robolyst/streetview. 


### **На примере Барнаула.**

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


Здесь представлен ноутбук с реализацией загрузки координат и лейблов изображений https://colab.research.google.com/drive/1BKJvkXlxelojyEPH_ZjippBw4HlrViX6?usp=sharing

Здесь представлен файл с загруженными лейблами https://disk.yandex.ru/d/DCx14ELm4Lg7Mg  Барнаула

### 1.1.2 Получение изображений

Получение изображений относительно простой но затратный по времени процесс. В данном случае также будет использован сервис "streetview" https://github.com/robolyst/streetview.  Было реализовано два варианта загрузки:
- изображения высокого разрешения 16300*8200 - https://colab.research.google.com/drive/1KAsBBn4Bf4ZxjmjQniubULUELyC1Rqeo?usp=sharing
- изображения низкого разрешения - https://colab.research.google.com/drive/1yG5Ja4PBmQVnT1ppAaOs0SxEO3aLBk9p?usp=sharing

Значительно ускорить получилось благодаря статье https://web.archive.org/web/20100818173558/http://jamiethompson.co.uk/web/2010/05/15/google-streetview-static-api/


Среднее время загрузки одного файла высокого разрешения- 1-3 мин. Для нашей задачи это неоправданно долго. Загрузка 900 000 изображений одним процессом заняла бы до 30 дней. Плюс изображения все равно бы уменьшались в процессе обучения. Поэтому датасет формировался из изображений низкого разрешения
Время загрузки панорам низкого разрешения составило порядка 7-8 часов для одного города. В дальнейшем Возникла идея, что такое количество изображений (от  400 000 до 600 000) для одного города избыточно, возможно вполне сократить количество картинок путем увеличения шага при подготовке координат на первом шаге.
Да и для загрузи изображений также в терминале было одновременно запущено 15 скриптов.


Когда изображения получены мы можем их нарезать
В ноутбуке представлен алгоритм https://colab.research.google.com/drive/1vegzIsvROEeCDyETkuCSxTfxXLYu8TMW?usp=sharing
<img src="./images/img_slice.PNG" width="600" />

Зеленый - фронт, белый - обратная сторона, красный - боковые стороны. Тонкие линии - центр среза


## 1.2. Формирование описания файлов

## 1.3. Поиск геоячеек

Пример ноутбука для получения геоячеек - классов https://colab.research.google.com/drive/1iEpay-tJHxELrxGen4_y9ktgtzq82R1i?usp=sharing

Наша задача весь объем полученных изображений относительно равномерно разделить на классы
<img src="./images/map_cluster.png" width="600" />

Задача будет решена в два этапа

- изначально с помощью алгоритма OPTICS мы извлекаем базовые кластеры. В полученных кластерах рассчитываем центры масс. OPTICS оставляет много шума - не класстеризованых точек.

  Соотношение точек кластеры/шум в городах Барнаул и Белгород
  <img src="./images/voronoi_step.png" width="600" />

  Получившиеся кластеры в городах Барнаул и Белгород
  <img src="./images/optic_claster.png" width="600" />
  
- передаем центры масс в алгоритм Вороного. Получаем новые замкнутые зоны на карте. Теперь относим точке на карте к соответствующему классу исхдя из полученных зон.

##
| <img src="./images/voronoi.png" width="200" /> | <img src="./images/foo.png" width="200" /> | <img src="./images/voronoi_cluster.png" width="200" /> |

##


## 

ГОТОВЫЙ ДАТАСЕТ - https://drive.google.com/file/d/1zB-L7LQgjfKftRoOOv4La_XaIyzzrXlQ/view?usp=drive_link

Наименгование колнки  | Описание значений в колонке
------------- | -------------
pano_id_crop  | Наименование иображения в составе датасета без расширения
pano_id  | ID панорамного изображения в GOOGLE MAPS
mark | Направление изображения относительно движения
city | Город в котором находится изображение
lat | Широта
lon | Долгота
heading | Азимут изображения
pitch | 
roll | 
date | Дата фотографии ( не всегда есть)
cluster | ID кластера изображения
kernel_lat | Широта центра кластера
kernel_lon | Долгота центра кластера



# 2. Модель

Реализация модели предсказания заключается в добавлении дополнительного линейного слоя к image encoder модели CLIP от Open AI. 
```python
!pip install git+https://github.com/openai/CLIP.git
import clip

model, preprocess = clip.load("ViT-B/32", device=device)
# Замораживаем слои CLIP
for child in model.children():
    for param in child.parameters():
        param.requires_grad = False
```
```python

class GeoModel(nn.Module):
    def __init__(self, n_classes):     
        super().__init__()    
        self.input = model.encode_image
        self.layer_1 = nn.Sequential(
            nn.Linear(512, 128),
            nn.ReLU(),
        )
        self.layer_2 = nn.Sequential(
            nn.Linear(128, 640),
            nn.ReLU(),
        )
        self.out = nn.Sequential(
            nn.Linear(640, n_classes),
            #nn.Softmax(),
        )

    def forward(self, x):
        x = self.input(x)
        x = x.to(torch.float32)
        x = self.layer_1(x)
        x = self.layer_2(x)
        x = self.out(x)
        return x

```
Помимо линейного слоя Классификации geocells, в оригиналной статье говориться о возможности реализации иерархической классификации (Страна -> Регион -> Город -> geocell). Данный этап пока в разработке. Нет общего понимания как это сделать.

## Loss Функция

Также необходимо подготовить свою функцию потерь. В оригинальной статье разработана следующая реализация:

 <img src="./images/loss_1.png" width="500" />  <img src="./images/loss_2.png" width="500" /> 

Суть заключается в следующем. Сначала расчитываем расстояние между двумя географическими  точками (haversine distance). 

Вторая особенность в том, что для стабилизации отклонений беруться два сравнения:
- между предсказанной точкой и реальной координатой
- между предсказанной точкой и реальным центром  геоячейки 
Эти значения мы обертываем в формулу с экспонентой. Делитель 65 в статье описан, как полученный экспериментальным путём. 

Последним шагом включаем логарифм вероятности каждого класса, умножаем его на результат расчёта отклонений и полученная сумма будет значением loss функции. 




```python
def hav_dist(points_1: torch.Tensor, points_2: torch.Tensor, earth_radius: int = 6371) -> torch.Tensor:
    """"Функция для расчета расстояния между точками в км."""  
    lat_1, lon_1 = points_1.T * math.pi / 180.0  # радианы
    lat_2, lon_2 = points_2.T * math.pi / 180.0  # радианы
    lat_delt = ((lat_2 - lat_1) / 2).sin() ** 2
    lon_delt = ((lon_2 - lon_1) / 2).sin() ** 2    
    return 2 * earth_radius * (lat_delt + lat_1.cos() * lat_2.cos() * lon_delt).sqrt().asin()


def new_loss(output: torch.Tensor, labels: torch.Tensor, labels_classes: torch.Tensor):
    tens_slice = 2
    true_points = labels[:, :tens_slice]
    true_kernels = labels[:, tens_slice:]

    probability = output.sigmoid()
    lbl_idx = probability.argmax(axis=1)
    prediction = labels_classes[lbl_idx,:]    
 
    point_distance = hav_dist(prediction, true_points)
    kernel_distance = hav_dist(prediction, true_kernels)
    distance = - ((kernel_distance - point_distance) / 65)
    distance = torch.exp(distance)

    loss = - (probability.log() * distance.unsqueeze(1)).sum()
    return loss
```

## Функции определения качества, точности

В оригинальной статье для определения точности и качества модели авторы используют показатель разности расстояния между предсказанной точкой и реальной. Попробуем реализовать. 

```python

def acc_foo(output: torch.Tensor, labels: torch.Tensor, labels_classes: torch.Tensor):
    tens_slice = 2
    true_points = labels[:, :tens_slice]
    true_kernels = labels[:, tens_slice:]

    probability = output.sigmoid()
    lbl_idx = probability.argmax(axis=1)
    prediction = labels_classes[lbl_idx,:]

    #point_distance = hav_dist(true_points, prediction)
    #kernel_distance = hav_dist(true_kernels, prediction)
    point_distance = hav_dist(prediction, true_points)
    return point_distance
```

## Мой custom dataloder

Наш датасет не стандартный. Поэтому реализуем свой dataloder

```python

class GeoImageDataset(Dataset):
    def __init__(
            self,
            annotations_file: Path,
            img_dir: Path,
            transform=None,
            target_transform=None,
            img_idx=0,
            labels_idx=0,  # индекс лейблов
            text_idx=1,  # индекс текста
        ):
        self.img_labels = pd.read_csv(annotations_file, sep=';')
        self.img_labels = self.img_labels[self.img_labels.cluster != -1]
        self.img_dir = img_dir
        self.transform = transform
        self.target_transform = target_transform
        self.img_idx = img_idx
        self.labels_idx = labels_idx
        self.text_idx = text_idx

    def __len__(self):
        return len(self.img_labels)

    def __getitem__(self, idx):
        img_path = self.img_dir / f'{self.img_labels.iloc[idx, self.img_idx]}.jpg'
        image = Image.open(img_path)
        label = self.img_labels.iloc[idx, self.labels_idx]
        text = self.img_labels.iloc[idx, self.text_idx]
        text = clip.tokenize(text)
        if self.transform:
            image = self.transform(image)
        if self.target_transform:
            label = self.target_transform(label)
        return image, text, label


def transform_label(labels: pd.Series):
    labels = labels.values.astype(np.float32)
    return torch.Tensor(labels)



dataset = GeoImageDataset(
    ds_info_file,  #
    images_data,
    preprocess,
    target_transform=transform_label,
    labels_idx=[4,5,11,12],
)

batch_size = 2500
train_dataset, val_dataset = torch.utils.data.random_split(dataset, [0.8, 0.2])
train_loader = torch.utils.data.DataLoader(train_dataset, batch_size=batch_size, shuffle=True)
val_loader = torch.utils.data.DataLoader(val_dataset, batch_size=batch_size, shuffle=True)
```


## 2.2. Обучение модели
Ноутбук обучения модели - https://colab.research.google.com/drive/1P5U1bTBNW7W3a6Udq_iU7Qfz0KrmDLnY?usp=sharing

 
 <img src="./images/train.png" width="600" /> 

 Результат обучния пока озадачивает. Обучение производится в 1 эпоху до 50 батча идет уверенный рост качества модели и снижения показателя лос, затем с 50 по 100 батч лосс функция выдает запредельные значения, качество останавливается и после 100 батча качество катастрофически обрушается. При этом значение лосс функции приближается к 0.Причина мне пока не ясна - либо кривая лосс функция, либо кривая моя модель. 
#

В настоящий момент продолжаю эксперименты:
- подключил scheduler - пытаюсь использовать разные варианты
- пробую разные варианты архитеткуры модели
- вероятно сильное влияние окажет изменение структуры датасета. На сегодня в датасете только два города и оба из России. Расстояние между ними порядка 3000 км. Необходимо дальше расширить датасет. Наполнить новыми городами и странами.

Т.к. colab ограничивает время и ресурсы, пока не получилось найти рабочую конфигурацию

#
После корректировки модели и добавления scheduler (scheduler меняет lr после каждого нового batch) получилось немного улулчшить результаты обучения. loss меняется более плавно, деградация качества также пропало
 <img src="./images/train_2.png" width="600" /> 
```python
class GeoModel(nn.Module):
    def __init__(self, n_classes):
        super().__init__()
        self.input = model.encode_image
        self.layer_1 = nn.Sequential(
            nn.Linear(512, 1000),
            nn.ReLU(),
        )
        self.layer_2 = nn.Sequential(
            nn.Linear(1000, 1500),
            nn.ReLU(),
        )
        self.out = nn.Sequential(
            nn.Linear(1500, n_classes),
            #nn.Softmax(),
        )

    def forward(self, x):
        x = self.input(x)
        x = x.to(torch.float32)
        x = self.layer_1(x)
        x = self.layer_2(x)
        x = self.out(x)
        return x


geo_model = GeoModel(n_classes=labels_classes.shape[0]).to(device)
loss_fn = new_loss
learning_rate = 1e-4
optimizer = torch.optim.Adam(geo_model.parameters(), lr=learning_rate)
scheduler = lr_scheduler.StepLR(optimizer, step_size=20, gamma=0.5)

```
#
