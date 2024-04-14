# wireguard_Debian
Установка WireGuard на Debian (Raspberry Pi)

// Обновляем систему
sudo apt update
sudo reboot

// Устанавливаем веб-сервер для тестов сети
sudo apt-get install apache2
sudo service apache2 restart
sudo service apache2 status
cd /var/www/html
sudo nano index.html
	<!DOCTYPE html>
	<html lang="ru">
	<head>
	<title>Заголовок страницы</title>
	</head>
	<body>
	<header>
	<h1>Личный сайт</h1>
	<p>Который сделан на основе готового шаблона</p>
	<nav>
	<ul>
	<li><a href="index.html">Эта страница</a></li>
	<li><a href="http://google.com">Другая страница</a></li>
	</ul>
	</nav>
	</header>
	<main>
	<article>
	<section>
	<h2>Первая секция</h2>
	<p>Она обо мне</p>
	<img src="images/image.png" alt="Человек и пароход">
	<p>Но может быть и о семантике, я пока не решил.</p>
	</section>
	<section>
	<h2>Вторая секция</h2>
	<p>Она тоже обо мне</p>
	</section>
	<section>
	<h2>И третья</h2>
	<p>Вы уже должны были начать догадываться.</p>
	</section>
	</article>
	</main>
	<footer>
	<p>Сюда бы я вписал информацию об авторе и ссылки на другие сайты</p>
	</footer>
	</body>
	</html>
sudo reboot

// Устанавливаем DNS-сервер на VPN-сервере
// domain-needed - настраивает DNS-сервер таким образом, чтобы он не пересылал имена без точки (.) или доменного имени вышестоящим серверам. Все имена без точки или домена остаются в локальной сети
// bogus-priv - запретить DNS-серверу пересылать запросы обратного просмотра локального диапазона IP-адресов на вышестоящие DNS-серверы. Это предотвращает утечку данных из локальной сети на вышестоящие серверы.
// no-resolv - прекращает чтение вышестоящих серверов имен из файла /etc/resolv.conf, полагаясь вместо этого на серверы в конфигурации DNSMasq.
sudo apt install dnsmasq
sudo nano /etc/dnsmasq.conf
	domain-needed
	bogus-priv
	no-resolv
	server=8.8.8.8
	server=8.8.4.4
	cache-size=1000
sudo reboot
sudo systemctl restart dnsmasq
sudo systemctl status dnsmasq

// Устанавливаем тестировщик DNS. Первый раз время поиска большое. Второй раз он равно 0 брагодаря кешированию
sudo apt install dnsutils
dig onliner.by @localhost
dig onliner.by @localhost

sudo apt-get install iptables
sudo iptables -I INPUT -p udp --dport 51820 -j ACCEPT

// Устанваливаем WireGuard
sudo apt install wireguard
// Генерируем ключи в файлах (похожим образом работает и 'echo $RANDOM | md5sum | head -c 32 | base64')
wg genkey | tee wg-server-private.key |  wg pubkey > wg-server-public.key | wg genpsk > wg-server-preshared.key
wg genkey | tee wg-mobile001-private.key |  wg pubkey > wg-mobile001-public.key | wg genpsk > wg-mobile001-preshared.key
// Генерируем доп.общий ключ (параметр PresharedKey в секции [Peer])
wg genpsk | tee wg-server-preshared.key
// Создаем конфигурацию для сервера
// Что такое [Interface]SaveConfig = true ???
// Номер порта может быть выбран произвольно, однако порт 443 редко блокируется брандмауэром в общедоступных сетях, таких как сети отелей // или общественного транспорта.
// Включение-отключение отладки: ifconfig wg0 debug или ifconfig wg0 -debug
// Поддерживается DNS в настройках для клиента
sudo nano /etc/wireguard/wg0.conf:
	[Interface]
	Address = 172.30.100.1/24
	ListenPort = 51820
	PrivateKey = <copy private key from wg-server-private.key>
	PostUp   = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
	PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
	[Peer]
	# device1
	PublicKey = <copy public key from wg-laptop-public.key>
	AllowedIPs = 172.30.100.2/32
	[Peer]
	# device2
	PublicKey = <copy public key from wg-mobile-public.key>
	AllowedIPs = 172.30.100.3/32
sudo wg-quick up wg0 (sudo wg-quick down wg0)
// А зачем это: sudo systemctl start wg-quick@wg0?
sudo systemctl enable wg-quick@wg0.service
ifconfig
// В более сложных случаях может помочь параметр FwMark, который задаётся в секции [Interface].
// Он задаёт метку пакетов (32-битное число) для исходящих пакетов WireGuard.
// По этой метке к пакетам можно применить дополнительные правила фильтрации или даже применить к ним другую таблицу маршрутизации

// Смотрим за состоянием сервера и потребленным трафиком
sudo wg

// Даем возможность вносить данные клиентов в КУЭР-код
sudo apt install qrencode
qrencode -t ansiutf8 < mobile001.conf
qrencode -o m001.png <  mobile001.conf

// Разрешаем перенаправление пакетов
sudo nano /etc/sysctl.conf
	net.ipv4.ip_forward = 1

// Устанавлием сервер доступа к файлам
sudo apt install vsftpd
systemctl status vsftpd
sudo nano /etc/vsftpd.conf
	anonymous_enable=NO
	local_enable=YES
	write_enable=YES
sudo systemctl restart vsftpd
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/vsftpd.pem -out /etc/ssl/private/vsftpd.pem
sudo nano /etc/vsftpd.conf
	rsa_cert_file=/etc/ssl/private/vsftpd.pem
	rsa_private_key_file=/etc/ssl/private/vsftpd.pem
	ssl_enable=YES
systemctl restart vsftpd

// Устанавливаем менеджер для файлов (разрешить права любому файлу: chmod 777 filename)
sudo apt install mc

// Пример конфигурации для клиента Android:
[Interface]
Address = 172.30.100.4/24
PrivateKey = mCJ9SNM7mBsBN1MzOmH81xGTZvRBJrc7ZlMBqH+gYE0=
DNS = 172.30.100.1
[Peer]
PublicKey = Xn3d/8mMqOK5oVL2C9NVItWoP4qWX63yWaxoOeGRvkQ=
AllowedIPs = 0.0.0.0/0 (весь трафик) или AllowedIPs = 172.30.100.0/24 (не весь трафик)
EndPoint = 78.10.222.186:51820 (поддерживается DNS!)
PersistentKeepalive = 20
