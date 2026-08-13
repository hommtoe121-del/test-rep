# Открытие портов

Источник: https://help.1forma.ru/maintenance_guide/installation/application/iis-deployment/setup/ports_open/

Для работы "Первой Формы" должны быть открыты следующие порты:
 Для чего Сервер Порты Для работы "Первой Формы" с web-cерверов до MS SQL Server 1433 с web-cерверов до PostgreSQL 5432 с web-серверов до MS SQL Server, PostgreSQL (обратное направление не требуется) 80, 443 Для Push уведомлений c web-серверов fcm.googleapis.com (Android Push) TCP/443, 5228-5230 c web-серверов android.googleapis.com (Android Push) TCP/443, 5228-5230, 5235, 5236 c web-серверов api.push.apple.com api.development.push.apple.com 443 Для работы системы Контур.Фокус с web серверов .kontur.ru .skbkontur.ru .kontur-extern.ru .kontur-ca.ru 443 Для работы сервиса DaData с web-серверов и sql-сервера suggestions.dadata.ru 443 Для работы системы SPARK с web-серверов и sql-сервера web-api.spark-marketing.ru:22222 22222 Для работы Redis с web-cерверов до Redis В зависимости от настроек Redis. По умолчанию  6379/tcp
