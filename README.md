# k8s-lab

Тестовый стенд Kubernetes, собранный вручную по методичке **Kubernetes The Hard Way**.
Используется как песочница для проверки и отработки реальных компонентов кластера:
хранение, ingress, observability, GitOps, безопасность, service mesh и т.д.

## Топология

| Роль         | Хост                | IP            | Pod CIDR      |
|--------------|---------------------|---------------|---------------|
| Control plane| `cp0.kubernetes.lab` | 172.24.254.214 | —             |
| Worker       | `w0.kubernetes.lab` | 172.24.254.216 | 10.200.0.0/24 |
| Worker       | `w1.kubernetes.lab` | 172.24.254.217 | 10.200.1.0/24 |

Внешний LoadBalancer (Traefik / MetalLB VIP): `172.24.254.221`.

## Структура репозитория

Репозиторий разбит по компонентам стенда. Каталог верхнего уровня — тема,
внутри — манифесты, values-файлы чартов и конфигурация.

### `install/` — базовая установка кластера
Артефакты развёртывания голого кластера (Kubernetes The Hard Way).
- `install/kubernetes-the-hard-way/` — конфиги и юниты установки:
  - `ca.conf`, `encryption-config.yaml` — конфигурация PKI и шифрования etcd;
  - `configs/` — конфиги бинарников (kubelet, kube-proxy, kube-scheduler,
    kube-apiserver-to-kubelet, containerd, сетевые конфиги bridge/loopback);
  - `units/` — systemd unit-файлы (etcd, kube-apiserver, kube-controller-manager,
    kube-scheduler, kubelet, kube-proxy, containerd);
  - `hosts`, `machines.txt`, `downloads-*.txt` — справочные списки нод и дистрибутивов;
- `calico.yaml` — CNI Calico;
- `api-server-cm.yaml` — CoreDNS/API-сервер конфигмапа (при ручной настройке DNS);
- `coredns-config.yaml`, `coredns-deploy.yaml`, `coredns-service.yaml` — развёртывание CoreDNS.

### `node-configs/` — конфигурация нод
Снимки реальной конфигурации каждой ноды кластера.
- `cp0/` — control plane: netplan, systemd-юниты control-plane компонентов (включая
  `kube-service-route.service`), конфиг kube-scheduler;
- `w0/`, `w1/` — workers: netplan, containerd `config.toml`, kubelet/kube-proxy
  конфиги, Calico conflist, journald-лимит.

### `csi/` — хранение и резервное копирование
- `local-path-provisioner-*` — локальный provisioner;
- `longhorn-values.yml`, `longhorn-ingress.yml`, `longhorn-backup-policy.yml` —
  Longhorn: values, ingress и политика бэкапов;
- `longhorn-s3-backup.yaml`, `longhorn-nfs-backup.yaml` — бэкап-таргеты S3 и NFS;
- `nfs-provisioner-values.yml`, `nfs-csi-test.yaml` — NFS CSI driver и тест;
- `test-nfs-backup.yaml`, `test-nfs-restore.yaml`, `test-s3-backup.yaml`,
  `test-s3-restore.yaml` — сценарии проверки бэкапа/восстановления;
- `coredns-corefile-kubernetes-lab.yml` — кастомизация CoreDNS для стенда.

### `registry/` — реестр образов
- `harbor/harbor-values-kubernetes-lab.yml` — values-файл развёртывания Harbor
  (ns `harbor`): longhorn-SC хранилище, ingress `harbor.kubernetes.lab`, TLS через
  cert-manager, trivy выключен.

### `ingress/` — входящий трафик
- `traefik/values.yml` — values Traefik (основной ingress-провайдер стенда);
- `traefik/longhorn-ingress.yml` — ingress для Longhorn через Traefik;
- `ingress-nginx.yml`, `ingress-nginx-new.yml` — альтернативный вариант ingress-nginx;
- `gateway-api/` — Kubernetes Gateway API:
  - `gateway-api-crds-v1.5.1-standard.yaml` — CRD стандартного канала Gateway API;
  - `gateway.yml` — Gateway/HTTPRoute с TLSPolicy (порт 8443);
  - `gw-demo-app.yml` — тестовое приложение `gw-demo` для проверки Gateway API.

### `certs/` — PKI и TLS
- `cert-manager-values.yml`, `cert-manager-defaults.yaml` — чарт cert-manager;
- `clusterissuer.yml` — промежуточный CA стенда (intermediate-ca-issuer);
- `root-ca-secret-template.yml` — шаблон секрета с корневым CA;
- `trust-manager.yml`, `trust-manager-defaults.yaml`, `trust-bundle.yml` —
  автоматическое распространение trust-бандла по кластеру;
- `test-certificate.yml` — тестовый сертификат для проверки выдачи.

### `identity/` — аутентификация и авторизация
- `keycloak/` — Keycloak как IdP:
  - `keycloak.yml` — развёртывание Keycloak;
  - `ingress.yml`, `netpol.yml`, `rbac.yml`, `servicemonitor.yml` — ingress,
    NetworkPolicy, RBAC и мониторинг;
  - `kubeconfig-oidc-template.yml` — шаблон kubeconfig через OIDC;
- `oidc/` — OIDC-интеграция API-сервера:
  - `oidc-flags.md` — заметки по флагам `--oidc-*` (включая грабли с группами);
  - `rbac-oidc.yml` — RBAC для OIDC-групп.

### `gitops/` — GitOps (Argo CD)
- `argocd/values.yml` — values Argo CD;
- `argocd/applications/lab-demo.yml` — Application для докена;
- `demo/` — пример приложения, управляемого через GitOps:
  - `lab-demo.yml` — базовый манифест;
  - `canary.yml` — canary-деплой (например для Progressive Delivery).

### `observability/` — мониторинг и логирование
- `kps-values.yml` — values kube-prometheus-stack;
- `alertmanager/basic-alerts.yml`, `alertmanager/webhook-sink.yml` — базовые алерты
  и webhook-получатель для проверки end-to-end;
- `blackbox/` — blackbox-exporter:
  - `blackbox-values.yml` — отдельный чарт (KPS не включает blackbox);
  - `blackbox-probes.yml` — Probe для проверки HTTPS-ingress стенда;
- `loki-values.yml`, `vector-values.yml`, `vector-agent-configmap.yml`,
  `loki-datasource.yml` — связка Loki + Vector (логирование);
- `dashboards/` — дашборды Grafana (k8s-views, istio, longhorn, traefik) и
  `dashboards-cm.yml` для их provision'а;
- `kubelet-endpoints.yml`, `service-monitors.yml`, `traefik-metrics.yml` —
  ServiceMonitor'ы и endpoints для сбора метрик.

### `security/` — безопасность
- `network-policies/` — NetworkPolicy (default-deny и выборочные разрешения):
  - `lab-demo-netpol.yml`, `lab-db-netpol.yml`, `lab-db-postgres.yml`,
    `observability-netpol.yml`.

### `mesh/` — service mesh
- `istio/istiod-minimal.generated.yaml` — минимальная установка istiod;
- `istio/peerauth-lab-demo.yml` — PeerAuthentication (mTLS) для докена.

### `kyverno/` — политики
- `kyverno-values.yml`, `kyverno-values-default.yml`, `kyverno-values-kubernetes-lab.yml`
  — values-варианты чарта Kyverno;
- `kyverno-policy.yml`, `kyverno-ca-policy.yml` — примеры политик (включая CA).

### `metallb/` — балансировка
- `metallb-values.yml` — values MetalLB;
- `metallb-pool.yml` — IPAddressPool (пул адресов LB);
- `metallb-test.yml` — проверка выдачи адреса.