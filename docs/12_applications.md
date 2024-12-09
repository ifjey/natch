<div style="page-break-before:always;">
</div>

# <a name="natch_applications"></a>12. Примеры использования Natch
## 12.1. Создание сетевого моста. Подключение эмулятора к рабочей сети.

Может возникнуть ситуация, когда компоненты анализируемой системы для корректной работы взаимодействуют с корпоративной сетью. Т.к. при обычном запуске qemu используется NAT, возникают сложности с настройкой vpn протоколов.

При данных настройках рекомендуется использовать подключение через Ethernet кабель.

Сначала определим сетевой интерфейс:
```bash
sudo lshw -class network
```

Данная команда отобразит название интерфейса и диапазон нашей рабочей сети, что упростит понимание дальнейших действий:
```bash
modprobe tun tap
sudo ip link add br0 type bridge
sudo ip tuntap add dev tap0 mode tap
sudo ip link set dev ens33 master br0
sudo ip link set dev tap0 master br0
sudo ip link set dev br0 up
sudo ip link set dev tap0 up
sudo ip address delete 192.168.159.131/32 dev ens33
sudo ip address delete 192.168.159.131 dev ens33
sudo ip address add 192.168.159.131/24 dev br0
```

Далее определим основной шлюз сети. Если хостовая операционная система - windows, то используем 'ipconfig'. Узнав нужный адрес, прописываем маршрут до шлюза при помощи ip route: 
```bash
sudo ip route add default via 192.168.159.2 dev br0
sudo resolvectl dns br0 192.168.159.2
```
Ждём 15 секунд, если установилось интернет соединение, то вы все сделали правильно.  

Для корректного проброса сети в qemu vm необходимо указать измененный mac tap интерфейса. (Таким образом можно избежать конфликт с dhcp сервером.) Генерируем mac, оставляя первые 3 октета от нашего tap интерфейса:
```bash
mac_address="fe:56:c2:$(dd if=/dev/urandom bs=512 count=1 2>/dev/null \
 | md5sum \
 | sed -E 's/^(..)(..)(..).*$/\1:\2:\3/')"
echo $mac_address```

Далее запустим qemu vm, используя полученный mac-адрес:
```bash
qemu-system-x86_64 \
-hda ubuntu.qcow2 \
-m 4G \
-enable-kvm \
-cpu host,nx \
-monitor stdio \
-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
-device e1000,netdev=net0,mac=fe:56:c2:f2:0b:e1
```

## 12.2. Соединение нескольких эмуляторов в сеть

Иногда анализируемая система состоит из компонентов, размещаемых на нескольких компьютерах,
объединённых в сеть. *Natch* напрямую не поддерживает такой режим, но его можно использовать
для каждой из отдельных виртуальных машин (или для всех одновременно):

- *Natch* настраивается для каждой из виртуальных машин, подлежащих анализу
- Запускаются все обычные виртуальные машины
- Запускаются машины под контролем *Natch* в режиме записи сценария
- Выполняется сценарий для анализа
- Завершается работа виртуальных машин
- По отдельности запускается воспроизведение сценария и получение поверхности атаки для каждой машины
- Полученные файлы поверхности атаки по отдельности анализируются в *SNatch*

Таким образом, при работе с несколькими виртуальными машинами нужно сконфигурировать их сетевые адаптеры,
чтобы они могли взаимодействовать между собой. Для этого можно использовать механизм туннелей.


Сконфигурировать туннель из tap-адаптеров на хосте можно с помощью следующих команд:
```bash
modprobe tun tap
sudo ip link add br0 type bridge
sudo ip tuntap add dev tap0 mode tap
sudo ip tuntap add dev tap1 mode tap
sudo ip link set dev ens33 master br0
sudo ip link set dev tap0 master br0
sudo ip link set dev tap1 master br0
sudo ip link set dev br0 up
sudo ip link set dev tap0 up
sudo ip link set dev tap1 up
sudo ip address delete 192.168.159.131/32 dev ens33
sudo ip address delete 192.168.159.131 dev ens33
sudo ip address add 192.168.159.131/24 dev br0
sudo ip route add default via 192.168.159.2 dev br0
sudo resolvectl dns br0 192.168.159.2
```
Каждый из эмуляторов нужно запускать с разными tap-адаптерами и mac-адресами.
В командной строке ниже используется `tap0` и `mac=50:54:00:00:00:43`:
```bash
qemu-system-x86_64 \
-hda ubuntu_nginx.qcow2  \
-m 4G \
-enable-kvm \
-cpu host,nx \
-monitor stdio \
-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
-device e1000,netdev=net0,mac=50:54:00:00:00:43
```
После создания проектов командой `natch create` нужно откорректировать параметры запуска эмулятора в файле
`qemu_opts.ini`.

Опции
```
-netdev user,id=net0
-device e1000,netdev=net0
```

нужно заменить следующие, как в скрипте выше:

```bash
-netdev tap,id=net0,ifname=tap0,script=no,downscript=no
-device e1000,netdev=net0,mac=50:54:00:00:00:43
```
для секций `record`, `replay` и, при необходимости, `kvm`.

Соответственно, для каждой из виртуальных машин tap-интерфейсы и mac-адреса должны быть разными,
если они запускаются одновременно.

[Страница на английском языке с описанием аналогичного опыта.](https://werewblog.wordpress.com/2015/12/31/create-a-virtual-network-with-qemukvm/comment-page-1/)

## 12.3. Запуск виртуальной машины в режиме VNC-сервера

Другой вариант запуска *Natch* на сервере без графического интерфейса -- это использование
встроенной возможности работы через протокол VNC. С помощью него можно работать
с графическим интерфейсом гостевой системы, подключаясь к эмулятору удалённо.

```bash
-vnc :0
```

С таким параметром эмулятор откроет порт для подключения через VNC-клиент.
При этом обычное окно с графическим интерфейсом исчезнет, останется только вывод в консоль.

Выбрать такой режим так же можно на этапе создания проекта (необходимо указать vnc):

```text
Select mode you want to run emulator: graphic [G/g] (default), text [T/t] or vnc [V/v] v
```

Дальше остаётся только подключиться к эмулятору по адресу `127.0.0.1:5900` через протокол
VNC (например, с помощью Remote Desktop Viewer) и оттуда работать с гостевой системой
как обычно. Если доступ нужен извне сервера, где запускается эмулятор, необходимо
соответствующим образом настроить маршрутизацию и брандмауэр.

## 12.4. Запуск Natch в контейнере

Для запуска *Natch* в контейнере, нужно использовать ОС, работающую в текстовом
режиме. В предыдущем разделе описано, как можно её настроить.

Для создания контейнера нужен файл `Dockerfile` с представленным ниже содержимым.
В этом же каталоге должен находиться пакет *Natch* для Ubuntu 20.04 --
`natch_X.X_ubuntu2004.deb`.
```bash
FROM ubuntu:20.04

#Set Timezone or get hang during the docker build...
ENV TZ=Europe/Moscow
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

RUN apt update
RUN apt install -y vim git cmake make clang zlib1g-dev unzip curl python3-pip
RUN apt install -y mc sudo
RUN apt install -y qemu-system libguestfs-tools zstd
RUN apt install -y netcat
RUN apt install -y nano
RUN apt install -y unzip
RUN apt install -y libsdl2-2.0-0

ARG cuidname=user
ARG cgidname=user

RUN groupadd $cgidname && useradd -m -g $cgidname -G sudo -p $cuidname -s /usr/bin/bash $cuidname
RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers
ENV PATH="${PATH}:/bin/natch/bin"

COPY natch_Ubuntu20_amd64.deb /home/user

RUN apt install /home/user/natch_X.X_ubuntu2004.deb

USER $cuidname
RUN /bin/natch/bin/natch_scripts/setup_requirements.sh
```

Создадим образ контейнера на основе `Dockerfile`:
```bash
sudo docker build -t docker /home/user/natch_quickstart
```

Последний параметр этой командой строки -- это каталог, где лежит `Dockerfile`.

Теперь запустим созданный контейнер:
```bash
docker run -v /home/user/natch_quickstart/:/mnt/ --network=host -it -u user docker
```

В папке `/home/user/natch_quickstart/` должен быть нужный для работы образ и объект оценки,
потому что она будет подмонтирована в каталог `/mnt` внутри контейнера.

Теперь можно запускать *Natch*:
```bash
cd /mnt
natch create project_name test_image_debian.qcow2
```

