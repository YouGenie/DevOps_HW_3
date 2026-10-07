# Домашнее задание 3: мониторинг ВМ и Docker-образ для бэкенда

# СДАВАТЬ СЮДА

[СЮДА](https://forms.gle/yj6UX2qztFbhM2aE6)

О том как сдавать написал [тут](../ABOUT.md)

> [!IMPORTANT]
> В этом ДЗ сдаётся **ссылка на репозиторий на GitHub** с результатами обоих заданий **и логи выполнения** в одном из форматов из [ABOUT.md](../ABOUT.md) (предпочтительно md или ipynb). Логи можно положить в тот же репозиторий, например в `README.md`.

Домашка состоит из двух заданий по **5 баллов** каждое.


| №   | Задание                                        | Баллы |
| --- | ---------------------------------------------- | ----- |
| 1   | Node Exporter → Prometheus → дашборд в Grafana | 5     |
| 2   | Три варианта Dockerfile для `counter-backend`  | 5     |


---



## Задание 1. Мониторинг ВМ

Стенд поднимается локально в Docker. Всё нужное лежит в [monitoring](./monitoring):

```
monitoring/
├── compose.yaml               # Prometheus, Node Exporter, Grafana
└── prometheus/
    └── prometheus.yml         # сюда нужно добавить задание для node_exporter
```

Это урезанная версия стенда из лекции: без Alertmanager, Blackbox и PostgreSQL.

```bash
cd monitoring
docker compose up -d
docker compose ps
```


| Сервис        | Адрес с хоста                                                    |
| ------------- | ---------------------------------------------------------------- |
| Prometheus    | [http://localhost:9090](http://localhost:9090)                   |
| Node Exporter | [http://localhost:9100/metrics](http://localhost:9100/metrics)   |
| Grafana       | [http://localhost:3000](http://localhost:3000) (`admin`/`admin`) |




### Что нужно сделать


| №   | Шаг                       | Требования                                                                                                                                                                             | Баллы |
| --- | ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----- |
| 1   | Node Exporter             | запущен, `curl localhost:9100/metrics` отдаёт метрики `node_*`                                                                                                                         | —     |
| 2   | Подключение к Prometheus  | в `prometheus.yml` есть задание `job_name: node`, у цели понятная метка `instance` (`vm1`, …); на [http://localhost:9090/targets](http://localhost:9090/targets) цель в состоянии `UP` | 1     |
| 3   | Источник данных в Grafana | Prometheus добавлен в **Connections → Data sources**, **Save & test** проходит успешно                                                                                                 | 1     |
| 4   | Свой дашборд              | **созданный вами**, а не импортированный: четыре панели, по 0,5 балла за каждую (см. таблицу ниже)                                                                                     | 2     |
| 5   | Переменная и экспорт      | на дашборде есть переменная `instance` (выбор ВМ), JSON дашборда выгружен (**Export → Export as JSON**) и лежит в репозитории                                                          | 1     |


Панели дашборда:


| Панель | Что показывает                          | Единицы          |
| ------ | --------------------------------------- | ---------------- |
| CPU    | загрузка процессора (всё, кроме `idle`) | %                |
| RAM    | занятая (или свободная) память          | байты или %      |
| Диск   | свободное место на файловых системах    | байты или %      |
| Сеть   | входящий и исходящий трафик             | бит/с или байт/с |


Все запросы панелей должны фильтроваться по переменной: `{instance=~"$instance"}`.

> [!NOTE]
> Если у вас остались ВМ с прошлых занятий, поставьте node_exporter и на них (например ролью `prometheus.prometheus.node_exporter` из лекции) и добавьте их целями в то же задание `node`. Тогда переменная `instance` станет по-настоящему полезной. Это не обязательно и на баллы не влияет. Если ВМ в облаке, не открывайте порт 9100 всему интернету.

---



## Задание 2. Docker-образ для counter-backend

Проект лежит в [counter-backend/](./counter-backend). Это HTTP-сервер на aiohttp, пакет собирается Poetry, код — в `src/counter_backend`. Сервер слушает порт из переменной `PORT` (по умолчанию `8080`):


| Метод  | Путь           | Ответ                |
| ------ | -------------- | -------------------- |
| `GET`  | `/api/counter` | `200 {"count": N}`   |
| `POST` | `/api/counter` | `201 {"count": N+1}` |


Базовый образ: `python:3.11.6-slim-bookworm`.

### Команды сборки

Установка зависимостей для сборки пакета:

```bash
python -m pip install \
  --no-color \
  --no-cache-dir \
  --disable-pip-version-check \
  --no-python-version-warning \
  --no-warn-script-location \
  --break-system-packages \
  --progress-bar off \
  poetry setuptools wheel
```

Загрузка всех зависимостей в папку `dist/vendor`:

```bash
mkdir -p dist
poetry export \
  --without-hashes \
  --format constraints.txt \
  --output dist/constraints.txt
poetry run \
  python -m pip wheel \
   --isolated \
   --requirement dist/constraints.txt \
   --wheel-dir dist/vendor
```

Пакетирование проекта в папку `dist`:

```bash
poetry build --format wheel
```

Установка всех зависимостей:

```bash
packages=$(\
  find 'dist' 'dist/vendor' \
    -maxdepth 1 \
    -iname '*.whl' \
    -exec realpath {} \; \
    -print0 \
   | xargs --null)
python -m pip install \
  --isolated \
  --no-index \
  --no-color \
  --no-cache-dir \
  --disable-pip-version-check \
  --no-python-version-warning \
  --no-warn-script-location \
  --no-deps \
  --break-system-packages \
  --progress-bar off \
  ${packages}
```

Эти команды находят все пакеты Python, собранные предыдущими командами, и устанавливают их.

Запуск модуля:

```bash
python -m counter_backend
```



### Что нужно сделать

Напишите три варианта Dockerfile:


| №   | Вариант                    | Требования                                                                                                                                                                                                                    | Баллы |
| --- | -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----- |
| 1   | `Dockerfile.single`        | **без** многоступенчатой сборки, процесс запускается от root                                                                                                                                                                  | 1,5   |
| 2   | `Dockerfile.multistage`    | многоступенчатая сборка: в стадии сборки выполнить первые три набора команд (установка Poetry, `export` + `wheel`, `build`), в финальную стадию скопировать **только** `dist` и установить пакеты там                         | 1,5   |
| 3   | `Dockerfile.nonroot`       | как вариант 2, плюс понижение привилегий: пользователь создаётся скриптом `addgroup`/`adduser` из раздела [«Понижение привилегий»](../../seminars/Seminar3.md#понижение-привилегий) семинара, процесс работает **не** от root | 1,5   |
| 4   | Сравнение размеров образов | таблица размеров трёх образов                                                                                                                                                                                                 | 0,5   |


> [!IMPORTANT]  
> **После сборки каждого образа** запустите из него контейнер и проверьте, что сервис доступен: `GET /api/counter` отвечает `{"count": 0}`, `POST /api/counter` увеличивает счётчик, а `docker exec ... id` показывает, от какого пользователя работает процесс. Вариант без такой проверки в логах оценивается в 0 баллов, даже если Dockerfile верный.

---



## Что сдать

Ссылку на репозиторий на GitHub примерно такой структуры:

```
.
├── monitoring/
│   ├── prometheus/
│   │   └── prometheus.yml       # с вашим заданием node
│   └── grafana/
│       └── dashboard.json       # экспорт вашего дашборда
├── counter-backend/             # исходники проекта из этой папки
│   ├── Dockerfile.single
│   ├── Dockerfile.multistage
│   ├── Dockerfile.nonroot
│   ├── .dockerignore
│   └── ...
├── screenshots/                 # /targets, Save & test, дашборд
└── README.md                    # логи выполнения
```

- Скриншоты и Readme можно заменить более привычным вам способом [ABOUT.md](../ABOUT.md) Скрины дашборда должны присутсвовать обязательно (Либо ссылку на развернутую вами графану)

---

**Максимальная оценка: 10 баллов**