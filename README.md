# kubernetes-bootstrap

Ansible playbooks для базового развертывания k3s-кластера SUNBOYS на готовых Linux VM.

## Что входит

- Установка k3s control-plane.
- Подключение worker-нод.
- Сохранение локального kubeconfig.
- Создание namespace `app`, `infra`, `monitoring`.
- Проверка доступности кластера через `kubectl` на control-plane.

## Требования

- Ansible на управляющей машине.
- Linux VM с SSH-доступом и sudo-правами.
- Открытый доступ worker-нод к control-plane по `6443/tcp`.
- Локальный `kubectl` не обязателен для playbook'ов, но полезен для ручной работы после bootstrap.

## Быстрый старт

```bash
cp ansible/inventory.example.yml ansible/inventory.yml
ansible-playbook -i ansible/inventory.yml ansible/playbooks/bootstrap-k3s.yml
ansible-playbook -i ansible/inventory.yml ansible/playbooks/create-namespaces.yml
ansible-playbook -i ansible/inventory.yml ansible/playbooks/verify-cluster.yml
```

Перед запуском замените IP-адреса, `k3s_node_ip`, SSH-пользователя и путь к ключу в `ansible/inventory.yml`.

Если sudo на удаленной VM требует пароль:

```bash
ansible-playbook -K -i ansible/inventory.yml ansible/playbooks/bootstrap-k3s.yml
```

Подробная инструкция находится в `docs/k3s-bootstrap.md`.
