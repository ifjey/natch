<div style="page-break-before:always;">
</div>

# <a name="natch_applications"></a>13. Примеры использования Natch

## 13.1. Соединение нескольких эмуляторов в сеть

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

Здесь `ens33` --- это реальный адаптер на хосте, который подключается к виртуальной сети.

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

Внутри виртуальной машины нужно настроить ip-адрес и маршрутизацию в соответствии с конфигурацией сети.

После использования скрипта `natch_run.py` нужно откорректировать все сгенерированные скрипты:
`run_qemu.sh`, `run_record.sh`, `run_replay.sh`. Строку вида `-netdev user,id=net0 -device e1000,netdev=net0`
нужно заменить на фрагмент, как в скрипте выше:
```bash
-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
-device e1000,netdev=net0,mac=50:54:00:00:00:43
```

Соответственно, для каждой из виртуальных машин tap-интерфейсы и mac-адреса должны быть разными,
если они запускаются одновременно.

[Страница на английском языке с описанием аналогичного опыта.](https://werewblog.wordpress.com/2015/12/31/create-a-virtual-network-with-qemukvm/comment-page-1/)

## 13.2. Natch в режиме командной строки

При записи/воспроизведении сценария на сервере без графической подсистемы,
можно использовать *Natch* в режиме командной строки.

В примере используется подготовленный разработчиком [тестовый комплект](https://nextcloud.ispras.ru/index.php/s/testing_natch),
включающий в себя минимизированный образ гостевой операционной системы Debian (размер qcow2-образа около 1 ГБ).

Приведённый ниже способ (тюнинг GRUB, сетевые настройки и т.д.) в вашем дистрибутиве Linux может отличаться от работающего в тестовом образе.

Создадим скрипт `run1.sh` для запуска виртуальной машины в эмуляторе. Сначала с использованием
графического режима, потому что мы его ещё не отключили:

```bash
natch-qemu-x86_64 \
-hda test_image_debian.qcow2 \
-enable-kvm \
-m 4G \
```

Дальше нужно проделать следующее.
Запустить скрипт `run1.sh`, залогиниться `root:root`
Установить неграфическую цель "по умолчанию": `systemctl set-default multi-user.target`

Открыть конфигурацию grub: `vim /etc/default/grub`

Изменить следующие строки (в скачанном образе эти строки уже раскомментированы):

- раскомментировать: `GRUB_TERMINAL=console`
- установить значение: `GRUB_CMDLINE_LINUX_DEFAULT="nomodeset"`
- установить значение: `GRUB_CMDLINE_LINUX="console=ttyS0"`

Сохранить, после этого выполнить: `update-grub`

Завершить работу: `shutdown 0`

Приведенные команды представлены в виде примера настройки образа. В тестовом образе уже все настроено.

Если нужно снова поработать в виртуальной машине без использования *Natch*, можно
изменить скрипт запуска эмулятора -- отключить графический вывод, используя опцию `nographic`.
Управлять виртуальной машиной в этом режиме работы можно за счет проброса монитора QEMU на свободный порт (в данном случае -- 7799)
и добавления ключей `server`,`nowait`, которые  указывают QEMU прослушивать соединения и запускать виртуальную машину, не дожидаясь подключения.
```bash
natch-qemu-x86_64 \
-hda test_image_debian.qcow2 \
-enable-kvm \
-m 4G \
-nographic \
-monitor tcp:0.0.0.0:7799,server,nowait \
```

Данные опции вставляются в Natch-скрипты записи/анализа сценариев автоматически, при указании соответствующего параметра при запуске `natch_run.py`.

Дальше нужно создать с помощью *Natch* проект, как описано в разделе [Создание проекта](6_create_project.md#create_project).

Только нужно будет выбрать текстовый, а не графический, режим работы эмулятора:
```text
Do you want to run emulator in graphic mode? [Y/n] n
```

Дальше, при работе с *Natch*, если нужно подключиться к эмулятору, например, для создания
снапшота, необходимо на хостовой машине открыть новое окно терминала и ввести команду:
```bash
nc -N 0.0.0.0 7799
```
После загрузки интерфейса управления *Natch*, выполнить команду `savevm ready`.

Там же можно завершить эмулятор с помощью команды `quit`.

## 13.3. Запуск виртуальной машины в режиме VNC-сервера

Другой вариант запуска *Natch* на сервере без графического интерфейса -- это использование
встроенной возможности работы через протокол VNC. С помощью него можно работать
с графическим интерфейсом гостевой системы, подключаясь к эмулятору удалённо.

```bash
-vnc :0
```

С таким параметром эмулятор откроет порт для подключения через VNC-клиент.
При этом обычное окно с графическим интерфейсом исчезнет, останется только вывод в консоль.

Дальше остаётся только подключиться к эмулятору по адресу `127.0.0.1:5900` через протокол
VNC (например, с помощью Remote Desktop Viewer) и оттуда работать с гостевой системой
как обычно. Если доступ нужен извне сервера, где запускается эмулятор, необходимо
соответствующим образом настроить маршрутизацию и брандмауэр.

## 13.4. Запуск Natch в контейнере

Для запуска *Natch* в контейнере, нужно использовать ОС, работающую в текстовом
режиме. В предыдущем разделе описано, как можно её настроить.

Для создания контейнера нужен файл `Dockerfile` со следующим содержимым.
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

