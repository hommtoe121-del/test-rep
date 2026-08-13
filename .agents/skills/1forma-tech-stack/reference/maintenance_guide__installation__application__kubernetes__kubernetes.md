# Запуск приложения "Первая форма" в Kubernetes

Источник: https://help.1forma.ru/maintenance_guide/installation/application/kubernetes/kubernetes/

ℹ️ Kubernetes (K8s) — это открытое программное обеспечение для автоматизации развёртывания, масштабирования, управления и контроля контейнеризованных приложений.

1. Получите учетные данные приватного реестра у сотрудников "Первой Формы".
 2. Создайте Namespace и imagePullSecret:
 `kubectl create ns 1forma

kubectl create secret docker-registry regcred \

  --docker-server=docker.1forma.ru \

  --docker-username='<USER>' \

  --docker-password='<PASS>' \

  --docker-email='<EMAIL>' -n 1forma
` 3. Подготовьте values.override.yaml (см. пример выше).
 4. (Опционально) Создайте TLS-секрет 1forma-tls.
 5. Установите:
 `helm install 1forma ./1forma -n 1forma -f values.override.yaml
` 6. Проверьте:
 `kubectl get pods,svc,ingress -n 1forma

helm status 1forma -n 1forma
`

Быстрое развертывание приложений
 При наличии нескольких серверов и необходимости обслуживать большее число пользователей, Kubernetes позволяет упростить процесс масштабирования. Обновление приложений происходит без необходимости отключения серверов, пропадает необходимость увеличивать мощность каждого сервера или добавлять новые серверы и переносить на них приложения.
 Управление версиями
 С помощью Kubernetes можно легко управлять версиями приложений. Даже после небольших обновлений новую версию можно быстро развернуть. В случае обнаружения проблемы также можно быстро вернуться на предыдущую версию без вреда для пользователей.
 Обеспечение отказоустойчивости
 Kubernetes упрощает обеспечение отказоустойчивости и поддерживается практически всеми крупными облачными провайдерами.

Архитектура Kubernetes включает в себя следующие компоненты:

-  Nodes (ноды, узлы) —  узлы, которые в зависимости от кластера, могут быть виртуальной или физической машиной. На узлах запускаются контейнеры приложений. Каждый узел содержит необходимые компоненты для запуска сервисов контейнеризации. Кластер имеет как минимум один рабочий узел.


-  Pods (поды)— базовые модули управления приложениями, которые могут содержать один или несколько контейнеров.


-  Volume (том) — ресурс, который позволяет нескольким контейнерам одновременно использовать общее хранилище.

  Мастер-нода (Master Node) — главный узел управления кластером Kubernetes. Он отвечает за координацию и контроль работы остальных узлов в кластере. Основные компоненты мастера включают:

-  Etcd — распределенное хранилище данных, используемое для хранения конфигурации кластера и состояния приложений.


-  API Server — компонент, обеспечивающий API для взаимодействия с Kubernetes. Он принимает команды управления и обновляет состояние кластера.


-  Scheduler — компонент, отвечающий за размещение подов на доступных рабочих узлах в соответствии с требованиями ресурсов и ограничениями.


-  Controller Manager — компонент, который мониторит состояние кластера и управляет работой других контроллеров, например, контроллеров репликации и контроллеров развертывания.

  Рабочие узлы (Worker Nodes) — вычислительные узлы, на которых размещаются контейнеры и выполняются приложения. Каждый рабочий узел имеет следующие компоненты:

-  Kubelet — агент, который запускает и управляет контейнерами на рабочих узлах, основываясь на инструкциях API сервера.


-  Kube-proxy — компонент, который ведет сетевой прокси-трафик и осуществляет балансировку нагрузки между сервисами.


-  Container Runtime — ответственный за запуск и управление контейнерами. Наиболее распространенным контейнерным runtime является Docker, но Kubernetes также поддерживает другие runtime, такие как containerd и CRI-O.

  Дополнительные компоненты:

-  Network Plugins (Плагины сети) — позволяют сетевым подсистемам в различных узлах коммуницировать друг с другом и соединять контейнеры с внешним миром.


-  Storage (Хранилище) — предоставляет множество опций для хранения данных, используемых Kubernetes, включая локальное хранилище, сетевые тома и распределенные файловые системы.


-  Dashboard — веб-интерфейс для визуального управления кластером и мониторинга его состояния.


-  Ingress Controller — позволяет управлять входящим сетевым трафиком в кластер и маршрутизировать его на соответствующие сервисы.

  `flowchart TD
    ING["Ingress Controller"] -- "входящий трафик" --> KP["Kube-proxy: маршрутизация и балансировка"]
    API["API Server"] -- "конфигурация и состояние" --> ETCD["etcd"]
    API --> SCH["Scheduler: размещение подов"]
    API --> CM["Controller Manager"]
    API -- "инструкции" --> KUBELET["Kubelet на рабочем узле"]
    KUBELET --> CR["Container Runtime"]
    CR --> PODS["Pods (контейнеры приложений)"]
    KP --> PODS`

Предполагается, что вы уверенно работаете с kubernetes/Helm и имеете доступ к кластеру.
 ℹ️ Учетные данные для приватного Docker Registry получите у сотрудников "Первой Формы"
 Ознакомьтесь с описанием сервисов и описанием переменных окружения:
 Описание Docker-образа
 Описание сервиса 1f-dbDeploy
 Описание сервиса 1f-core
 Описание сервиса 1f-spa

`1forma/

├─ Chart.yaml                 # метаданные: имя, версия, appVersion

├─ values.yaml                # значения по умолчанию

├─ templates/                 # манифесты Kubernetes (Go templates)

│  ├─ \_helpers.tpl            # вспомогательные шаблоны/имена

│  ├─ namespace.yaml          # опциональное создание Namespace

│  ├─ configmaps/office-proxy-headers.yaml

│  ├─ secrets/…               # registry, tls, component secrets

│  ├─ services/…              # Service для каждого компонента

│  ├─ deployments/…           # Deployment'ы (core, spa, redis, …)

│  ├─ statefulsets/…          # StatefulSet'ы (core, core-job, …)

│  ├─ hpa/…                   # HorizontalPodAutoscaler для core/spa

│  ├─ ingress/…               # основной Ingress + office-ingress

│  ├─ networkpolicy/…         # пример NetworkPolicy

│  └─ NOTES.txt
` Рекомендации по структуре Helm-чартов и работе с templates/ и values — в официальной документации Helm.

- Компоненты приложения (включаемые флагами в values.yaml):
  ocore (API/бэкенд) — по умолчанию enabled: true, может работать как StatefulSet (useStatefulSet: true) или Deployment.
 ocoreJob — экземпляр core в режиме Job/worker (CORE_IS_JOB_SERVER: true).
 ospa — фронтенд (статический SPA).
 oredis — внутренний Redis (можно отключить и использовать внешний).
 odocumentserver — OnlyOffice DocumentServer (по умолчанию выключен).
 odbdeploy — job для выката схемы/миграций БД (может запускаться как hook).

-  Ingress (NGINX) с настраиваемыми таймаутами (proxy-read-timeout, proxy-send-timeout) и TLS-секретом.


-  Autoscaling (HPA) для core и spa — autoscaling.enabled.


-  Гибкая работа с секретами: можно создавать секреты чартом (`createSecret: true`) или использовать уже существующие (createSecret: false и подставить secretName).


-  Поддержка приватного реестра образов через imagePullSecrets.

Основные поля values.yaml (приведены только наиболее важные).
 Namespace
 `namespace:

 name: "1forma"

 # Если ns уже создан вручную — установите create=false (см. комментарии в values.yaml)
` Registry / imagePullSecrets
 `registry:

 server: docker.1forma.ru

 secretName: regcred

 createSecret: false     # если секрет создадите заранее

 username: ""            # НЕ храните пароли в гите; используйте внешние секреты

 password: ""

 email: ""

imagePullSecrets:

 - name: regcred
` Ingress
 `ingress:

 enabled: true

 className: nginx

 host: "example.host"

 tls:

   enabled: true

   create: true

   secretName: "1forma-tls"

   crt: ""    # если создаёт чарт — поместите PEM в values или используйте внеш. secret

   key: ""

 timeouts:

   proxyRead: "3600"

   proxySend: "3600"
` Core
 `core:

 enabled: true

 useStatefulSet: true

 image: { repository: docker.1forma.ru/1f/1f-core, tag: "tag", pullPolicy: IfNotPresent }

 replicaCount: 2

 port: 8000

 resources:

   requests: { cpu: "4000m", memory: "4Gi" }

   limits:   { cpu: "8000m", memory: "8Gi" }

 autoscaling:

   enabled: false

   minReplicas: 2

   maxReplicas: 10

   targetCPUUtilizationPercentage: 80

 env:

   CORE\_DB\_TYPE:    null

   CORE\_DB\_SERVER:  null

   CORE\_DB\_NAME:    null

 secretName: core-secrets

 createSecret: true

 secretData:

   CORE\_DB\_PASSWORD:   null

   REBUS\_DB\_PASSWORD:  null
` CoreJob (worker)
 Параметры аналогичны core. Важно: CORE_IS_JOB_SERVER: true, отдельный secretName (corejob-secrets).
 SPA (frontend)
 `spa:

 enabled: true

 image: { repository: docker.1forma.ru/1f/1f-spa, tag: "..." }

 port: 80

 autoscaling:

   enabled: true

   minReplicas: 2

   maxReplicas: 5

   targetCPUUtilizationPercentage: 70
` Redis
 `redis:

 enabled: true

 image: { repository: docker.1forma.ru/1f/1f-redis, tag: "latest" } # Рекомендуется указывать конкретную версию вместо latest, например: "7.2"

 port: 6379

 secretName: redis-secret

 createSecret: true

 secretData:

   REDIS\_PASSWORD: null
` DocumentServer (OnlyOffice)
 `documentserver:

 enabled: false

 image: { repository: onlyoffice/documentserver, tag: "9.0.0" }

 port: 80

 env:

   JWT\_ENABLED: true

   JWT\_HEADER: "Authorization"

   JWT\_IN\_BODY: true

   JWT\_SECRET: null

   WOPI\_ENABLED: true

   ALLOW\_PRIVATE\_IP\_ADDRESS: true

   ALLOW\_META\_IP\_ADDRESS: true

   USE\_UNAUTHORIZED\_STORAGE: true

   TZ: "Europe/Moscow"

 secretName: docs-secrets

 createSecret: true

 secretData:

   JWT\_SECRET: null
` DB Deploy (миграции)
 `dbdeploy:

 enabled: false

 useHook: true                 # запускать миграции как Helm hook

 ttlSecondsAfterFinished: 86400

 keepLogs: true

 env:

   DBDEPLOY\_DB\_TYPE:   null

   DBDEPLOY\_DB\_SERVER: null

   DBDEPLOY\_DB\_NAME:   null

 secretName: dbdeploy-secrets

 createSecret: true

 secretData:

   DBDEPLOY\_DB\_PASSWORD: null
`

Включается шаблоном-примером (networkPolicy.enabled: false по умолчанию). Настройте под вашу сетевую модель CNI.



1. Запросите **логин/пароль** и адрес реестра у сотрудников "Первой Формы".
 2. Создайте секрет одного из способов:

- Командой:
  `kubectl create secret docker-registry regcred \

  --docker-server=docker.1forma.ru \

  --docker-username='<USER>' \

  --docker-password='<PASS>' \

  --docker-email='<EMAIL>' \

  -n 1forma
`
- Из локального ~/.docker/config.json:
  `kubectl create secret generic regcred \

  --type=kubernetes.io/dockerconfigjson \

  --from-file=.dockerconfigjson=$HOME/.docker/config.json \

  -n 1forma
` Затем укажите imagePullSecrets: [{name: regcred}] в values.yaml.

-  Для каждого сервиса предусмотрен secretName и блок secretData.


-  Если createSecret: true, чарт создаст Secret (данные берутся из values.yaml — безопасно только при использовании шифрования репозитория/CI-переменных).


-  Если секреты уже существуют в кластере, установите createSecret: false и укажите готовый secretName.

`core:

 createSecret: false

 secretName: "core-secrets-ext"

 # secret core-secrets-ext должен существовать в ns

redis:

 createSecret: true

 secretData:

   REDIS\_PASSWORD: "s3cr3t"    # лучше подставлять через helm --set-file или CI vars
`

-  Подставляйте чувствительные данные через --set-string / --set-file в CI, а не сохраняйте в git.


-  Рассмотрите использование External Secrets или Sealed Secrets в вашей организации.

-  Kubernetes v1.24+ (рекомендовано актуальное LTS в вашей среде).


-  Helm v3.x.


-  Доступный Ingress Controller (NGINX), класс по умолчанию — nginx.


-  Установлен metrics-server (для HPA).


-  Ресурсы узлов (см. resources.requests/limits в values.yaml).

1. Создайте Namespace (если не поручаете это чарту):
 `kubectl create ns 1forma
` 2. Создайте imagePullSecrets (regcred) для приватного реестра.
 3. (Опционально) Создайте TLS-секрет заранее:
 `kubectl create secret tls 1forma-tls \

  --cert=server.crt --key=server.key -n 1forma
` или позвольте чарту создать его из полей ingress.tls.crt/key.

`# посмотреть, что будет применено

helm install 1forma ./1forma -n 1forma --dry-run --debug

# применить

helm install 1forma ./1forma -n 1forma
`

`# предварительный просмотр диффа

helm upgrade 1forma ./1forma -n 1forma --dry-run

# обновление

helm upgrade 1forma ./1forma -n 1forma
`

`helm history 1forma -n 1forma

helm rollback 1forma <REVISION> -n 1forma
`

Быстрый чек-лист

-  Подпись образов/доступ к реестру: ошибки ImagePullBackOff → проверьте imagePullSecrets и regcred.


-  Ingress 4xx/5xx/таймауты: проверьте аннотации/таймауты NGINX, соответствие host, TLS-секрет.


-  HPA не масштабирует: проверьте наличие metrics-server и корректность метрик/ресурсных запросов.

`# Рендеринг шаблонов локально

helm template 1forma ./1forma -n 1forma -f values.override.yaml | less

# Состояние релиза

helm status 1forma -n 1forma

helm get values 1forma -n 1forma

helm get manifest 1forma -n 1forma

# Диагностика подов

kubectl get pods -n 1forma -o wide

kubectl describe pod <pod> -n 1forma

kubectl logs -n 1forma <pod> --all-containers=true --tail=200

# Сервисы и сетевуха

kubectl get svc,ep -n 1forma

kubectl get ingress -n 1forma

kubectl describe ingress 1forma -n 1forma

# Проверка HPA

kubectl get hpa -n 1forma

kubectl describe hpa <name> -n 1forma

# События

kubectl get events -n 1forma --sort-by=.lastTimestamp
`

Если createSecret: true и значения пустые (null), Kubernetes создаст пустые ключи — контейнер может не запустится. Проверьте обязательные переменные (_PASSWORD, JWT_SECRET, параметры БД).
