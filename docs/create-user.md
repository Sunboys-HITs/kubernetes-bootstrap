# Создание пользователя на сервере

## 1. Зайти на машину по SSH (под root)

```bash
ssh root@1.1.1.1
```

## 2. Создать пользователя

```bash
useradd -m -s /bin/bash deployer
```

- `-m` — создать домашнюю папку `/home/deployer`
- `-s /bin/bash` — установить bash по умолчанию

## 3. Дать sudo без пароля

```bash
echo "deployer ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/deployer
chmod 0440 /etc/sudoers.d/deployer
```

## 4. Создать папку .ssh и добавить публичный ключ

На локальной машине прочитай свой публичный ключ:

```bash
cat ~/.ssh/id_ed25519.pub
```

Скопируй вывод. На сервере (под root) выполни:

```bash
mkdir -p /home/deployer/.ssh
echo "<вставь_публичный_ключ>" >> /home/deployer/.ssh/authorized_keys
chmod 700 /home/deployer/.ssh
chmod 600 /home/deployer/.ssh/authorized_keys
chown -R deployer:deployer /home/deployer/.ssh
```

## 5. Проверить

```bash
ssh deployer@1.1.1.1 "sudo whoami"
```

Должно вернуть `root` без запроса пароля.
