<div align="center">

# 🤖 Архитектура Telegram-систем

**Bot API против MTProto** — почему это две разные инфраструктуры

[![Node.js](https://img.shields.io/badge/Node.js-22-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)](https://nodejs.org)
[![PHP](https://img.shields.io/badge/PHP-8.3-777BB4?style=for-the-badge&logo=php&logoColor=white)](https://php.net)
![Чатов](https://img.shields.io/badge/архив-141_800_сообщений-0ea5e9?style=flat-square)

</div>

---

## ⚖️ Два разных Telegram

Их постоянно путают, а требования к инфраструктуре у них противоположные.

| | **Bot API** | **MTProto (userbot)** |
|---|---|---|
| Кто вы для Telegram | бот | обычный пользователь |
| Транспорт | HTTPS, webhook или polling | постоянное TCP-соединение |
| Процесс | не нужен, хватает PHP на хостинге | обязателен постоянно живущий процесс |
| Масштабирование | горизонтальное, сколько угодно | **строго один экземпляр на сессию** |
| Видит | только то, что написали боту | всю переписку аккаунта |
| Ломается | таймаутом webhook | обрывом соединения и `AUTH_KEY_DUPLICATED` |

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'ui-sans-serif,-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Helvetica,Arial,sans-serif','fontSize':'15px','textColor':'#1e1b4b','lineColor':'#4338ca','primaryColor':'#e0e7ff','primaryTextColor':'#1e1b4b','primaryBorderColor':'#4338ca','secondaryColor':'#ede9fe','tertiaryColor':'#f5f3ff','mainBkg':'#c7d2fe','nodeBorder':'#4338ca','nodeTextColor':'#1e1b4b','edgeLabelBackground':'#e0e7ff','attributeBackgroundColorOdd':'#eef2ff','attributeBackgroundColorEven':'#e0e7ff','noteBkgColor':'#fef3c7','noteTextColor':'#1a1a1a','noteBorderColor':'#b45309','clusterBkg':'#f5f3ff','clusterBorder':'#4338ca','labelBoxBkgColor':'#e0e7ff','labelBoxBorderColor':'#4338ca','labelTextColor':'#1e1b4b','actorBkg':'#c7d2fe','actorBorder':'#4338ca','actorTextColor':'#1e1b4b','actorLineColor':'#4338ca','signalColor':'#4338ca','signalTextColor':'#1e1b4b','sequenceNumberColor':'#ffffff','activationBkgColor':'#ddd6fe','activationBorderColor':'#4338ca','transitionColor':'#4338ca','transitionLabelColor':'#1e1b4b','stateBkg':'#c7d2fe','stateLabelColor':'#1e1b4b','altBackground':'#f5f3ff','compositeBackground':'#f5f3ff','compositeBorder':'#4338ca','compositeTitleBackground':'#e0e7ff','specialStateColor':'#4338ca','innerEndBackground':'#4338ca'}}}%%
flowchart LR
    subgraph BotAPI["Bot API — запрос-ответ"]
        U1[Пользователь] -->|сообщение| TG1[Telegram]
        TG1 -->|HTTPS webhook| S1[PHP на обычном хостинге]
        S1 -->|200 OK| TG1
    end

    subgraph MTProto["MTProto — постоянное соединение"]
        TG2[Telegram DC] <-->|долгоживущий TCP| S2[Node.js процесс 24/7]
        S2 --> DB[(локальная БД)]
    end

    style BotAPI fill:#6366f1,stroke:#4338ca,color:#ffffff
    style MTProto fill:#f59e0b,stroke:#b45309,color:#1a1a1a
```

### Практическое следствие для выбора хостинга

Бот на Bot API живёт на любом PHP-хостинге: webhook — это обычный HTTPS-запрос.
Userbot на MTProto **нельзя** развернуть на serverless (Vercel, Netlify,
Cloudflare Workers): там нет постоянного процесса и нет долгоживущих соединений.

И нельзя запускать в двух экземплярах с одной строкой сессии — Telegram выдаст
`AUTH_KEY_DUPLICATED` и сессию придётся восстанавливать входом по коду. Это
исключает автоскейлинг и реплики: ровно один процесс.

---

## 🔁 Повторная доставка: почему бот начисляет дважды

Telegram повторит апдейт, если бот не ответил вовремя. Пользователь нажал
«оплатить» один раз — обработчик отработал дважды.

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'ui-sans-serif,-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Helvetica,Arial,sans-serif','fontSize':'15px','textColor':'#1e1b4b','lineColor':'#4338ca','primaryColor':'#e0e7ff','primaryTextColor':'#1e1b4b','primaryBorderColor':'#4338ca','secondaryColor':'#ede9fe','tertiaryColor':'#f5f3ff','mainBkg':'#c7d2fe','nodeBorder':'#4338ca','nodeTextColor':'#1e1b4b','edgeLabelBackground':'#e0e7ff','attributeBackgroundColorOdd':'#eef2ff','attributeBackgroundColorEven':'#e0e7ff','noteBkgColor':'#fef3c7','noteTextColor':'#1a1a1a','noteBorderColor':'#b45309','clusterBkg':'#f5f3ff','clusterBorder':'#4338ca','labelBoxBkgColor':'#e0e7ff','labelBoxBorderColor':'#4338ca','labelTextColor':'#1e1b4b','actorBkg':'#c7d2fe','actorBorder':'#4338ca','actorTextColor':'#1e1b4b','actorLineColor':'#4338ca','signalColor':'#4338ca','signalTextColor':'#1e1b4b','sequenceNumberColor':'#ffffff','activationBkgColor':'#ddd6fe','activationBorderColor':'#4338ca','transitionColor':'#4338ca','transitionLabelColor':'#1e1b4b','stateBkg':'#c7d2fe','stateLabelColor':'#1e1b4b','altBackground':'#f5f3ff','compositeBackground':'#f5f3ff','compositeBorder':'#4338ca','compositeTitleBackground':'#e0e7ff','specialStateColor':'#4338ca','innerEndBackground':'#4338ca'}}}%%
sequenceDiagram
    participant U as Пользователь
    participant TG as Telegram
    participant B as Бот
    participant DB as База

    U->>TG: нажал «оплатить»
    TG->>B: update_id = 512
    B->>DB: INSERT processed_updates(512)
    DB-->>B: ok, первый раз
    B->>DB: создать начисление
    B--xTG: ответ не дошёл (таймаут)

    TG->>B: update_id = 512 повторно
    B->>DB: INSERT processed_updates(512)
    DB-->>B: конфликт уникальности
    B-->>TG: 200 OK, начисления нет
```

Защита — таблица обработанных апдейтов с уникальным индексом по `update_id`.
Именно индекс, а не проверка `SELECT` перед `INSERT`: два одновременных повтора
пройдут проверку оба.

Рядом — ограничение частоты на пользователя: без него один человек, зажавший
кнопку, кладёт обработчик.

---

## 🚧 Где заканчивается конструктор и начинается разработка

Честная граница, которую стоит проговаривать клиенту до начала работы:

**Собирается без программиста:** меню из кнопок, приветствие, FAQ, форма
«имя-телефон-комментарий» с отправкой в чат.

**Нужен программист:**

- приём оплаты и связь платежа с конкретной услугой или периодом;
- состояние диалога, которое переживает перезапуск (пользователь на третьем шаге
  из пяти, сервер перезапустился — шаг не должен теряться);
- права и роли: ученик, преподаватель, администратор видят разное;
- связь с учётом — остатки, расписание, задолженности;
- рассылка по базе без попадания под флудлимит;
- повторная доставка, гонки, идемпотентность.

---

## 💾 Состояние диалога в базе, а не в памяти

Частая ошибка — держать шаг диалога в переменной процесса. Перезапуск, второй
воркер, и пользователь висит в никуда.

```php
// Состояние диалога — строка в БД, привязанная к чату.
// Переживает перезапуск, видно в админке, чинится руками при необходимости.
$session = $sessions->findByChat($chatId) ?? $sessions->start($chatId);

match ($session->step) {
    'await_phone'   => $this->handlePhone($session, $message),
    'await_subject' => $this->handleSubject($session, $message),
    default         => $this->showMainMenu($chatId),
};
```

---

## 🧩 Связка «бот + панель» вместо бота в одиночку

Бот удобен ученику или покупателю, но администратору в нём работать нельзя:
списки, фильтры, отчёты и массовые операции в чате невозможны.

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'ui-sans-serif,-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Helvetica,Arial,sans-serif','fontSize':'15px','textColor':'#1e1b4b','lineColor':'#4338ca','primaryColor':'#e0e7ff','primaryTextColor':'#1e1b4b','primaryBorderColor':'#4338ca','secondaryColor':'#ede9fe','tertiaryColor':'#f5f3ff','mainBkg':'#c7d2fe','nodeBorder':'#4338ca','nodeTextColor':'#1e1b4b','edgeLabelBackground':'#e0e7ff','attributeBackgroundColorOdd':'#eef2ff','attributeBackgroundColorEven':'#e0e7ff','noteBkgColor':'#fef3c7','noteTextColor':'#1a1a1a','noteBorderColor':'#b45309','clusterBkg':'#f5f3ff','clusterBorder':'#4338ca','labelBoxBkgColor':'#e0e7ff','labelBoxBorderColor':'#4338ca','labelTextColor':'#1e1b4b','actorBkg':'#c7d2fe','actorBorder':'#4338ca','actorTextColor':'#1e1b4b','actorLineColor':'#4338ca','signalColor':'#4338ca','signalTextColor':'#1e1b4b','sequenceNumberColor':'#ffffff','activationBkgColor':'#ddd6fe','activationBorderColor':'#4338ca','transitionColor':'#4338ca','transitionLabelColor':'#1e1b4b','stateBkg':'#c7d2fe','stateLabelColor':'#1e1b4b','altBackground':'#f5f3ff','compositeBackground':'#f5f3ff','compositeBorder':'#4338ca','compositeTitleBackground':'#e0e7ff','specialStateColor':'#4338ca','innerEndBackground':'#4338ca'}}}%%
flowchart TB
    U[Ученик в Telegram] --> BOT[Бот: запись, оплата, напоминания]
    BOT --> CORE[(Общая база и доменная логика)]
    ADMIN[Администратор в браузере] --> PANEL[Веб-панель: группы, расписание, долги]
    PANEL --> CORE
    PAY[Payme / Click] -->|webhook| CORE
    CORE --> NOTIFY[Уведомления обратно в бот]
    NOTIFY --> U

    style CORE fill:#6366f1,stroke:#4338ca,stroke-width:2px,color:#ffffff
```

Ключевое: **бот и панель — два интерфейса к одной доменной логике**, а не две
системы. Начисление задолженности считается в одном месте и одинаково, независимо
от того, кто спросил.

---

## 📢 Рассылка: ограничение как часть логики, а не рекомендация

Массовая отправка с личного аккаунта через MTProto — быстрый способ получить
флудлимит и потерять аккаунт вместе со всей перепиской. С ботом мягче, но лимиты
тоже есть.

Практика: суточный лимит зашивается в код и проверяется по журналу отправок,
между сообщениями пауза с разбросом, отправка только в рабочее окно. Не как
настройка, которую можно выкрутить, а как жёсткое ограничение.

---

## 🔍 Диагностика: почему «сессия жива, но не работает»

Реальный случай, стоивший времени. Клиентская библиотека возвращала
«не авторизован», хотя сохранённая сессия была в порядке.

Причина: библиотека **проглатывает все ошибки внутри** проверки авторизации и
возвращает `false` без исключения. Соединение умерло после многодневного простоя,
а наружу это выглядело как «войдите заново».

Лечение — не доверять одиночному `false`: сбросить закэшированный клиент,
переподключиться и перепроверить на свежем соединении. После этого «мёртвая»
сессия оживает без повторного входа по коду.

---

## 🔗 Смежные заметки

- [db-schema-notes](https://github.com/Shohruh1997/db-schema-notes) — схемы, включая `tg_chats` и составной ключ
- [payment-integration-notes](https://github.com/Shohruh1997/payment-integration-notes) — приём оплат из бота
- [php-layered-architecture-notes](https://github.com/Shohruh1997/php-layered-architecture-notes) — общая логика для бота и панели

---

<div align="center">

**Шохрух Рузиев** · backend-разработчик, Ташкент

[![Сайт](https://img.shields.io/badge/ecomdev.uz-6366f1?style=for-the-badge&logo=googlechrome&logoColor=white)](https://ecomdev.uz)
[![Telegram](https://img.shields.io/badge/Telegram-2CA5E0?style=for-the-badge&logo=telegram&logoColor=white)](https://t.me/EcomDev_uz)

Исходный код систем — в приватных репозиториях, доступ по запросу.

</div>
