# siamese-network-triplets
Данный проект нацелен на классификацию реальных изображений и заполнителей для определения достоверности фотографии при загрузке на различные платформы и сервисы.
# Оценка сходства изображений с использованием сиамской сети с функцией тройных потерь
**Дата создания:** 2024/05/15<br>
**Исходник:** https://keras.io/examples/vision/siamese_network/<br>
**Описание:** Обучение сиамской сети на созданном наборе данных (100 якорных картинок) с использованием ResNet50, функция потерь - Triplet Loss, метод поиска сходства - вычисление косинусного сходства <br>
**Примечание:** Используется версия keras 3.1.0, картинки в формате JPEG

## Введение

[Сиамская сеть](https://en.wikipedia.org/wiki/Siamese_neural_network) - это тип сетевой архитектуры, который содержит две или более идентичных подсети, используемых для генерации векторов признаков для каждого входного сигнала и их сравнения.

Сиамские сети могут применяться для различных целей, таких как обнаружение дубликатов, поиск аномалий и распознавание лиц.

В этом примере используется сиамская сеть с тремя идентичными подсетями. Мы предоставим модели три изображения, на которых два из них будут похожи (выборки _anchor_ и _positive_), а третий будет не связан (_negative_). Наша цель состоит в том, чтобы модель научилась оценивать сходство между изображениями на основе косинусного расстояния.

Чтобы сеть могла обучаться, мы используем функцию потери триплетов. В этом примере мы определяем функцию потерь триплета следующим образом:

`L(A, P, N) = max(‖f(A) - f(P)‖2 - ‖f(A) - f(N)‖2 + margin, 0)`


## Импорт библиотек

pip install keras==3.1.0 <br>

import matplotlib.pyplot as plt<br>
import numpy as np<br>
import os<br>
import random<br>
import tensorflow as tf<br>
import keras<br>
from pathlib import Path<br>
from keras import applications<br>
from keras import layers<br>
from keras import losses<br>
from keras import ops<br>
from keras import optimizers<br>
from keras import metrics<br>
from keras import Model<br>
from keras.applications import resnet<br>
from tensorflow.keras import regularizers<br>

import glob<br>
from PIL import Image<br>
from tensorflow.keras.preprocessing import image<br>
from tensorflow.keras.preprocessing.image import ImageDataGenerator<br>
import pathlib<br>
import shutil<br>

## Аугментация изображений
Аугментация изображений - это методика создания дополнительных данных из имеющихся данных. В данном примере мы создадим набор якорных изображений и аументируем их для получения набора позитивных изображений.

В наборе данных archive.zip находится 100 якорных картинок, сделаем для каждой картинки одну аугментированную. Чтобы картинки отличались, рассмотрим несколько способов аугментации картинок - поворот, смещение по ширине, смещение по высоте, изменение яркости изображения, сдвиг, масштабирование, отзеркаливание по горизонтали и вертикали.


## Загрузка набора данных

Будет загружен *полностью похожий* набор данных и распаковать его в каталог "~/.keras" в локальной среде.

Набор данных состоит из двух отдельных файлов:

* `left.zip содержит изображения, которые мы будем использовать в качестве якоря (*anchor*).
* `right.zip содержит изображения, которые мы будем использовать в качестве положительного образца, то есть изображения, похожие на якорь (*positive*).


## Подготовка данных

Рассмотрим несколько примеров триплетов. Обратите внимание, что первые два изображения выглядят похоже, в то время как третье всегда отличается.<br>

![image](https://github.com/user-attachments/assets/6dcc0356-299a-417f-9f20-de7817c20113)

## Настройка модели генератора вложений

Сиамская сеть будет генерировать эмбеддинги для каждого из изображений триплета. Для этого используем модель ResNet50, предварительно обученную на ImageNet, и подключим к ней несколько Dense - слоев, чтобы научиться работать с эмбеддингами триплетов.

Зафиксируем веса всех слоев модели вплоть до слоя `conv 5_block 1_out`. Это необходимо для того, чтобы не повлиять на веса, которые модель уже изучила. Таким образом оставим несколько нижних слоев способными к обучению, чтобы точно настроить их вес.

## Настройка модели сиамской сети

Сиамская сеть получит каждое из триплетных изображений в качестве входных данных, сгенерирует эмбеддинги и выведет расстояние между якорем и
положительным эмбеддингом, а также расстояние между якорем и отрицательным эмбеддигом.

Чтобы вычислить расстояние, мы можем использовать пользовательский слой `DistanceLayer`, который возвращает оба значения в виде кортежа.

## Создание модели сиамской сети

Реализуем модель с пользовательским циклом обучения, чтобы мы могли вычислить потерю триплета, используя три эмбеддинга, созданные сиамской сетью.

Создадим экземпляр `Mean` метрики для отслеживания потерь в процессе обучения, а также вычислим точность. Для этого будем сравнивать расстояние от anchor до positive и negative. Чем меньше расстояние, тем картинки считаются наиболее похожими, а, значит, для похожих картинок выполняется следующее неравенство: `an_distance >  ap_distance` (расстояние между anchor и negative больше чем между anchor и positive)

## Обучение
Обучение сети будет происходить на 30 эпохах с learning rate = 0.00001. В конце обучения выведем графики loss и accuracy, а также сохраним модель для дальнейшего использования<br>
![image](https://github.com/user-attachments/assets/6111ebe5-98c3-45d4-bf9f-c9384610cb8e)<br>
![image](https://github.com/user-attachments/assets/d7941f9c-770d-486e-bda1-84e6c2e45d9b)<br>
![image](https://github.com/user-attachments/assets/07a0a3cb-152b-481d-9541-2c4909d60dbc)<br>

## Тестирование сети
На этом этапе проверяем, как сеть научилась отличать эмбеддинги
в зависимости от того, принадлежат ли они похожим изображениям или нет.

Используем [косинусное сходство](https://en.wikipedia.org/wiki/Cosine_similarity) для измерения сходства между эмбеддингами.

Для этого выберем образец из набора данных, чтобы проверить сходство между эмбеддингами, сгенерированными для каждого изображения. Визуализируем несколько триплетов и проверим сходство для первого триплета<br>
![image](https://github.com/user-attachments/assets/cc706759-4034-480f-a928-5cd8abf89fd9)

Вычисляем косинусное сходство между якорем и позитивными изображениями и сравниваем его со сходством между якорем и негативными изображениями.

Следует ожидать, что сходство между якорем и позитивными изображениями будет
больше, чем сходство между якорем и негативными изображениями.<br>
![image](https://github.com/user-attachments/assets/55132fca-971e-430f-9ea7-f8344146f970)<br>
Для пользовательского изображения:<br>
![image](https://github.com/user-attachments/assets/166a6189-a275-4ec6-b6e8-e43aa1516692)<br>

## Создание интерфейса с помощью Gradio<br>
![test_14_no](https://github.com/user-attachments/assets/0fbdc0cb-abec-48db-83e3-af5fde14893f)<br>
![test_6_art](https://github.com/user-attachments/assets/c6913906-57da-4617-b1bc-49e2ee0d24e6)<br>

