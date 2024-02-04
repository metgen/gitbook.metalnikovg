---
description: >-
  Обновление установленных приложени в Windows при помощи Windows Package
  Manager (winget)
---

# Обновление приложений Windows при помощи Winget

Windows Package Manager (winget) — это менеджер пакетов пакетов предустановленный в Windows 10 и Windows 11 позволяет устанавливать, обновлять программы и пакеты из командной строки.

Для ее использования нужно, открыть Powershell или Терминал от имени администратора. И ввести команду

```powershell
winget
```

После этого вы увидите подсказки для использования данной консольной утилиты.

<figure><img src="../../.gitbook/assets/winget-help.png" alt=""><figcaption></figcaption></figure>

В данной статье мы рассмотрим обновление установленных приложений при помощи нее. чтобы проверить список доступных обновлений, введите

```powershell
winget upgrade
```

<figure><img src="../../.gitbook/assets/winget-upgrade.png" alt=""><figcaption></figcaption></figure>

Чтобы установить все доступные обновления приложений используйте команду

```powershell
winget upgrade --all
```

или

```powershell
winget upgrade --all -h
```

для тихой установки

Чтобы обновить конкретное приложение, например браузер Mozilla Firefox, нужно указать ID пакета

<figure><img src="../../.gitbook/assets/winget-upgrade-firefox.png" alt=""><figcaption></figcaption></figure>

```powershell
winget upgrade "Mozilla.Firefox"
```
