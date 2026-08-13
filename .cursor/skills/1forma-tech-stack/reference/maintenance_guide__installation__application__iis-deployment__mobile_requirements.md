# Требования для работы мобильных приложений

Источник: https://help.1forma.ru/maintenance_guide/installation/application/iis-deployment/mobile_requirements/

1. Приложение "Первая Форма" должно быть опубликовано по протоколу HTTPS с использованием SSL-сертификата (тип "для домена и www"). SSL-сертификат должен быть действующим, доверенным и соответствовать URL сайта приложения "Первая Форма".
 2. Сервер "Первой Формы" должен поддерживать Forms-авторизацию.
 3. Для работы push-уведомлений нужно указать публичный стек сетей.

-  для Android-устройств: fcm.googleapis.com: TCP/443, 5228-5230, android.googleapis.com: TCP/443, 5228-5230, 5235, 5236.


-  для iOS-устройств: api.push.apple.com: 443, api.development.push.apple.com: 443.

  4. На сервере "Первой Формы" в файле web.config должны быть указаны следующие настройки (по умолчанию настроено).
 `<location path="iOSClientServices/Auth.ashx">
<system.web>
  <authorization>
    <allow users="?" />
  </authorization>
</system.web>
</location>

<location path="iOSClientServices/Report4NumberCall.ashx">
  <system.web>
    <authorization>
      <allow users="?" />
    </authorization>
  </system.web>
</location>

<location path="iOSClientServices/Apps">
  <system.web>
      <authorization>
      <allow users="?" />
      </authorization>
  </system.web>
</location>
` 5. В настройках PUSH должны быть добавлены сертификаты для приложения OneFChat.
 ℹ️ Данный пункт является устаревшим. Windows Server 2016+ и в версиях приложения от 2020 года и выше сертификат вшит в приложении и используется протокол h2 для push уведомления.
 Сертификаты также требуется установить на веб-сервер в папку Личное для пользователя, от имени которого запущен пул приложения "Первая Форма". Посмотреть, от чьего имени запущен пул, можно в настройках Application Pools.
 Если пул запущен от имени LocalSystem, то в Мастере импорта сертификатов выберите расположение Локальный Компьютер. Если пул запущен от имени конкретного пользователя, то нужно авторизоваться в Windows под именем этого пользователя и выбрать расположение Текущий пользователь.
 ℹ️ Чтобы получить сертификаты, обратитесь в техническую поддержку "Первой Формы", оставив заявку в системе Help Desk
