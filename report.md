---
## Front matter
title: "Системы инициализации Upstart"
subtitle: "Лабораторная работа по операционным системам"
author: "Зюков Александр Валерьевич"

## Generic otions
lang: ru-RU
toc-title: "Содержание"

## Bibliography
bibliography: bib/cite.bib
csl: pandoc/csl/gost-r-7-0-5-2008-numeric.csl

## Pdf output format
toc: true # Table of contents
toc-depth: 2
lof: true # List of figures
lot: true # List of tables
fontsize: 12pt
linestretch: 1.5
papersize: a4
documentclass: scrreprt
## I18n polyglossia
polyglossia-lang:
  name: russian
  options:
	- spelling=modern
	- babelshorthands=true
polyglossia-otherlangs:
  name: english
## I18n babel
babel-lang: russian
babel-otherlangs: english
## Fonts
mainfont: IBM Plex Serif
romanfont: IBM Plex Serif
sansfont: IBM Plex Sans
monofont: IBM Plex Mono
mathfont: STIX Two Math
mainfontoptions: Ligatures=Common,Ligatures=TeX,Scale=0.94
romanfontoptions: Ligatures=Common,Ligatures=TeX,Scale=0.94
sansfontoptions: Ligatures=Common,Ligatures=TeX,Scale=MatchLowercase,Scale=0.94
monofontoptions: Scale=MatchLowercase,Scale=0.94,FakeStretch=0.9
mathfontoptions:
## Biblatex
biblatex: true
biblio-style: "gost-numeric"
biblatexoptions:
  - parentracker=true
  - backend=biber
  - hyperref=auto
  - language=auto
  - autolang=other*
  - citestyle=gost-numeric
## Pandoc-crossref LaTeX customization
figureTitle: "Рис."
tableTitle: "Таблица"
listingTitle: "Листинг"
lofTitle: "Список иллюстраций"
lotTitle: "Список таблиц"
lolTitle: "Листинги"
## Misc options
indent: true
header-includes:
  - \usepackage{indentfirst}
  - \usepackage{float} # keep figures where there are in the text
  - \floatplacement{figure}{H} # keep figures where there are in the text
---

# Цель работы

1. Изучить принципы работы системы инициализации Upstart
2. Освоить методы создания и управления сервисами
3. Сравнить Upstart с традиционным SysVinit и современным systemd
4. Получить практические навыки конфигурирования системы инициализации

# Задание

1. Установить Upstart в тестовой среде
2. Создать и настроить демонстрационный сервис
3. Исследовать механизм обработки событий
4. Провести сравнительный анализ с другими системами инициализации

# Теоретическое введение

## Основные концепции Upstart

Upstart — событийно-ориентированная система инициализации, разработанная Canonical в 2006 году как замена традиционному SysVinit. Основные особенности:

- Параллельный запуск независимых сервисов
- Реакция на системные события в реальном времени
- Автоматический перезапуск упавших процессов

# Системы инициализации Upstart

## Архитектура Upstart

### Основные компоненты системы

Upstart представляет собой модульную систему, состоящую из нескольких ключевых компонентов:

1. **Главный демон (upstart-init)**:
   - Занимает место PID 1 в системе
   - Обрабатывает все события и управляет жизненным циклом процессов
   - Реализует механизм respawn для критически важных сервисов

2. **Подсистема событий**:
   - Включает в себя генераторы событий (event emitters)
   - Обеспечивает доставку событий подписчикам
   - Поддерживает как синхронные, так и асинхронные события

3. **Менеджер сессий**:
   - Управляет пользовательскими сеансами
   - Обеспечивает изоляцию процессов разных пользователей
   - Реализует механизм cgroups для контроля ресурсов

### Механизм обработки событий

Upstart использует сложную систему обработки событий, включающую:

- **Встроенные события**:
  - `startup` - инициализация системы
  - `filesystem` - готовность файловых систем
  - `net-device-up` - появление сетевого интерфейса

- **Пользовательские события**:
  - Могут генерироваться любым процессом через D-Bus
  - Пример: `initctl emit custom-event`

- **Зависимости между событиями**:
  ```conf
  start on (started network-manager and filesystem)
  stop on (stopped dbus or runlevel [06])
  ```

## Формат конфигурационных файлов

### Структура job-файла

Полный синтаксис конфигурационного файла включает следующие секции:

1. **Мета-информация**:
   ```conf
   description "Сервис управления веб-сервером"
   version "1.2"
   author "Отдел DevOps <devops@company.com>"
   ```

2. **Переменные окружения**:
   ```conf
   env NGINX_BIN=/usr/sbin/nginx
   env CONFIG_FILE=/etc/nginx/nginx.conf
   ```

3. **Условия запуска**:
   ```conf
   start on (local-filesystems and net-device-up IFACE=eth0)
   stop on runlevel [016]
   ```

4. **Предварительные скрипты**:
   ```conf
   pre-start script
       [ -x $NGINX_BIN ] || { stop; exit 0; }
       mkdir -p /var/log/nginx
       chown www-data:www-data /var/log/nginx
   end script
   ```

5. **Основная команда**:
   ```conf
   exec $NGINX_BIN -c $CONFIG_FILE -g "daemon off;"
   ```

6. **Пост-скрипты**:
   ```conf
   post-stop script
       rm -f /var/run/nginx.pid
   end script
   ```

### Особые директивы

1. **Обработка процессов**:
   ```conf
   expect fork    # Для демонов, делающих fork
   expect daemon  # Для обычных демонов
   respawn        # Автоматический перезапуск
   respawn limit 5 10  # 5 попыток за 10 секунд
   ```

2. **Управление окружением**:
   ```conf
   env PORT=8080
   export PORT
   umask 022
   oom never  # Защита от OOM-killer
   ```

3. **Работа с консолью**:
   ```conf
   console output  # Перенаправление вывода
   console owner   # Владелец терминала
   ```

## Управление сервисами

### Команды администрирования

1. **Базовые операции**:
   ```bash
   initctl start servicename
   initctl stop servicename
   initctl restart servicename
   initctl status servicename
   ```

2. **Диагностика**:
   ```bash
   initctl list                # Список всех задач
   initctl show-config servicename  # Показать конфигурацию
   initctl log-priority debug  # Изменить уровень логирования
   ```

3. **Работа с событиями**:
   ```bash
   initctl emit --no-wait EVENT_NAME  # Асинхронная генерация
   initctl emit EVENT_NAME PARAM=value  # С параметрами
   ```

### Отладка сервисов

1. **Просмотр логов**:
   ```bash
   tail -f /var/log/upstart/servicename.log
   ```

2. **Тестовый запуск**:
   ```bash
   initctl check-config servicename  # Проверка синтаксиса
   initctl start servicename debug   # Запуск с отладкой
   ```

3. **Анализ зависимостей**:
   ```bash
   initctl graph-events | dot -Tpng > deps.png  # Визуализация
   ```

## Внутренние механизмы

### Обработка зависимостей

Upstart использует сложный алгоритм разрешения зависимостей:

1. **Топологическая сортировка** задач при запуске
2. **Параллельное выполнение** независимых задач
3. **Ожидание условий**:
   ```conf
   start on (started postgresql and net-device-up IFACE=eth0)
   ```

### Безопасность

1. **Изоляция процессов**:
   - Использование cgroups
   - Ограничение ресурсов
   - Запуск от разных пользователей

2. **Защитные механизмы**:
   ```conf
   limit nofile 4096 4096  # Ограничение файловых дескрипторов
   oom score -100         # Приоритет при нехватке памяти
   ```

## Пример сложной конфигурации

```conf
description "Многоуровневый сервис приложения"
author "Зюков А.В. <1132241589@pfur.ru>"
version "2.1"

env APP_HOME=/opt/myapp
env USER=appuser
env GROUP=appgroup
env JAVA_OPTS="-Xms512m -Xmx1024m"

start on (started postgresql and net-device-up IFACE=eth0)
stop on (runlevel [016] or stopped postgresql)

respawn
respawn limit 10 60
console output

pre-start script
    # Проверка зависимостей
    [ -d $APP_HOME ] || { stop; exit 0; }
    [ -x $APP_HOME/bin/start.sh ] || { stop; exit 0; }
    
    # Подготовка окружения
    mkdir -p $APP_HOME/logs
    chown $USER:$GROUP $APP_HOME/logs
    chmod 750 $APP_HOME/logs
end script

script
    exec start-stop-daemon --start --chuid $USER:$GROUP \
         --make-pidfile --pidfile $APP_HOME/run/app.pid \
         --exec $APP_HOME/bin/start.sh -- $JAVA_OPTS
end script

post-stop script
    rm -f $APP_HOME/run/app.pid
    # Очистка временных файлов
    find $APP_HOME/tmp -type f -mtime +7 -delete
end script
```

## Проблемы и решения

### Типичные проблемы

1. **Циклические зависимости**:
   - Решение: анализ графа зависимостей
   - Инструмент: `initctl graph-events`

2. **Конфликты ресурсов**:
   - Решение: правильная настройка pre-start скриптов
   - Пример: блокировка файлов `.lock`

3. **Проблемы с правами**:
   - Решение: использование `chuid` и `start-stop-daemon`

### Оптимизация производительности

1. **Параллелизация запуска**:
   ```conf
   start on (started service1 and started service2)
   ```

2. **Отложенный запуск**:
   ```conf
   start on started network-manager
   task
   script
       sleep 10  # Дать время для инициализации сети
       exec /usr/bin/myapp
   end script
   ```

3. **Приоритизация**:
   ```conf
   nice -10  # Установка приоритета
   ```

## Интеграция с другими системами

### Совместная работа с Systemd

1. **Режим совместимости**:
   - Upstart может работать как подсистема в Systemd
   - Конвертация job-файлов в unit-файлы

2. **Гибридные конфигурации**:
   ```bash
   systemctl start upstart-compat.service
   ```

### Работа с Docker

1. **Использование в контейнерах**:
   ```dockerfile
   RUN apt-get install -y upstart
   COPY myapp.conf /etc/init/
   CMD ["/sbin/init"]
   ```

2. **Ограничения**:
   - Отсутствие полноценной поддержки событий
   - Проблемы с изоляцией процессов

## Перспективы развития

### Современное состояние

1. **Поддержка**:
   - Только критические исправления безопасности
   - Нет активной разработки новых функций

2. **Альтернативы**:
   - Systemd для сложных систем
   - Runit для минималистичных решений

### Области применения

1. **Legacy-системы**:
   - Промышленные контроллеры
   - Устаревшие серверные платформы

2. **Образовательные цели**:
   - Изучение эволюции систем инициализации
   - Пример event-driven архитектуры
