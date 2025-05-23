<p align="center">
  <img src="https://github.com/kamil1403/otus_mdadm_RAID/raw/main/screenshots/Banner.jpg" alt="RAID Banner" width="800">
</p>

<h1 align="center">otus_mdadm_RAID</h1>
<p align="center">Дата: 11-05-2025<br>Автор: Kamil Ibragimov</p>

## Домашнее задание: работа с mdadm   
Задание:   
• Добавить в виртуальную машину несколько дисков   
• Собрать RAID-0/1/5/10 на выбор   
• Сломать и починить RAID   
• Создать GPT таблицу, пять разделов и смонтировать их в системе   

На проверку отправьте:   
• Cкрипт для создания рейда   
• Отчет по командам для починки RAID и созданию разделов   

Результат:   
• Cкрипт для создания RAID 1 написан. Результат см. на скриншотах ["create_RAID_1_1"](https://github.com/kamil1403/otus_mdadm_RAID/blob/main/screenshots/create_RAID_1_1.png) и ["create_RAID_1_2"](https://github.com/kamil1403/otus_mdadm_RAID/blob/main/screenshots/create_RAID_1_2.png)   
• Отчет по командам для починки RAID см. на скриншотах ["fail_RAID_1_1"](https://github.com/kamil1403/otus_mdadm_RAID/blob/main/screenshots/fail_RAID_1_1.png) и ["fail_RAID_1_2"](https://github.com/kamil1403/otus_mdadm_RAID/blob/main/screenshots/fail_RAID_1_2.png)   
• Отчет по командам для создания GPT разделов на RAID см. на скриншотах ["GPT_RAID_1_1"](https://github.com/kamil1403/otus_mdadm_RAID/blob/main/screenshots/GPT_RAID_1_1.png), ["GPT_RAID_1_2"](https://github.com/kamil1403/otus_mdadm_RAID/blob/main/screenshots/GPT_RAID_1_2.png) и ["GPT_RAID_1_3"](https://github.com/kamil1403/otus_mdadm_RAID/blob/main/screenshots/GPT_RAID_1_3.png)   



## 🧭 Оглавление

- [✍🏻 Скрипт для сборки RAID 1](#script)
- [✍🏻 Скрипт симуляции отказа диска](#fixraid)
- [📀 Диски и устройства](#hard)
- [🧱 Разметка и подготовка диска](#partition)
- [🗃️ Сборка RAID 1](#setup)
- [💥 Симуляция отказа диска](#failure)
- [⚙️ Создание разделов на RAID](#GPT)
- [🛠️ RAID 1 без потери данных](#preserve)
- [✂️ Уменьшение размера раздела](#resize)

---

<a id="script"></a>
## ✍🏻 Скрипт для сборки RAID 1
Запуск через sudo bash script.sh   
Дать права chmod +x script.sh

```bash
#!/bin/bash

mkdir -p /mnt/raid01
apt-get install -y mdadm
mdadm --create --verbose /dev/md127 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdb2
mkfs.ext4 /dev/md127
mount /dev/md127 /mnt/raid01
echo "/dev/md127 /mnt/raid01 ext4 defaults 0 0" >> /etc/fstab
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u
```

---

<a id="fixraid"></a>
## ✍🏻 Скрипт симуляции отказа диска
Запуск через sudo bash script.sh   
Дать права chmod +x script.sh

```bash
#!/bin/bash

mdadm /dev/md127 --fail /dev/sdb1
mdadm /dev/md127 --remove /dev/sdb1
mdadm --zero-superblock /dev/sdb3 >/dev/null 2>&1
mdadm /dev/md127 --add /dev/sdb3
watch -n 1 cat /proc/mdstat
```

---

<a id="hard"></a>
## 📀 Диски и устройства

```bash
# Показывает структуру дисков и разделов в виде дерева
lsblk
lsblk | grep -E 'sd|NAME'
# Выводит список физических дисков и их разделов в системе
ls -la /dev/sd*
```

---

<a id="partition"></a>
## 🧱 Разметка и подготовка диска

```bash
# Показывает текущую таблицу разделов диска /dev/sdb (размеры, типы, начальные сектора)
fdisk -l /dev/sdb
# Запуск интерактивного режима для управления разделами диска
fdisk /dev/sdb
# Действия внутри fdisk:
# m — меню команд
# p — текущая таблица разделов
# n — создать раздел (primary/extended)
# w — записать изменения и выйти
# Форматирует раздел /dev/sdb1 в файловую систему ext4
mkfs.ext4 /dev/sdb1
# Отображает древовидную структуру дисков и разделов (имена, размеры, точки монтирования)
lsblk
# Показывает UUID и тип файловой системы для всех разделов (нужен для настройки /etc/fstab)
blkid
# Создает директории /mnt/disk1, /disk2 и т.д. для монтирования разделов
# Флаг -p игнорирует ошибки, если директории уже существуют
mkdir -p /mnt/disk1 /mnt/disk2 /mnt/disk3 /mnt/disk4
# Монтирует раздел /dev/sdb1 в директорию /mnt/disk1 (временное подключение)
mount /dev/sdb1 /mnt/disk1
# Редактирует файл fstab для автоматического монтирования разделов при загрузке
nano /etc/fstab
# Пример строки
UUID=1234-ABCD /mnt/disk1 ext4 defaults 0 2
```

---

<a id="setup"></a>
## 🗃️ Сборка RAID 1 с использованием mdadm

```bash
# Создает директорию /mnt/raid01 для монтирования RAID-массива
mkdir /mnt/raid01
# Показывает статус RAID-массивов в системе (активные массивы, процесс синхронизации, устройства)
cat /proc/mdstat
# Устанавливает утилиту mdadm для управления программными RAID-массивами в Linux
apt install mdadm
# Создает RAID 1 (зеркало) из двух разделов, где /dev/md127 — имя массива, -l 1 — уровень RAID (зеркалирование), -n 2 — количество устройств
mdadm --create /dev/md127 -l 1 -n 2 /dev/sdb1 /dev/sdb2
# Проверка прогресса синхронизации RAID (например, resync = 50%).
cat /proc/mdstat
# Выводит детальную информацию о массиве:
# Состояние (active, degraded)
# Список устройств (/dev/sdb1, /dev/sdb2)
# UUID массива
mdadm -D /dev/md127
# Форматирует RAID-массив в файловую систему ext4 (готов к хранению данных)
mkfs.ext4 /dev/md127
# Монтирует массив в директорию /mnt/raid01 для доступа к данным
mount /dev/md127 /mnt/raid01
# Показывает свободное место на всех смонтированных файловых системах, включая RAID
df -h
# Копирует содержимое /var/log/ в RAID-массив для резервирования логов
cp -r /var/log/* /mnt/raid01
# Редактирует файл fstab для автоматического монтирования разделов при загрузке
nano /etc/fstab
# Пример строки для автоматического монтирования RAID
/dev/md127 /mnt/raid01 ext4 defaults 0 0
```

---

<a id="failure"></a>
## 💥 Симуляция отказа диска в RAID 1 и его замена

```bash
# Выводит детальную информацию о массиве:
# Состояние (active, degraded)
# Список устройств (/dev/sdb1, /dev/sdb2)
# UUID массива
mdadm -D /dev/md127
# Помечает диск /dev/sdb1 как неисправный (инициирует процесс замены)
mdadm /dev/md127 --fail /dev/sdb1
# Удаляет неисправный диск /dev/sdb1 из массива (после --fail)
mdadm /dev/md127 --remove /dev/sdb1
# Проверяет метаданные RAID на диске (например, был ли диск частью массива)
mdadm --examine /dev/sdX
# Очищает суперблок RAID на диске перед повторным использованием (удаляет следы участия в массиве)
mdadm --zero-superblock /dev/sdX
# Добавляет новый диск /dev/sdb3 в массив для восстановления (начнется ресинхронизация данных)
mdadm /dev/md127 --add /dev/sdb3
# Отображает текущий статус RAID, включая прогресс ресинхронизации (например, resync = 75%)
cat /proc/mdstat
```

---

<a id="GPT"></a>
## ⚙️ Создание разделов на RAID

```bash
# Размонтирует RAID
sudo umount /mnt/raid01 
# Создаст GPT разметку
sudo parted -s /dev/md127 mklabel gpt
# Создаст 5 равных разделов по 20%
sudo parted /dev/md127 mkpart primary ext4 0% 20%
sudo parted /dev/md127 mkpart primary ext4 20% 40%
sudo parted /dev/md127 mkpart primary ext4 40% 60%
sudo parted /dev/md127 mkpart primary ext4 60% 80%
sudo parted /dev/md127 mkpart primary ext4 80% 100%
# Создает файловые системы (ext4)
for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md127p$i; done
# Создает точки монтирования
sudo mkdir -p /raid/part{1,2,3,4,5}
# Монтирует разделы 
for i in $(seq 1 5); do mount /dev/md127p$i /raid/part$i; done
# Редактирует файл fstab для автоматического монтирования разделов при загрузке
nano /etc/fstab
# Пример строки для автоматического монтирования разделов
for i in {1..5}; do echo "/dev/md127p$i /mnt/raid_part$i ext4 defaults 0 2" | sudo tee -a /etc/fstab; done

```

---

<a id="preserve"></a>
## 🛠️ Создание RAID 1 без потери данных

```bash
# Создать RAID 1 из двух дисков, где один диск уже содержит данные (например, `/dev/sdb1`)    
# Эти данные нужно сохранить, объединив его с новым диском (`/dev/sdb2`) в зеркальный RAID-массив   

# Проверяет список доступных дисков и их разделов перед началом работы.
lsblk
# Создает RAID 1 с параметрами:
# /dev/md127 — имя массива
# -l 1 — уровень RAID (зеркало)
# -n 2 — количество устройств
# /dev/sdb2 — активный диск
# missing — второй диск добавлен позже (чтобы не стирать данные на /dev/sdb1)
mdadm --create /dev/md127 -l 1 -n 2 /dev/sdb2 missing
# Проверяет состояние массива: убедиться, что массив создан, но работает в degraded mode (только с одним диском)
mdadm -D /dev/md127
# Форматирует RAID-массив в файловую систему ext4 (данные на /dev/sdb1 не затрагиваются)
mkfs.ext4 /dev/md127
# Создает директорию /mnt/raid02 для монтирования RAID-массива
mkdir /mnt/raid02
# Подключает массив для копирования данных
mount /dev/md127 /mnt/raid02
# Копирует данные с исходного диска (/mnt/raid01, где смонтирован /dev/sdb1) в новый RAID-массив
cp -r /mnt/raid01/* /mnt/raid02
# Отключает исходный диск (/dev/sdb1) перед добавлением его в массив
umount /mnt/raid01
# Добавляет диск /dev/sdb1 в массив, запуская синхронизацию данных (зеркалирование с /dev/sdb2)
mdadm /dev/md127 --add /dev/sdb1
# Показывает статус массива (прогресс синхронизации)
mdadm -D /dev/md127
# Отслеживает процесс в реальном времени (например, resync = 75%)
watch cat /proc/mdstat
# Отключает временную точку монтирования
umount /mnt/raid02
# Подключает RAID-массив к исходной директории (/mnt/raid01), заменяя старый диск
mount /dev/md127 /mnt/raid01
```

---

<a id="resize"></a>
## ✂️ Уменьшение размера последнего раздела (sdb4 → 5G)

```bash
# sdb      8:16   0   25G  0 disk   
# ├─sdb1   8:17   0    5G  0 part   
# ├─sdb2   8:18   0    5G  0 part   
# ├─sdb3   8:19   0    5G  0 part   
# └─sdb4   8:20   0   10G  0 part   

# Чтобы разбить раздел /dev/sdb4 (10 GB) на два раздела по 5 GB, необходимо удалить текущий sdb4 (если на нём нет важных данных!):
# Запуск утилиты для редактирования таблицы разделов диска /dev/sdb
sudo fdisk /dev/sdb
# Действия внутри fdisk:
# d → 4 — удаление раздела /dev/sdb4
# n → p → 4 → +5G — создание нового первичного раздела (p) /dev/sdb4 размером 5 GB
# n → p → 5 → Enter — создание раздела /dev/sdb5 на оставшемся пространстве (5 GB)
# w — сохранение изменений и выход
# Важно: После этого будет 4 primary раздела, sdb1–sdb4
# Обновление информации о разделах в ядре системы без перезагрузки
sudo partprobe
# Форматирование новых разделов в файловую систему ext4
sudo mkfs.ext4 /dev/sdb4
sudo mkfs.ext4 /dev/sdb5
# Проверка изменений или дополнительная настройка разделов
fdisk /dev/sdb
```

---
