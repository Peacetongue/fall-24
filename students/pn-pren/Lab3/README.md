# Отчет по лабораторной работе: Реализация алгоритмов дерева решений и их оценка

## Описание задачи
Задача состоит в реализации алгоритма построения дерева решений (ID3) с различными критериями разбиения, решении задач классификации и регрессии, редукции дерева и сравнении результатов с эталонной реализацией.

## Используемый датасет
Был выбран датасет Titanic с Kaggle:
- Содержит пропуски в признаках `Age` и `Embarked`.
- Имеет как количественные (например, `Age`, `Fare`), так и категориальные признаки (`Sex`, `Embarked`).

Обработка данных:
1. Удалены неинформативные столбцы: `PassengerId`, `Name`, `Ticket`, `Cabin`.
2. Пропуски в `Age` заменены на медианное значение, в `Embarked` — на моду.
3. Категориальные признаки преобразованы в числовые с использованием one-hot encoding.

## Реализация алгоритмов

### Алгоритм ID3

1. **Проверка глубины (max_depth)**  
   Если текущая глубина превышает max_depth, алгоритм возвращает наиболее частый класс (максимум функцией np.bincount) и его вероятности (количество каждого класса / общее количество меток).

2. **Однородность меток**  
   Если все метки одинаковы (или меньше min_samples_split), возвращаем наиболее частый класс и вероятности аналогично пункту 1.

3. **Поиск лучшего разделения**  
   Перебираем по признаку и по всем допустимым уникальным значениям (threshold), вычисляя прирост информации через энтропию (self._calculate_information_gain) или критерий Донского (self._d_criterion). Максимальное значение указывает на лучшее разделение.

```python
def _id3_classification(self, features, labels, depth, max_depth, min_samples_split):

    if max_depth is not None and depth >= max_depth:
        counts = np.bincount(labels)
        probabilities = counts / len(labels)
        return counts.argmax(), None, None, None, probabilities

    # Если все метки одинаковы -> возвращаем класс + вероятности
    unique_labels = np.unique(labels)
    if len(unique_labels) == 1:
        counts = np.bincount(labels)
        probabilities = counts / len(labels)
        return counts.argmax(), None, None, None, probabilities
    
    if len(labels) < min_samples_split:
        counts = np.bincount(labels)
        probabilities = counts / len(labels)
        return counts.argmax(), None, None, None, probabilities

    best_feature = None
    best_threshold = None
    best_info_gain = -np.inf

    for feature_index in range(features.shape[1]):
        thresholds = np.unique(features[:, feature_index])
        for threshold in thresholds:
            mask = features[:, feature_index] <= threshold

            # Пропускаем разделения, которые не делят данные
            if np.sum(mask) == 0 or np.sum(~mask) == 0:
                continue

            if self.criterion == 'entropy':
                info_gain = self._calculate_information_gain(labels, mask)
            else:
                info_gain = self._d_criterion(labels, mask)

            if info_gain > best_info_gain:
                best_info_gain = info_gain
                best_feature = feature_index
                best_threshold = threshold
```

### Редукция дерева
Реализованы алгоритмы обрезки (pruning) для двух типов деревьев: DecisionTreeClassifier и TreeRegressor. Цель обрезки заключается в сокращении сложности модели путем удаления менее значимых ветвей дерева, что помогает предотвратить переобучение и улучшить обобщающую способность модели.

#### DecisionTreeClassifier

Функция _prune_tree для DecisionTreeClassifier выполняет следующие шаги:

1. **Проверка листового узла**  
   Если текущий узел является листовым (то есть `threshold` равен `None`), возвращается сам узел без изменений.

2. **Разделение данных**  
   Валидационные данные разделяются на две подмножества на основе порогового значения текущего признака:  
   ```python
   mask = validation_features[:, feature] <= threshold
   left_val_features = validation_features[mask]
   left_val_labels = validation_labels[mask]
   right_val_features = validation_features[~mask]
   right_val_labels = validation_labels[~mask]
   ```
   Левая ветвь: данные, удовлетворяющие условию.  
   Правая ветвь: данные, не удовлетворяющие условию.  

3. **Рекурсивная обрезка поддеревьев**  
   Функция рекурсивно вызывается для левой и правой подветвей, если они существуют.

4. **Сравнение точности**  
   Вычисляется точность модели до и после обрезки:
   ```python
   original_accuracy = accuracy(tree, validation_features, validation_labels)
   pruned_accuracy = accuracy(pruned_tree, validation_features, validation_labels)
   ```

5. **Замена узла**
   Если обрезка ухудшает точность, текущий узел заменяется на листовой узел с наиболее частым классом:
   ```python
   class_label = majority_class(validation_labels)
   ```
   
#### TreeRegressor

Функция _prune_tree для TreeRegressor выполняет аналогичные шаги с некоторыми отличиями:

1. **Проверка листового узла**  
   Если текущий узел является листовым (`node[1]` равен `None`), возвращается узел без изменений.

2. **Разделение данных**  
   Данные разделяются по текущему признаку и пороговому значению:  
   ```python
   mask = validation_features[:, feat] <= thresh
   ```
   Левая ветвь: данные, удовлетворяющие условию.  
   Правая ветвь: данные, не удовлетворяющие условию.

3. **Рекурсивная обрезка поддеревьев**  
   Функция рекурсивно вызывается для левой и правой подветвей.

4. **Сравнение ошибок**  
   Вычисляются ошибки до и после обрезки:  
   ```python
   current_error = MSE(node, X_val, y_val)
   pruned_error = MSE(pruned_node, X_val, y_val)
   ```
   Если ошибка после обрезки не увеличивается, дерево обрезается.

5. **Замена узла**  
   Если обрезка ухудшает ошибку, узел заменяется на листовой узел со средним значением целевой переменной:  
   ```python
   mean_value = sum(y_i) / N
   ```

### Метрики
1. DecisionTreeClassifier  
   Собственная реализация  
   - критерий - Энтропия:
      ```bash
      Время обучения: 0.3640 секунд
      Время предсказания: 0.0027 секунд
      Точность модели до обрезки: 75.42%
      Время обрезки дерева: 0.0290 секунд
      Время предсказания: 0.0027 секунд
      Точность модели после обрезки: 75.98%
      ```
   - критерий Донского:
      ```bash
      Время обучения: 0.4178 секунд
      Время предсказания: 0.0037 секунд
      Точность модели до обрезки: 75.42%
      Время обрезки дерева: 0.0428 секунд
      Время предсказания: 0.0039 секунд
      Точность модели после обрезки: 75.98%
      ```
   Sklearn реализация (критейрий Донского не используется в стандартных реализациях DecisionTreeClassifier из sklearn, для сравнения был взят дефолтный критерий Gini):
   ```bash
   Время обучения модели Gini: 0.0018 секунд
   Время обучения модели Entropy: 0.0018 секунд
   Точность модели Gini до обрезки: 76.54%
   Время предсказания модели Gini: 0.0004 секунд
   Точность модели Entropy до обрезки: 77.65%
   Время предсказания модели Entropy: 0.0002 секунд
   ```
   Модели с критерием Донского и Энтропии в собственной реализации показывают схожие результаты (после обрезки точность возрастает до 75.98%). При этом sklearn с Gini и Entropy обучается быстрее (около 0.0018 с) и даёт более высокую точность (до 77.65%).

2. TreeRegressor  
   Собственная реализация:
      ```bash
      MSE до обрезки: 2112.1791806705496
      MSE после обрезки: 2112.1791806705496
      Замеры времени выполнения:
      Обучение модели: 0.3450 секунд
      Предсказание до обрезки: 0.0030 секунд
      Обрезка дерева: 0.4034 секунд
      Предсказание после обрезки: 0.0000 секунд
      ```
   Sklearn реализация:
   ```bash
   Время обучения: 0.0020 секунд
   Время предсказания: 0.0000 секунд
   MSE: 2469.4859128214207
   ```
   У кастомной реализации TreeRegressor более низкое значение MSE по сравнению с реализацией sklearn, что указывает на лучшую точность   модели. Однако, время обучения и обрезки дерева существенно дольше. Sklearn предоставляет значительно более быструю обработку, но с несколько худшими показателями точности.