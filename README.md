# Проект: Алгоритм автоматической застройки участка (urban_0.2.ipynb)

**Описание:**

Этот проект представляет собой реализацию алгоритма автоматического распределения территорий на заданном участке с учетом градостроительных ограничений. Алгоритм принимает на вход геоданные участка и ограничения, а на выходе генерирует план застройки с размещением различных объектов (жилые дома, коммерческие здания, парки, дороги) и файл GeoJSON с координатами этих объектов.

**Входные данные (input.geojson):**

*   **Границы участка:** Задаются в формате GeoJSON (FeatureCollection с Polygon).
*   **Ограничения по плотности застройки:** Процент площади участка, доступный для застройки каждым типом объектов (жилые, коммерческие, парки).
*   **Минимальное расстояние между объектами:** Задается в метрах для каждого типа объектов.
*   **Дополнительные ограничения:** Зоны, на которых нельзя строить (леса, водоемы, существующие дороги), также в формате GeoJSON.

**Выходные данные:**

*   **Файл GeoJSON (output.geojson):** Содержит информацию о расположении сгенерированных объектов (здания, дороги, парки) в формате GeoJSON (FeatureCollection).  Каждый объект имеет тип (`type`), `id` и геометрию (Polygon для зданий/парков, LineString для дорог).
*   **Изображение плана застройки (output.png):** Визуализация сгенерированного плана застройки с цветовым кодированием разных типов объектов.

**Используемые библиотеки:**

*   **geopandas:** Для работы с геоданными (чтение, запись, манипуляции).
*   **shapely:** Для работы с геометрическими объектами (создание, буферизация, пересечения, объединения).
*   **matplotlib:** Для визуализации плана застройки.
*   **numpy:** Для численных операций (работа с массивами координат).
*   **pyproj:** Для преобразования координат между системами координат (WGS 84 и UTM).
*   **networkx:** Для построения графа дорожной сети (минимальное остовное дерево).
*   **sklearn.neighbors:** Для построения k-NN графа (для генерации дорог).
*   **json:** Для записи выходных данных в формате GeoJSON.
*   **itertools:** combinations (для перебора пар)

**Алгоритм:**

1.  **Загрузка и предобработка данных:**
    *   Загрузка входных данных из файла `input.geojson` с помощью `geopandas`.
    *   Проверка геометрий на корректность с помощью `shapely.validation.make_valid`.
    *   Извлечение зоны застройки (`buildable_area`), зон с ограничениями (`restricted_areas`) и главной дороги (`main_road`).
    *   Преобразование координат из WGS 84 в UTM с помощью `pyproj`.

2.  **Создание сетки:**
    *   Создание регулярной сетки точек внутри зоны застройки. Шаг сетки определяется максимальным размером здания и максимальным минимальным расстоянием между объектами.
    *   Фильтрация сетки: удаление точек, попадающих в зоны с ограничениями или находящихся слишком близко к границе зоны застройки.

3.  **Размещение дорог:**
    *   **Главная дорога:** Буферизуется с заданной шириной.
    *   **Второстепенные дороги:**
        *   Строится k-NN граф на прореженной сетке точек (и точках главной дороги), с добавлением случайного "шума" к координатам.
        *   Находится минимальное остовное дерево (MST) на этом графе.
        *   Тупиковые вершины MST соединяются с ближайшими ребрами исходного k-NN графа.
    *   **Перекрестки:**  Находятся точки пересечения второстепенных дорог. Близко расположенные точки объединяются, и для каждой группы создается кольцевой перекресток (буфер вокруг центроида группы).
    *   **Локальные дороги:** Для каждого размещенного объекта находится ближайшая точка на второстепенных дорогах/главной дороге, и строится отрезок, соединяющий объект с этой точкой.  Проверяется, чтобы локальная дорога не пересекала другие здания.

4.  **Размещение объектов (зданий и парков):**
    *   Жадный алгоритм: точки сетки перебираются в случайном порядке.
    *   Для каждого типа объектов (жилые, коммерческие, парки):
        *   Вычисляется доступная площадь для застройки данным типом объектов (с учетом заданной плотности).
        *   Для каждой точки сетки:
            *   Создается геометрия объекта (прямоугольник).
            *   Проверяется:
                *   Входит ли объект в зону застройки (с учетом отступа от краев).
                *   Не пересекается ли объект с зонами с ограничениями.
                *   Не пересекается ли объект с другими уже размещенными объектами (с учетом минимального расстояния).
                *   Не пересекается ли объект с дорогами.
            *   Если все проверки пройдены, объект размещается, и доступная площадь уменьшается.
            *   Размещение объектов данного типа прекращается, когда достигнут предел плотности застройки.

5.  **Генерация GeoJSON:** Создание GeoJSON-файла с информацией о размещенных объектах (здания, дороги, перекрестки).

6.  **Визуализация:** Отрисовка плана застройки с помощью `matplotlib`:
    *   Зона застройки (светло-серая).
    *   Зоны с ограничениями (серые).
    *   Здания (красные, синие, зелёные – в зависимости от типа).
    *   Главная дорога (черная, буферизованная).
    *   Второстепенные дороги (синие).
    *   Локальные дороги (серые).
    *   Перекрестки (зеленые).
    *   Буферизованные дороги (красным, полупрозрачным).
    *   Легенда.

**Структура кода (функции):**

*   `to_utm`, `from_utm`: Преобразование координат между WGS 84 и UTM.
*   `calculate_degree_distance`: Вычисление эквивалента расстояния в метрах в градусах.
*   `load_and_validate_data`: Загрузка данных и проверка геометрий.
*   `get_buildable_area`, `get_restricted_areas`, `get_main_road`: Извлечение геометрий из данных.
*   `buffer_road`: Создание буфера для дороги.
*   `create_intersection`: Создание кольцевого перекрестка.
*   `create_grid`: Создание сетки точек.
*   `filter_grid`: Фильтрация сетки.
*   `generate_secondary_roads`: Генерация второстепенных дорог (k-NN + MST).
*   `connect_dead_ends`: Соединение тупиков второстепенных дорог.
*   `generate_intersections`: Генерация перекрестков (с объединением близких).
*   `generate_local_roads`: Генерация локальных дорог.
*   `place_objects`: Размещение объектов (жадный алгоритм).
*   `generate_geojson`: Генерация выходного GeoJSON.
*   `visualize_results`: Визуализация результатов.

**Запуск:**

Код запускается в Jupyter Notebook (`urban_0.2.ipynb`).  Входной файл `input.geojson` должен находиться в папке `./data/`.  Результаты сохраняются в файлы `output.geojson` и `output.png`.
