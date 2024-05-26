# Включение SSH

Если вы столкнулись с ошибкой при подключении по ssh к Fedora Linux&#x20;

`ssh: connect to host 'hostname/ip' port 22: Connection refused`

Проверьте установлен ли SSH сервер в вашей системе

```bash
rpm -qa | grep openssh-server
```

В ответ вы должны получить версию SSH сервера, если в ответ ничего не получили, то установите его

```bash
sudo dnf install openssh-server
```

Когда выполнится установка или если он уже был установлен — включить службу systemd sshd , чтобы демон SSH запускался после каждой перезагрузки:

```bash
sudo systemctl enable sshd
```

После включения службы `SSHD` еще раз используйте команду `systemclt` для запуска SSH-сервера:

```bash
sudo systemctl start sshd
```

Когда все будет готово, проверьте состояние SSH-сервера, используя следующую команду:

```bash
sudo systemctl status sshd
```

Теперь вы должны увидеть, что порт ssh открыт для новых входящих соединений:

```bash
sudo ss -lt
```

<figure><img src="../../../.gitbook/assets/ssh_check.png" alt=""><figcaption></figcaption></figure>

Теперь с подключением по ssh не должно возникнуть проблем.
