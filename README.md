# Предсказание оттока клиентов в пекарне.  
  
## Описание проекта
Закзачик: Крупная сеть кофеен-пекарен.  
Задача: предсказать, на основании данных, что клиент уйдет в отток.

Описание данных:  
Таблица `data`:  
- clnt_ID - уникальный айди юзера, str   
- timestamp - дата и время совершения покупки, datetime
- gest_Sum - сумма покупки, float
- gest_Discount - сумма скидки, float
 
Таблица `target`
- clnt_ID - уникальный айди юзера, str
- target - флаг оттока, int: 1 если юзер ушел в отток | 0 если НЕ отток


## Алгоритм решения: 
1. **Предобработка данных**
- Проверка данных
- Создание новых признаков

2. **Исследовательский анализ данных**
В ходе проведенного анализа, удалось заметить, что:  
- В среднем, траты клиентов из "не-отток" в рамках недели, месяца и года выше, нежели траты "ушедших" клиентов.  При этом в рамках 1-го дня клиенты "не-оттока" в среднем тратят немногим меньше, нежели клиенты из "оттока".  
- Размер скидки у "не-ушедших" гостей выше, чем у гостей "ушедших". То же справдливо и в отношении доли покупок со скидкой в общем количестве покупок за месяц.  
- "Не-ушедшие" клиенты в среднем посещают кофейню чаще и с меньшими интревалами, нежели клиенты, которые ушли в "отток".

3. **Оценка обобщающей способности алгоритмов машинного обучения с настройками по умолчанию.**
- Подготовка данных для алгоритмов машинного обучения и создание необходимых функций предобработки данных¶
- Оценка работы Random Forest / Random Forest with polynomial features (intersections only)
- Оценка работы K-Nearest Neighbors
- Оценка работы Logistic Regression / Logistic Regression with polynomial features
- Оценка работы Support Vector Machine / Support Vector Machine with polynomial features
- Оценка работы Gradient Boosting / Gradient Boosting with polynomial features (intersections only)  
  
Оценка производилась с помощью техники кросс-валидации LeaveOneGroupOut. Результаты работы моделей представлены в таблице ниже:  

|                                     |   ROC-AUC |   F1 |   Accuracy |   Precision |   Recall |
|:------------------------------------|----------:|-----:|-----------:|------------:|---------:|
| Random Forest                       |      68.5 | 66.7 |       63.8 |        65.5 |     68   |
| Random Forest with Polynom          |      68.2 | 66.2 |       63.4 |        65.3 |     67.1 |
| KNN                                 |      64.6 | 64.4 |       61.5 |        63.6 |     65.3 |
| Logistic Regression                 |      72.2 | 70.2 |       67   |        67.8 |     72.7 |
| Logistic_Regression with Polynom    |      72.5 | 70.6 |       67.3 |        67.8 |     73.6 |
| SVM Classifier                      |      71.1 | 71.1 |       66.9 |        66.5 |     76.4 |
| SVM Classifier with Polynom         |      71.3 | 70.3 |       67.1 |        67.8 |     72.9 |
| CatBoost (max roc_auc)              |      71.3 | 68.5 |       66   |        67.8 |     69.3 |
| CatBoost (max f1)                   |      71.3 | 68.5 |       66   |        67.8 |     69.3 |
| CatBoost with Polynom (max roc_auc) |      70.9 | 67.6 |       65.4 |        67.6 |     67.7 |

Для дальнейшей работы была выбрана модель логистической регрессии. 

4. **Оптимизация гиперпараметров.**  
На данном этапе для модели логистической регрессии подбиралось наиболее оптимальное сочетание гиперапараметров `C` и `l1_ratio`. Оптимизация гиперпараметров проводилась с использованием библиотеки HyperOpt.  
Наиболее оптимальными гиперпараметрами оказались С=0.001, l1_ratio=0.9

5. **Калибровка вероятностей.**  
На данном этапе для получившегося классификатора была произведена калибровка вероятностей.  

6. **Проверка откалиброванного и неоткалиброванного классификатора на тестовой выборке.**  
На данном этапе была произведена оценка обобщающей способности алгоритмов на тестовой выборке. Получившиеся результаты представлены ниже:   

|                                |   ROC-AUC |   F1 |   Accuracy |   Precision |   Recall |
|:-------------------------------|----------:|-----:|-----------:|------------:|---------:|
| Logistic_Regression            |      82.6 | 21.5 |       60.5 |        12.2 |     88.8 |
| Logistic_Regression calibrated |      82.6 | 22.4 |       63.5 |        12.8 |     86.3 |
| Dummy-estimator                |      49.8 | 10.8 |       49.9 |         6   |     49.7 |

С одной стороны, есть причины предполагать, что наша модель плохо выявила закономерности в данных. Однако на кросс-валидации показатель F1 был близок к 70%, и точность моделей в два раза выше нежели у константного алгоритма. Более того, тот факт, что с приближением к концу года баланс классов в данных очень сильно меняется заставляет задаться вопросом, действительно ли все данные были выгружены и размечены корректно, нет ли каких-то аномалий, и в чем вообще может быть причина такого дисбаланса.   
  
Если же все размечено корректно, то одной из причин, почему поведение "не-отточных" гостей для модели становится похожим на поведение "отточных" также может быть тот факт, что 2020 год - это год, когда действовали ковидные ограничения на общепит. Это могло отразиться на частоте посещение кафе даже со стороны "не-отточных" клиентов (следует принять во внимание, что одни из наиболее важных признаков у нас связаны как раз таки с количеством посещений в месяц, а также с количеством дней, которое прошло от последнего посещения клиента до 31 числа месяца).  
Возможно, отчасти поэтому на данных за октябрь модель показала такой результат.  

7. **Интрерпретация модели.** 
В результате интерпретации коэффициентов логистической регрессии было выявлено, что чем больше "дней от последнего посещения гостя до конца месяца", а также чем меньше "посещений в течение месяца" совершил клиент, тем выше вероятность того, что он уйдет в отток. При этом, наиболее весомым признаком является "количество посещений за месяц".