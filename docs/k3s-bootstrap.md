# K3s Bootstrap

Runbook описывает базовое развертывание Kubernetes-кластера k3s для SUNBOYS на готовых Linux VM через Ansible.

## Назначение

Репозиторий закрывает начальный bootstrap кластера:

- установка k3s control-plane;
- подключение worker-нод;
- настройка единого входа API через Ingress;
- сохранение kubeconfig на управляющую машину;
- создание namespace `app`, `infra`, `monitoring`;
- проверка состояния кластера.

На этом этапе не устанавливаются приложения, monitoring stack, PostgreSQL, RabbitMQ и storage class. В k3s используется встроенный Traefik Ingress Controller.

## Требования

На управляющей машине:

- Ansible;
- SSH-ключ для подключения к VM;
- доступ к репозиторию `kubernetes-bootstrap`.

Локальный `kubectl` не обязателен для запуска playbook'ов `create-namespaces.yml` и `verify-cluster.yml`: они выполняют `/usr/local/bin/kubectl` на control-plane. После `bootstrap-k3s.yml` kubeconfig копируется локально, поэтому локальный `kubectl` нужен только для ручной работы с кластером с управляющей машины.

На каждой VM:

- Linux с systemd;
- пользователь с sudo-правами;
- доступ в интернет для скачивания k3s installer;
- доступность control-plane для worker-нод по `6443/tcp`;
- доступ между нодами по `8472/udp`, если используется стандартный Flannel VXLAN.

## Inventory

Создайте рабочий inventory из примера:

```bash
cp ansible/inventory.example.yml ansible/inventory.yml
```

Заполните реальные адреса и SSH-пользователя:

```yaml
all:
  vars:
    ansible_user: deployer
    ansible_ssh_private_key_file: ~/.ssh/id_ed25519

  children:
    k3s_server:
      hosts:
        control-plane-1:
          ansible_host: 31.192.111.254
          k3s_node_ip: 10.0.0.10

    k3s_agents:
      hosts:
        worker-1:
          ansible_host: 31.192.111.255
          k3s_node_ip: 10.0.0.11
```

`ansible_host` - адрес, по которому Ansible подключается к серверу по SSH.

`k3s_node_ip` - адрес, который k3s использует как IP Kubernetes-ноды. Если у сервера есть публичный и приватный IP, обычно `ansible_host` указывает на публичный SSH-адрес, а `k3s_node_ip` - на приватный адрес внутри сети кластера.

Не задавайте `ansible_become: true` глобально в `all.vars`: это может заставить delegated-задачи на `localhost` выполнять sudo. Playbook'и сами включают `become: true` там, где он нужен.

## Переменные

Основные переменные лежат в `ansible/group_vars/all.yml`.

| Переменная | Назначение |
| --- | --- |
| `k3s_version` | Версия k3s для установки. |
| `k3s_server_url` | URL Kubernetes API control-plane для подключения worker-нод. |
| `kubeconfig_local_path` | Локальный путь, куда будет сохранен kubeconfig. |
| `cluster_namespaces` | Namespace, которые создает `create-namespaces.yml`. |
| `ingress_host` | Единый hostname для публичного API. По умолчанию используется `sslip.io` от IP control-plane. |
| `api_ingress_routes` | Список маршрутов `/api/*` на Kubernetes Service. |
| `api_ingress_health_checks` | HTTP-проверки API через единый ingress-адрес. |

По умолчанию kubeconfig сохраняется в `~/.kube/sunboys-k3s.yaml` или в путь из переменной окружения `KUBECONFIG`, если она задана.

## Запуск

Установите k3s control-plane, подключите worker-ноды и сохраните kubeconfig:

```bash
ansible-playbook -i ansible/inventory.yml ansible/playbooks/bootstrap-k3s.yml
```

Создайте базовые namespace:

```bash
ansible-playbook -i ansible/inventory.yml ansible/playbooks/create-namespaces.yml
```

Настройте единый вход для API через Traefik Ingress:

```bash
ansible-playbook -i ansible/inventory.yml ansible/playbooks/install-ingress.yml
```

По умолчанию playbook создает Ingress `sunboys-api` в namespace `app` и маршрутизирует:

| Path | Service |
| --- | --- |
| `/api/auth` | `auth-service:80` |
| `/api/classrooms` | `main-service:80` |
| `/api/tasks` | `main-service:80` |
| `/api/submissions` | `main-service:80` |

Перед запуском убедитесь, что эти `Service` уже существуют, или измените `api_ingress_routes` в `ansible/group_vars/all.yml`.

Проверьте кластер:

```bash
ansible-playbook -i ansible/inventory.yml ansible/playbooks/verify-cluster.yml
```

`verify-cluster.yml` печатает вывод команд:

- `kubectl cluster-info`;
- `kubectl get nodes -o wide`;
- `kubectl get namespace app infra monitoring`;
- `kubectl -n kube-system get pods`.
- `kubectl -n app get ingress sunboys-api -o wide`;

Если sudo на удаленной VM требует пароль, добавьте `-K`:

```bash
ansible-playbook -K -i ansible/inventory.yml ansible/playbooks/bootstrap-k3s.yml
```

## Ручная проверка

После bootstrap можно работать с кластером с управляющей машины, если установлен локальный `kubectl`:

```bash
kubectl --kubeconfig ~/.kube/sunboys-k3s.yaml cluster-info
kubectl --kubeconfig ~/.kube/sunboys-k3s.yaml get nodes -o wide
kubectl --kubeconfig ~/.kube/sunboys-k3s.yaml get ns app infra monitoring
kubectl --kubeconfig ~/.kube/sunboys-k3s.yaml -n kube-system get pods
```

Без локального `kubectl` можно выполнить проверку на control-plane:

```bash
ssh deployer@<control-plane-ip>
sudo /usr/local/bin/kubectl --kubeconfig /etc/rancher/k3s/k3s.yaml get nodes -o wide
```

## Ожидаемый результат

Кластер считается готовым, когда:

- `verify-cluster.yml` завершается без ошибок;
- control-plane и worker-ноды находятся в статусе `Ready`;
- namespace `app`, `infra`, `monitoring` существуют;
- pod'ы в `kube-system` находятся в `Running` или `Completed`.

## Troubleshooting

### `No such file or directory: kubectl`

Это означает, что playbook пытается выполнить локальный `kubectl`, которого нет на управляющей машине. Актуальные `create-namespaces.yml` и `verify-cluster.yml` выполняют kubectl на control-plane, поэтому обновите репозиторий и повторите запуск.

### `sudo: a password is required` на localhost

Проверьте, что в `ansible/inventory.yml` нет глобального `ansible_become: true` в `all.vars`. Для удаленных задач become уже включен в playbook'ах.

### Worker-нода не подключается

Проверьте доступность Kubernetes API control-plane с worker-ноды:

```bash
curl -k https://<control-plane-ip>:6443
```

Если соединения нет, проверьте firewall/security group для `6443/tcp`.

### Node в статусе `NotReady`

На control-plane проверьте системные pod'ы:

```bash
sudo /usr/local/bin/kubectl --kubeconfig /etc/rancher/k3s/k3s.yaml -n kube-system get pods
```

На проблемной worker-ноде проверьте агент:

```bash
sudo systemctl status k3s-agent
sudo journalctl -u k3s-agent -n 100 --no-pager
```

На control-plane проверьте server:

```bash
sudo systemctl status k3s
sudo journalctl -u k3s -n 100 --no-pager
```

### Kubeconfig смотрит не в тот кластер

Передайте kubeconfig явно:

```bash
kubectl --kubeconfig ~/.kube/sunboys-k3s.yaml get nodes
```

Или задайте переменную окружения:

```bash
export KUBECONFIG=~/.kube/sunboys-k3s.yaml
```

### K3s уже установлен

Playbook проверяет наличие systemd unit-файлов `k3s.service` и `k3s-agent.service`. Если нужно полностью пересоздать кластер, удаляйте k3s вручную штатными uninstall-скриптами и только после отдельного плана, потому что это разрушительная операция.

