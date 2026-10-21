## Общая конфигурация
Две виртуальные машины:
backup 192.168.56.160 Ubuntu 22.04
client 192.168.56.150 Ubuntu 22.04

В vagrantfile это будет выглядеть следующим образом (часть полноценного скрипта):
```yaml
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  config.vm.provider :virtualbox do |v|
    v.memory = 1024
    v.cpus = 2
  end

  boxes = [
    { name: "backup", ip: "192.168.56.160" },
    { name: "client", ip: "192.168.56.150" }
  ]

  boxes.each do |opts|
    config.vm.define opts[:name] do |srv|
      srv.vm.hostname = opts[:name]
      srv.vm.network "private_network", ip: opts[:ip]
```

## Настройка сервера backup
provision для этой машины будет выглядеть следующим образом:
```yaml
      # Провижининг конкретно для backup
      if opts[:name] == "backup"
        srv.vm.disk :disk, size: "2GB", name: "disk1"
        srv.vm.provision "shell", inline: <<-SHELL
           # Ищем добавленные диски (обычно это /dev/sdb и /dev/sdc)
           # Используем `lsblk` для поиска дисков, у которых нет файловой системы
           DISK1=$(lsblk -dpno NAME | grep -E "/dev/sd[c-d]" | head -n 1)
           # Форматируем в ext4
           mkfs.ext4 -F "$DISK1"
           apt-get update
           apt-get install borgbackup -y
           # Создаём пользователей с домашними каталогами
           # -m создает домашнюю директорию
           # -s /bin/bash задает шелл
           useradd -m -s /bin/bash borg
           echo "borg:borg" | chpasswd
           usermod -U borg
           # Создаем папку и делаем владельцем пользователя и группу borg
           mkdir /var/backup
           # Монтируем диск в созданную директорию
           mount "$DISK1" /var/backup
           # Добавляем записи в /etc/fstab (используем UUID для надежности)
           echo "UUID=$(blkid -s UUID -o value "$DISK1") /var/backup ext4 defaults 0 2" >> /etc/fstab
           chown borg:borg /var/backup/
           rm -rf /var/backup/*
           # Создаем папку для ssh ключей
           su - borg -c "mkdir /home/borg/.ssh"
           cp /vagrant/ssh_key/authorized_keys /home/borg/.ssh/
           chown -R borg:borg /home/borg/.ssh
           # Выдаем права пользователю borg на папку и файл authorized_keys
           su - borg -c "chmod 700 .ssh"
           su - borg -c "chmod 600 .ssh/authorized_keys"
        SHELL
      end
```

### Объяснение блока с настройкой сервера backup
- **Добавляем диск для резервных копий:**
```
# Ищем добавленные диски (обычно это /dev/sdb и /dev/sdc)
           # Используем `lsblk` для поиска дисков, у которых нет файловой системы
           DISK1=$(lsblk -dpno NAME | grep -E "/dev/sd[c-d]" | head -n 1)
           # Форматируем в ext4
           mkfs.ext4 -F "$DISK1"
```

- **Устанавливаем на client и backup сервере borgbackup**

```shell
apt install borgbackup
```

- **Создаём пользователя `borg`**
```
# Создаём пользователей с домашними каталогами
           # -m создает домашнюю директорию
           # -s /bin/bash задает шелл
           useradd -m -s /bin/bash borg
           echo "borg:borg" | chpasswd
           usermod -U borg
```

- **Создаем папку и делаем владельцем пользователя и группу borg**
```
mkdir /var/backup
```

- **Монтируем диск в созданную директорию**
```bash
mount "$DISK1" /var/backup
```
- **Добавляем записи в `/etc/fstab` (используем UUID для надежности)**
```bash
echo "UUID=$(blkid -s UUID -o value "$DISK1") /var/backup ext4 defaults 0 2" >> /etc/fstab
```

- **Выдаем права на директорию `/var/backup` пользователю borg и чистим все содержимое данной папки:**
```bash
chown borg:borg /var/backup/
rm -rf /var/backup/*
```

```ad-info
Чистить директорию нужно потому что при инициализации программе `borgbackup` нужна чистая папка без файлов и папок (==даже скрытых==)
```

- **Создаем папку для ssh ключей:**
```
# Создаем папку для ssh ключей
           su - borg -c "mkdir /home/borg/.ssh"
           cp /vagrant/ssh_key/authorized_keys /home/borg/.ssh/
           chown -R borg:borg /home/borg/.ssh
           #su - borg -c "touch .ssh/authorized_keys"
           su - borg -c "chmod 700 .ssh"
           su - borg -c "chmod 600 .ssh/authorized_keys"
```

```ad-info
Здесь отметим что ключи были созданы ранее и копированы в отдельную подпапку проекта `./ssh-key/`.
```

```ad-warning
Содержимое `authorized_keys` скопировано из файла `id_rsa.pub` который создавался для сервера `client`. 
```

**Команда для создания ключей**
```bash
ssh-keygen
```

- **Выдаем права пользователю borg на папку и файл authorized_keys:**
```
su - borg -c "chmod 700 .ssh"
su - borg -c "chmod 600 .ssh/authorized_keys"
```

**На этом можно считать настройку сервера `backup` законченной.**

## Настройка сервера client
```yaml
      # Провижининг конкретно для client
      if opts[:name] == "client"
        srv.vm.provision "shell", inline: <<-SHELL
           # Устанавливаем borgbackup
           apt-get update
           apt-get install borgbackup sshpass -y
           # Копируем заранее заготовленные ssh ключи 
           cp /vagrant/ssh_key/id_rsa.pub /home/vagrant/.ssh/
           cp /vagrant/ssh_key/id_rsa /home/vagrant/.ssh/
           # Назначаем владельцем пользователя vagrant           
           chown vagrant:vagrant /home/vagrant/.ssh/id_rsa.pub
           chown vagrant:vagrant /home/vagrant/.ssh/id_rsa
           # Выдаем права пользователю vagrant
           chmod 600 /home/vagrant/.ssh/id_rsa
           chmod 644 /home/vagrant/.ssh/id_rsa.pub
           # Добавляем ключи в known_hosts
           ssh-keyscan -p 22 -H 192.168.56.160 >> /home/vagrant/.ssh/known_hosts 2>/dev/null
           # Инициализируем репозиторий borg на backup сервере с client сервера
           # borg init --encryption=repokey borg@192.168.56.160:/var/backup/
           # 1. Создаём unit-файл сервиса
           cat > /etc/systemd/system/borg-backup.service <<'EOF'
Description=Borg Backup

[Service]
Type=oneshot

User=vagrant
Group=vagrant
Environment="HOME=/home/vagrant"

# Парольная фраза
Environment="BORG_PASSPHRASE=borgpass"
# Репозиторий
Environment=REPO=borg@192.168.56.160:/var/backup/
# Что бэкапим
Environment=BACKUP_TARGET=/etc

# Создание бэкапа
ExecStart=-/bin/borg create \
    --stats                \
    ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

# Проверка бэкапа
ExecStart=/bin/borg check ${REPO}

# Очистка старых бэкапов
ExecStart=-/bin/borg prune \
    --keep-daily  90      \
    --keep-monthly 12     \
    --keep-yearly  1       \
    ${REPO}
EOF
           # 2. Создаём unit-файл таймера
           cat > /etc/systemd/system/borg-backup.timer <<'EOF'
[Unit]
Description=Borg Backup

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
EOF
        SHELL
      end
```

**Разберем некоторые этапы подробнее**
Все основное действие происходит в разделе `shell`
- **Установка borgbackup**
```shell
apt-get update
apt-get install borgbackup -y
```

```ad-info
Ключ `-y` нужен для того что бы автоматически согласиться на запрос об установке пакета.
```

- **Копируем заранее заготовленные ssh ключи:**
```shell
cp /vagrant/ssh_key/id_rsa.pub /home/vagrant/.ssh/
cp /vagrant/ssh_key/id_rsa /home/vagrant/.ssh/
```

Тут стоит прояснить пару моментов:
Скрипты создаются отдельно и укладываются в отдельную подпапку проекта  `./ssh-key`, оттуда они копируются уже в определенные папки на создаваемых виртуальных машинах.

```ad-warning
Содержимое ключа `id_rsa.pub` заранее скопировано в файл /home/borg/.ssh/authorized_keys (который в свою очередь тоже был создан заранее и помещен в отдельную подпапку проекта  `./ssh-key`)
```

```ad-info
Папка с проектом обычно монтируется в корневую систему создаваемой виртуальной машины и поэтому путь при копировании указывается сразу из папки `/vagrant` просто `cp /vagrant/ssh_key/id_rsa.pub /home/vagrant/.ssh/`
```

- **Назначаем владельцем пользователя vagrant**
```
chown vagrant:vagrant /home/vagrant/.ssh/id_rsa.pub
chown vagrant:vagrant /home/vagrant/.ssh/id_rsa
```

- **Выдаем права пользователю vagrant**
```
chmod 600 /home/vagrant/.ssh/id_rsa
chmod 644 /home/vagrant/.ssh/id_rsa.pub
```

- **Добавляем ключи в known_hosts**
Это нужно для автоматизации что бы ключ уже был в файле `known_hosts` и не требовал подтверждения при первом подключении. 
```shell
ssh-keyscan -p 22 -H 192.168.56.160 >> /home/vagrant/.ssh/known_hosts 2>/dev/null
```

```ad-note
Возможно этот пункт не обязателен потому что в дальнейшем нам пришлось вручную заходить на сервер `client` и проводить инициализацию удаленной директории `borgbackup`
```

- **Создаём unit-файл сервиса**
```
# 1. Создаём unit-файл сервиса
           cat > /etc/systemd/system/borg-backup.service <<'EOF'
Description=Borg Backup

[Service]
Type=oneshot

# Этот блок нужен для того чтобы указать от какого пользователя будет запущен сервис.
User=vagrant
Group=vagrant
Environment="HOME=/home/vagrant"

# Парольная фраза
Environment="BORG_PASSPHRASE=borgpass"
# Репозиторий
Environment=REPO=borg@192.168.56.160:/var/backup/
# Что бэкапим
Environment=BACKUP_TARGET=/etc

# Создание бэкапа
ExecStart=-/bin/borg create \
    --stats                \
    ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

# Проверка бэкапа
ExecStart=/bin/borg check ${REPO}

# Очистка старых бэкапов
ExecStart=-/bin/borg prune \
    --keep-daily  90      \
    --keep-monthly 12     \
    --keep-yearly  1       \
    ${REPO}
EOF
```

```ad-info
В этом скрипте стоит обратить внимание на пару деталей:
- Скрипт запускается от пользователя `vagrant` и это указано в отдельном блоке.
- В блоке `ExecStart=-/bin/borg create \
    --stats                \
    ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}` команда начинается дефис `-`.  Дефис означает «игнорировать ненулевой код выхода этой команды» 
```

- **Создаём unit-файл таймера**
```
# 2. Создаём unit-файл таймера
           cat > /etc/systemd/system/borg-backup.timer <<'EOF'
[Unit]
Description=Borg Backup

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
EOF
        SHELL
      end
```

Проясним несколько моментов по этому скрипту:
- Добавлена в секцию [Timer] настройка, которая даст таймеру первую точку отсчёта. Самый простой вариант — использовать OnBootSec, чтобы первый запуск произошёл через некоторое время после загрузки системы: `OnBootSec=5min`.
Связано это добавление с тем что таймер после запуска службы не начинал отсчет.
Объяснение `deepseek`:
```ad-note
Ваш таймер не работает, потому что использует OnUnitActiveSec=5min без начальной точки отсчёта. В systemd эта настройка означает «запустить через 5 минут после того, как сервис был активен в последний раз», но у только что запущенного таймера такой «последней активации» ещё нет, поэтому systemd не знает, от чего отсчитывать эти 5 минут, и не планирует запуск .
```

- Предложено тем же `deepseek` еще более изящное решение.
Если вам нужно, чтобы бэкап запускался строго по расписанию (например, каждый час), а не «плавал» в зависимости от времени выполнения предыдущего задания, лучше использовать OnCalendar. Он не зависит от предыдущих активаций и работает предсказуемо :
```
[Timer]
OnCalendar=*:0/5
Persistent=true
```

`OnCalendar=*:0/5` означает «каждые 5 минут» (в 00, 05, 10... минут каждого часа). Persistent=true позволит выполнить пропущенный запуск, если система была выключена.

## Инициализация папки для копии borgbackup
После того как vagrant отработал и машины созданы нужно будет выполнить инициализацию папки для копии borgbackup:
- **Заходим на сервер `client`**
```
vagrant ssh client
```
- **Инициализируем папку `/var/backup` на удаленном сервере `backup`**
```
borg init --encryption=repokey borg@192.168.56.160:/var/backup/
```

```ad-info
На этом этапе `borgbackup` потребует задать пароль. Его нужно будет указать таким же как в сервисе `borg-backup.service` т.е. строка `Environment="BORG_PASSPHRASE=borgpass"`
```

## Запуск сервиса borg-backup.timer
После того как папка инициализирована, можно включить и запустить сервис `borg-backup.timer` который будет по времени запускать сервис `borg-backup.service`

```bash
systemctl enable borg-backup.timer 
systemctl start borg-backup.timer
```

- **Проверить работу таймера можно следующей командой:**
```
systemctl list-timers --all
```

- **Ошибки при запуске или работе можно проверить с помощью:**
```bash
sudo journalctl -u borg-backup.service -n 30 --no-pager
```


## Итоговый vagrantfile
```yaml
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  config.vm.provider :virtualbox do |v|
    v.memory = 1024
    v.cpus = 2
  end

  boxes = [
    { name: "backup", ip: "192.168.56.160" },
    { name: "client", ip: "192.168.56.150" }
  ]

  boxes.each do |opts|
    config.vm.define opts[:name] do |srv|
      srv.vm.hostname = opts[:name]
      srv.vm.network "private_network", ip: opts[:ip]

      # Провижининг конкретно для backup
      if opts[:name] == "backup"
        srv.vm.disk :disk, size: "2GB", name: "disk1"
        srv.vm.provision "shell", inline: <<-SHELL
           # Ищем добавленные диски (обычно это /dev/sdb и /dev/sdc)
           # Используем `lsblk` для поиска дисков, у которых нет файловой системы
           DISK1=$(lsblk -dpno NAME | grep -E "/dev/sd[c-d]" | head -n 1)
           # Форматируем в ext4
           mkfs.ext4 -F "$DISK1"
           apt-get update
           apt-get install borgbackup -y
           # Создаём пользователей с домашними каталогами
           # -m создает домашнюю директорию
           # -s /bin/bash задает шелл
           useradd -m -s /bin/bash borg
           echo "borg:borg" | chpasswd
           usermod -U borg
           # Создаем папку и делаем владельцем пользователя и группу borg
           mkdir /var/backup
           # Монтируем диск в созданную директорию
           mount "$DISK1" /var/backup
           # Добавляем записи в /etc/fstab (используем UUID для надежности)
           echo "UUID=$(blkid -s UUID -o value "$DISK1") /var/backup ext4 defaults 0 2" >> /etc/fstab
           chown borg:borg /var/backup/
           rm -rf /var/backup/*
           # Создаем папку для ssh ключей
           su - borg -c "mkdir /home/borg/.ssh"
           cp /vagrant/ssh_key/authorized_keys /home/borg/.ssh/
           chown -R borg:borg /home/borg/.ssh
           # Выдаем права пользователю borg на папку и файл authorized_keys
           su - borg -c "chmod 700 .ssh"
           su - borg -c "chmod 600 .ssh/authorized_keys"
        SHELL
      end
      # Провижининг конкретно для client
      if opts[:name] == "client"
        srv.vm.provision "shell", inline: <<-SHELL
           # Устанавливаем borgbackup
           apt-get update
           apt-get install borgbackup -y
           # Копируем заранее заготовленные ssh ключи 
           cp /vagrant/ssh_key/id_rsa.pub /home/vagrant/.ssh/
           cp /vagrant/ssh_key/id_rsa /home/vagrant/.ssh/
           # Назначаем владельцем пользователя vagrant           
           chown vagrant:vagrant /home/vagrant/.ssh/id_rsa.pub
           chown vagrant:vagrant /home/vagrant/.ssh/id_rsa
           # Выдаем права пользователю vagrant
           chmod 600 /home/vagrant/.ssh/id_rsa
           chmod 644 /home/vagrant/.ssh/id_rsa.pub
           # Добавляем ключи в known_hosts
           ssh-keyscan -p 22 -H 192.168.56.160 >> /home/vagrant/.ssh/known_hosts 2>/dev/null
           # Инициализируем репозиторий borg на backup сервере с client сервера
           # borg init --encryption=repokey borg@192.168.56.160:/var/backup/
           # 1. Создаём unit-файл сервиса
           cat > /etc/systemd/system/borg-backup.service <<'EOF'
Description=Borg Backup

[Service]
Type=oneshot

# Этот блок нужен для того чтобы указать от какого пользователя будет запущен сервис.
User=vagrant
Group=vagrant
Environment="HOME=/home/vagrant"

# Парольная фраза
Environment="BORG_PASSPHRASE=borgpass"
# Репозиторий
Environment=REPO=borg@192.168.56.160:/var/backup/
# Что бэкапим
Environment=BACKUP_TARGET=/etc

# Создание бэкапа
ExecStart=-/bin/borg create \
    --stats                \
    ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

# Проверка бэкапа
ExecStart=/bin/borg check ${REPO}

# Очистка старых бэкапов
ExecStart=-/bin/borg prune \
    --keep-daily  90      \
    --keep-monthly 12     \
    --keep-yearly  1       \
    ${REPO}
EOF
           # 2. Создаём unit-файл таймера
           cat > /etc/systemd/system/borg-backup.timer <<'EOF'
[Unit]
Description=Borg Backup

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
EOF
        SHELL
      end
    end
  end
end

```
## Готово
