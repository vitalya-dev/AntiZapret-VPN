# 🚀 Настройка sing-box + WARP

### Шаг 0: Подготовка АнтиЗапрета (Изоляция и маркировка)

Нам нужно разрешить файрволу АнтиЗапрета принимать ответный трафик от sing-box и научить OpenVPN маркировать свои пакеты для защиты от зацикливания.

**1. Отключаем изоляцию клиентов:**
По умолчанию `iptables` в АнтиЗапрете блокирует трафик, идущий к VPN-клиентам не из основного интерфейса. Чтобы клиенты могли получать ответы от sing-box, эту блокировку нужно снять.
Откройте конфигурационный файл установщика:

```bash
nano /root/antizapret/setup
```

Найдите параметр `CLIENT_ISOLATION` и измените его значение с `y` на `n`:

```ini
CLIENT_ISOLATION=n
```

Сохраните файл (`Ctrl+O`, `Enter`, `Ctrl+X`).

**2. Добавляем метку ВО ВСЕ конфиги OpenVPN:**
> **💡 Зачем нам метка `mark 555`?**
> Метка нужна для защиты от бесконечной петли маршрутизации (routing loop).

Чтобы защитить все возможные виды подключений, добавим `mark 555` во все конфигурационные файлы OpenVPN, которые лежат на сервере. Выполните одну быструю команду, которая добавит эту строчку во все `.conf` файлы разом:

```bash
echo "mark 555" | tee -a /etc/openvpn/server/antizapret-udp.conf /etc/openvpn/server/antizapret-tcp.conf /etc/openvpn/server/vpn-udp.conf /etc/openvpn/server/vpn-tcp.conf
```

**3. Перезагрузка:**
Чтобы новые правила `iptables`, маршрутизации и настройки OpenVPN применились начисто, **полностью перезагрузите сервер**:

```bash
reboot
```

---

### Шаг 1: Установка sing-box

После перезагрузки подключаемся к серверу и устанавливаем актуальную версию sing-box одним скриптом:

```bash
curl -fsSL https://sing-box.app/install.sh | bash

```

---

### Шаг 2: Получаем ключи WARP через wgcf

Чтобы sing-box подключился к Cloudflare, нужен личный приватный ключ и IP-адрес. Добудем их через официальную утилиту `wgcf`.

Выполните команды по очереди:

```bash
# Скачиваем утилиту и даем права на запуск
wget -O wgcf https://github.com/ViRb3/wgcf/releases/download/v2.2.22/wgcf_2.2.22_linux_amd64
chmod +x wgcf

# Регистрируем аккаунт и генерируем конфиг
./wgcf register --accept-tos
./wgcf generate
```

Откройте созданный файл:

```bash
cat wgcf-profile.conf
```

Вам нужно скопировать оттуда **два значения** для следующего шага:

* **PrivateKey** (строка вида `uBW0nm7U...=`)
* **Address** (ваш IPv4, обычно это `172.16.0.2/32`)

---

### Шаг 3: Настройка config.json

Откройте файл конфигурации sing-box:

```bash
nano /etc/sing-box/config.json
```

Удалите всё содержимое и вставьте код ниже.
⚠️ **Обязательно замените `ВАШ_IP_АДРЕС` и `ВАШ_ПРИВАТНЫЙ_КЛЮЧ` на данные, которые вы скопировали на Шаге 2!**

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "endpoints": [
    {
      "type": "wireguard",
      "tag": "warp",
      "name": "warp-tun",
      "system": false,
      "mtu": 1280,
      "address": [
        "ВАШ_IP_АДРЕС"
      ],
      "private_key": "ВАШ_ПРИВАТНЫЙ_КЛЮЧ",
      "peers": [
        {
          "address": "162.159.192.1",
          "port": 2408,
          "public_key": "bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=",
          "allowed_ips": [
            "0.0.0.0/0",
            "::/0"
          ],
          "reserved": [0, 0, 0]
        }
      ]
    }
  ],
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "singbox-tun",
      "address": [
        "172.19.0.1/30"
      ],
      "auto_route": true,
      "strict_route": false,
      "stack": "system",
      "sniff": true,
      "sniff_override_destination": true,
      "mtu": 1280
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ],
  "route": {
    "rule_set": [
      {
        "tag": "geosite-openai",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-openai.srs",
        "download_detour": "direct"
      },
      {
        "tag": "geosite-gemini",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-google-gemini.srs",
        "download_detour": "direct"
      }
    ],
    "rules": [
      {
        "ip_is_private": true,
        "outbound": "direct"
      },
      {
        "rule_set": [
          "geosite-openai",
          "geosite-gemini"
        ],
        "outbound": "warp"
      }
    ],
    "auto_detect_interface": true,
    "final": "direct"
  }
}
```

*(Сохраните файл: нажмите `Ctrl+O`, затем `Enter`, затем `Ctrl+X`)*

---

### Шаг 4: Настройка системной службы (systemd)

Чтобы предотвратить зацикливание трафика и заставить пакеты с меткой `555` идти к клиентам в обход туннеля, редактируем файл службы:

```bash
nano /usr/lib/systemd/system/sing-box.service
```

Приведите его к следующему виду:

```ini
[Unit]
Description=sing-box service
Documentation=https://sing-box.sagernet.org
After=network.target nss-lookup.target network-online.target

[Service]
User=sing-box
StateDirectory=sing-box
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_RAW CAP_NET_BIND_SERVICE CAP_SYS_PTRACE CAP_DAC_READ_SEARCH
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW CAP_NET_BIND_SERVICE CAP_SYS_PTRACE CAP_DAC_READ_SEARCH
ExecStart=/usr/bin/sing-box -D /var/lib/sing-box -C /etc/sing-box run
ExecStartPost=/sbin/ip rule add fwmark 555 table main pref 99
ExecStopPost=-/sbin/ip rule del fwmark 555 table main pref 99
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=10s
LimitNOFILE=infinity

[Install]
WantedBy=multi-user.target
```

*(Сохраните файл: `Ctrl+O`, `Enter`, `Ctrl+X`)*

---

### Шаг 5: Запуск

Применяем настройки и запускаем наш туннель:

```bash
systemctl daemon-reload
systemctl enable sing-box
systemctl restart sing-box
```

Убедитесь, что сервис работает (должен гореть зеленый статус `active (running)`):

```bash
systemctl status sing-box
```

---

### ✅ Проверка работоспособности

Чтобы убедиться, что трафик до OpenAI действительно идет через Cloudflare WARP, выполните в терминале сервера команду:

```bash
curl -s https://chatgpt.com/cdn-cgi/trace | grep warp
```

Если в ответ вы увидели строку `warp=on` или `warp=plus`, значит всё настроено идеально! Теперь трафик для ChatGPT и Gemini успешно заворачивается в WARP.
