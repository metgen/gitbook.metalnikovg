---
description: GRUB (GRand Unified Bootloader)
---

# Смена темы Grub

GRUB (GRand Unified Bootloader) — загрузчик операционной системы от проекта GNU. GRUB позволяет пользователю иметь несколько установленных операционных систем и при включении компьютера выбирать одну из них для загрузки. Помимо этого в нем можно выбирать ядро с которого хотим загрузить ОС.

В обычном виде он представляет собой просто черный экран с меню загрузки. Но его можно немного кастомизировать темами, их существует огромное количество, просмотреть и выбрать понравившуюся можно например на этом [сайте](https://www.gnome-look.org/browse?cat=109\&ord=latest)

Мне например понравилась тема [CyberRe](https://www.gnome-look.org/p/1420727).

<figure><img src="../../../.gitbook/assets/cyberre.png" alt=""><figcaption></figcaption></figure>

Но она имеет один небольшой недостаток, при выборе пункта загрузки появлялось черное окно терминала на несколько секунд, только потом начиналась загрузка системы. Но я исправил этот недостаток, скачать ее доработанную версию можно с [GitHub](https://github.com/metgen/CyberReFersh), для этого введите в терминале

```bash
git clone https://github.com/metgen/CyberReFersh.git
```

Затем перейдите в папку с темой и введите команду, для копирования темы в каталог /boot/grub2/themes

```bash
sudo mkdir -p /boot/grub2/themes/CyberRe && sudo cp -r CyberRe /boot/grub2/themes
```

Теперь нужно отредактировать конфигурацию Grub

```bash
sudo nano /etc/default/grub
```

Добавляем или изменяем следующие строки:

<mark style="color:purple;">GRUB\_TIMEOUT=30 #время в секундах до начала загрузки ОС</mark>

<mark style="color:purple;">GRUB\_THEME="/boot/grub2/themes/Insert-Theme/theme.txt" #путь до темы</mark>

<mark style="color:purple;">#GRUB\_TERMINAL\_OUTPUT="console"</mark>

<mark style="color:purple;">GRUB\_GFXMODE=1920x1080,auto</mark>

<mark style="color:purple;">GRUB\_GFXPAYLOAD\_LINUX="keep"</mark>

Далее сохраняем файл Ctrl+S и выполняем обновление конфигурации загрузчика

```bash
sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg
```

{% hint style="info" %}
В будущем планирую сделать скрипт автоматической установки
{% endhint %}
