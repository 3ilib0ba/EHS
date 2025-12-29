# Linux работа с памятью и процессами

---

## Задание 1. Systemd

* Создайте bash-скрипт /usr/local/bin/homework_service.sh с содержанием:

```bash
#!/bin/bash
echo "My custom service has started."
while true; do
  echo "Service heartbeat: $(date)" >> /tmp/homework_service.log
  sleep 15
done
```

* Создайте systemd unit файл для скрипта, который бы переживал любые обновления системы. Убедитесь, что сервис сам перезапускается в случае падения через 15 секунд.

* Запустите сервис и убедитесь, что он работает.

* Используя systemd-analyze, покажите топ-5 systemd unit`ов стартующих дольше всего.

---

Сначала я создал скрипт и выдал ему права на исполнение:

```txt
user@operation-highload-systems:~$ sudo cat /usr/local/bin/homework_service.sh
#!/bin/bash
echo "My custom service has started."
while true; do
 echo "Service heartbeat: $(date)" >> /tmp/homework_service.log
 sleep 15
done
user@operation-highload-systems:~$ sudo chmod +x /usr/local/bin/homework_service.sh 
user@operation-highload-systems:~$ sudo ls -l /usr/local/bin/homework_service.sh 
-rwxr-xr-x 1 root root 144 дек 29 00:54 /usr/local/bin/homework_service.sh
```

---

Далее создаю systemd файла для сервиса + добавляю его в рабочие сервисы (фоновые) и запускаю:

```txt
user@operation-highload-systems:~$ sudo cat /etc/systemd/system/homework.service 
[Unit]
Description=Homework_service
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
ExecStart=/usr/local/bin/homework_service.sh
Restart=on-failure
RestartSec=15
StartLimitBurst=5

[Install]
WantedBy=multi-user.target

user@operation-highload-systems:~$ sudo systemctl daemon-reload

user@operation-highload-systems:~$ sudo systemctl enable --now homework.service
Created symlink /etc/systemd/system/multi-user.target.wants/homework.service -> /etc/systemd/system/homework.service.

user@operation-highload-systems:~$ systemctl status homework.service
* homework.service - Homework_service
     Loaded: loaded (/etc/systemd/system/homework.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2025-12-29 01:05:48 MSK; 12s ago
   Main PID: 27963 (homework_servic)
      Tasks: 2 (limit: 4599)
     Memory: 648.0K
        CPU: 2ms
     CGroup: /system.slice/homework.service
             |-27963 /bin/bash /usr/local/bin/homework_service.sh
             `-27965 sleep 15

user@operation-highload-systems:~$ tail -n 3 /tmp/homework_service.log 
Service heartbeat: Пн 29 дек 2025 01:06:03 MSK
Service heartbeat: Пн 29 дек 2025 01:06:18 MSK
Service heartbeat: Пн 29 дек 2025 01:06:33 MSK
```

---

Далее сымитирую падение сервиса и проверю его автоматический рестарт. Для этого убью его процесс и проверю по логам хартбиты:

```txt
user@operation-highload-systems:~$ systemctl show -p MainPID --value homework.service
27963

user@operation-highload-systems:~$ tail -n 1 /tmp/homework_service.log 
Service heartbeat: Пн 29 дек 2025 01:31:34 MSK

user@operation-highload-systems:~$ sudo kill -9 27963
[sudo] password for user: 

user@operation-highload-systems:~$ systemctl status homework.service
* homework.service - Homework_service
     Loaded: loaded (/etc/systemd/system/homework.service; enabled; vendor preset: enabled)
     Active: activating (auto-restart) (Result: signal) since Mon 2025-12-29 01:31:45 MSK; 14s ago
    Process: 27963 ExecStart=/usr/local/bin/homework_service.sh (code=killed, signal=KILL)
   Main PID: 27963 (code=killed, signal=KILL)
        CPU: 190ms

user@operation-highload-systems:~$ tail -n 5 /tmp/homework_service.log 
Service heartbeat: Пн 29 дек 2025 01:30:49 MSK
Service heartbeat: Пн 29 дек 2025 01:31:04 MSK
Service heartbeat: Пн 29 дек 2025 01:31:19 MSK
Service heartbeat: Пн 29 дек 2025 01:31:34 MSK
Service heartbeat: Пн 29 дек 2025 01:32:00 MSK

user@operation-highload-systems:~$ tail -n 5 /tmp/homework_service.log 
Service heartbeat: Пн 29 дек 2025 01:31:19 MSK
Service heartbeat: Пн 29 дек 2025 01:31:34 MSK
Service heartbeat: Пн 29 дек 2025 01:32:00 MSK
Service heartbeat: Пн 29 дек 2025 01:32:15 MSK
Service heartbeat: Пн 29 дек 2025 01:32:30 MSK
```

Видно, что авто-рестарт был в 01:31:45 и после по логам ясно что сервис работает

---

Покажу самые долгие по старту systemd юниты при помощи `systemd-analyze`:

```txt
user@operation-highload-systems:~$ systemd-analyze blame | head -n 5
1.685s plymouth-quit-wait.service
1.323s snapd.seeded.service
1.320s snapd.service
 271ms dev-sda3.device
 269ms dev-loop8.device
```

--- 

## Задание 2. Межпроцессное взаимодействие (IPC) с разделяемой памятью

* Создайте шареную память:

    * На любом языке прогаммирования создайте програму использующую шареную память. Например `shm_creator.c`:
```
#include <stdio.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <unistd.h>

int main() {
    key_t key = ftok("homework_key", 65); // Generate a unique key
    int shmid = shmget(key, 1024, 0666|IPC_CREAT); // Create 1KB segment
    if (shmid == -1) {
        perror("shmget");
        exit(1);
    }
    printf("Shared memory segment created.\n");
    printf("ID: %d\nKey: 0x%x\n", shmid, key);
    printf("Run 'ipcs -m' to see it. Process will exit in 60 seconds...\n");
    sleep(60);
    shmctl(shmid, IPC_RMID, NULL); // Clean up
    printf("Shared memory segment removed.\n");
    return 0;

}
```


* Скомпилируйте и запустите
  * `gcc shm_creator.c -o shm_creator`
  * `touch homework_key`
  * `./shm_creator`
* Пока программа запущена (60 секунд), проанализируйте вывод: в соседнем терминале запустите ipcs -m. 
Обратите внимание на  nattch (number of attached processes) проанализируйте вывод

---

Создал, скомпилировал и запустил программу:

```
user@operation-highload-systems:~$ cat shm_creator.c  
#include <stdio.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <unistd.h>

int main() {
    key_t key = ftok("homework_key", 65); // Generate a unique key
    int shmid = shmget(key, 1024, 0666|IPC_CREAT); // Create 1KB segment
    if (shmid == -1) {
        perror("shmget");
        exit(1);
    }
    printf("Shared memory segment created.\n");
    printf("ID: %d\nKey: 0x%x\n", shmid, key);
    printf("Run 'ipcs -m' to see it. Process will exit in 60 seconds...\n");
    sleep(60);
    shmctl(shmid, IPC_RMID, NULL); // Clean up
    printf("Shared memory segment removed.\n");
    return 0;
}

user@operation-highload-systems:~$ gcc shm_creator.c -o shm_creator
user@operation-highload-systems:~$ touch homework_key
user@operation-highload-systems:~$ ./shm_creator
Shared memory segment created.
ID: 2
Key: 0x4103c04d
Run 'ipcs -m' to see it. Process will exit in 60 seconds...
Shared memory segment removed.
```

Параллельно с этим в другом терминале получаю информацию о разделяемой памяти в системе:

```
user@operation-highload-systems:~$ ipcs -m
------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      
0x4103c04d 2          user       666        1024       0                       

user@operation-highload-systems:~$ ipcs -m -i 2
Shared memory Segment shmid=2
uid=1000 gid=1000 cuid=1000 cgid=1000
mode=0666 access_perms=0666
bytes=1024 lpid=0 cpid=31318 nattch=0
att_time=Not set                   
det_time=Not set                   
change_time=Mon Dec 29 02:01:35 2025
```

Видно сгенерированный ключ, `shmid` (ID сегмента в таблице ядра), права на чтение/запись (`rw-rw-rw-`), размер сегмента в байтах.

Поле `nattch`, равное 0, показывает кол-во подключенных к сегменту процессов. Если их 0, это значит что работа уже завершена и процессы не подключены, 
что в данной ситуации нормально.

---

## Задание 3. Анализ памяти процессов (VSZ vs RSS)

* Откройте 1 окно терминала и запустите питон скрипт, который запрашивает 250 MiB памяти и держит ее 2 минуты
  * python3 -c "print('Allocating memory...'); a = 'X' * (250 * 1024 * 1024); import time; print('Memory allocated. Sleeping...'); time.sleep(120);"
* Пока скрипт запущен, откройте вторую вкладку, найдите там PID запущенного скрипта и проанализируйте использование RSS и VSZ:
  * ps -o pid,user,%mem,rss,vsz,comm -p %%YOUR_PID%%
* Объясните почему vsz больше rss, и почему rss далеко не 0

---

Запускаю скрипт в одном окне и смотрю анализ vsz, rss в другом:

```
user@operation-highload-systems:~$ python3 -c "print('Allocating memory...'); a = 'X' * (250 * 1024 * 1024); import time; print('Memory allocated. Sleeping...'); time.sleep(120);"
Allocating memory...
Memory allocated. Sleeping...
```

```
user@operation-highload-systems:~$ ps -o pid,user,%mem,rss,vsz,comm -p 32408
    PID USER     %MEM     RSS     VSZ COMMAND
  32408 user      6.5  264064  275128 python3
```

VSZ это виртуальная память, выделенная процессу (включает озу, свопы, разделяемые библиотеки и используемые файлы)

RSS это объём физической памяти, реально используемой процессом (то есть только та память, что загружена в ОЗУ для процесса)

VSZ не равен RSS так как ещё содержит и другие файлы, которые сейчас не находятся в ОЗУ полностью или разделяемые с другими процессами библиотеки.

RSS не равен 0, так как скрипт создаёт большую строку `a`, которая лежит в памяти, как и другие необходимые питону части процесса

---

## Задание 4. NUMA и cgroups

* Продемонстрируйте количество NUMA нод на вашем сервере и количество памяти для каждой NUMA ноды
* Убедитесь что вы можете ограничивать работу процессов при помощи systemd.
* Запустите
  * sudo systemd-run --unit=highload-stress-test --slice=testing.slice \
    --property="MemoryMax=150M" \
    --property="CPUWeight=100" \
    stress --cpu 1 --vm 1 --vm-bytes 300M --timeout 30s

* Будет ли работать тест если мы запрашиваем 300М оперативной памяти, а ограничивыем 150М?
* В соседней вкладке проследите за testing.slice при помощи systemd-cgls. Привысило ли использование памяти 150М ? Что происходит с процессом при превышении? Попробуйте использовать разные значения
* Опишите что делает и для ччего можно использовать MemoryMax and CPUWeight.

---

Демонстрация всех NUMA-нод на машине:

```
user@operation-highload-systems:~$ numactl --hardware
available: 1 nodes (0)
node 0 cpus: 0 1 2 3
node 0 size: 3915 MB
node 0 free: 357 MB
node distances:
node   0 
  0:  10
```

Есть только 1 нода с ОЗУ ~ в 4 ГБ

---

Демонстрация возможности ограничения работы процессов (ограничение в 100МБ):

```
user@operation-highload-systems:~$ sudo systemd-run --unit=limit-demo --property="MemoryMax=100M" sleep 60
Running as unit: limit-demo.service

user@operation-highload-systems:~$ systemctl show limit-demo | grep -E 'MemoryMax'
MemoryMax=104857600

----- [[СПУСТЯ ВРЕМЯ]] -----

user@operation-highload-systems:~$ systemctl show limit-demo | grep -E 'MemoryMax'
MemoryMax=infinity
```

---

Запуск highload-стресс-теста:

```
user@operation-highload-systems:~$ sudo systemd-run --unit=highload-stress-test --slice=testing.slice --property="MemoryMax=150M" --property="CPUWeight=100" stress --cpu 1 --vm 1 --vm-bytes 300M --timeout 30s
Running as unit: highload-stress-test.service
```

Анализ

```
user@operation-highload-systems:~$ systemctl status highload-stress-test
* highload-stress-test.service - /usr/bin/stress --cpu 1 --vm 1 --vm-bytes>
     Loaded: loaded (/run/systemd/transient/highload-stress-test.service; >
  Transient: yes
     Active: active (running) since Mon 2025-12-29 02:55:27 MSK; 2s ago
   Main PID: 33655 (stress)
      Tasks: 3 (limit: 4599)
     Memory: 141.5M (max: 150.0M available: 8.4M)
        CPU: 4.256s
     CGroup: /testing.slice/highload-stress-test.service
             |-33655 /usr/bin/stress --cpu 1 --vm 1 --vm-bytes 300M --time>
             |-33656 /usr/bin/stress --cpu 1 --vm 1 --vm-bytes 300M --time>
             `-33657 /usr/bin/stress --cpu 1 --vm 1 --vm-bytes 300M --time>
```

Видно, что запрос на 300 МБ памяти для процесса выполняется с отказом, то есть ограничение на размер памяти работает строго.

При этом процесс не получает запрашиваемую память и завершается корректно, после создаётся новый

Таким образом, `MemoryMax` ставит жёсткое ограничение на используемую сервисом память

Параметр `CPUWeight` определяет вес при делении процессорного времени между группами процессов. Его можно использовать для определения приоритетов между ними

---
