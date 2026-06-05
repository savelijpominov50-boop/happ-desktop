# 🔐 project_muam — Централизованный анализ логов безопасности (Auditd & SIEM-лайт)

> Курсовой проект по курсу «Администрирование ОС Linux»  
> Тема №29: Сбор и централизованный анализ логов безопасности (Auditd & SIEM-лайт)

---

## 📋 Описание проекта

Проект реализует автоматизированную систему мониторинга безопасности Linux-инфраструктуры на базе стека **Auditbeat + Elasticsearch + Kibana** (ELK), полностью развёрнутой в изолированных Docker-контейнерах.

Система осуществляет:
- Сбор и анализ событий аудита ядра Linux через подсистему **auditd**
- Мониторинг целостности критических системных файлов (`/etc/passwd`, `/etc/shadow`, `/etc/sudoers`)
- Централизованную индексацию и визуализацию инцидентов безопасности в **Kibana**
- Отправку алертов о критических событиях в **Telegram** через Webhook API

---

## 👤 Состав команды и роли

| ФИО | Роль | Зона ответственности |
|-----|------|----------------------|
| Поминов Савелий | DevOps/IaC Engineer + SRE + Security Engineer | Вся инфраструктура (одиночный проект) |

> Проект выполнен одним студентом. Согласно методическому руководству, студент совмещает все три роли в усечённом объёме.

---

## 🛠️ Технологический стек

| Компонент | Технология | Версия |
|-----------|-----------|--------|
| Аудит ядра | Auditbeat | 8.12.0 |
| Поисковый движок | Elasticsearch | 8.12.0 |
| Визуализация | Kibana | 8.12.0 |
| Контейнеризация | Docker + Docker Compose | latest LTS |
| ОС (базовые образы) | Ubuntu/Alpine | LTS |
| Язык автоматизации | Bash | 5.0+ |

---

## 📁 Структура репозитория

```
project_muam/
├── .github/
│   └── workflows/
│       └── deploy-docs.yml         # Деплой документации в gh-pages
├── docker/
│   ├── auditbeat/
│   │   ├── Dockerfile              # Сборка образа Auditbeat
│   │   └── auditbeat.yml           # Конфигурация правил аудита и модуля file_integrity
│   └── kibana/
│       └── (конфигурация Kibana)
├── scripts/
│   └── (скрипты автоматизации и тестирования)
├── .env.example                    # Шаблон переменных окружения
├── docker-compose.yml              # Главный файл оркестрации
├── README.md                       # Данный файл
└── LICENSE.txt                     # Лицензия MIT
```

---

## ⚙️ Требования к окружению

- **ОС:** Ubuntu Server 22.04 LTS / Debian 12 (64-bit)
- **Docker:** >= 24.0
- **Docker Compose Plugin:** >= 2.20
- **RAM:** не менее 4 ГБ (Elasticsearch требует минимум 2 ГБ heap)
- **Дисковое пространство:** не менее 10 ГБ свободного места

---

## 🚀 Быстрый старт

### 1. Клонирование репозитория

```bash
git clone https://github.com/savelijpominov50-boop/project_muam.git
cd project_muam
```

### 2. Настройка переменных окружения

```bash
cp .env.example .env
```

Отредактируйте файл `.env`, заполнив обязательные переменные:

```bash
# Версия стека ELK
ELASTIC_VERSION=8.12.0

# Пароль пользователя elastic (обязательно сменить!)
ELASTIC_PASSWORD=<ваш_безопасный_пароль>

# Пароль Kibana (обязательно сменить!)
KIBANA_PASSWORD=<ваш_безопасный_пароль_kibana>

# Telegram-бот для алертов
ALERT_TELEGRAM_TOKEN=<токен_вашего_бота>
ALERT_TELEGRAM_CHAT_ID=<id_чата>
```

> ⚠️ **ВНИМАНИЕ:** Файл `.env` содержит чувствительные данные. Он добавлен в `.gitignore` и **категорически запрещён** к коммиту в репозиторий.

### 3. Запуск инфраструктуры

```bash
cd deploy
docker compose up -d --build
```

### 4. Проверка статуса контейнеров

```bash
docker compose ps
```

Все контейнеры должны иметь статус `healthy`.

### 5. Остановка

```bash
docker compose down
```

Для остановки с удалением томов данных:

```bash
docker compose down -v
```

---

## 🔍 Проверка работоспособности

### Elasticsearch

```bash
curl -u elastic:${ELASTIC_PASSWORD} http://localhost:9200/_cluster/health?pretty
```

Ожидаемый ответ: `"status": "green"` или `"yellow"`.

### Kibana

Откройте в браузере: [http://localhost:5601](http://localhost:5601)

Войдите под пользователем `elastic` с паролем из `.env`.

### Auditbeat

```bash
docker compose logs auditbeat --tail=50
```

Проверка отправки событий в Elasticsearch:

```bash
curl -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/auditbeat-*/_count?pretty"
```

---

## 🛡️ Мониторинг безопасности

### Правила аудита ядра

Система отслеживает следующие события (конфигурация в `docker/auditbeat/auditbeat.yml`):

| Правило | Описание | Ключ |
|---------|----------|------|
| `-a always,exit -F arch=b64 -S execve` | Запуск любых команд (64-бит) | `user_commands` |
| `-a always,exit -F arch=b32 -S execve` | Запуск любых команд (32-бит) | `user_commands` |
| `-w /hostfs/etc/passwd -p wa` | Изменение базы пользователей | `identity_changes` |
| `-w /hostfs/etc/shadow -p wa` | Изменение хешей паролей | `security_alert` |
| `-w /hostfs/etc/sudoers -p wa` | Изменение привилегий sudo | `privilege_changes` |

### Мониторинг целостности файлов

Модуль `file_integrity` Auditbeat отслеживает изменения в:
- `/hostfs/etc/passwd`
- `/hostfs/etc/shadow`
- `/hostfs/etc/sudoers`
- `/hostfs/etc/sudoers.d`

Хеш-алгоритм: **SHA-256**.

---

## 📊 Дашборды Kibana

После запуска стека перейдите в Kibana → **Discover** → выберите индекс `auditbeat-*`.

Для просмотра событий безопасности используйте фильтры:
- `event.module: auditd` — события аудита ядра
- `event.module: file_integrity` — изменения файлов
- `tags: security_alert` — критические инциденты

---

## 🔧 Конфигурационные файлы

### `docker/auditbeat/auditbeat.yml`

Файл содержит:
- Правила аудита системных вызовов (`auditd` модуль)
- Настройку мониторинга целостности файлов (`file_integrity` модуль)
- Параметры подключения к Elasticsearch
- Настройки ротации и хранения логов Auditbeat

### `.env.example`

Шаблон всех необходимых переменных окружения. Перед запуском **обязательно** скопировать в `.env` и заполнить реальными значениями.

---

## 🧪 Тестирование

### Проверка обнаружения изменений файлов

```bash
# Симуляция инцидента — изменение /etc/passwd (выполнять ТОЛЬКО в тестовой среде!)
docker exec -it <container_id> sh -c \
  "echo '# test' >> /hostfs/etc/passwd"

# Проверка, что событие зафиксировано в Elasticsearch
curl -u elastic:${ELASTIC_PASSWORD} \
  "http://localhost:9200/auditbeat-*/_search?q=tags:identity_changes&pretty" \
  | grep "_source" | head -20
```

### Проверка алертов

Скрипты автоматического тестирования расположены в `deploy/scripts/tests/`.

---

## 🔐 Безопасность

- Все сервисы запускаются от **непривилегированных пользователей** (non-root)
- Пароли и токены передаются исключительно через `.env` (в `.gitignore`)
- Конфигурационные файлы монтируются в контейнеры в режиме **read-only** (`:ro`)
- Межконтейнерное взаимодействие осуществляется через **изолированные Docker-сети**
- Файл `.env` **запрещено** добавлять в Git-репозиторий

---

## 🔄 CI/CD

### `.github/workflows/deploy-docs.yml`

Автоматически деплоит документацию из каталога `docs/` в ветку `gh-pages` при слиянии в `main`.

---

## 📄 Лицензирование

| Артефакт | Лицензия |
|----------|----------|
| Исходный код скриптов автоматизации | [MIT License](LICENSE.txt) |
| Инфраструктурный код (Docker, Compose) | [GPLv3](LICENSE.txt) |
| Техническая документация и отчёт | [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) |

Полный текст лицензии: [LICENSE.txt](LICENSE.txt)

---

## 📚 Использованные источники

1. [Elastic Auditbeat Reference 8.12](https://www.elastic.co/guide/en/beats/auditbeat/current/index.html)
2. [Linux Audit Documentation — kernel.org](https://github.com/linux-audit/audit-documentation)
3. [Docker Compose Specification](https://docs.docker.com/compose/compose-file/)
4. [Elasticsearch Docker Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html)
5. ГОСТ 2.105-2019 «ЕСКД. Общие требования к текстовым документам»
6. ГОСТ 7.32-2017 «СИБИД. Отчёт о научно-исследовательской работе»

---

*Курсовой проект по курсу «Администрирование ОС Linux» | 2025–2026 учебный год*
