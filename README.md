# eMudhra CA mitigation for Yandex Managed Kubernetes

DaemonSet отключает доверие к двум корневым сертификатам, через которые может
проверяться скомпрометированная цепочка eMudhra, на **worker-нодах** Managed
Kubernetes. Работает без `apt`, обновления ОС и скачивания пакетов на хост.

Основание: [рекомендация Ubuntu](https://ubuntu.com/blog/emudhra-ca-compromise-fixes-available),
[USN-8944-1](https://ubuntu.com/security/notices/USN-8944-1).

## Область действия

- Системное хранилище CA Ubuntu/Debian на worker-нодах и использующий его
  `containerd`.
- Все ноды выбранного кластера, включая ноды с taints и вновь добавленные.
- Не обновляет образы контейнеров, уже работающие
  приложения, Java/NSS trust stores и собственные CA bundles приложений.
  Их необходимо обновить отдельно. Это **не полное устранение инцидента во всём кластере**.

## Что меняется

1. В `/etc/ca-certificates.conf` отключаются только
   `mozilla/emSign_ECC_Root_CA_-_G3.crt` и `mozilla/Comodo_AAA_Services_root.crt`.
2. При необходимости выполняется `update-ca-certificates --fresh`.
   Остальные выбранные CA и локальные CA сохраняются; стандартные hooks
   `update-ca-certificates` выполняются на хосте.
3. Проверяется отсутствие сертификатов с SHA-256 отпечатками двух отозванных
   корневых CA в системном bundle и файлах непосредственно в `/etc/ssl/certs`.
4. `containerd` перезапускается для сброса кэша CA.
   Хэш успешно обработанного bundle сохраняется в
   `/var/lib/yc-emudhra-mitigation/restarted-bundle.sha256`.
   Повторное применение к неизменному bundle не перезапускает сервис.
   `kubelet` не перезапускается: в стандартной конфигурации mk8s он использует
   отдельный CA из kubeconfig для соединения с Kubernetes API.
5. Проверка и восстановление выполняются каждые 5 минут, readiness — каждые
   30 секунд. Ошибка применения завершает контейнер с ненулевым кодом;
   Kubernetes повторит запуск. Pod не готов, пока проверка не пройдёт.

Если отозванный сертификат установлен повторно как пользовательский CA,
проверка завершится ошибкой с его путём. Такой файл нужно удалить из источника
локальных CA отдельно; пример не удаляет произвольные пользовательские файлы.

## Перед применением

Нужны права на создание privileged DaemonSet и ConfigMap в `kube-system`.
На хосте нужны `python3`, `update-ca-certificates`, `systemctl` и systemd-сервис
`containerd`. Образ `cr.yandex/mirror/ubuntu:22.04` должен быть доступен нодам;
для закрытого контура зеркалируйте его в доступный registry и замените `image`.

Перезапуск `containerd` временно прерывает операции управления контейнерами и
может вызвать кратковременный `NotReady`. Действующие контейнеры обычно
продолжают работать через containerd-shim, но это зависит от конфигурации узла.
Соединения с легитимными сервисами, использующими отозванные цепочки,
перестанут проходить TLS-проверку.

Сначала проверьте на тестовом кластере / одной ноде, затем выберите окно работ.
`maxUnavailable: 1` ограничивает **обновление** DaemonSet, но не его первое
развёртывание и не одновременные изменения CA на нескольких нодах.
Для поэтапного первого запуска добавьте в `spec.template.spec.nodeSelector`
манифеста `emudhra-mitigation: enabled` и последовательно помечайте ноды:

```bash
kubectl label node <node-name> emudhra-mitigation=enabled
```

После проверки всех нод уберите дополнительный selector из манифеста и
повторите `apply`, чтобы покрыть также будущие ноды.

## Установка и проверка

Скачайте этот каталог и проверьте выбранный Kubernetes context:

```bash
kubectl config current-context
kubectl apply -f emudhra-mitigation-daemonset.yaml
kubectl -n kube-system rollout status daemonset/emudhra-ca-mitigation --timeout=10m
kubectl -n kube-system get ds emudhra-ca-mitigation
kubectl -n kube-system get pods -l app=emudhra-ca-mitigation -o wide
kubectl -n kube-system logs -l app=emudhra-ca-mitigation --prefix --tail=50
```

Убедитесь, что `DESIRED`, `CURRENT`, `READY`, `AVAILABLE` совпадают с числом
ожидаемых Linux-нод. В логах каждой ноды должно появиться:

```text
PASS: affected roots absent from system bundle and certificate directory
```

Принудительная read-only проверка конкретного пода:

```bash
kubectl -n kube-system exec <pod-name> -- bash -ec \
  'nsenter --target 1 --mount --uts --ipc --net --pid --root --wd -- python3 - check < /opt/mitigation/mitigate.py'
```

## Обновление скрипта

Код митигации редактируется непосредственно в `data.mitigate.py` ConfigMap
в файле `emudhra-mitigation-daemonset.yaml`.
Изменение только ConfigMap не вызывает автоматического перезапуска подов:
новый код начнёт применяться после обновления смонтированного ConfigMap
и очередного пятиминутного цикла.

Чтобы применить изменённый код без ожидания очередного цикла, обновите
манифест и перезапустите DaemonSet:

```bash
kubectl apply -f emudhra-mitigation-daemonset.yaml
kubectl -n kube-system rollout restart daemonset/emudhra-ca-mitigation
kubectl -n kube-system rollout status daemonset/emudhra-ca-mitigation --timeout=10m
```

## Удаление

```bash
kubectl delete -f emudhra-mitigation-daemonset.yaml
```

Удаление DaemonSet останавливает мониторинг, но **не возвращает доверие** к
отозванным CA. Настройки на нодах сохраняются после перезагрузки.
При замене ноды нужна установка исправленного базового образа либо работающий
DaemonSet. Не восстанавливайте старую конфигурацию целиком: это может затереть
последующие изменения администратора. Исходная копия для диагностики хранится
в `/var/lib/yc-emudhra-mitigation/ca-certificates.conf.before`.
