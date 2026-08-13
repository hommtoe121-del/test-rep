# Настройка IIS для использования TCWebService

Источник: https://help.1forma.ru/maintenance_guide/installation/application/iis-deployment/tcwebservice/

ℹ️ "Первая Форма" может взаимодействовать с другими системами с помощью веб-сервисов. Работа с веб-сервисами описана в Методическом руководстве администратора.
 ℹ️ Приведённый ниже способ установки через pkgmgr.exe относится к устаревшим версиям Windows. На Windows Server 2016 и новее используйте Server Manager или PowerShell: `Install-WindowsFeature Web-Server -IncludeManagementTools`.

1. Установите .NET Framework, если он ещё не установлен. Обратите внимание: если IIS устанавливается после .NET Framework, пул приложений ASP.NET не создаётся автоматически — его потребуется создать вручную для работы веб-сервисов.
 2. По умолчанию в Windows Server не устанавливаются службы IIS. Установите их с помощью мастера добавления ролей диспетчера сервера или с помощью командной строки.
 В процессе установки Windows Server можно выбрать для установки Server Core, в результате будет произведена минимальная серверная установка Windows Server. Например, при такой установке обычный интерфейс Windows не устанавливается, поэтому настройка сервера должна производиться из командной строки.
 Чтобы выполнить данную процедуру, необходимо быть членом административной роли IIS (администратор веб-сервера).
 3. Выполните настройку IIS с помощью пользовательского интерфейса или сценария.
 Использование пользовательского интерфейса
 1. Нажмите кнопку Пуск, введите administration и выберите Диспетчер сервера.
 2. В разделе Сводка ролей выберите Добавить роли.
 3. Воспользуйтесь мастером добавления ролей, чтобы добавить роль веб-сервера.
 ℹ️ При использовании мастера добавления ролей для установки служб IIS установка происходит по умолчанию, то есть с минимальным набором компонентов. Если требуется установить дополнительные роли служб IIS, такие как Разработка приложения или Проверка работоспособности и диагностика, не забудьте включить парметры, связанные с этими компонентами, на странице Выбор служб ролей данного мастера.
 Использование сценария
 В командной строке выполните следующую команду:
 CMD /C START /w PKGMGR.EXE /l:log.etw /iu:IIS-WebServerRole;IIS-WebServer;IIS-CommonHttpFeatures;IIS-StaticContent;IIS-DefaultDocument;
 IIS-DirectoryBrowsing;IIS-HttpErrors;IIS-HttpRedirect;IIS-ApplicationDevelopment;IIS-ASP;IIS-CGI;IIS-ISAPIExtensions;IIS-ISAPIFilter;
 IIS-ServerSideIncludes;IIS-HealthAndDiagnostics;IIS-HttpLogging;IIS-LoggingLibraries;IIS-RequestMonitor;IIS-HttpTracing;IIS-CustomLogging;
 IIS-ODBCLogging;IIS-Security;IIS-BasicAuthentication;IIS-WindowsAuthentication;IIS-DigestAuthentication;IIS-ClientCertificateMappingAuthentication;
 IIS-IISCertificateMappingAuthentication;IIS-URLAuthorization;IIS-RequestFiltering;IIS-IPSecurity;IIS-Performance;IIS-HttpCompressionStatic;
 IIS-HttpCompressionDynamic;IIS-WebServerManagementTools;IIS-ManagementScriptingTools;IIS-IIS6ManagementCompatibility;IIS-Metabase;IIS-WMICompatibility;
 IIS-LegacyScripts;WAS-WindowsActivationService;WAS-ProcessModel;IIS-FTPServer;IIS-FTPSvc;IIS-FTPExtensibility;IIS-WebDAV;IIS-ASPNET;
 IIS-NetFxExtensibility;WAS-NetFxEnvironment;WAS-ConfigurationAPI;IIS-ManagementService;MicrosoftWindowsPowerShell
 ℹ️ При использовании этого сценария выполняется полная установка IIS, что приводит к установке всех доступных пакетов. Если какие-либо пакеты средств не нужны, следует отредактировать сценарий таким образом, чтобы устанавливались только необходимые пакеты.

ℹ️ tcwebservice.asmx доступен только при установке TaskCenter. Является устаревшим и не рекомендуется к использованию. Для обращения к методам рекомендуется использовать API (Swagger).
 Это адрес необходимо указать в режиме администрирования в разделе misc -> Системные настройки -> Общие настройки приложения в поле "Настройки веб-сервиса".
 Поле должно выглядеть следующим образом:
 http://%адрес1Формы%/tcwebservice.asmx;login;password;domen
 (домен указывать необязательно)
