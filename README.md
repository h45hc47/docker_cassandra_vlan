Все действия выполнялись на чистой копии Ubuntu 24.04, все пакеты обновлены до последних версий.


---
<details> 
  <summary>за кадром</summary>
   Настройка NAT-сети VMWare
</details>

# Выдача статического IP
Для начала поднимем и добавим в автозапуск службу systemd-networkd:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now systemd-networkd
sudo systemctl status systemd-networkd --no-pager
```
<img width="964" height="308" alt="изображение" src="https://github.com/user-attachments/assets/f261de18-e3bd-4a72-a858-27113468ec5b" /><br/><br/>

Посмотрим имя сетевого интерфейса. В нашем случае — enp2s1 на обеих машинах.
```bash
ip link
```
<img width="1133" height="136" alt="изображение" src="https://github.com/user-attachments/assets/2eacb6ce-8445-4057-8bed-38c7977af3f9" /><br/><br/>

#### На машине А — 192.168.1.197
Создадим конфиг netplan /etc/netplan/01-netcfg.yaml и внесём в него:
```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [192.168.1.197/24]
      routes:
        - to: 0.0.0.0/0
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```
#### На машине Б — 192.168.1.198
Создадим конфиг netplan /etc/netplan/01-netcfg.yaml и внесём в него:
```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [192.168.1.198/24]
      routes:
        - to: 0.0.0.0/0
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```
<br/>

Ограничим права на его чтение/запись только для root:
```bash
sudo chmod 600 /etc/netplan/01-netcfg.yaml
```
<br/>

Применим и проверим:
```bash
sudo netplan try
ip addr show enp2s1
```
#### На машине А — 192.168.1.197
<img width="1003" height="131" alt="изображение" src="https://github.com/user-attachments/assets/a2301c60-6a78-45b8-82fc-2c7db5aa163b" />

#### На машине Б — 192.168.1.198
<img width="1008" height="130" alt="изображение" src="https://github.com/user-attachments/assets/b266e695-03dd-4ac9-8258-7a433169432b" />


# Установка SSH
На обеих машинах выполним:
```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```
<img width="834" height="288" alt="изображение" src="https://github.com/user-attachments/assets/1bd6319f-af66-4f77-97d4-ab7a1af17156" /><br/><br/>

Проверим подключение:
#### На машине А — 192.168.1.197
```bash
ssh vm-b@192.168.1.198
```
<img width="803" height="480" alt="изображение" src="https://github.com/user-attachments/assets/2b3a0c89-d008-4a16-8c25-3f7440a221dd" />

#### На машине Б — 192.168.1.198
```bash
ssh vm-a@192.168.1.197
```
<img width="808" height="640" alt="изображение" src="https://github.com/user-attachments/assets/aa284ab1-96b0-4a89-82e5-9b95d99c2654" />


# Установка Docker и Docker Compose на машине А
Выполним установку:
```bash
sudo apt install docker-compose pip
```
<br/>

Добавим пользователя в группу docker:
```bash
sudo usermod -aG docker vm-a
```
<br/>

Проверим, что всё установилось и работает:<br/>
<img width="474" height="612" alt="изображение" src="https://github.com/user-attachments/assets/2e8191ea-c58c-45b6-91e3-893958469299" /><br/>
<img width="478" height="107" alt="изображение" src="https://github.com/user-attachments/assets/858c69a4-dc02-4369-a302-0657757cd4c3" />


# Создание Docker ipvlan сети, чтобы контейнеры получили IP 192.168.1.200–202
На машине А выполним:
```bash
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=enp2s1 \
  --ip-range=192.168.1.200/29 \
  --aux-address="host_reserved=192.168.1.199" \
  cassandra-net
```
Проверим:
```bash
docker network ls
```
<img width="475" height="130" alt="изображение" src="https://github.com/user-attachments/assets/41f6e1a3-4c58-42f1-a1cd-fe3f261a93d6" /><br/><br/>

Так как хост не «видит» контейнеры на macvlan. Чтобы можно было SSH с 192.168.1.197 подключаться к 192.168.1.200-202, создадим виртуальный интерфейс на хосте:
```bash
sudo ip link add ipvlan0 link enp2s1 type ipvlan mode l2
sudo ip addr add 192.168.1.199/24 dev ipvlan0
sudo ip link set ipvlan0 up
sudo ip route add 192.168.1.200/29 dev ipvlan0
```
<br/>

Чтобы он создавался автоматически при каждом входе в систему добавим сервис systemd:
```ini
[Unit]
Description=Create ipvlan0 on enp2s1
After=network.target
Wants=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/modprobe ipvlan
ExecStart=/sbin/ip link add ipvlan0 link enp2s1 type ipvlan mode l2
ExecStart=/sbin/ip addr add 192.168.1.199/24 dev ipvlan0
ExecStart=/sbin/ip link set ipvlan0 up
ExecStart=/sbin/ip route add 192.168.1.200/29 dev ipvlan0
ExecStop=/sbin/ip link delete ipvlan0 || true

[Install]
WantedBy=multi-user.target
```
Включаем и запускаем:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ipvlan0.service
```


# Подготавливаем Dockerfile и docker-compose.yml
В стандартном образе cassandra нету SSH сервера, так что соберём образ с ним и доверенным ключом, оформив Dockerfile:
```Dockerfile
FROM cassandra:5.0.5

# Устанавливаем openssh-server
RUN apt-get update && apt-get install -y openssh-server && \

# Подготовка директорий
RUN mkdir -p /var/run/sshd /root/.ssh && \
    chmod 700 /root/.ssh && chown root:root /root/.ssh

# Копируем публичный ключ
COPY public_key.pub /root/.ssh/authorized_keys

# Права и владелец для authorized_keys
RUN chmod 600 /root/.ssh/authorized_keys && chown root:root /root/.ssh/authorized_keys



EXPOSE 22

CMD ["/usr/sbin/sshd","-D"]
```














docker-compose.yml
```yaml
version: '3.8'
services:
  cassandra1:
    image: cassandra:5.0.5
    container_name: cassandra1
    networks:
      cassandra-net:
        ipv4_address: 192.168.1.200
    environment:
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_DC=DC1
      - CASSANDRA_RACK=RACK1
      - CASSANDRA_BROADCAST_ADDRESS=192.168.1.200
      - CASSANDRA_LISTEN_ADDRESS=192.168.1.200
    volumes:
      - cassandra1-data:/var/lib/cassandra

  cassandra2:
    image: cassandra:5.0.5
    container_name: cassandra2
    networks:
      cassandra-net:
        ipv4_address: 192.168.1.201
    environment:
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_DC=DC1
      - CASSANDRA_RACK=RACK1
      - CASSANDRA_SEEDS=cassandra1
      - CASSANDRA_BROADCAST_ADDRESS=192.168.1.201
      - CASSANDRA_LISTEN_ADDRESS=192.168.1.201
    volumes:
      - cassandra2-data:/var/lib/cassandra

  cassandra3:
    image: cassandra:5.0.5
    container_name: cassandra3
    networks:
      cassandra-net:
        ipv4_address: 192.168.1.202
    environment:
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_DC=DC1
      - CASSANDRA_RACK=RACK1
      - CASSANDRA_SEEDS=cassandra1
      - CASSANDRA_BROADCAST_ADDRESS=192.168.1.202
      - CASSANDRA_LISTEN_ADDRESS=192.168.1.202
    volumes:
      - cassandra3-data:/var/lib/cassandra

volumes:
  cassandra1-data:
  cassandra2-data:
  cassandra3-data:

networks:
  cassandra-net:
    external: true
```
