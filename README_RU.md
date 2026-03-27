
# Universal NUT UPS Template for Zabbix 7.x

![Zabbix](https://img.shields.io/badge/Zabbix-7.0%2B-red) ![License](https://img.shields.io/badge/License-GPLv3-blue)

English documentation: [README.md](./README.md).

Универсальный шаблон для мониторинга источников бесперебойного питания (ИБП) через **Network UPS Tools (NUT)** в системе мониторинга **Zabbix 7.0+**.

Шаблон оптимизирован для **APC Smart-UPS**, но подходит для большинства ИБП, поддерживаемых NUT. Решает проблему с "мусорным" выводом консоли (`Init SSL without certificate database`), корректно обрабатывает JSON Discovery и поддерживает автоматическое обнаружение параметров.

## 📊 Возможности

Шаблон автоматически обнаруживает и собирает следующие метрики:

* **Батарея:**
    * Уровень заряда (%)
    * Напряжение (V)
    * Температура (°C)
    * **Runtime** (Остаток времени работы в секундах/минутах)
    * Статус (Зарядка, Разрядка, Требуется замена и т.д.)
* **Вход / Выход:**
    * Входящее/Исходящее напряжение (V)
    * Частота (Hz)
    * Сила тока (A)
* **Нагрузка:**
    * Текущая нагрузка на ИБП (%)
* **Инвентаризация:**
    * Модель, Серийный номер, Дата производства батареи.

## 🛠️ Требования

1.  Zabbix Server 7.0 или выше.
2.  Zabbix Agent (установленный на машине, где подключен ИБП).
3.  Настроенный пакет `nut-server` и `nut-client` (проверьте командой `upsc -L`).

## ⚙️ Установка и настройка (Linux)

Для корректной работы Discovery и очистки вывода от SSL-ошибок используется специальный wrapper-скрипт.

### 1. Создание скрипта-обработчика

Создайте файл скрипта на хосте с ИБП:

```bash
sudo nano /etc/zabbix/scripts/ups_handler.sh

```

Вставьте в него следующий код:

```bash
#!/bin/bash

# IP адрес вашего сервера NUT (обычно localhost или IP сервера)
NUT_HOST="127.0.0.1" 
# Если NUT требует авторизации, используйте формат: username:password@127.0.0.1

# --- РЕЖИМ 1: Автообнаружение (Discovery) ---
if [ "$1" = "ups.discovery" ]; then
    /usr/bin/upsc -L $NUT_HOST 2>/dev/null | sed 's/://' | awk 'BEGIN {printf "{\"data\":["} {printf "{\"{#UPSNAME}\":\""$1"\"},"} END {print "]}"}' | sed 's/,]}/]}/'
    exit 0
fi

# --- РЕЖИМ 2: Получение метрик ---
if [ -z "$3" ]; then
    # Вариант: Пришло 2 параметра (Имя, Метрика)
    UPS_NAME=$1
    METRIC=$2
    TARGET_HOST=$NUT_HOST
else
    # Вариант: Пришло 3 параметра (Имя, IP, Метрика) - для совместимости
    UPS_NAME=$1
    TARGET_HOST=$2
    METRIC=$3
fi

# Выполняем запрос, подавляя ошибки SSL
/usr/bin/upsc "${UPS_NAME}@${TARGET_HOST}" "$METRIC" 2>/dev/null

```

> **Важно:** Если ваш NUT слушает внешний IP, измените переменную `NUT_HOST` внутри скрипта (например, на `192.168.1.10`).

Дайте права на выполнение:

```bash
sudo chmod +x /etc/zabbix/scripts/ups_handler.sh

```

### 2. Настройка Zabbix Agent

Откройте конфигурацию агента (или создайте отдельный конфиг в `/etc/zabbix/zabbix_agentd.d/nut.conf`):

```bash
sudo nano /etc/zabbix/zabbix_agentd.conf

```

Добавьте в конец файла строку **UserParameter**:

```ini
UserParameter=upsmon[*],/etc/zabbix/scripts/ups_handler.sh "$1" "$2" "$3"

```

Перезапустите агента:

```bash
sudo systemctl restart zabbix-agent

```

### 3. Проверка работы (Тест)

Вы можете проверить работу скрипта с сервера Zabbix или локально через `zabbix_get`:

```bash
# Проверка обнаружения (должен вернуть чистый JSON)
zabbix_get -s 127.0.0.1 -k upsmon[ups.discovery]

# Проверка получения данных (замените 'apc' на имя вашего ИБП)
zabbix_get -s 127.0.0.1 -k upsmon[apc,battery.charge]

```

---

## 📦 Установка в Zabbix (Web Interface)

1. Скачайте файл `zabbix_nut_universal_ru.xml` из этого репозитория.
2. Перейдите в веб-интерфейс Zabbix: **Data collection** -> **Templates**.
3. Нажмите **Import** (справа сверху).
4. Выберите скачанный файл и нажмите **Import**.
5. Перейдите к настройке хоста (**Hosts**), к которому подключен ИБП.
6. Добавьте шаблон **NUT UPS Universal RU**.
7. Через 1-2 минуты (или после нажатия "Execute now" в Discovery rules) появятся метрики.

## 📝 Лицензия

Этот проект распространяется под лицензией **GPL v3**.
Вы можете свободно использовать, модифицировать и распространять данный шаблон.

Автор: [didimozg](https://github.com/didimozg)
