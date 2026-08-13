# Установка обязательных программных компонент

Источник: https://help.1forma.ru/maintenance_guide/installation/application/required_software_components/



Необходимо установить:

-  Windows6.1-KB2506143-x64.msu


-  Windows6.1-KB2592525-x64.msu


-  Windows6.1-KB2670838-x64.msu


-  dotNetFx45_Full_setup.exe


-  FileFormatConverters.exe

  Примечание: в случае ошибки при установке Windows6.1-KB2592525-x64.msu его требуется установить принудительно:
 1. Скачайте KB2592525 в любую папку на сервере
 2. Создайте подпапку "files"
 3. Распакуйте MSU в Powershell: expand Windows6.1-KB2592525-x64.msu –F:* .\files
 4. В powershell перейдите в папку "files"
 5. Установите с помощью Pkgmgr: pkgmgr /ip /m:Windows6.1-KB2592525-x64.cab

Для установки необходимых компонентов Windows выполните следующую команду в PowerShell:
 Import-Module ServerManager
 Add-WindowsFeature Web-Server,Web-WebServer,Web-Common-Http,Web-Static-Content,Web-App-Dev,Web-Asp-Net,Web-Net-Ext,Web-ISAPI-Ext,Web-ISAPI-Filter,Web-Includes,Web-Security,Web-Windows-Auth,Web-Filtering,Web-Stat-Compression,Web-Dyn-Compression,Web-Mgmt-Console,Ink-Handwriting,IH-Ink-Support
