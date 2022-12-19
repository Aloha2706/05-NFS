# 05-nfs

## Создаём тестовые виртуальные машины:

Берем предварительно настроенный Vagranfile:

        https://cdn.otus.ru/media/private/b7/1b/Vagrantfile-37455-b71b83?hash=25QFt2MfOJX8l2QTLyaMzw&expires=1671393392

коментируем эти строки и запускаемся

        #nfsc.vm.provision "shell", path: "nfsc_script.sh"
## 3) Настраиваем сервер NFS

        vagrant ssh nfss

ставим утилиты:

        yum install -y nfs-utils

Включаем фаервол:
        systemctl enable firewalld --now
        systemctl status firewalld

разрешаем в firewall доступ к сервисам NFS

        firewall-cmd --add-service="nfs3" \
        --add-service="rpc-bind" \
        --add-service="mountd" \
        --permanent

        firewall-cmd --reload

        [root@nfss vagrant]# firewall-cmd --list-all
        public (active)
        target: default
        icmp-block-inversion: no
        interfaces: eth0 eth1
        sources:
        services: dhcpv6-client mountd nfs3 rpc-bind ssh
        ports:
        protocols:
        masquerade: no
        forward-ports:
        source-ports:
        icmp-blocks:
        rich rules:


включаем сервер NFS (для конфигурации NFSv3 over UDP он не требует
дополнительнойнастройки, однако вы можете ознакомиться с
умолчаниями в файле __/etc/nfs.conf__)

        systemctl enable nfs --now

проверяем что необходимые порты прослушиваются.  

        ss -tnplu |grep '111\|2049\|20048'

создаём и настраиваем директорию, которая будет экспортирована
в будущем

        mkdir -p /srv/share/upload
        chown -R nfsnobody:nfsnobody /srv/share
        chmod 0777 /srv/share/upload

создаём в файле __/etc/exports__ структуру, которая позволит экспортировать ранее созданную директорию. вот таким вот хитрым способом. Надо запомнить. )))

        cat << EOF > /etc/exports
        /srv/share 192.168.50.11/32(rw,sync,root_squash)
        EOF

где:
rw — разрешение на запись
ro — только чтение
sync — синхронный режим доступа. sync (async) — указывает, что сервер должен отвечать на запросы только после записи на диск изменений, выполненных этими запросами. Опция async указывает серверу не ждать записи информации на диск, что повышает производительность, но понижает надежность
no_root_squash — по умолчанию пользователь root на клиентской машине не будет иметь доступа к разделяемой директории сервера. Этой опцией мы снимаем это ограничение.
no_all_squash — включение пользовательской авторизации
all_squash — все подключения будут выполнятся от анонимного пользователя.


экспортируем ранее созданную директорию,
exportfs - ведение таблицы экспортированных файловых систем NFS


        exportfs -r
        exportfs -av перечитываем файл конфига /etc/exports
        проверяем:
        exportfs -s
        /srv/share  192.168.50.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,root_squash,no_all_squash)

## 4) Настраиваем клиент NFS

Логинимся 

        vagrant ssh nfsc

Стартуем нужные сервисы

        systemctl start rpcbind
        systemctl enable rpcbind

и настраиваем фаервол.

        systemctl enable firewalld --now
        systemctl status firewalld

Настраиваем fstab

        echo "192.168.50.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0" >> /etc/fstab

и запускаем вот это :

        systemctl daemon-reload
        systemctl restart remote-fs.target

Отметим, что в данном случае происходит автоматическая генерация
systemd units в каталоге `/run/systemd/generator/`, которые производят
монтирование при первом обращении к каталогу `/mnt/` 

Проверяем примонтировалась ли у нас fs

        mount |grep mnt

и видим что ничего не примонтировалось. 
При подробном расследовании оказалось что при записи в /etc/fstab строка  x-systemd.automount скопировалась без дефиса. 
x-systemd.automount Это функция автомонтирования обязательно ее изучи.

/etc/systemd/system/ здесь опции монтирования. 

ну и это до кучи 
man systemd-fstab-generator

исправляем проверяем получаем:

        systemd-1 on /mnt type autofs (rw,relatime,fd=25,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=20721)
        192.168.50.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.50.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.50.10)

значит все примонтировалось как надо. 

создаем файлы на сервере в директории /srv/share/upload
и на клиенте в директории /mnt/upload
если все файлы созданы, значит права доступа настроены правильно.

Проверяем сервер:
- заходим на сервер в отдельном окне терминала
- перезагружаем сервер
- заходим на сервер
- проверяем наличие файлов в каталоге `/srv/share/upload/`
- проверяем статус сервера NFS `systemctl status nfs`
- проверяем статус firewall `systemctl status firewalld`
- проверяем экспорты `exportfs -s`
- проверяем работу RPC `showmount -a 192.168.50.10`

Проверяем клиент:
- возвращаемся на клиент
- перезагружаем клиент
- заходим на клиент
- проверяем работу RPC `showmount -a 192.168.50.10`
- заходим в каталог `/mnt/upload`
- проверяем статус монтирования `mount | grep mnt`
- проверяем наличие ранее созданных файлов
- создаём тестовыйфайл `touch final_check`
- проверяем, что файл успешно создан

После этого создаем два  bash-скрипта, `nfss_script.sh` и `nfsc_script.sh` 
Уничтожаем тестовый стенд командой `vagrant destory -f` 
И снова создаем виртуальные машины. 
После загрузки проверяем что скрипты отработали и машины настроены. 







