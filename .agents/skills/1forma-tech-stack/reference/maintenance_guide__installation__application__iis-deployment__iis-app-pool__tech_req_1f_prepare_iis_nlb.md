# Настройка серверов, работающих под управлением NLB

Источник: https://help.1forma.ru/maintenance_guide/installation/application/iis-deployment/iis-app-pool/tech_req_1f_prepare_iis_nlb/

Для нескольких серверов в одном NLB важно установить одинаковый ключ MachineKey. Для этого выполните следующие действия:
 1. По очереди зайдите на каждый сервер и откройте настройки IIS.
 2. В блоке ASP.Net выберите MachineKey:
 3. Отключите все параметры.
 4. Выставите:

-  Encryption method > SHA1.


-  Decryption method > AES.


-  Сгенерируйте ключи validationKey и decryptionKey на одном из серверов IIS и скопируйте их на все остальные.

  Альтернативный способ: сгенерируйте ключи на сайте http://foxtools.ru/MachineKey и вставьте полученные строки в web.config каждого сайта.
 ℹ️ Сразу после внесения изменений приложение может выдавать friendlyerror. Подождите несколько минут и перезапустите IIS.
 5. Нажмите Apply.
 6. Перезапустите IIS.
