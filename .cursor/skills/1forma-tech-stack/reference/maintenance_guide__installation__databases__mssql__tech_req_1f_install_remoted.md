# Настройка дистанционного обновления версии приложения

Источник: https://help.1forma.ru/maintenance_guide/installation/databases/mssql/tech_req_1f_install_remoted/

Для дистанционного обновления версии приложения выполните следующие действия:
 1. Создайте административный общий ресурс (Administrative Shares) на сервере C:\TCTemp$\;
 2. Перенесите туда файлы nakat_server.bat и unzip.exe (это утилита разархивирует, она есть в сборке в папке _nakat);
 3. Разрешите выполнение скриптов. Для этого на сервере в Windows PowerShell выполните следующую команду: Set-ExecutionPolicy Bypass;
 4. В папку _nakat сборки перенесите файл nakat_client.bat (в папке должны быть файлы unzip.exe, psexec.exe);
 5. Выполните curr.sql на БД.
 ℹ️ nakat_client.bat нужно запустить на обоих серверах.

1. nakat_client.bat
 Создайте новый текстовый при помощи приложения "Блокнот", в файл скопируйте следующие строки:
 `echo start
del tc.zip
zip -q -r tc.zip ..\\\* -x ..\\Web.config
copy /y tc.zip \\\\\[ИМЯ\_ПЕРВОГО\_СЕРВЕРА\]\\TCTemp$
psexec \\\\\[ИМЯ\_ПЕРВОГО\_СЕРВЕРА\] -w C:\\TCTemp$\\  C:\\TCTemp$\\nakat.bat
copy /y tc.zip \\\\\[ИМЯ\_ВТОРОГО\_СЕРВЕРА2\]\\TCTemp
psexec \\\\\[ИМЯ\_ВТОРОГО\_СЕРВЕРА\] -w C:\\TCTemp$\\  C:\\TCTemp$\\nakat.bat
echo done!
pause
` Сохраните файл под названием "nakat_client.bat"

- nakat_server.bat
  Создайте новый текстовый при помощи приложения "Блокнот", в файл скопируйте следующие строки:
 `nlb.exe stop
%windir%\\system32\\inetsrv\\appcmd stop apppool /apppool.name:1Forma
unzip -o -q  C:\\TCTemp$\\tc.zip -x ..\\Web.config -d C:\\inetpub\\wwwroot\\com
%windir%\\system32\\inetsrv\\appcmd start apppool /apppool.name:1Forma
nlb.exe start
` Сохраните файл под названием "nakat_server.bat"
