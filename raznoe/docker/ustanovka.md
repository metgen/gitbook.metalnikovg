---
description: Установка Docer
---

# Установка

### Установка на Ubuntu

Для установки docker и docker-compose выполните следующие команды

```bash
curl -sSL https://get.docker.com | sh
sudo groupadd docker
sudo usermod -aG docker ${USER}
sudo apt install docker-compose
sudo systemctl enable docker
sudo reboot
```

Проверьте работу Docker

```
docker run hello-world
```

В ответ вы должны получить следующее сообщение:

> Hello from Docker! This message shows that your installation appears to be working correctly.
>
> To generate this message, Docker took the following steps:
>
> 1. The Docker client contacted the Docker daemon.
> 2. The Docker daemon pulled the "hello-world" image from the Docker Hub. (amd64)
> 3. The Docker daemon created a new container from that image which runs the executable that produces the output you are currently reading.
> 4. The Docker daemon streamed that output to the Docker client, which sent it to your terminal.
