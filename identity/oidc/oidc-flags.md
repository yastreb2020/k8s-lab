# Фаза 12 — OIDC-флаги kube-apiserver (cp0)

OIDC-аутентификация к8s через Keycloak. На cp0 kube-apiserver запущен
systemd-сервисом (НЕ static pod, KTHW) — флаги добавлены в
`/etc/systemd/system/kube-apiserver.service`, секция `[Service] ExecStart`:

```
--oidc-ca-file=/var/lib/kubernetes/keycloak-ca.crt \
--oidc-client-id=kubectl \
--oidc-issuer-url=https://keycloak.kubernetes.lab/realms/kubernetes.lab \
--oidc-groups-claim=groups \
--oidc-username-claim=preferred_username \
```

## Развёртывание (выполнено 2026-10-07)

1. **CA Keycloak на cp0** (для верификации discovery-endpoint'а):
   ```
   cp test-dir/neurotech-ca.crt cp0:/var/lib/kubernetes/keycloak-ca.crt
   chmod 644 /var/lib/kubernetes/keycloak-ca.crt   # apiserver читает как root
   ```
   Файл `neurotech-ca.crt` — root CA (Neurotech CA) стенда. Keycloak лист
   выпускается K8s Intermediate CA (дочерний от Neurotech), chain из 2
   сертификатов — intermediate подается сервером в TLS-рукопожатии.

2. **/etc/hosts на cp0**:
   ```
   172.24.254.221 keycloak.kubernetes.lab
   ```
   (Keycloak за Ingress/Traefik, MetalLB-IP.)

3. **Правка юнита** + `systemctl daemon-reload && systemctl restart kube-apiserver`.
   Бэкап юнита: `/etc/systemd/system/kube-apiserver.service.bak-oidc`.

## Профилирование Keycloak realm (скрипт /tmp/kc_provision.py, 2026-10-07)

- realm `kubernetes.lab` (displayName "Kubernetes Lab OIDC")
- клиент `kubectl`: **public** (без client-secret), `standardFlowEnabled`,
  PKCE (`s256`), `directAccessGrantsEnabled=false`, `serviceAccountsEnabled=false`
- `redirectUris`: `http://127.0.0.1:8080`, `http://localhost:8080`,
  `http://172.22.119.118:8080` (WSL eth0) + `http://127.0.0.1:8000`,
  `http://localhost:8000`, `http://172.22.119.118:8000` (kubelogin)
- `webOrigins`: `+`
- protocol-mapper'ы (claim `groups` → к8s-группа **`developers`**, БЕЗ префикса):
  | mapper | type | config |
  |---|---|---|
  | `groups` | oidc-group-membership-mapper | `claim.name=groups, full.path=false, multivalued=true` |
  | `preferred-username` | oidc-usermodel-attribute-mapper | `claim.name=preferred_username, user.attribute=username` |
  | `email` | oidc-usermodel-attribute-mapper | `claim.name=email, user.attribute=email` |
  | `audience-kubectl` | oidc-audience-mapper | `included.client.audience=kubectl` |
- группа `developers` (path `/developers`)
- пользователь `kate` (email `kate@kubernetes.lab`), в группе `developers`,
  firstName/lastName **обязательны** (KC26 user-profile: required roles=[user])

> **ГРЯБО (KC26, 2026-10-07):** protocol-mapper'ы `oidc-usermodel-attribute-mapper`
> и `oidc-group-membership-mapper` в config используют ключ **`claim.name`**,
> НЕ `claim`. С `claim` mapper молча не выдаёт значение (claim в токене = null),
> при этом token exchange всё равно PASS — ловится только декодом access_token.
> Подтверждено: `groups=['developers']` в токене.

> **ГРЯБО (KC26, 2026-10-07):** создание realm = `POST /admin/realms`
> (не PUT, как в старых версиях). Join пользователя в группу =
> `PUT /admin/realms/{r}/users/{u}/groups/{g}` (не POST).
> Обновление пользователя = `PUT` (не PATCH).
> ClientRepresentation: поле `redirectUris` (не `validRedirectUris`),
> `accessTokenLifespan` в клиенте не принимается (400).

> **ГРЯБО (aud, 2026-10-07):** API-сервер сверяет JWT `aud` с
> `--oidc-client-id`. Валидный client id попадает в `aud` автоматически
> только если realm-клиент не переопределяет audience. Явный
> audience-mapper `included.client.audience=kubernetes-api` ЛОМАЛ aud
> (aud к8s не виден, token 401). Решение: mapper
> `included.client.audience=kubectl` — token принимает API-сервер.
> Проверено: `/version`, `namespaces`, `nodes`, `deployments` → 200.

> **ГРЯБО (groups prefix, 2026-10-07):** API-сервер k8s v1.32 к значениям
> `--oidc-groups-claim` **НЕ добавляет** префикс `system:groups:` (миф из
> старых туториалов). SelfSubjectReview: groups=`[developers
> system:authenticated]`. ClusterRoleBinding на `system:groups:developers`
> молча не даёт прав (403, но impersonation can-i путает — проверять
> SelfSubjectReview). Правильный subject: group **`developers`**.

## Пароль пользователя kate

Не в git. Сгенерирован при провижне, записан в:
`test-dir/kate-oidc.local.txt` (gitignored).

Для сброса:
```
kubectl -n identity exec deploy/keycloak -- /opt/keycloak/bin/kcadm.sh \
  update credentials --username kate --new-password '<new>' \
  --realm kubernetes.lab --token <admin-token>
```

## kubeoidc.conf (test-dir, gitignored)

Используется **exec-плагин kubelogin** (int128/kubelogin, v1.36.4,
`~/.local/bin/kubelogin`), не built-in `authProvider` — built-in в kubectl 1.35
требует legacy-блок с дефисом `auth-provider` + `idp-issuer-url`, а его
callback на фиксированном `localhost`-порту несовместим с WSL2 NAT + случайный
порт. kubelogin: `--listen-address`, refresh-токены, нормальный кэш.

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://api-server.kubernetes.lab:6443
    certificate-authority-data: <base64 CLUSTER CA (из kubeadmin.conf)>
  name: kubernetes.lab
contexts:
- context:
    cluster: kubernetes.lab
    user: kate
  name: kubernetes.lab
current-context: kubernetes.lab
users:
- name: kate
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubelogin
      interactive: true
      args:
      - get-token
      - --oidc-issuer-url=https://keycloak.kubernetes.lab/realms/kubernetes.lab
      - --oidc-client-id=kubectl
      - --certificate-authority-data=<base64 NEUROTECH CA (root CA стенда)>
      - --grant-type=authcode
      - --listen-address=0.0.0.0:8000
      - --skip-open-browser
```

**Две РАЗНЫЕ CA — не путать:**
- в `exec`-блоке (kubelogin): **Neurotech CA** (root CA стенда, ~1.5 KB b64) —
  для TLS discovery `https://keycloak.kubernetes.lab` (cert-manager лист
  выпущен K8s Intermediate → Neurotech).
- в `cluster` (kubectl): **cluster CA** (из kubeadmin.conf, ~2.5 KB b64) —
  для TLS до `api-server.kubernetes.lab:6443`.

Использование:
```
KUBECONFIG=~/test-dir/kubeoidc.conf kubectl get nodes
```
Первый запуск: `kubelogin` слушает `0.0.0.0:8000`, в Windows-браузере открыть
`http://localhost:8000/` → 302 на Keycloak → логин `kate` → callback на
`localhost:8000` → токен в кэш `~/.kube/cache/oidc-login` (access ~15 мин +
refresh_token — продлевается автоматически, браузер больше не нужен до logout).
`--skip-open-browser` — URL выводится в stderr, открывать вручную (или
`explorer.exe http://localhost:8000/` из WSL).

## RBAC

`rbac-oidc.yml` (эта папка): ClusterRole `lab-oidc-view` (read-only: nodes,
namespaces, pods, deployments, services, configmaps, PVC, ingress, gateway) +
ClusterRoleBinding на группу **`developers`** (без префикса, см. гряду выше).

Проверка (как admin, impersonate OIDC-identity; реальный username =
`issuer#kate`):
```
kubectl auth can-i get pods --as="https://keycloak.kubernetes.lab/realms/kubernetes.lab#kate" \
  --as-group=developers    # → yes
kubectl auth can-i get secrets --as="https://keycloak.kubernetes.lab/realms/kubernetes.lab#kate" \
  --as-group=developers    # → no
kubectl auth can-i create deployments --as="https://keycloak.kubernetes.lab/realms/kubernetes.lab#kate" \
  --as-group=developers    # → no
```
Живая проверка (2026-10-07, через kubeoidc.conf + кэшированный токен kate):
```
kubectl auth whoami
# Username  https://keycloak.kubernetes.lab/realms/kubernetes.lab#kate
# Groups    [developers system:authenticated]
kubectl get nodes       # → 2 (w0, w1)
kubectl get pods -A     # → 80
kubectl get secrets     # → Forbidden (как задумано)
```

## Откат OIDC

```
ssh cp0 'sudo cp /etc/systemd/system/kube-apiserver.service.bak-oidc \
  /etc/systemd/system/kube-apiserver.service && sudo systemctl daemon-reload \
  && sudo systemctl restart kube-apiserver'
```
