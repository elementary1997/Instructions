# Установка Wireguard + WGDasboard

Эта инструкция позволяет установить и обеспечить автозапуск WGDashboard, настроить NAT и безопасный доступ с HTTPS через nginx.

1. Установка WGDashboard:

- Обновите пакеты и установите зависимости:

```bash
sudo apt update
sudo apt install -y wireguard git net-tools python3 python3-pip iptables-persistent nginx python3-certbot-nginx
```

- Склонируйте репозиторий WGDashboard:

```bash
git clone https://github.com/WGDashboard/WGDashboard.git
```

- Перейдите в каталог с исходниками:

```bash
cd WGDashboard/src
```

- Дайте права на исполнение скрипту и установите WGDashboard:

```bash
sudo chmod u+x wgd.sh
sudo ./wgd.sh install
```

- Дайте права на чтение и выполнение конфигурационным файлам WireGuard:

```bash
sudo chmod -R 755 /etc/wireguard
```


2. Включение маршрутизации IPv4 (рекомендуется):
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p /etc/sysctl.conf
```

3. Настройка NAT и сохранение правил iptables:
```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i wg0 -j ACCEPT
sudo iptables -A FORWARD -o wg0 -j ACCEPT

sudo netfilter-persistent save
```

4. Создание systemd-сервиса для автозапуска WGDashboard:

- Создайте файл `/etc/systemd/system/wgd.service` со следующим содержимым:

```nginx
[Unit]
Description=WGDashboard startup service
After=network.target

[Service]
Type=oneshot
ExecStart=/full/path/to/WGDashboard/src/wgd.sh start
WorkingDirectory=/full/path/to/WGDashboard/src
User=root
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

(Замените `/full/path/to/WGDashboard/src` на фактический путь к скрипту)
- Активируйте сервис и запустите его:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wgd.service
sudo systemctl start wgd.service
```


5. Настройка HTTPS с помощью nginx и Let's Encrypt:

- Создайте конфигурацию nginx, например `/etc/nginx/sites-available/wgdashboard`:

```nginx
server {
    listen 80;
    server_name vpn.example.com;

    location / {
        proxy_pass http://127.0.0.1:10086;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

(Замените `vpn.example.com` на ваш реальный домен)
- Активируйте конфигурацию и перезапустите nginx:

```bash
sudo ln -s /etc/nginx/sites-available/wgdashboard /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

- Установите SSL сертификат Let’s Encrypt через certbot:

```bash
sudo certbot --nginx -d vpn.example.com
```

Certbot автоматически настроит HTTPS и обновление сертификатов.

6. Доступ к панели WGDashboard:

- Откройте в браузере:

```bash
https://vpn.example.com
```

- Логин: admin
- Пароль: admin

***

Обратите внимание:

- DNS домена vpn.example.com должен указывать на IP вашего сервера.
- В systemd-сервисе не используйте sudo в ExecStart, сервис запускается от root.
- После изменений в systemd или путях всегда выполняйте `daemon-reload` и перезапускайте сервисы.
