#hardware
***
## Сервер не отвечает, как можно получить доступ к серверу, не находясь непосредственно в ЦОДЕ

Через IPMI или kvm если оно есть на сервере.
Ну или звонить дежурному инженера ЦОДА, если иных опций нет.

***
## Что такое kvm(не гипервизор)? Как можно его использовать?

KVM (или kvm over ip) — устройство, позволяющее передавать видеосигнал и ввод с мыши/клавиатуры по сети с использованием IP-протокола от вашего сервера. При помощи KVM вы можете перезагрузить сервер, получить доступ в BIOS сервера и к другим функциям, которые невозможно выполнить на сервере через терминал. То есть он обособлен от операционной системы.

Часто используется как последнее средств. Когда сервер не грузится, или есть ещё какой-то программный или системный сбой.

KVM - это аббревиатура, состоящая из слов. Клавиатура, монитор(видео), мышь.

***
## Что такое IPMI? Какие подсистемы он в себя включает?

IPMI (Intelligent Platform Management Interface) – это интерфейс для удаленного мониторинга и управления физическим состоянием сервера.

Как я понимаю, модуль находится внутри самого сервера. И называется он BMC. Контроллер управления. 

В случае утраты контроля над работой сервера, можно удаленно управлять его работой, а именно:

- получить доступ к консоли, изменить настройки BIOS;
- перезагрузить, включить/выключить сервер;
- ознакомиться с состоянием сервера (слежение за температурными датчиками, датчиками напряжения, состояние блока питания, скорость 
вращения вентиляторов);
- подключение образов .iso.

Но вообще, BMC ― это отдельный компьютер со своим программным обеспечением и сетевым интерфейсом, который распаивают на материнской плате или подключают как плату расширения по шине PCI management bus.

К BMC контроллеры подключаются через интерфейс IPMB (Intelligent 
Platform Management Bus ― шина интеллектуального управления платформой).
 IPMB ― это шина на основе I2C (Inter-Integrated Circuit), по которой 
BMC перенаправляет команды управления к различным частям архитектуры:

- Общается с дополнительными контроллерами (MCs)
- Считывает данные сенсоров (Sensors)
- Обращается к энергонезависимому хранилищу (Non-Volatile Storage)

Архитектура IPMI реализована так, что удаленный администратор не 
имеет прямого доступа к компонентам системы. Например, чтобы получить 
данные с сенсоров, удаленный администратор посылает команду на BMC, а 
BMC в свою очередь обращается к сенсорам.

Подробнее о технологии можно почитать по ссылке:  
[https://selectel.ru/blog/ipmi-obzor-texnologii/](https://selectel.ru/blog/ipmi-obzor-texnologii/)

***
## Какие преимущества предоставляет IPMI в сравнении с kvm?

Недостатки модуля IP-KVM в сравнении с IPMI

Традиционные внешние IP-KVM устройства позволяют вам удаленно работать 
только с консолью своего сервера, отсутствует возможность управления 
питанием, монтирования образов и контроля состояния датчиков сервера.

У IP-KVM есть несколько ключевых недостатков:

- отсутствие постоянного доступа к управлению сервером (чтобы
воспользоваться IP-KVM, вам нужно создать запрос в техподдержку с
просьбой подключить к вашему серверу временный IP-KVM в датацентре;
заявку желательно подавать заранее, подключение занимает от 15 до 30
минут в лучшем случае; в подключении KVM может быть отказано, если
сейчас в наличии нет свободного оборудования);
- отсутствие возможности управлять питанием, монтировать образы и контролировать состояние датчиков сервера.

***
## Просмотр информации о железной составляющей сервера

## Модели процессора, количестве физических и логических ядер, поддерживаемых инструкциях, режиме работы?


- Модель процессора `cat /proc/cpuinfo`

```
model name	: Intel(R) Xeon(R) CPU E5-2620 0 @ 2.00GHz
model		: 45
```

Та же модель процессора через `lscpu`

```
Model name:          Intel(R) Xeon(R) CPU E5-2620 0 @ 2.00GHz
```

Ядра физические и логические

```
vm13 : ~ [0] # grep "cpu cores" /proc/cpuinfo |sort -u |cut -d":" -f2
 4
vm13 : ~ [0] # grep -c "processor" /proc/cpuinfo
4
```

Поддерживаемые инструкции /proc/cpuinfo и lscpu

Но возможно стоит также в спецификацию посмотреть

```
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon rep_good nopl xtopology pni pclmulqdq vmx ssse3 cx16 pcid sse4_1 sse4_2 x2apic popcnt tsc_deadline_timer aes xsave avx hypervisor lahf_lm kaiser tpr_shadow vnmi flexpriority ept vpid tsc_adjust xsaveopt arat
```

режимы работы процессора

`find / -name scaling_governor`

`find / -name scaling_max_freq`

`cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor`

- **powersave** — режим энергосбережения, ядро будет работать на пониженных частотах
- **ondemand** — режим зависящей от текущей нагрузки на ядро
- **performance** — режим максимальной мощности, выставляет максимально возможную частоту

- Физические ядра - это число физических ядер, реальных аппаратных компонентов.

Логические ядра - это число физических ядер, умноженное на количество потоков, которые могут выполняться на каждом ядре с помощью гиперпотока.
например, мой 4-ядерный процессор запускает два потока на ядро, поэтому у меня есть 8 логических процессоров.

Узнать сколько ядер доступно можно командой:

```
dmidecode -t processor | grep "Core Enabled:"
Core Enabled: 6
Core Enabled: 6
```

Видим, что на данной системе находится 12 физических ядер (6+6). Соответственно, нормальный показатель LA должен быть менее 12. Однако, на процессорах Intel используется технология [Hyper-Threading](http://www.intel.ru/content/www/ru/ru/architecture-and-technology/hyper-threading/hyper-threading-technology.html), которая делит одно физическое ядро на два логических.

```
dmidecode -t processor | grep "Thread Count:"
Thread Count: 12
Thread Count: 12
```

Соответственно, в данном случае в системе может быть одновременно 24 виртуальных процессора (потока).

Технология Turbo Boost позволяет процессору «разгоняться» и работать на частоте выше заявленной (т.е. выше 100%, выше единицы). Какой показатель LA считать нормальным в данном случае является предметом споров.

***
## Типы оперативной памяти, модели материнской платы, версии BIOS?  

**Оперативная память** 

```
dmidecode --type memory 
# dmidecode --type 17
# dmidecode 2.11
SMBIOS 2.8 present.

Handle 0x1100, DMI type 17, 40 bytes
Memory Device
	Array Handle: 0x1000
	Error Information Handle: Not Provided
	Total Width: Unknown
	Data Width: Unknown
	Size: 9216 MB
	Form Factor: DIMM
	Set: None
	Locator: DIMM 0
	Bank Locator: Not Specified
	Type: RAM
	Type Detail: Other
	Speed: Unknown
	Manufacturer: QEMU
	Serial Number: Not Specified
	Asset Tag: Not Specified
	Part Number: Not Specified
	Rank: Unknown
	Configured Clock Speed: Unknown

Handle 0x004F, DMI type 17, 34 bytes
Memory Device
	Array Handle: 0x003F
	Error Information Handle: Not Provided
	Total Width: 72 bits
	Data Width: 64 bits
	Size: 16384 MB
	Form Factor: DIMM
	Set: None
	Locator: P2_DIMMH2
	Bank Locator: Node1_Bank0
	Type: DDR3
	Type Detail: Registered (Buffered)
	Speed: 1333 MHz
	Manufacturer: Samsung            
	Serial Number: 85D3A920     
	Asset Tag: Dimm10_AssetTag
	Part Number: M393B2G70BH0-Y
	Rank: 2
	Configured Clock Speed: 1333 MHz
```

Материнская плата

```

dmidecode --type baseboard
Handle 0x0002, DMI type 2, 15 bytes
Base Board Information
	Manufacturer: Supermicro
	Product Name: X9DR3-F
	Version: 0123456789
	Serial Number: VM16BS021748
	Asset Tag: To be filled by O.E.M.
	Features:
		Board is a hosting board
		Board is replaceable
	Location In Chassis: To be filled by O.E.M.
	Chassis Handle: 0x0003
	Type: Motherboard
	Contained Object Handles: 0
```

Версия bios

```bash
storage8 : ~ [130] # dmidecode --type BIOS
# dmidecode 2.11
SMBIOS 2.7 present.

Handle 0x0000, DMI type 0, 24 bytes
BIOS Information
	Vendor: American Megatrends Inc.
	Version: 3.2a
```

***
## Текущих значениях датчиков напряжения, температуры, оборотов вентиляторов?

, нужно дополнить

Температура и всякие такие штуки можно смотреть в `sensors`

```bash
sorsstorage13 : ~ [0] # sensors
coretemp-isa-0001
Adapter: ISA adapter
Core 0:       +32.0°C  (high = +85.0°C, crit = +95.0°C)
Core 1:       +36.0°C  (high = +85.0°C, crit = +95.0°C)
Core 9:       +27.0°C  (high = +85.0°C, crit = +95.0°C)
Core 10:      +39.0°C  (high = +85.0°C, crit = +95.0°C)

intel5500-pci-00a3
Adapter: PCI adapter
temp1:        +65.5°C  (high = +100.0°C, hyst = +95.0°C)
					   (crit = +110.0°C)

coretemp-isa-0000
Adapter: ISA adapter
Core 0:       +39.0°C  (high = +85.0°C, crit = +95.0°C)
Core 1:       +38.0°C  (high = +85.0°C, crit = +95.0°C)
Core 9:       +31.0°C  (high = +85.0°C, crit = +95.0°C)
Core 10:      +29.0°C  (high = +85.0°C, crit = +95.0°C)
```

Но большая часть информации через ipmicfg может взяться

```bash
sorshocking : ~ [0] # ipmicfg -pminfo
 [SlaveAddress = 78h] [Module 1]
 Item                           |                          Value
 ***-                           |                          ***--
 Status                         |              [STATUS OK] (00h)
 Input Voltage                  |                        227.2 V
 Input Current                  |                         0.52 A
 Main Output Voltage            |                        12.09 V
 Main Output Current            |                         9.37 A
 Temperature 1                  |                        33C/91F
 Temperature 2                  |                       41C/106F
 Fan 1                          |                       3968 RPM
 Fan 2                          |                          0 RPM
 Main Output Power              |                          113 W
 Input Power                    |                          126 W
 PMBus Revision                 |                           0x22
 PWS Serial Number              |                P7061VF28GT1194
 PWS Module Number              |                    PWS-706P-1R
 PWS Revision                   |                            1.1
 Current Sharing Control        |           Active - Active (80)

 [SlaveAddress = 7Ah] [Module 2]
 Item                           |                          Value
 ***-                           |                          ***--
 Status                         |              [STATUS OK] (00h)
 Input Voltage                  |                        226.5 V
 Input Current                  |                         0.59 A
 Main Output Voltage            |                        12.09 V
 Main Output Current            |                        10.50 A
 Temperature 1                  |                        35C/95F
 Temperature 2                  |                       41C/106F
 Fan 1                          |                       4384 RPM
 Fan 2                          |                          0 RPM
 Main Output Power              |                          127 W
 Input Power                    |                          147 W
 PMBus Revision                 |                           0x22
 PWS Serial Number              |                P7061VF28GT1193
 PWS Module Number              |                    PWS-706P-1R
 PWS Revision                   |                            1.1
 Current Sharing Control        |           Active - Active (80)
```

***
## Типе используемого сетевого адаптера и состоянии его интерфейсов? 

```bash
sorsstorage13 : ~ [0] # lspci | grep net
01:00.0 Ethernet controller: Intel Corporation 82576 Gigabit Network Connection (rev 01)
01:00.1 Ethernet controller: Intel Corporation 82576 Gigabit Network Connection (rev 01)
```

```bash
vm13 : ~ [0] # lshw -class network -short
H/W path      Device       Class      Description
=================================================
/0/100/12     eth0         network    Virtio network device
/0/100/13     eth1         network    Virtio network device
/1            vethd2a5488  network    Ethernet interface
/2            vethd29a7c3  network    Ethernet interface
/3            veth12f3f63  network    Ethernet interface
/4            veth39864d5  network    Ethernet interface
/5            vethbe4b44c  network    Ethernet interface
/6            veth62b367d  network    Ethernet interface
/7            veth26cd9d1  network    Ethernet interface
/8            veth2ab7aa7  network    Ethernet interface
/9            vethb7e97f1  network    Ethernet interface
```

```bash
vm13 : ~ [255] # ip a s eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
	link/ether 3e:8e:90:47:59:70 brd ff:ff:ff:ff:ff:ff
	inet 5.101.156.76/24 brd 5.101.156.255 scope global eth0
```

Или так

```bash
vm13 : ~ [0] # ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1
	link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
	link/ether 3e:8e:90:47:59:70 brd ff:ff:ff:ff:ff:ff
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
	link/ether 2e:84:f2:32:05:88 brd ff:ff:ff:ff:ff:ff
```

Здесь можно увидеть понятие up, означает что интерфейс поднят

- UP — устройство подключено и готово принимать и отправлять фреймы;
- LOOPBACK — интерфейс является локальным и не может взаимодействовать с другими узлами в сети;
- BROADCAST - устройство способно отправлять широковещательные фреймы;
- POINTTOPOINT — соединение типа «точка-точка»
- PROMISC — устройство находится в режиме «прослушивания» и принимает все фреймы.
- NOARP — отключена поддержка разрешения имен сетевого уровня.
- ALLMULTI — устройство принимает все групповые пакеты.
- NO-CARRIER — нет связи (не подключен кабель).
- DOWN — устройство отключено.

Еще можно вот так посмотреть:

```bash
ls /sys/class/net
additional-docker-sys-sys    docker0  eth1  veth12f3f63  veth2ab7aa7  veth62b367d  vethbe4b44c  vethd2a5488
br-a67eedd3a789  eth0     lo    veth26cd9d1  veth39864d5  vethb7e97f1  vethd29a7c3
```

***
## Подключённых USB и PCI устройствах? 

PC

```bash
lspci -vvv
```

USB

```bash
lsusb -vvv
```


***