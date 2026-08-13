# Телефония: Администрирование и конфигурация

Источник: https://help.1forma.ru/domains/telephony/admin/

Документ описывает администрирование и конфигурацию телефонии в 1Форме: типы PBX (FreeSwitch, Oktell, Простые звонки), разделение SipProviders и TelephonySettings, параметры Click-to-Call и WebRTC, SIP-аккаунты пользователей, события SmartPack и типовую нагрузку для сайзинга. Для администраторов площадки и инженеров внедрения.

Платформа поддерживает три типа интеграции PBX с разными моделями инициации звонка.
 Тип ServerType Примечание FreeSwitch 1 Основной. Callback-модель. Типовая конфигурация в production Oktell 2 Требует профиль в доп. учётных записях Простые звонки 3 Минимальная интеграция SipProviders — конфигурация SIP-сервера для прямых звонков (WebRTC из браузера, нативный SIP с мобильного). Звонок идёт напрямую между клиентом и SIP-сервером.
 TelephonySettings — конфигурация PBX для Click-to-Call. Платформа отправляет HTTP-запрос к PBX, PBX инициирует callback. Клиент в этом не участвует.

Основные параметры:
 Параметр Описание Пример ServerType Тип PBX (1=FreeSwitch, 2=Oktell, 3=Простые звонки) 1 (FreeSwitch) ServerUrl URL для callback-запросов `https://<sip-сервер>/fs1f/click2call/` ServerLogin / ServerPassword Авторизация (пароль хранится в зашифрованном виде) — InternalPhoneMatchRegExp Regex внутреннего номера `^\d{5}$` FirstDigitInPhone Замена первой цифры при городском вызове — DefaultPhonePrefix Префикс по умолчанию — DefaultIntlPhonePrefix Международный префикс — AllowClientCallControl Разрешить управление звонком с клиента — CallRedirectSmartExpressionId SmartExpression для переадресации null

Типовая конфигурация — несколько провайдеров, отличающихся доменом SIP-сервера. Стандартные порты: SIP `5061`, WSS `7443`.
 Поле Назначение `WSAddress` WebSocket для WebRTC `Realm` SIP realm `ICEServers` JSON конфигурация ICE/TURN серверов `SIPServer` / `SIPPort` Основной SIP-сервер `SIPServerReserve` / `SIPPortReserve` Резервный SIP-сервер Привязка пользователя к SIP-провайдеру (UserSipSettings). Два типа:

- TypeId=1 (Mobile) — нативный SIP-клиент на мобильном

- TypeId=2 (Web) — WebRTC из браузера
  Бизнес-правило: одна `IsDefault = true` настройка на каждый TypeId.

Привязка SmartScript-пакетов к событиям телефонии. Механизм реализован, но в типовых инсталляциях используется редко.
 Событие Параметры IncomingCallStarted user, callId, phone, incoming, redirected, external, startTime IncomingTalkStarted + talkStartTime IncomingTalkComplete + talkEndTime, recordingUrl OutgoingCallStarted аналогично incoming OutgoingTalkStarted + talkStartTime OutgoingTalkComplete + talkEndTime, recordingUrl Метрика Порядок значений Поток звонков тысячи в месяц SIP-провайдеры от 1 до 5 TelephonySettings 1 рабочая конфигурация TelephonyActionsPacks редко используется, обычно 0 Документ Содержание `../users-and-groups/admin.md` Настройка SIP-номеров пользователя (скриншоты UI) `../auth/admin.md` Настройки IP-телефонии (FreeSwitch, Oktell, Простые звонки) `docs/domains/conferences/mobile.md` SIP-телефония на мобильных (iOS/Android endpoints)
