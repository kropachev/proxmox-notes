# Подготовка Proxmox к пробросу устройств в виртуальную машину (PCIe Passthrough)

Когда виртуальной машине нужен прямой доступ к физическому оборудованию хоста - например, запустить **локальную LLM** на видеокарте или отдать NVMe-диск файловому хранилищу TrueNAS, стандартная виртуализация не подходит: GPU нужно увидеть в VM как настоящее железо со своими драйверами и полной производительностью, а не как эмулированный VirtIO-адаптер.

Для этого есть **PCIe Passthrough** - передача физического PCIe-устройство в монопольное использование виртуальной машине. Хост в этот момент с устройством не работает - все запросы идут напрямую из гостевой ОС.

В этой статье разберем подготовку **хоста Proxmox VE 9** к проброс GPU - на примере **NVIDIA RTX**.

При подготовке использованы материалы по ссылкам (и различные форумы на которых поднимались вопросы устаревших инструций и исправления ошибок):
* [PCI(e) Passthrough - Proxmox VE Wiki](https://pve.proxmox.com/wiki/PCI(e)_Passthrough)
* [Proxmox VE Admin Guide - PCI Passthrough](https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_pci_passthrough)


## Предпосылки

Чтобы все заработало, нам понадобится:

* **CPU с поддержкой Intel VT-d / AMD-Vi**. В примере используется Intel 12-го поколения - все десктопные модели поддерживают, нужно только включить в BIOS.
* **Материнская плата с IOMMU** - современные чипсеты Intel (B660/H670/Z690/B760/Z790) и AMD (B550/X570/B650/X670) подходят.
* **Хосту нужна встроенная графика**. На Intel это встроенная **Intel UHD** (в моделях без суффикса `F`). После проброса дискретной GPU хост продолжит работать на встроенной.

## Настройка BIOS

В BIOS нужно включить следующие опции, названия зависят от вендора материнской платы.

| Опция | Значение | Зачем |
|---|---|---|
| **Intel Virtualization Technology (VT-x)** | Enabled | Аппаратная виртуализация |
| **Primary Grapic Adapter** | Onboard / Internal Graphics | Чтобы хост грузился на встроенной графике, а не на NVIDIA |
| **Above 4G Decoding** | Enabled | Требуется современным GPU с большим BAR |
| **Clever Access Memory (C.A.M.)** / **Resizable BAR (ReBAR)** | Enabled | Одна и та же технология под разными названиями: AMD - SAM, MSI - C.A.M., остальные - ReBAR. RTX 50-series поддерживает, дает прирост на LLM |
| **VT-d / IOMMU** | Enabled | **Обязательно** для passthrough |
| **Secure Boot** | Disabled | Упрощает работу с модулями ядра на хосте |
| **CSM (Compatibility Support Module)** | Disabled | Только UEFI, без Legacy BIOS |

Сохраняем настройки и загружаемся в Proxmox.

## Параметры ядра (GRUB)

Чтобы ядро Linux включило IOMMU и зарезервировало устройство для VM, нужно изменить параметры загрузки.

В рассматриваемом примере Proxmox VE установлен на **ext4/LVM**, и используется **GRUB**. Если у вас Proxmox на **ZFS** - то ниже я приведу пару строк о systemd-boot (по идее должно сработать, но не проверял).

Открываем файл настроек загрузчика

```bash
nano /etc/default/grub
```

Ищем строку `GRUB_CMDLINE_LINUX_DEFAULT`, в нее нужно дописать `intel_iommu=on iommu=pt`:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```

Параметр `iommu=pt` (passthrough mode) снижает накладные расходы для устройств, которые **не** пробрасываются в VM, то есть для всего остального оборудования хоста.

Применяем
```bash
update-grub
```

<details>
<summary>Если Proxmox на ZFS (systemd-boot)</summary>

На ZFS-инсталляциях Proxmox VE 9 использует не GRUB, а **systemd-boot**, и `update-grub` ничего не сделает. Проверьте тип загрузчика:

```bash
proxmox-boot-tool status
```

Если в выводе `systemd-boot`, отредактируйте `/etc/kernel/cmdline` (одна строка!):

```
root=ZFS=rpool/ROOT/pve-1 boot=zfs intel_iommu=on iommu=pt
```

Затем:

```bash
proxmox-boot-tool refresh
```

</details>

Перезагружаем сервер:

```bash
reboot
```

После перезагрузки проверяем, что IOMMU поднялся:

```bash
dmesg | grep -e DMAR -e IOMMU
cat /proc/cmdline
```

В выводе `dmesg` должны быть строки вида `DMAR: IOMMU enabled` и `DMAR: Intel(R) Virtualization Technology for Directed I/O`. В `/proc/cmdline` - наши параметры.

## Загрузка модулей VFIO

**VFIO** (Virtual Function I/O) - подсистема ядра, через которую устройство передается в QEMU/KVM. Нужно загружать ее при старте системы.

В прошлых версиях использовался файл `/etc/modules`, который объявлен устаревшим - вместо него используется каталог `/etc/modules-load.d/` с отдельными `.conf`-файлами на каждую группу модулей.

Создаем файл
```bash
nano /etc/modules-load.d/vfio.conf
```

Содержимое:
```
vfio
vfio_iommu_type1
vfio_pci
```

> В старых гайдах часто встречается еще `vfio_virqfd` - в современных ядрах он слит с `vfio` и **больше не нужен**.

## Blacklist драйверов NVIDIA на хосте

Чтобы хост не подхватил GPU своими драйверами и не "забрал" ее себе, отключаем их на уровне модулей.

Создаем файл 
```bash
nano /etc/modprobe.d/blacklist-nvidia.conf
```
Добавляем строки:
```
blacklist nouveau
blacklist nvidia
blacklist nvidiafb
blacklist nvidia_drm
```

`nouveau` - открытый драйвер NVIDIA в ядре, `nvidia*` - проприетарные модули, которые могут подтянуться, если "кто-то" по ошибке поставит пакет.

## Определение PCI-адреса и ID видеокарты

Смотрим, как ядро видит нашу видеокарту NVIDIA:

```bash
lspci -nn | grep -i nvidia
```

Пример вывода для RTX 5060:

```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GB206 [GeForce RTX 5060 Ti] [10de:2d04] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:22eb] (rev a1)
```

Здесь:

* `01:00.0` и `01:00.1` - **PCI-адреса** двух функций карты (видео и HDMI-аудио);
* `[10de:2d04]` и `[10de:22eb]` - **vendor:device ID**, понадобятся для привязки к vfio-pci. У вашей карты они могут отличаться - всегда сверяйтесь с реальным `lspci`.

## Привязка GPU к vfio-pci

Теперь говорим ядру перенаправлять устройства в VFIO.

Создаем конфиг
``` bash
nano /etc/modprobe.d/vfio.conf
```
Вставляем строку
```
options vfio-pci ids=10de:2d04,10de:22eb disable_vga=1
```

Нужно подставить свои ID из `lspci -nn`. Указываем оба ID - и видео, и HDMI-аудио, иначе аудио-функция останется висеть на стандартном драйвере.

## Применение и перезагрузка

Пересобираем initramfs, чтобы все правки попали в загрузочный образ:

```bash
update-initramfs -u -k all
```

Перезагружаемся:

```bash
reboot
```

## Проверка IOMMU-групп

После перезагрузки проверяем, что GPU попала в **изолированную** IOMMU-группу:

```bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
  n=${d#*/iommu_groups/*}; n=${n%%/*}
  printf 'IOMMU Group %s ' "$n"; lspci -nns "${d##*/}"
done | sort -n
```

Ищем группу с нашей NVIDIA. В группе должно быть только две карты (`01:00.0` и `01:00.1`).  
Если в одной группе с GPU оказались сетевые карты, USB-контроллеры и т.п. - так бывает на бюджетных платах, когда GPU стоит в слоте от чипсета. В этом случае могу предложить два варианта:
1. Переставить GPU в верхний x16-слот от CPU (`PCIEX16_1`) - самый надежный способ.
2. Обновить BIOS - иногда вендоры исправляют разметку IOMMU-групп.

> ⚠️ В интернете встречается **ACS Override Patch** - он искусственно режет группы. **Не рекомендуется** новичкам. Этот вопрос выходит за границы текущего гайда...

## Проверка привязки

Финальная проверка - убеждаемся, что драйвер GPU именно `vfio-pci`:

```bash
lspci -nnk -s 01:00
```

В выводе должна быть строка:

```
Kernel driver in use: vfio-pci
```

Если так - все готово, можно переходить к следующему шагу, созданию виртуальной машины и пробросу GPU.
