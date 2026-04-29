# Apache OpenOffice Vulnerabilities Collector

Лабораторная работа №2: сбор данных об уязвимостях Apache OpenOffice, обогащение информации через MITRE CVE API, конвертация результата в XML, валидация JSON и загрузка данных в PostgreSQL.

## Тема

Apache OpenOffice Security Bulletin
Источник данных: https://www.openoffice.org/security/bulletin.html

## Структура проекта

.
├── collector.py          # сбор CVE и обогащение данных через MITRE API
├── converter.py          # конвертация result_task_2.json в XML
├── validate_task.py      # проверка JSON по json_schema.json
├── db_filler.py          # загрузка данных в PostgreSQL
├── json_schema.json      # JSON Schema для проверки result_task_2.json
├── init.sql              # создание таблиц БД
├── Dockerfile            # контейнер Python-приложения
├── docker-compose.yml    # запуск приложения и PostgreSQL
├── requirements.txt      # зависимости Python
├── README.md
└── .gitignore

## Запуск проекта

1. Собрать и запустить контейнеры:

sudo docker compose up -d --build

2. Выполнить сбор CVE и обогащение данных через MITRE API:

sudo docker compose run --rm app python3 collector.py --task all

3. Выполнить конвертацию JSON в XML:

sudo docker compose run --rm app python3 converter.py

4. Выполнить проверку JSON по схеме:

sudo docker compose run --rm app python3 validate_task.py

5. Загрузить данные в PostgreSQL:

sudo docker compose run --rm app python3 db_filler.py

## Результаты выполнения

После запуска создаются файлы:

result_task_1.json
result_task_2.json
result_task_3.xml

Файлы с итоговыми результатами не выгружаются в репозиторий, так как они создаются автоматически при запуске программы.

## Task 1

На первом этапе скрипт collector.py парсит страницу Apache OpenOffice Security Bulletin. Из страницы извлекаются идентификаторы CVE, дата релиза версии OpenOffice, в которой была исправлена уязвимость, и ссылка на страницу уязвимости на сайте поставщика.

Результат сохраняется в файл result_task_1.json.

Формат результата:

[
    {
        "ID": "CVE-2025-64401",
        "vendor_release_date": "2025-11-10",
        "vendor_release_url": "https://www.openoffice.org/security/cves/CVE-2025-64401.html"
    }
]

## Task 2

На втором этапе collector.py берет CVE-ID из result_task_1.json и отправляет запросы в MITRE CVE API.

Для каждой уязвимости собираются дополнительные данные:

- ссылка на CVE;
- дата публикации;
- дата обновления;
- описание;
- CVSS;
- CPE;
- CWE.

Результат сохраняется в файл result_task_2.json.

Если в MITRE CVE API отсутствуют CVSS, CPE или CWE, программа подставляет значения-заглушки. Это нужно для сохранения целостности структуры данных и дальнейшей валидации.

## Task 3

На третьем этапе converter.py преобразует result_task_2.json в XML-файл result_task_3.xml.

При конвертации сохраняется вложенность данных:

- cvss_list преобразуется в элементы cvss;
- cpe_list преобразуется в элементы cpe;
- cwe преобразуется в элементы cwe.

Пример структуры XML:

<cvss version="cvss31" score="7.5" severity="HIGH">CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:N</cvss>
<cpe>cpe:2.3:a:apache:openoffice:*:*:*:*:*:*:*:*</cpe>
<cwe id="CWE-862" name="CWE-862 Missing Authorization">CWE-862 Missing Authorization</cwe>

## Task 4

На четвертом этапе validate_task.py проверяет файл result_task_2.json по JSON-схеме json_schema.json.

Схема проверяет:

- наличие обязательных полей первого уровня;
- наличие минимум одного элемента в cvss_list;
- наличие минимум одного элемента в cpe_list;
- наличие минимум одного элемента в cwe;
- заполненность дочерних элементов.

Также выполняется дополнительная проверка соответствия количества записей в JSON и XML.

## Task 5

На пятом этапе через Docker Compose поднимается PostgreSQL. Таблицы создаются автоматически из файла init.sql.

База данных нормализована до третьей нормальной формы. Основная информация об уязвимости хранится отдельно от CVSS, CPE и CWE.

Используемые таблицы:

vulnerabilities
cvss_metrics
cpe_entries
vulnerability_cpe
cwe_entries
vulnerability_cwe

Скрипт db_filler.py читает result_task_2.json и загружает данные в PostgreSQL.

## Проверка базы данных

Посмотреть список таблиц:

sudo docker compose exec db psql -U openoffice_user -d openoffice_vulnerabilities -c "\dt"

Проверить количество записей:

sudo docker compose exec db psql -U openoffice_user -d openoffice_vulnerabilities -c "SELECT COUNT(*) FROM vulnerabilities;"
sudo docker compose exec db psql -U openoffice_user -d openoffice_vulnerabilities -c "SELECT COUNT(*) FROM cvss_metrics;"
sudo docker compose exec db psql -U openoffice_user -d openoffice_vulnerabilities -c "SELECT COUNT(*) FROM cpe_entries;"
sudo docker compose exec db psql -U openoffice_user -d openoffice_vulnerabilities -c "SELECT COUNT(*) FROM cwe_entries;"

## Используемые технологии

Python 3.12
Requests
BeautifulSoup4
lxml
jsonschema
psycopg2-binary
PostgreSQL 15
Docker Compose

## Особенности и проблемы

При работе с MITRE CVE API часть уязвимостей может не содержать полного набора данных. Например, для некоторых записей отсутствуют CVSS-метрики, CPE-строки или CWE-классификация.

Для обработки таких случаев в программе используются значения-заглушки:

n/a
0.0
NONE

Это позволяет сохранить одинаковую структуру итогового JSON-файла, выполнить XML-конвертацию, пройти валидацию и загрузить данные в базу данных без потери записей.

## Очистка проекта

Остановить контейнеры:

sudo docker compose down

Остановить контейнеры и удалить volume базы данных:

sudo docker compose down -v
