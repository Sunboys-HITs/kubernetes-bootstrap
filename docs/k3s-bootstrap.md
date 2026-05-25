# K3s Bootstrap

Этот runbook описывает развертывание базового Kubernetes-кластера k3s для системы автотестирования SUNBOYS на готовых Linux VM.

## Что разворачивается

- Один control-plane узел k3s.
- Один или несколько worker-узлов k3s.
- Локальный kubeconfig для доступа к кластеру.
- Базовые namespace:
  - `app`
  - `infra`
  - `monitoring`

На этом этапе не устанавливаются приложения, ingress controller, мониторинг, PostgreSQL, RabbitMQ или Helm charts.

## Требования

- Linux VM с systemd.
- SSH-доступ к каждой ноде.
- Пользователь с sudo-правами.
- Доступ worker-нод к control-plane по `6443/tcp`.
- Доступ между нодами по `8472/udp`, если используется стандартный Flannel VXLAN.
- На управляющей машине установлены Ansible и kubectl.

## Подготовка inventory

Скопируйте пример inventory:

```bash
cp ansible/inventory.example.yml ansible/inventory.yml
```

Отредактируйте `ansible/inventory.yml`: замените IP-адреса, SSH-пользователя и при необходимости путь к приватному ключу.

Минимальная структура групп:

```yaml
k3s_server:
  hosts:
    control-plane-1:

k3s_agents:
  hosts:
    worker-1:
    worker-2:
```

## Переменные

Основные переменные находятся в `ansible/group_vars/all.yml`.

| Переменная | Назначение |
| --- | --- |
| `k3s_version` | Версия k3s для установки. |
| `k3s_server_url` | URL Kubernetes API control-plane. |
| `kubeconfig_local_path` | Локальный путь для kubeconfig. |
| `cluster_namespaces` | Список namespace, которые нужно создать. |

## Запуск

Установите control-plane, подключите worker-ноды и сохраните kubeconfig:

```bash
ansible-playbook -i ansible/inventory.yml ansible/playbooks/bootstrap-k3s.yml
```

Создайте базовые namespace:

```bash
ansible-playbook -i ansible/inventory.yml ansible/playbooks/create-namespaces.yml
```

Проверьте кластер:

```bash
ansible-playbook -i ansible/inventory.yml ansible/playbooks/verify-cluster.yml
```

Можно также выполнить проверки вручную:

```bash
kubectl --kubeconfig ~/.kube/sunboys-k3s.yaml cluster-info
kubectl --kubeconfig ~/.kube/sunboys-k3s.yaml get nodes -o wide
kubectl --kubeconfig ~/.kube/sunboys-k3s.yaml get ns app infra monitoring
kubectl --kubeconfig ~/.kube/sunboys-k3s.yaml -n kube-system get pods
```

## Ожидаемый результат

`kubectl get nodes` должен показать control-plane и все worker-ноды в статусе `Ready`.

Namespace `app`, `infra`, `monitoring` должны существовать:

```bash
kubectl --kubeconfig ~/.kube/sunboys-k3s.yaml get ns app infra monitoring
```

## Troubleshooting

### Worker-нода не подключается

Проверьте доступность Kubernetes API control-plane с worker-ноды:

```bash
curl -k https://<control-plane-ip>:6443
```

Проверьте firewall/security group для `6443/tcp`.

### Node в статусе NotReady

Проверьте системные pod'ы:

```bash
kubectl --kubeconfig ~/.kube/sunboys-k3s.yaml -n kube-system get pods
```

На проблемной ноде проверьте сервис:

```bash
sudo systemctl status k3s-agent
sudo journalctl -u k3s-agent -n 100 --no-pager
```

Для control-plane:

```bash
sudo systemctl status k3s
sudo journalctl -u k3s -n 100 --no-pager
```

### Kubectl смотрит не в тот кластер

Передайте kubeconfig явно:

```bash
kubectl --kubeconfig ~/.kube/sunboys-k3s.yaml get nodes
```

Или задайте переменную окружения:

```bash
export KUBECONFIG=~/.kube/sunboys-k3s.yaml
```

### K3s уже установлен

Playbook проверяет наличие systemd unit-файлов `k3s.service` и `k3s-agent.service`. Если нужно полностью пересоздать кластер, сначала вручную удалите k3s штатными uninstall-скриптами на соответствующих нодах. Не делайте это на рабочем кластере без отдельного плана миграции.

### Sudo требует пароль

Запустите playbook с запросом become-пароля:

```bash
ansible-playbook -K -i ansible/inventory.yml ansible/playbooks/bootstrap-k3s.yml
```
