# kubernetes-bootstrap

Ansible playbooks для базового развертывания k3s-кластера SUNBOYS на готовых Linux VM.

## Что входит

- Установка k3s control-plane.
- Подключение worker-нод.
- Сохранение локального kubeconfig.
- Создание namespace `app`, `infra`, `monitoring`.
- Проверка доступности кластера через `kubectl`.

## Быстрый старт

```bash
cp ansible/inventory.example.yml ansible/inventory.yml
ansible-playbook -i ansible/inventory.yml ansible/playbooks/bootstrap-k3s.yml
ansible-playbook -i ansible/inventory.yml ansible/playbooks/create-namespaces.yml
ansible-playbook -i ansible/inventory.yml ansible/playbooks/verify-cluster.yml
```

Перед запуском замените IP-адреса и SSH-пользователя в `ansible/inventory.yml`.

Подробная инструкция находится в `docs/k3s-bootstrap.md`.
