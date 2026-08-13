# Шифрование строки подключения

Источник: https://help.1forma.ru/maintenance_guide/installation/databases/mssql/tech_req_1f_crypt/

Для регистрации приложений ASP.NET в службах IIS используется утилита aspnet_regiis.exe. Для обеспечения дополнительного уровня безопасности используется шифрование строки подключения.

- Чтобы зашифровать строку подключения, нужно в командной строке Windows (PowerShell) выполнить команду:
  aspnet_regiis -pef "connectionStrings" <адрес_файлов_приложения>

- Чтобы расшифровать строку подключения, нужно в командной строке Windows выполнить команду:
  aspnet_regiis -pdf "connectionStrings" <адрес_файлов_приложения>
 где <адрес_файлов_приложения>, например, это "C:\inetpub\wwwroot\1Forma".
