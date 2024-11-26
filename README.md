# Proximal Policy Optimization

## Постановка задачи
С помощью алгоритма Proximal Policy Optimization нужно обучить политику, решающую следующие подзадачи

1. Подъем маятника из нижнего положения в верхнее с последующей стабилизацией 
2. Постановка конца маятника в соответствии с заданным положением в мировых координатах - далее **таргет**
3. Стабилизация маятника с произвольной массой

С имплементацией PPO проблем нет. Основная задача состоит в разработке функции награды и определении данных доступных моделей. 

## Теория
### Proximal Policy Optimization

Этот алгоритм, основанный на статье [Proximal Policy Optimization Algorithms](https://arxiv.org/pdf/1707.06347),  был реализован на основе кода к домашней работе из  [курса ШАДА](https://github.com/yandexdataschool/Practical_RL/tree/master/week09_policy_II). Также написание алгоритма опиралось на [следующие лекции](https://github.com/FortsAndMills/RL-Theory-book/blob/main/RL_Theory_Book.pdf)

### Наблюдения

Следующий момент состоит в том, как можно определить состояние.
  
Базовы  вектор наблюдений:

- положение тележки относительно таргета - $х - target$ м
- угол маятника с вертикалью - $\theta$ радиан
- угловая скорость тележки - $\frac{dx}{dt}$ м/с
- скорость конца маятника - $\frac{d\theta}{dt}$ рад/с

Расширенный вектор наблюдений:

- положение тележки - $x$ м
- угол маятника с вертикалью - $\theta$ радиан
- скорость тележки - $\frac{dx}{dt}$ м/с
- угловая скорость конца маятника - $\frac{d\theta}{dt}$ рад/с
- положение таргета - $target$ м
  
Для решения задачи с произвольной массой будем давать модели несколько последовательных наблюдений с последнего действия. Так можно дать возможность по данным движений предсказать соотношение масс тележки и маятника.

### Функция награды

Функция награды в этой задач должна влиять на:  

- размещение конца маятника около таргета  
- раскручивание в нижний точках(приложение силы и достижение нужной скорости для подъема)  
- стабилизацию при раскручивании в верхней точке

Начнем с первого пункта. В похожих окружениях агент получает награду, если удерживает объект в пределах 0.2 радиан(угол с вертикалью). Используем самое простое - косинус. Для размещения около таргета используем поощрение -  функцию от нормы расстояния от центра текущей системы координат(таргета или начала координат).

```python
relative_posistion_cart = x - target + np.sin(theta) * len_pole
```

```python
reward = np.cos(theta) + np.exp(-np.abs(relative_posistion_cart))
```

```python
reward = np.cos(theta) + np.exp(-np.abs(relative_posistion_cart)) * (relative_posistion_cart) < 0.1)
```

Для чего начали с этой модели? Для того чтобы:

-  определить подходящее поведение награды в пределах 0.2 радиан
-  выбрать L1 или L2 норму
-  выбратьпоощрение в небольшей окрестности или везде
-  сравнить поведение модели  на двух вариантах определения состояния
-  настроить гиперпараметры (количество эпох, $\lambda$)



Второй пункт немного сложнее. Мы должны поощрять скорость по направлению вверх, но тогда есть субоптимальная стратегия -  это бесконечно крутиться. C последним будем бороться тем, что при прохождении нижнего части круга суммарная награда дает неположительную величину. Проще всего будет наказывать за движение вниз, а на движение вверх не реагировать. Также чтобы агент начал пробовать двигаться добавим награду за работу мотора.
```python
reward = min(-theta**2 + 0.1 * dtheta**2 + 0.001 * a**2, 0.0)
```
В третьем пункте будем наказывать за падение в верхней части и поощрять за возвышение
```python
reward = np.cos(theta) - max(dtheta * theta, 0.0)
```

Итоговый вариант функции награды после проведения экспериментов выглядит так:

### Дополнительные параметры


Для этой и других моделей установим максимальное время жизни. Для агентов, чья задача ограничивается размещением около таргета, добавим допустимые углы маятника с вертикалью в 0.2 радиана. Агенту не нужно много времени для раскачивания и стабилизации, и нет смысла исследовать мир вдоль оси x.

Для задачи с уравновешиванием маятника с произвольной массой. Масса маятника, по умолчанию установленная в среде, менялась максимум в 10 раз(уменьшение и увеличение).

## Эксперименты

В начале обучались агенты для уравновешивания маятника и постановки конца маятника в таргет. Вариант с расширенным вектором наблюдений давал стабильнее обучение чем с базовым вектором наблюдений но сходился дольше. Количество эпох из этих экспериментов нужно было брать небольшое - 8-32. 

На видео можно сравнить поведение моделей с
[L1](https://github.com/Pikudan/PPO/blob/51f794d113a53735c0851f2a9985dbfdd922debc/video/%D1%83%D1%80%D0%B0%D0%B2%D0%BD%D0%BE%D0%B2%D0%B5%D1%88%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5_L1.mov)
и
[L2](https://github.com/Pikudan/PPO/blob/51f794d113a53735c0851f2a9985dbfdd922debc/video/%D1%83%D1%80%D0%B0%D0%B2%D0%BD%D0%BE%D0%B2%D0%B5%D1%88%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5_L2.mov)
 нормой. Можно также заметить что агент не стремиться попасть прямо в таргет, а останавливается около него. $L_1$ помогает лучше помогает достигать лучших результатов.


Также важным оказалось
[ограничение](https://github.com/Pikudan/PPO/blob/51f794d113a53735c0851f2a9985dbfdd922debc/video/%D1%83%D1%80%D0%B0%D0%B2%D0%BD%D0%BE%D0%B2%D0%B5%D1%88%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5_L1.mov)
области получения награды за близость к таргету - агент быстрее хотел попасть туда.
[Без ограничения](https://github.com/Pikudan/PPO/blob/51f794d113a53735c0851f2a9985dbfdd922debc/video/%D1%83%D1%80%D0%B0%D0%B2%D0%BD%D0%BE%D0%B2%D0%B5%D1%88%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5_L2.mov)
агент возможно хотел снизить вероятность падения и действовал осторожно. Но как было замечено далее для задачи upswing ограничение только мешало моделе понять куда надо переместить маятник.

Далее обучение стартовало уже с нижней точки c вышепоказаной моделью. Расширенный вектор наблюдений здесь уже лучше справлялся. По датасету нужно пройтись при этом несколько раз, теоретически — хочется как можно больше, но понятно, что чем больше расходится $\pi_{\theta}$ и $\pi_{old}$, тем менее эффективна «нижняя оценка» и тем больше данных будет резать наш клиппинг. Соответственно, количество эпох — сколько раз пройтись по датасету — является ключевым гиперпараметром. Занятно было видеть этого
[агента](https://github.com/Pikudan/PPO/blob/51f794d113a53735c0851f2a9985dbfdd922debc/video/%D1%83%D1%80%D0%B0%D0%B2%D0%BD%D0%BE%D0%B2%D0%B5%D1%88%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5_L2.mov)
, обученного с 8 эпохами. Он быстрее всех поднимал маятник но не следовал к таргету. Остальные сильно себя раскручивали и несколько раз крутились, но все же выполняли обе первые подзадачи, например, следующий
[агент](https://github.com/Pikudan/PPO/blob/51f794d113a53735c0851f2a9985dbfdd922debc/video/%D1%83%D1%80%D0%B0%D0%B2%D0%BD%D0%BE%D0%B2%D0%B5%D1%88%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5_L2.mov)
, обученный с 128 эпохами.


## Репозиторий

Контент репозитория
```bash
- main
- video # записи игр
- weight # записи игр
```
Контент файлов (ветка main)
```bash
- main.py
- train.py   # train
- test.py    # inference 
- envs.py    # environment
- agent.py   # PPO agent
- network.py # Policy and Value Network
- runners.py # run game in environment
- utils
  - AsArray               # Transform to ndarray
  - Policy                # Policy PPO
  - GAE                   # Generalized Advantage Estimator
  - TrajectorySampler     # Samples minibatch
  - NormalizeAdvantages   # Normalize advantages
  - make_ppo_runner       # Create runner
  - evaluate              # Play games on inference
- arguments  # Arguments parse
- writer     # Create a SummaryWriter object for logging
```

## Использование

Настройка виртуального окружения
```bash
virtualenv -p python3 venv
source venv/bin/activate
pip install -r requirements.txt
```

Для тренировки:
```bash
mjpython main.py --mode=train --upswing=True --extended_observation=True
```

Для тестирования:
```bash
mjpython main.py --mode=test --upswing=True --extended_observation=True --policy_model=policy.pth --value_model=value.ptр
```

Для исследования связки двух моделей:
```bash
mjpython main.py --mode=research--policy_model=policy.pth --value_model=value.pth
```

```bash
Options:
    --mode                    str       train, test - тренировка, тестирование
    --upswing                 bool      флаг включения подзадачи upswing (default setting = False)
    --target                  bool      флаг включения подзадачи установки конца маятника в заданных координатах (default setting = False)
    --extended_observation    bool      флаг расширения размерности наблюдений до 5 (default setting = False)
    --mass_use                bool      флаг использования модели для маятника с произвольной массой (default setting = False)
    --policy_model            str       путь до весов актера модели
    --value_model             str       путь до весов критика модели
    --num_observations        int       количество последовательных наблюдений, произошедших между действиями и переданных агенту
    --num_epochs              int       количество проходов по траектории
    --num_runner_steps        int       количество шагов в среде
    --mass                    float     масса маятника. Используется только при mode=test. По умолчанию береться из созданной среды
    --gamma                   float     gamma
    --lambda                  float     lambda
    --num_minibatches         int       размер батча траекторий
```
