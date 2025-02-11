# Otus Homework 22. VPN.
### Цель домашнего задания
Создать домашнюю сетевую лабораторию. Научится настраивать VPN-сервер в Linux-based системах.
### Описание домашнего задания
1. Настроить VPN между двумя ВМ в tun/tap режимах. Замерить скорость в туннелях. Сделать вывод об отличающихся показателях
2. Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на ВМ
## Выполнение
### VPN между двумя ВМ
С помощью _vagrant_ развернем тестовый стенд из двух виртуальных машин:
|Имя|IP-адрес|
|-|-|
|Server|192.168.56.10|
|Client|192.168.56.20|
  
На каждую ВМ установить _OpenVPN_, а также _iperf_ для тестирования пропускной способности туннеля.
```bash
apt update && apt install -y openvpn iperf3
```
На певром сервере сгенерируем файл ключ для впн сервера и скопируем его на второй
```bash
openvpn --genkey secret /etc/openvpn/static.key
```
Для _tap_ режиме конфигурационный файл сервера **server** будет выглядеть следующим образом:
#### /etc/openvpn/server.conf
```bash
dev tap 
ifconfig 10.10.10.1 255.255.255.0 
topology subnet 
secret /etc/openvpn/static.key 
comp-lzo 
status /var/log/openvpn-status.log 
log /var/log/openvpn.log  
verb 3 
```
На сервере **client** необходимо заменить IP-адрес на _10.10.10.2_. Создадим systemd unit для запуска сервера:
#### /etc/systemd/system/openvpn@.service
```bash
[Unit] 
Description=OpenVPN Tunneling Application On %I 
After=network.target 
[Service] 
Type=notify 
PrivateTmp=true 
ExecStart=/usr/sbin/openvpn --cd /etc/openvpn/ --config %i.conf 
[Install] 
WantedBy=multi-user.target
```
После чего запустим запустим OpenVPN сервер на обеих ВМ:
```bash
systemctl daemon-reload
systemctl start openvpn@server.service
systemctl enable openvpn@server.service
```
С помощью iperf замери скорость через впн туннель. На **server** запустим в режиме сервера:
```bash
iperf3 -s
```
На **client**:
```bash
iperf3 -c 10.10.10.1 -t 40 -i 5
```
![iperf1](img/iperf1.jpg)  
  
Для выполнения задания с помощью _ansible_ запустим плэйбуки:
```bash
ansible-playbook vpn-tap.yml
```
Для изменения режима на _tun_, необходимо в обоих конфигурационных файлах изменить строчку `dev tun`.  

---
Ключевое отличие этих режимов в том, что **TAP** оперирует Ethernet кадрами работает на кальном уровне модели OSI, в то время как **TUN** работает непосредственно с IP-адресами на сетевом уровне.

---
Применим изменения, перезапустив службу и снова измерим пропускную способность:
```bash
systemctl restart openvpn@server.service
```
![iperf2](img/iperf2.jpg)

В нашем случае в рамках тестовой среды результаты примерно одинаковые. На практике _tun_ режим обладает более высокой производительность и испытывает меньшую нагрузку, так как он работает работает только с IP-пакетами и не занимается обработкой широковещательных пакетов, которые могут потребовать дополнительных ресурсов.

Для выполнения задания с помощью _ansible_ запустим плэйбук:
```bash
ansible-playbook vpn-tun.yml
```
### RAS VPN
**Remote Access Server** VPN используется для получения удаленного доступа клиентами к внутренней сети предприятия.
Установим пакет _easy-rsa_, создадим центр сертификации и сгенерируем сертифкаты сервера и клиента:
```bash
apt install - y easy-rsa
cd /etc/openvpn
/usr/share/easy-rsa/easyrsa init-pki
/usr/share/easy-rsa/easyrsa build-ca
/usr/share/easy-rsa/easyrsa gen-req server nopass
/usr/share/easy-rsa/easyrsa gen-req client nopass
/usr/share/easy-rsa/easyrsa sign-req server server 
/usr/share/easy-rsa/easyrsa sign-req client client
```
#### /etc/openvpn/server.conf
port 1194
proto udp
dev tun
server 10.10.10.0 255.255.255.0
topology subnet
ca /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key /etc/openvpn/server.key
cipher AES-256-GCM
auth SHA1
dh none
persist-key
persist-tun
client-to-client
keepalive 10 120
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```
Запустим сервер:
```bash
systemctl start openvpn@server.service
```
Добавим конфгурационный файл в OpenVPN клиент, установленный на нашем хосте. Для удобства добавим содержимо сертификатов несопредственно в конфигурационный файл:
```bash
port 1194
proto udp
dev tun
remote 192.168.56.10 1194
client
resolv-retry infinite
#route 192.168.56.0 255.255.255.0
cipher AES-256-GCM
auth SHA1
persist-key
persist-tun
comp-lzo
verb 3
<ca>
-----BEGIN CERTIFICATE-----
MIIDMDCCAhigAwIBAgIUCGRMVFjIlwvqrrsHe828NayYViowDQYJKoZIhvcNAQEL
BQAwDTELMAkGA1UEAwwCQ0EwHhcNMjQwNzI5MDg0MDExWhcNMzQwNzI3MDg0MDEx
WjANMQswCQYDVQQDDAJDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
AMO0p8T+it9pKksxvbqp1xUnpf9PJGD/qD/fdBN8vQ9XdZtRqijX4VIQk3f5bknu
fPZkjnSzfCgOH79ITvVO362hllip1QRtbFmhmQVkN0v3kw0iLCMiopDcfsettFHT
RTlO56RVDLdtPvOLMfOaRprM1Q14Tc4IuXuBCpEFGFsIeK9E7mqReBR19lNKscvx
EBF51CbqqKJOfMlvp+nJSF4OnS73koMnx290Zi+ZPq6VLb7sIuFN9RlwhYVZY6dG
Pc7z6B+/YPM/U7GcHjDQ3FvBowyxaxCvSuvYJUDtC1hsz91xfujgfWjGd0psztIe
KXo0IgHF9XilbfaDW9dhLJUCAwEAAaOBhzCBhDAdBgNVHQ4EFgQUD7V+G19y0YK6
JMbcPSjB/IRS5yEwSAYDVR0jBEEwP4AUD7V+G19y0YK6JMbcPSjB/IRS5yGhEaQP
MA0xCzAJBgNVBAMMAkNBghQIZExUWMiXC+quuwd7zbw1rJhWKjAMBgNVHRMEBTAD
AQH/MAsGA1UdDwQEAwIBBjANBgkqhkiG9w0BAQsFAAOCAQEAnkws99pTyLcLPRi9
1Fo+jyNAeVVVwB18eTSwfQLwWL33bokGcqsy3KCBDD2zOBViOe55LjMk+it+U+CE
T+3BRbHJflGitGrhez0Td6nfzdnPQDeCyVVqSkkXpdjSKx03GLwZ/eA6nAPyscPm
aD75N436XKJlVH7ALQbJ658WGaB3uurdp3dI2ckMItspOlU5hqUMEgaynaFc0+qs
Y8+4/d0kh3OSi1MyqYNfrpuzcW5ythOcVBv94I0SZyBnxDMh61l9lqZ0y2mRwMPA
Pu8O9j5g/Y0EgG8EBypNEipTjceVjKpkMG6GSvRxP6b5XkX8NkOW6Swb9O5nT8Uu
DdlKmg==
-----END CERTIFICATE-----
</ca>
<cert>
-----BEGIN CERTIFICATE-----
MIIDQjCCAiqgAwIBAgIQZO9en+XqIf4iE01Q28CvFTANBgkqhkiG9w0BAQsFADAN
MQswCQYDVQQDDAJDQTAeFw0yNDA3MjkwODQyMjNaFw0yNjExMDEwODQyMjNaMBEx
DzANBgNVBAMMBmNsaWVudDCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
AKkP9Atbs+8+QqD8PP6tziES3/Uc7fK1/Dz9jlOe3AaF+A3PQ/PPrT2ogPsXL6ms
S757MAT7Av70iTIde+KlYSyTpblkmByDEL2+/NeWnOy3CVwWEjXF3uMWWWE5OMbw
MhNADfyX+VrVlsvI4N1Uw6LfKv/cpgnwxI4Qtgzeg2H/Tx9H5UkXnJhn39XQAwhN
wjJyLvB4oQsNTSILNCtHhMG9rmNu9R17fmoB4JsZs2AZlkdmHvF/uJlVaE+g6TRW
bZB+W6zPAnLY44SHvGTayeW3mIE5ZTdHB4Wu1sI1ZrD6qPgfsBezLNxqihQwzAgx
6EhC9Qqw/NBAtZsflkc4nGcCAwEAAaOBmTCBljAJBgNVHRMEAjAAMB0GA1UdDgQW
BBSxCD6/7+rDjuK5s83yTa2x4vQIaTBIBgNVHSMEQTA/gBQPtX4bX3LRgrokxtw9
KMH8hFLnIaERpA8wDTELMAkGA1UEAwwCQ0GCFAhkTFRYyJcL6q67B3vNvDWsmFYq
MBMGA1UdJQQMMAoGCCsGAQUFBwMCMAsGA1UdDwQEAwIHgDANBgkqhkiG9w0BAQsF
AAOCAQEAsxbEoEYkv4G37kyPfMCW/J6C5grqvnhcanOE/DcO5jzx5RP4Sk9hZ6lv
3KtyKMXXj1OTW0b9KuGUVNCUsObZPbM8bw6dHmcpxGN+U/h9RjJ5NJflVJsCVA3g
/gzzR0bsqkwtwjbYYbxg29W7G8h/G6kAQSo8UdQa1FHH9EHbU3rU9Ysgu8S+VZ/b
gRfyx0QZ4cmTG8ORs1DRuxZvulDL62n+ipSqaq18eLk/7nXvE8XHOHgyvmfyH3Fu
tJWNbObyetIqkgCLecNq6T3wITxy7sMjN9BjeTXdJC43XV4woD3cN485+3Ck4s7d
3+RaG11ye7z5wkX3dfW8XuAOMLYyIg==
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQCpD/QLW7PvPkKg
/Dz+rc4hEt/1HO3ytfw8/Y5TntwGhfgNz0Pzz609qID7Fy+prEu+ezAE+wL+9Iky
HXvipWEsk6W5ZJgcgxC9vvzXlpzstwlcFhI1xd7jFllhOTjG8DITQA38l/la1ZbL
yODdVMOi3yr/3KYJ8MSOELYM3oNh/08fR+VJF5yYZ9/V0AMITcIyci7weKELDU0i
CzQrR4TBva5jbvUde35qAeCbGbNgGZZHZh7xf7iZVWhPoOk0Vm2QfluszwJy2OOE
h7xk2snlt5iBOWU3RweFrtbCNWaw+qj4H7AXsyzcaooUMMwIMehIQvUKsPzQQLWb
H5ZHOJxnAgMBAAECggEABQzuyky924aoIJK4fcE9IwbN0vmUxYdDQaKE1yGr9V6Y
3Pdqp8TLVPK3kQi4dKCq40YJ1rRAUCkyVLbFxde78e7NPh8aXjZaJdj9SQbSVE4E
L7s8ZbKDaJROgkAz3oQ8HWVwpYb4EjXG8KU6dnzlmp7WdC39ddblVaXRJ8xD74Wl
CTAHxjgRYOZGnVhUhWyPpk99NG3Uq2BzsUb7yRxnRqVNW4+PLbEfbOSxV/ZB4m13
aQzWWJ5tdfYhmc6mlZ1mW4RGop2nPmCZMMR0bafxw0IdiDiMOGOCalyQkZ00L5y5
oCGdnQfT18sGJnkRHRYX8xn9BcwiAe75+BHSjWWCpQKBgQC8XdP0igDZjjJ+HZzW
nE4gnzGfo1/6EJtw036bIM1l40RmUdrtFs9voQpR0bMy4a8vkT6iNaNpzBYgoaje
n+LXx7khOcr+gH63mXAUXF0RTxABxQRZMFYDAX55NAYFQ6y3VnfHhsZatxTquybs
xbfsxu0l8kITOGi05pLxbbenVQKBgQDlw7uDUGFeHycrxDybAkVbxJKsS9l2SbB2
VITrJNJUsNnpDFZIaajAcBSewy0eqd5ASeYZwbV2b4Ebn2Dg/H4MnAi+jLGGuKTN
krQf91tXtv1DDOrUEk8fbPBILNKigKV0V3algyjl+7pE3VBymkCppdMOx7xpT+LT
IWjub+M8ywKBgQCMPO7IaNYpIozFCBb0UHp6Hws65s9VxXd0kID5zXoeGQ2bf+WW
Dh1x5ltgftcDUrKyn1gaPATlh2QR90laNX8VV0SlT/mpcNDmr/2Zqwo/ELXCG4QZ
QrtGkZ4vbmPtF21HMcELc3PJpfSUrbFVJf7A8XktfydiV+Tcia1swVqx4QKBgFHq
L38IeD47MxbqdoT5EUs/UN92h0ghy3TUezLuRMKG7pmkmVpluREqpF9ZzEtDWoZn
Ek8afZyE8m2rq7lqq3HJa2Cr/lq+l5rm86r14C3sgmyWPV5wTJ8ykpPYzxu6a8KH
sDggA8PCtEz67kR9dBJHmXCKi0Ssg3ysS6G+aDBzAoGANna7c3qsB0CTFjqgjQK0
5HtwPglH8jrG195VI12/aj+ejFxRGcWDQ0j46Ugt0phgDa0eByB23ZknaJb2i37b
cGqk0GIWRy4iZOYJWuhpLT/fMmtlhJBlR4F8L4Qtjt+6NIz7SUSDgWt3YFwFB6u1
8bh7I1mGQAWDtgfjaT4i6oI=
-----END PRIVATE KEY-----
</key>
```
Мы успешно подключились:  
  
![ras](img/ras.jpg)

Для выполнения задания с помощью _ansible_ запустим плэйбук:
```bash
ansible-playbook vpn-ras.yml
```
