<!-- README-ru.md -->

# Руководство по Настройке WireGuard для Arch Linux

Это руководство предоставляет упрощённый пошаговый процесс настройки безопасного VPN WireGuard на Arch Linux. Оно обеспечивает правильную конфигурацию публичных и приватных ключей, чтобы избежать распространённых проблем, связанных с аутентификацией и маршрутизацией трафика. Конфигурация клиента охватывает различные платформы, включая Linux, Windows и мобильные устройства (iOS/Android).

[English version](README.md)
## Содержание

1. [Требования](#требования)
2. [Установка](#установка)
3. [Генерация Ключей](#генерация-ключей)
4. [Конфигурация Сервера](#конфигурация-сервера)
5. [Конфигурация Клиента](#конфигурация-клиента)
    - [Linux](#linux)
    - [Windows](#windows)
    - [iOS/Android](#iosandroid)
6. [Брандмауэр и Маршрутизация](#брандмауэр-и-маршрутизация)
7. [Запуск WireGuard](#запуск-wireguard)
8. [Проверка](#проверка)
9. [Диагностика](#диагностика)

## Требования

- **Arch Linux** установлен на сервере.
- **Права суперпользователя** (root) на сервере.
- **Публичный IP-адрес** для сервера.
- **Приложение WireGuard** установлено на клиентском устройстве (Linux, Windows, iOS, Android).

## Установка

### На Сервере

1. **Обновите систему:**

    ```bash
    sudo pacman -Syu
    ```

2. **Установите WireGuard и текстовый редактор Nano:**

    Nano — удобный текстовый редактор, облегчающий редактирование конфигурационных файлов.

    ```bash
    sudo pacman -S wireguard-tools nano
    ```

### На Клиенте

- **Linux:**

    ```bash
    sudo pacman -S wireguard-tools nano
    ```

- **Windows, iOS, Android:**

    Установите [приложение WireGuard](https://www.wireguard.com/install/) с официального сайта или из магазина приложений вашего устройства.

## Генерация Ключей

### На Сервере

1. **Перейдите в директорию WireGuard:**

    ```bash
    sudo mkdir -p /etc/wireguard
    cd /etc/wireguard
    ```

2. **Сгенерируйте ключи сервера:**

    ```bash
    umask 077
    wg genkey | tee server_privatekey | wg pubkey > server_publickey
    ```

    - `server_privatekey`: Приватный ключ сервера.
    - `server_publickey`: Публичный ключ сервера.

### На Клиенте

#### Linux

1. **Перейдите в директорию WireGuard:**

    ```bash
    mkdir -p ~/wireguard
    cd ~/wireguard
    ```

2. **Сгенерируйте ключи клиента:**

    ```bash
    wg genkey | tee client_privatekey | wg pubkey > client_publickey
    ```

    - `client_privatekey`: Приватный ключ клиента.
    - `client_publickey`: Публичный ключ клиента.

#### Windows, iOS, Android

1. **Используйте приложение WireGuard для генерации ключей:**

    - Откройте приложение WireGuard.
    - Нажмите "Add Tunnel" > "Add empty tunnel..."
    - Приложение сгенерирует приватный и публичный ключи.
    - Сохраните приватный ключ в безопасном месте и скопируйте публичный ключ для конфигурации сервера.

## Конфигурация Сервера

1. **Создайте/Отредактируйте конфигурационный файл WireGuard:**

    ```bash
    sudo nano /etc/wireguard/wg0.conf
    ```

2. **Добавьте следующую конфигурацию:**

    ```ini
    [Interface]
    Address = 10.0.0.1/24
    ListenPort = 51820
    PrivateKey = <server_privatekey>

    # Включение IP-переадресации и настройка NAT
    PostUp = sysctl -w net.ipv4.ip_forward=1
    PostUp = iptables -t nat -A POSTROUTING -o <external_interface> -j MASQUERADE
    PostUp = iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
    PostUp = iptables -A FORWARD -s 10.0.0.0/24 -j ACCEPT
    PostDown = sysctl -w net.ipv4.ip_forward=0
    PostDown = iptables -t nat -D POSTROUTING -o <external_interface> -j MASQUERADE
    PostDown = iptables -D FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
    PostDown = iptables -D FORWARD -s 10.0.0.0/24 -j ACCEPT

    [Peer]
    PublicKey = <client_publickey>
    AllowedIPs = 10.0.0.2/32
    ```

    - Замените `<server_privatekey>` содержимым файла `server_privatekey`.
    - Замените `<external_interface>` на внешний сетевой интерфейс сервера (например, `ens1`, `eth0`). Посмотреть интерфейсы команда: ip a
    - Замените `<client_publickey>` на публичный ключ клиента.

3. **Сохраните и выйдите** (`Ctrl + O`, `Enter`, `Ctrl + X`).

## Конфигурация Клиента

### Linux

1. **Создайте/Отредактируйте конфигурационный файл:**

    ```bash
    sudo nano /etc/wireguard/wg0.conf
    ```

2. **Добавьте следующую конфигурацию:**

    ```ini
    [Interface]
    PrivateKey = <client_privatekey>
    Address = 10.0.0.2/24
    DNS = 8.8.8.8

    [Peer]
    PublicKey = <server_publickey>
    Endpoint = <server_public_ip>:51820
    AllowedIPs = 0.0.0.0/0, ::/0
    PersistentKeepalive = 25
    ```

    - Замените `<client_privatekey>` содержимым файла `client_privatekey`.
    - Замените `<server_publickey>` на публичный ключ сервера.
    - Замените `<server_public_ip>` на публичный IP-адрес сервера.

3. **Сохраните и выйдите** (`Ctrl + O`, `Enter`, `Ctrl + X`).

### Windows

1. **Откройте приложение WireGuard.**

2. **Добавьте новый туннель:**

    - Нажмите "Add Tunnel" > "Add empty tunnel..."

3. **Вставьте конфигурацию клиента:**

    ```ini
    [Interface]
    PrivateKey = <client_privatekey>
    Address = 10.0.0.2/24
    DNS = 8.8.8.8

    [Peer]
    PublicKey = <server_publickey>
    Endpoint = <server_public_ip>:51820
    AllowedIPs = 0.0.0.0/0, ::/0
    PersistentKeepalive = 25
    ```

    - Замените `<client_privatekey>` на приватный ключ клиента.
    - Замените `<server_publickey>` на публичный ключ сервера.
    - Замените `<server_public_ip>` на публичный IP-адрес сервера.

4. **Сохраните и активируйте туннель.**

### iOS/Android

1. **Откройте приложение WireGuard.**

2. **Добавьте новый туннель:**

    - Нажмите "+" > "Create from scratch" или "Import from file/device."

3. **Введите конфигурацию клиента:**

    ```ini
    [Interface]
    PrivateKey = <client_privatekey>
    Address = 10.0.0.2/24
    DNS = 8.8.8.8

    [Peer]
    PublicKey = <server_publickey>
    Endpoint = <server_public_ip>:51820
    AllowedIPs = 0.0.0.0/0, ::/0
    PersistentKeepalive = 25
    ```

    - Замените `<client_privatekey>` на приватный ключ клиента.
    - Замените `<server_publickey>` на публичный ключ сервера.
    - Замените `<server_public_ip>` на публичный IP-адрес сервера.

4. **Сохраните и активируйте туннель.**

## Брандмауэр и Маршрутизация

### На Сервере

1. **Настройте правила iptables:**

    ```bash
    sudo iptables -t nat -A POSTROUTING -o <external_interface> -j MASQUERADE
    sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
    sudo iptables -A FORWARD -s 10.0.0.0/24 -j ACCEPT
    ```

2. **Сохраните правила iptables для сохранения после перезагрузки:**

    ```bash
    sudo iptables-save | sudo tee /etc/iptables/iptables.rules
    sudo systemctl enable iptables
    sudo systemctl start iptables
    ```

3. **Включите IP-переадресацию:**

    ```bash
    echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/99-sysctl.conf
    sudo sysctl -p /etc/sysctl.d/99-sysctl.conf
    ```

## Запуск WireGuard

### На Сервере и Linux Клиенте

1. **Запустите и включите WireGuard:**

    ```bash
    sudo systemctl start wg-quick@wg0
    sudo systemctl enable wg-quick@wg0
    ```

### На Windows, iOS, Android

1. **Активируйте туннель:**

    - Откройте приложение WireGuard.
    - Включите переключатель рядом с вашим настроенным туннелем.

## Проверка

1. **Проверьте статус WireGuard:**

    ```bash
    sudo wg show
    ```

    - Убедитесь, что интерфейс `wg0` активен и пэры перечислены.

2. **Тестирование подключения:**

    - **Пинг сервера с клиента:**

        ```bash
        ping 10.0.0.1
        ```

    - **Пинг внешнего IP с клиента:**

        ```bash
        ping 8.8.8.8
        ```

    - **Тестирование разрешения DNS:**

        ```bash
        nslookup google.com
        ```

    - **Доступ к веб-сайтам:**

      Откройте веб-браузер и перейдите на любой сайт (например, [https://www.google.com](https://www.google.com)).

## Диагностика

- **Неправильное Сопоставление Ключей:**
  
  - Убедитесь, что `[Peer] PublicKey` на сервере содержит **публичный ключ клиента**.
  - Убедитесь, что `[Peer] PublicKey` на клиенте содержит **публичный ключ сервера**.

- **Правила Брандмауэра:**
  
  - Проверьте правила iptables:

    ```bash
    sudo iptables -L -v
    sudo iptables -t nat -L -v
    ```

- **IP-Переадресация:**
  
  - Подтвердите, что IP-переадресация включена:

    ```bash
    sysctl net.ipv4.ip_forward
    ```

    Должно вернуть `net.ipv4.ip_forward = 1`.

- **Просмотр Логов:**
  
  - Проверьте логи WireGuard на сервере:

    ```bash
    sudo journalctl -u wg-quick@wg0
    ```

- **Доступность Порта:**
  
  - Убедитесь, что UDP-порт `51820` открыт и прослушивается:

    ```bash
    sudo ss -ulnp | grep 51820
    ```

- **Проблемы с DNS:**
  
  - Если разрешение DNS не работает, попробуйте другие DNS-серверы (например, `1.1.1.1`, `8.8.4.4`).

## Частые Проблемы и Решения

### Причина: Неправильно Настроенные Публичные Ключи

**Проблема:** Клиент использовал приватный ключ сервера вместо его публичного ключа, что мешало корректной аутентификации.

**Решение:**
- Убедитесь, что `[Peer] PublicKey` на клиенте установлен на **публичный ключ сервера**.
- Убедитесь, что `[Peer] PublicKey` на сервере установлен на **публичный ключ клиента**.

### Причина: Дублирующиеся Правила iptables

**Проблема:** Несколько одинаковых правил `MASQUERADE` вызывали конфликты маршрутизации.

**Решение:**
- Удалите дублирующиеся правила iptables и оставьте только одно правило `MASQUERADE`.
  
    ```bash
    sudo iptables -t nat -F POSTROUTING
    sudo iptables -t nat -A POSTROUTING -o <external_interface> -j MASQUERADE
    ```

### Причина: Отключённая IP-Переадресация

**Проблема:** IP-переадресация была отключена, блокируя маршрутизацию трафика через VPN.

**Решение:**
- Включите IP-переадресацию постоянно.
  
    ```bash
    echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/99-sysctl.conf
    sudo sysctl -p /etc/sysctl.d/99-sysctl.conf
    ```

## Заключение

Правильная настройка публичных и приватных ключей, а также корректные настройки брандмауэра и маршрутизации, являются ключевыми для функционирования VPN WireGuard на Arch Linux. Следуя этому руководству, вы сможете настроить WireGuard безопасно и эффективно, минимизируя потенциальные проблемы, связанные с аутентификацией и маршрутизацией трафика.

Для дополнительной помощи обратитесь к [документации WireGuard](https://www.wireguard.com/#documentation) или обратитесь за помощью к сообществу Arch Linux.
