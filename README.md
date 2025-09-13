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
<img width="964" height="308" alt="изображение" src="https://github.com/user-attachments/assets/f261de18-e3bd-4a72-a858-27113468ec5b" /><br/>

Посмотрим имя сетевого интерфейса. В нашем случае — enp2s1 на обеих машинах.
```bash
ip link
```
<img width="1133" height="136" alt="изображение" src="https://github.com/user-attachments/assets/2eacb6ce-8445-4057-8bed-38c7977af3f9" /><br/>

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

Ограничим права на его чтение/запись только для root:
```bash
sudo chmod 600 /etc/netplan/01-netcfg.yaml
```

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
На машине А выполним:
```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```
<img width="834" height="288" alt="изображение" src="https://github.com/user-attachments/assets/1bd6319f-af66-4f77-97d4-ab7a1af17156" /><br/>

Так же нам понадобится сгенерировать ключ, он понадобится позже для подключению к контейнеру:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
ssh-add
```
<img width="786" height="441" alt="изображение" src="https://github.com/user-attachments/assets/43c98615-9696-4993-9207-b27f1aaeb788" />


# Установка Docker и Docker Compose на машине А
Выполним установку:
```bash
sudo apt install docker-compose pip
```

Добавим пользователя в группу docker:
```bash
sudo usermod -aG docker vm-a
```

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
<img width="475" height="130" alt="изображение" src="https://github.com/user-attachments/assets/41f6e1a3-4c58-42f1-a1cd-fe3f261a93d6" /><br/>

Так как хост не «видит» контейнеры на ipvlan. Чтобы можно было SSH с 192.168.1.197 подключаться к 192.168.1.200-202, создадим виртуальный интерфейс на хосте:
```bash
sudo ip link add ipvlan0 link enp2s1 type ipvlan mode l2
sudo ip addr add 192.168.1.199/24 dev ipvlan0
sudo ip link set ipvlan0 up
sudo ip route add 192.168.1.200/29 dev ipvlan0
```

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
В стандартном образе cassandra нету SSH сервера, так что соберём образ с ним и доверенными ключами (положить в папку public_keys/), оформив Dockerfile. Так как Docker-контейнер может корректно работать только с одним PID 1 процессом, мы будем использовать супервизор supervisord:
```Dockerfile
FROM cassandra:5.0.5

# Устанавливаем openssh-server и supervisor; удаляем ненужные больше списки apt
RUN apt-get update && apt-get install -y openssh-server supervisor && \
    rm -rf /var/lib/apt/lists/*

# Подготовка директорий
RUN mkdir -p /var/run/sshd /root/.ssh /var/log/supervisor && \
    chmod 700 /root/.ssh && chown root:root /root/.ssh && \
    chmod 777 /var/lib/cassandra /var/log/cassandra && chown -R cassandra:cassandra /var/lib/cassandra /var/log/cassandra

# Копируем публичные ключи
COPY public_keys/ /etc/ssh/public_keys/
RUN cat /etc/ssh/public_keys/*.pub > /root/.ssh/authorized_keys
# Права и владелец для authorized_keys
RUN chmod 600 /root/.ssh/authorized_keys && chown root:root /root/.ssh/authorized_keys

# Запрещаем парольную аутентификацию по SSH
RUN sed -i 's/^#PasswordAuthentication.*$/PasswordAuthentication no/' /etc/ssh/sshd_config
    sed -i 's/^#PubkeyAuthentication.*$/PubkeyAuthentication yes/' /etc/ssh/sshd_config
    sed -i 's/^#PermitRootLogin.*$/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
    sed -i 's/^#AuthorizedKeysFile.*$/AuthorizedKeysFile .ssh\/authorized_keys/' /etc/ssh/sshd_config

# Копируем конфиг supervisord
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# SSH — 22, cassandra — 7000 7001 7199 9042
EXPOSE 22 7000 7001 7199 9042

CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```
Правильней было бы вынести копирование сертификатов в отдельный entrypoint, который бы копировал их при запуске кластера, а не при сборке контейнера. 
<br/>

Создаём docker-compose.yml
```yaml
ersion: '3.8'

services:
  cassandra1:
    build:
      context: .
      dockerfile: Dockerfile.cassandra.ssh
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
      - CASSANDRA_RPC_ADDRESS=192.168.1.200
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
      - CASSANDRA_SEEDS=192.168.1.200
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
      - CASSANDRA_SEEDS=192.168.1.200
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
А так же конфиг supervisord.conf:
```ini
[supervisord]
nodaemon=true
user=root
logfile=/var/log/supervisord.log

[program:sshd]
command=/usr/sbin/sshd -D
autorestart=true
stdout_logfile=/var/log/sshd.log
stderr_logfile=/var/log/sshd.err

[program:cassandra]
command=./usr/local/bin/docker-entrypoint.sh
user=cassandra
group=cassandra
autorestart=true
stdout_logfile=/var/log/cassandra.out
stderr_logfile=/var/log/cassandra.err
```


# Поднимаем кластер
Поднять кластер можно коммандой:
```bash
docker-compose up -d
```

Проверим кластер:
```bash
docker-compose ps
docker-compose network inspect cassandra-net
```
<img width="599" height="131" alt="изображение" src="https://github.com/user-attachments/assets/3491ca7d-fee4-451b-8817-d336d74777e8" />
<img width="894" height="490" alt="изображение" src="https://github.com/user-attachments/assets/38eb1d55-942d-4157-9580-d6a8cedacff2" />


Проверим статус cassandra:
```bash
docker-compose exec -it cassandra1 nodetool status
```
<details> 
  <summary>спойлер</summary>
   На самом деле поднимал только одну ноду из-за ограничений компьютера
  <img width="1138" height="649" alt="изображение" src="https://github.com/user-attachments/assets/b258ac6b-36cc-4d4a-ab99-eaa7e30973bd" />
</details>
<img width="1010" height="197" alt="изображение" src="https://github.com/user-attachments/assets/77c51e98-84b9-49a3-a5c8-8a7fe51ef5a0" />




#  Подключение к контейнеру по SSH с машины А
Для подключения по SSH достаточно набрать комманду:
```bash
ssh root@192.168.1.200
```
<img width="809" height="548" alt="изображение" src="https://github.com/user-attachments/assets/0d35ed27-4f4f-40dc-81c3-dc7dbf6840be" />

# Подключение к контейнеру через cqlsh с машины Б
cqlsh удобно установить и использовать через pipx:
```bash
sudo apt install pipx
pipx install cqlsh
```
Для подключения через cqlsh надо набрать комманду:
```bash
cqlsh root@192.168.1.200
```
<img width="695" height="109" alt="изображение" src="https://github.com/user-attachments/assets/72943103-d7ff-44bf-b498-451000e6362d" />

