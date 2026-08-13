# Запуск ВКС в Kubernetes

Источник: https://help.1forma.ru/maintenance_guide/installation/application/kubernetes/kubernetes_vks/



Для запуска ВКС в K8s используется менеджер пакетов Helm. Перед запуском Helm необходимо изменить значения переменных (Секретов, Доменных имен и IP адресов) в файле values.yaml.
 values.yaml
 Скрипт gen-passwords.sh
 1f-meet-web.yaml
 1f-meet-prosody.yaml
 1f-meet-jvb.yaml
 1f-meet-jicofo.yaml
 1f-meet-jibri.yaml
 1f-meet-ingress.yml
 `apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: 1forma-ingress
spec:
 ingressClassName: nginx
 rules:
   - host: {{ .Values.domainName }}
     http:
       paths:
         - pathType: Prefix
           backend:
             service:
               name: web
               port:
                 number: 80
           path: /
 tls:
   - hosts:
     - {{ .Values.domainName }}
     secretName: 1forma-tls

---
apiVersion: v1
kind: Secret
metadata:
 name: 1forma-tls
data:
 tls.crt: {{ .Values.tlsCertBase64 }}
 tls.key: {{ .Values.tlsKeyBase64 }}
type: kubernetes.io/tls
` Если по какой-то причине у вас нет возможности использовать Helm в вашем K8s, вы можете отредактировать yaml-файлы и работать с ними как с классическими манифестами. Для этого замените значения {{ .Values. }} на значения переменных в файле values.yaml.
 Обратите внимание: в параметре advertiseIpList  необходимо через запятую передать список IP адресов для формирования ICE Candidate pair в WebRTC. Как правило, это Real IP  с которого проброшен 10000/UDP порт или локальный адрес Ноды.
 Необходимо пробросить порт 10000 к поду JVB. В настройках нашей конфигурации мы используем сервис JVB типа Load Balancer с возможностью балансировки UDP. Для вашей конфигурации можно использовать NodePort. Описание сервиса JVB находится в файле 1f-meet-jvb.yaml (см. шаблоны выше). Доступ по протоколам HTTP/HTTPS осуществляется через Ingress, который в нашей конфигурации реализован с использованием Ingress Nginx.

Запустите команду для установки и настройки ВКС:
 `helm install 1f-meet 1f-meet-k8s/
`

В режиме администрирования Первой Формы перейдите в раздел "Сервисы".
 Если в списке существующих сервисов есть сервис с типом "ВКС", перейдите к его настройкам. Если такого сервиса нет, создайте его по кнопке "Добавить сервис".
 Заполните настройки сервиса:

-  Сервис: ВКС.


-  Описание: Имя сервиса ВКС.


-  Домен: Доменное имя сервиса ВКС.


-  Секретный ключ: Значение jwtAppSecret переменной из предыдущего шага.


-  Operation Address: /app/v1.2/api/publications/action/conference-actions.


-  Группа пользователей с комнатами: ID группы, для которых будет доступен сервис. Обычно это группа "Все пользователи".


-  Префикс комнаты: Оставьте пустым.


-  Смарт-фильтр проверки модератора: создайте новый фильтр. Задайте произвольное имя (например, ВКС модератор) Выберите тип TSQL.

  Смарт-выражение:
 `select dbo.fnIsConferenceModerator(@eventParam1, @eventParam0)
` Сохраните. Выберите созданный фильтр в настраиваемом сервисе.

- Расширенные настройки (json):
  {"externalModuleKey":"formaApiSecret",
 "roomCreationKey":"creationKey",
 "recordHost":"https://records.domain.ru",
 "recordsApiKey":"formaRecordSecret"}
 где:

-  roomCreationKey - Ключ, необходимый для работы плагина Outlook. Он служит для аутентификации плагина при генерации временной комнаты. Cгенерируйте случайное значение.


-  externalModuleKey - Ключ для доступа к api модуля ВКС. Значение formaApiSecret.


-  recordHost - Адрес хоста ссылок на записи ВКС.


-  recordsApiKey - Ключ аутентификации для приема событий записи ВКС. Значение formaRecordSecret.

  Сохраните настройки Сервиса.
 Включите сервис ВКС в настройках приложения. Для этого перейдите в раздел "Общие настройки приложения".
 Найдите пункт "ВКС" и во всплывающем меню выберите созданный сервис.
 Сохраните изменения.
