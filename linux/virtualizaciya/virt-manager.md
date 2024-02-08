# VirtManager

`virt-manager`— графическая консоль для управления виртуальными машинами через [libvirt](./) от компании RedHat. Чаще всего используются виртуальные машины QEMU/KVM, но контейнеры Xen и libvirt LXC хорошо поддерживаются.

С помощью Virt-Manager можно, создавать, редактировать, запускать и останавливать виртуальные машины на гипервизоре KVM. Можно выполнять настройку параметров виртуальных машин, что значительно упрощает работу по сравнению с управлением KVM из интерфейса командной строки.

<figure><img src="../../.gitbook/assets/virt-manager-mainpng" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
Red Hat Linux изменила статус virt-manager в RHEL 8 на deprecated (устарел), и возможно в следующих релизах OC этот пакет будет недоступен. Вместо него предлагается использовать веб интерфейс **Cockpit**. Однако на данный момент в модуле управления KVM в [Cockpit](cockpit.md) пока не хватает всех необходимых функций, доступных в virt-manager.
{% endhint %}

## Установка

### DNF

```bash
sudo dnf install virt-manager
```

### APT

```bash
sudo apt-get install virt-manager
```

## Установка виртуальной машины
