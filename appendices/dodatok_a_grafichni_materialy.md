# ДОДАТОК А

# ГРАФІЧНІ МАТЕРІАЛИ ДО ІНФОРМАЦІЙНОЇ СИСТЕМИ GAMESINFOSYS

Додаток сформовано за принципом графічного додатка з дипломної роботи Д. О. Чуба: функціональні, процесні, інформаційні та UML-моделі винесено в окремий блок. Усі діаграми адаптовано до фактичної архітектури GamesInfoSys.

## А.1 Функціональні моделі IDEF0

Джерело моделі: `diagrams/06_idef0.md`.

# IDEF0-ДІАГРАМИ ІНФОРМАЦІЙНОЇ СИСТЕМИ `GamesInfoSys`

Діаграми побудовано за структурою рисунків 2.1–2.3 з оригінального DOCX. Збережено ICOM-логіку: входи розташовано зліва, виходи — справа, керування — зверху, механізми — знизу. Після експорту в DOCX потрібно перевірити, щоб стрілки входили саме у відповідні сторони функціональних блоків.

## Рисунок 2.1 – Контекстна діаграма «Управління інформаційним ресурсом відеоігор»

```mermaid
flowchart TB
    subgraph CONTROL["КЕРУВАННЯ"]
        direction LR
        C1["Правила валідації<br/>пошукових запитів та URL"]
        C2["Конфігурація API,<br/>регіонів і валют"]
        C3["Правила зіставлення<br/>зовнішніх ідентифікаторів"]
        C4["Модель даних та правила<br/>оновлення пропозицій"]
    end

    subgraph FUNCTION[" "]
        direction LR
        subgraph INPUT["ВХОДИ"]
            direction TB
            I1["Пошуковий запит"]
            I2["RAWG ID / Steam App ID"]
            I3["Метадані, ціни та курси валют"]
        end

        A0["A0<br/><br/>УПРАВЛІННЯ ІНФОРМАЦІЙНИМ<br/>РЕСУРСОМ ВІДЕОІГОР"]

        subgraph OUTPUT["ВИХОДИ"]
            direction TB
            O1["Результати пошуку"]
            O2["Сторінка деталей гри"]
            O3["Нормалізовані цінові пропозиції"]
            O4["Історія цін та зовнішні посилання"]
        end
    end

    subgraph MECHANISM["МЕХАНІЗМИ"]
        direction LR
        M1["Користувач"]
        M2["ASP.NET Core<br/>Razor Pages"]
        M3["RAWG / Steam /<br/>CheapShark / НБУ"]
        M4["Entity Framework Core<br/>та SQLite"]
    end

    C1 --> A0
    C2 --> A0
    C3 --> A0
    C4 --> A0
    I1 --> A0
    I2 --> A0
    I3 --> A0
    A0 --> O1
    A0 --> O2
    A0 --> O3
    A0 --> O4
    M1 --> A0
    M2 --> A0
    M3 --> A0
    M4 --> A0

    style A0 fill:#fff,stroke:#111,stroke-width:2px,color:#111
    style CONTROL fill:#fff,stroke:#fff
    style MECHANISM fill:#fff,stroke:#fff
    style FUNCTION fill:#fff,stroke:#fff
    style INPUT fill:#fff,stroke:#fff
    style OUTPUT fill:#fff,stroke:#fff
```

На контекстній діаграмі вся система подана як одна функція `A0`. Вона перетворює запити, ідентифікатори та зовнішні дані на результати пошуку, сторінки деталей і збережені пропозиції. Виконання функції регулюється конфігурацією й правилами обробки, а забезпечується програмними компонентами, зовнішніми сервісами та базою даних.

## Рисунок 2.2 – Декомпозиція контекстної діаграми IDEF0

```mermaid
flowchart LR
    I1["Пошуковий запит"] --> A1
    I2["RAWG ID гри"] --> A2
    I3["Steam App ID або URL"] --> A3
    I4["Дані магазинів"] --> A4
    I5["Курси валют"] --> A5

    A1["A1<br/>Пошук і перегляд<br/>каталогу"]
    A2["A2<br/>Отримання детальної<br/>інформації про гру"]
    A3["A3<br/>Визначення зовнішніх<br/>ідентифікаторів"]
    A4["A4<br/>Агрегація цінових<br/>пропозицій"]
    A5["A5<br/>Конвертація валют"]
    A6["A6<br/>Збереження пропозицій<br/>та історії цін"]
    A7["A7<br/>Формування результату<br/>для користувача"]

    A1 -- "вибрана гра" --> A2
    A2 -- "метадані та посилання" --> A3
    A2 -- "дані гри" --> A4
    A3 -- "зовнішні ID" --> A4
    A4 -- "нормалізовані пропозиції" --> A5
    A4 -- "дані для збереження" --> A6
    A5 -- "ціни у гривні" --> A7
    A6 -- "актуальні та історичні дані" --> A7
    A2 -- "опис, платформи, зображення" --> A7

    C1["Правила пошуку<br/>і валідації"] --> A1
    C1 --> A3
    C2["Налаштування API<br/>та регіонів"] --> A2
    C2 --> A4
    C3["Правила конвертації"] --> A5
    C4["Схема БД та унікальні індекси"] --> A6

    M1["RawgClient"] --> A1
    M1 --> A2
    M2["OfferAggregator<br/>і клієнти магазинів"] --> A3
    M2 --> A4
    M3["NbuRatesClient<br/>CurrencyConverter"] --> A5
    M4["EF Core / SQLite"] --> A6
    M5["Razor Pages"] --> A7

    A1 --> O1["Результати пошуку"]
    A7 --> O2["Сторінка деталей,<br/>пропозиції та посилання"]

    style A1 fill:#fff,stroke:#111
    style A2 fill:#fff,stroke:#111
    style A3 fill:#fff,stroke:#111
    style A4 fill:#fff,stroke:#111
    style A5 fill:#fff,stroke:#111
    style A6 fill:#fff,stroke:#111
    style A7 fill:#fff,stroke:#111
```

Декомпозиція зберігає граничні стрілки контекстної діаграми та розподіляє загальну функцію між сімома підпроцесами. Основний інформаційний потік проходить від пошуку до формування сторінки, а процеси `A4–A6` відповідають за отримання, перетворення й накопичення цінових даних.

## Рисунок 2.3 – Декомпозиція процесу «Агрегація цінових пропозицій»

```mermaid
flowchart LR
    I1["RAWG ID, назва гри<br/>та метадані"] --> A41
    I2["Steam App ID або URL"] --> A42

    A41["A41<br/>Знайти або створити<br/>відстежувану гру"]
    A42["A42<br/>Визначити й перевірити<br/>Steam App ID"]
    A43["A43<br/>Отримати пропозицію<br/>Steam"]
    A44["A44<br/>Отримати пропозиції<br/>CheapShark"]
    A45["A45<br/>Отримати пропозиції<br/>українських джерел"]
    A46["A46<br/>Нормалізувати та<br/>усунути дублікати"]
    A47["A47<br/>Зберегти пропозиції<br/>й точки історії"]

    A41 -- "TrackedGame" --> A42
    A42 -- "перевірений Steam App ID" --> A43
    A42 -- "перевірений Steam App ID" --> A44
    A41 -- "назва та платформа" --> A45
    A43 -- "дані Steam" --> A46
    A44 -- "дані CheapShark" --> A46
    A45 -- "локальні пропозиції" --> A46
    A46 -- "StoreOffer" --> A47
    A47 --> O1["Актуальні пропозиції<br/>та історія цін"]

    C1["Правила валідації URL"] --> A42
    C2["Регіон ціноутворення"] --> A43
    C3["Правила унікальності<br/>Store + ExternalId + Region"] --> A46
    C4["Схема TrackedGame,<br/>StoreOffer, OfferPricePoint"] --> A47

    M1["OfferAggregator"] --> A41
    M2["SteamStoreClient"] --> A43
    M3["CheapSharkClient"] --> A44
    M4["UaMarketplaceScraper"] --> A45
    M5["AppDbContext / SQLite"] --> A47

    style A41 fill:#fff,stroke:#111
    style A42 fill:#fff,stroke:#111
    style A43 fill:#fff,stroke:#111
    style A44 fill:#fff,stroke:#111
    style A45 fill:#fff,stroke:#111
    style A46 fill:#fff,stroke:#111
    style A47 fill:#fff,stroke:#111
```

Процеси отримання пропозицій із Steam, CheapShark та українських джерел можуть виконуватися незалежно, але їхні результати надходять до спільного етапу нормалізації. Після цього дані зберігаються в єдиній моделі, що забезпечує однакове відображення незалежно від джерела.

## Відповідність програмному коду

- `A1–A2` — `RawgClient.SearchGamesAsync`, `RawgClient.GetGameAsync`, `RawgClient.GetScreenshotsAsync`.
- `A3` — `DetailsModel.TryParseSteamAppId`, `OfferAggregator.TryInferSteamAppId`.
- `A4` — `OfferAggregator.SyncOffersForRawgGameAsync`.
- `A5` — `CurrencyConverter.ToUahAsync`, `NbuRatesClient`.
- `A6` — `AppDbContext`, `TrackedGame`, `StoreOffer`, `OfferPricePoint`.
- `A7` — `Pages/Games/Details.cshtml.cs` і `Pages/Games/Details.cshtml`.

## А.2 Процесні моделі IDEF3

Джерело моделі: `diagrams/08_idef3.md`.

# IDEF3-ДІАГРАМИ ОСНОВНИХ СЦЕНАРІЇВ `GamesInfoSys`

Діаграми 2.4–2.9 замінюють шість IDEF3-прикладів з оригінального DOCX. Блоки `UOB` описують одиниці роботи, а вузли `J` показують розгалуження або об’єднання потоків.

## Рисунок 2.4 – IDEF3-діаграма процесу «Пошук і вибір відеогри»

```mermaid
flowchart LR
    U1["UOB1<br/>Відкрити каталог"] --> U2["UOB2<br/>Ввести пошуковий запит"]
    U2 --> U3["UOB3<br/>Передати запит RawgClient"]
    U3 --> J1{"J1<br/>Режим даних"}
    J1 -- "RAWG API" --> U4["UOB4<br/>Отримати відповідь API"]
    J1 -- "demo" --> U5["UOB5<br/>Відфільтрувати demo-games.json"]
    U4 --> J2{"J2<br/>Об'єднання"}
    U5 --> J2
    J2 --> U6["UOB6<br/>Відобразити результати"]
    U6 --> U7["UOB7<br/>Вибрати гру"]
```

## Рисунок 2.5 – IDEF3-діаграма процесу «Отримання детальної інформації»

```mermaid
flowchart LR
    U1["UOB1<br/>Передати RAWG ID"] --> J1{"J1<br/>Паралельний запуск"}
    J1 --> U2["UOB2<br/>Отримати опис, жанри,<br/>платформи та рейтинг"]
    J1 --> U3["UOB3<br/>Отримати знімки екрана"]
    J1 --> U4["UOB4<br/>Отримати зовнішні посилання"]
    U2 --> J2{"J2<br/>Синхронізація"}
    U3 --> J2
    U4 --> J2
    J2 --> U5["UOB5<br/>Сформувати модель сторінки"]
```

## Рисунок 2.6 – IDEF3-діаграма процесу «Визначення Steam App ID»

```mermaid
flowchart LR
    U1["UOB1<br/>Перевірити TrackedGame"] --> J1{"J1<br/>ID уже відомий?"}
    J1 -- "так" --> U5["UOB5<br/>Використати збережений ID"]
    J1 -- "ні" --> U2["UOB2<br/>Перевірити посилання RAWG"]
    U2 --> J2{"J2<br/>ID знайдено?"}
    J2 -- "так" --> U3["UOB3<br/>Зберегти визначений ID"]
    J2 -- "ні" --> U4["UOB4<br/>Прийняти ID або URL від користувача"]
    U4 --> U6["UOB6<br/>Перевірити формат і домен"]
    U6 --> U3
    U3 --> U5
```

## Рисунок 2.7 – IDEF3-діаграма процесу «Отримання цінових пропозицій»

```mermaid
flowchart LR
    U1["UOB1<br/>Підготувати ідентифікатори<br/>та пошукові назви"] --> J1{"J1<br/>Паралельне розгалуження"}
    J1 --> U2["UOB2<br/>Запитати Steam Store"]
    J1 --> U3["UOB3<br/>Запитати CheapShark"]
    J1 --> U4["UOB4<br/>Пошукати UA-пропозиції"]
    U2 --> J2{"J2<br/>Об'єднання відповідей"}
    U3 --> J2
    U4 --> J2
    J2 --> U5["UOB5<br/>Передати сирі пропозиції<br/>на нормалізацію"]
```

## Рисунок 2.8 – IDEF3-діаграма процесу «Конвертація вартості у гривню»

```mermaid
flowchart LR
    U1["UOB1<br/>Перевірити наявність<br/>ціни та валюти"] --> J1{"J1<br/>Валюта UAH?"}
    J1 -- "так" --> U2["UOB2<br/>Використати поточне значення"]
    J1 -- "ні" --> U3["UOB3<br/>Отримати курс НБУ з кешу"]
    U3 --> J2{"J2<br/>Курс наявний?"}
    J2 -- "так" --> U4["UOB4<br/>Обчислити орієнтовну<br/>вартість у гривні"]
    J2 -- "ні" --> U5["UOB5<br/>Позначити конвертацію<br/>як недоступну"]
    U2 --> U6["UOB6<br/>Передати результат у UI"]
    U4 --> U6
    U5 --> U6
```

## Рисунок 2.9 – IDEF3-діаграма процесу «Збереження та відображення пропозицій»

```mermaid
flowchart LR
    U1["UOB1<br/>Нормалізувати пропозицію"] --> U2["UOB2<br/>Знайти StoreOffer за<br/>Store + ExternalId + Region"]
    U2 --> J1{"J1<br/>Запис існує?"}
    J1 -- "ні" --> U3["UOB3<br/>Створити StoreOffer"]
    J1 -- "так" --> U4["UOB4<br/>Оновити ціну, URL<br/>та час спостереження"]
    U3 --> J2{"J2<br/>Об'єднання"}
    U4 --> J2
    J2 --> U5["UOB5<br/>Додати OfferPricePoint"]
    U5 --> U6["UOB6<br/>Прочитати актуальні пропозиції"]
    U6 --> U7["UOB7<br/>Відобразити таблицю,<br/>UAH та зовнішні посилання"]
```

## Рекомендації до експорту

- використовувати білий фон, чорні контури та однаковий розмір блоків;
- розміщувати сценарій зліва направо, як у зразках оригінального DOCX;
- не зменшувати текст нижче читабельного розміру після вставлення на сторінку А4;
- під кожним експортованим зображенням додати відповідний підпис `Рисунок 2.4 – ...`;
- за потреби розмістити широкі діаграми на альбомній сторінці з коректними полями шаблону.

## А.3 Діаграми потоків даних DFD

Джерело моделі: `diagrams/07_dfd.md`.

# DFD-ДІАГРАМИ ІНФОРМАЦІЙНОЇ СИСТЕМИ `GamesInfoSys`

Комплект відповідає рисункам 2.10–2.12 оригінального DOCX: контекст системи, рівень підсистеми та деталізація одного процесу. На відміну від IDEF0, стрілки тут позначають саме дані, а не керування або механізми.

## Рисунок 2.10 – Контекстна діаграма DFD рівня системи

```mermaid
flowchart LR
    U["Користувач"]
    E1["RAWG / локальний<br/>демонстраційний набір"]
    E2["Steam Store,<br/>CheapShark,<br/>UA-маркетплейси"]
    E3["API НБУ"]
    P0(["0<br/>Інформаційна система<br/>GamesInfoSys"])

    U -- "пошуковий запит;<br/>вибір гри;<br/>Steam App ID або URL" --> P0
    P0 -- "результати пошуку;<br/>деталі гри;<br/>ціни та посилання" --> U
    P0 -- "запит метаданих гри" --> E1
    E1 -- "каталог, опис,<br/>платформи, зображення" --> P0
    P0 -- "ідентифікатори та<br/>запити пропозицій" --> E2
    E2 -- "ціни, валюти,<br/>назви й URL" --> P0
    P0 -- "запит валютних курсів" --> E3
    E3 -- "курси валют до гривні" --> P0

    style P0 fill:#fff,stroke:#111,stroke-width:2px
```

Зовнішньою сутністю, що ініціює основні сценарії, є користувач. Інші зовнішні сутності є постачальниками даних. База даних на контекстному рівні не показується як зовнішня сутність, оскільки є внутрішньою частиною `GamesInfoSys`.

## Рисунок 2.11 – Діаграма потоків даних рівня підсистеми

```mermaid
flowchart LR
    U["Користувач"]
    R["RAWG / demo data"]
    S["Steam Store"]
    C["CheapShark"]
    M["UA-маркетплейси"]
    N["API НБУ"]

    P1(["1.0<br/>Пошук ігор"])
    P2(["2.0<br/>Отримання деталей"])
    P3(["3.0<br/>Агрегація пропозицій"])
    P4(["4.0<br/>Конвертація валют"])
    P5(["5.0<br/>Формування сторінки"])

    D1[("D1<br/>TrackedGames")]
    D2[("D2<br/>StoreOffers")]
    D3[("D3<br/>OfferPricePoints")]

    U -- "пошуковий запит" --> P1
    P1 -- "запит каталогу" --> R
    R -- "список ігор" --> P1
    P1 -- "результати пошуку" --> U
    U -- "RAWG ID вибраної гри" --> P2
    P2 -- "запит деталей і знімків" --> R
    R -- "метадані гри" --> P2
    P2 -- "RAWG ID, назва,<br/>зовнішні посилання" --> P3
    U -- "Steam App ID або URL" --> P3
    P3 -- "запит Steam" --> S
    S -- "метадані та ціна Steam" --> P3
    P3 -- "запит пропозицій" --> C
    C -- "пропозиції CheapShark" --> P3
    P3 -- "пошукові запити" --> M
    M -- "локальні пропозиції" --> P3
    P3 <--> D1
    P3 <--> D2
    P3 --> D3
    D2 -- "актуальні пропозиції" --> P4
    P4 -- "запит курсу валюти" --> N
    N -- "курс до UAH" --> P4
    P2 -- "опис, платформи,<br/>жанри, зображення" --> P5
    P3 -- "магазини, ціни,<br/>URL та регіони" --> P5
    P4 -- "орієнтовні ціни у гривні" --> P5
    P5 -- "сторінка деталей гри" --> U

    style P1 fill:#fff,stroke:#111
    style P2 fill:#fff,stroke:#111
    style P3 fill:#fff,stroke:#111
    style P4 fill:#fff,stroke:#111
    style P5 fill:#fff,stroke:#111
```

На цьому рівні видно, що дані гри не записуються безпосередньо до `StoreOffers`. Спочатку процес `3.0` створює або знаходить запис у `TrackedGames`, після чого пов’язує з ним нормалізовані пропозиції та історичні точки.

## Рисунок 2.12 – Діаграма DFD процесу «Агрегація цінових пропозицій»

```mermaid
flowchart LR
    IN1["RAWG ID, назва та<br/>метадані гри"]
    IN2["Steam App ID або URL"]
    P31(["3.1<br/>Отримати запис<br/>відстежуваної гри"])
    P32(["3.2<br/>Визначити Steam App ID"])
    P33(["3.3<br/>Отримати зовнішні<br/>пропозиції"])
    P34(["3.4<br/>Нормалізувати й<br/>зіставити дані"])
    P35(["3.5<br/>Оновити поточні<br/>пропозиції"])
    P36(["3.6<br/>Додати точки<br/>історії цін"])
    EXT1["Steam / CheapShark /<br/>UA-маркетплейси"]
    D1[("D1<br/>TrackedGames")]
    D2[("D2<br/>StoreOffers")]
    D3[("D3<br/>OfferPricePoints")]

    IN1 --> P31
    P31 -- "пошук за RAWG ID" --> D1
    D1 -- "наявна гра" --> P31
    P31 -- "нова або наявна<br/>TrackedGame" --> D1
    P31 -- "дані відстежуваної гри" --> P32
    IN2 --> P32
    P32 -- "оновлений Steam App ID" --> D1
    P32 -- "ідентифікатори й назва" --> P33
    P33 -- "запити цін і товарів" --> EXT1
    EXT1 -- "неоднорідні відповіді" --> P33
    P33 -- "сирі пропозиції" --> P34
    P34 -- "нормалізовані StoreOffer" --> P35
    P35 -- "пошук дубліката" --> D2
    D2 -- "наявна пропозиція" --> P35
    P35 -- "нова або оновлена пропозиція" --> D2
    P35 -- "ціна, валюта,<br/>StoreOffer ID" --> P36
    P36 -- "нова часова точка" --> D3
    D2 -- "актуальний список пропозицій" --> OUT["Пропозиції для сторінки деталей"]
    D3 -- "історичні значення" --> OUT

    style P31 fill:#fff,stroke:#111
    style P32 fill:#fff,stroke:#111
    style P33 fill:#fff,stroke:#111
    style P34 fill:#fff,stroke:#111
    style P35 fill:#fff,stroke:#111
    style P36 fill:#fff,stroke:#111
```

Балансування DFD дотримано: дані, що входять до процесу `3.0` на рисунку 2.11, присутні на детальній діаграмі як `RAWG ID`, назва, метадані та Steam App ID. Виходом є список актуальних пропозицій і збережена історія.

## Словник потоків даних

| Потік | Склад даних |
|---|---|
| Пошуковий запит | текст назви або її частина |
| Метадані гри | RAWG ID, назва, опис, дата випуску, рейтинг, жанри, платформи, URL, зображення |
| Зовнішній ідентифікатор | Steam App ID або інший ID магазину |
| Пропозиція | магазин, платформа, регіон, зовнішній ID, назва товару, URL, валюта, поточна та початкова ціна |
| Валютний курс | код валюти та коефіцієнт перерахунку до гривні |
| Точка історії | StoreOffer ID, момент часу UTC, валюта, ціна та початкова ціна |

## А.4 UML-діаграма варіантів використання

Джерело моделі: `diagrams/01_use_case.md`.

# Use case diagram for `GamesInfoSys`

```mermaid
flowchart LR
    user["User"]
    rawg["RAWG API"]
    steam["Steam Store"]
    cheap["CheapShark"]
    nbu["NBU rates service"]
    ua["UA marketplaces"]

    uc1(("Search games"))
    uc2(("Browse catalog"))
    uc3(("View game details"))
    uc4(("Refresh offers"))
    uc5(("Set Steam App ID"))
    uc6(("Convert prices to UAH"))
    uc7(("Open external stores"))
    uc8(("View price history data"))

    user --> uc1
    user --> uc2
    user --> uc3
    user --> uc4
    user --> uc5
    user --> uc6
    user --> uc7
    user --> uc8

    uc1 --> rawg
    uc2 --> rawg
    uc3 --> rawg
    uc4 --> steam
    uc4 --> cheap
    uc4 --> ua
    uc6 --> nbu
```

## Notes from code

- Search and catalog browsing are handled by `RawgClient.SearchGamesAsync(...)`.
- Game details are loaded by `RawgClient.GetGameAsync(...)`.
- Offer refresh is coordinated by `OfferAggregator.SyncOffersForRawgGameAsync(...)`.
- Manual Steam mapping is handled by `DetailsModel.OnPostSetSteamAsync(...)`.
- UAH conversion is handled by `CurrencyConverter.ToUahAsync(...)`.

## А.5 Компонентна архітектура

Джерело моделі: `diagrams/02_component_architecture.md`.

# Component architecture for `GamesInfoSys`

```mermaid
flowchart TD
    browser["Browser / User UI"]

    subgraph RazorPages["ASP.NET Core Razor Pages"]
        indexPage["Games/Index PageModel"]
        detailsPage["Games/Details PageModel"]
    end

    subgraph Services["Application services"]
        rawg["RawgClient"]
        offers["OfferAggregator"]
        fx["CurrencyConverter"]
        regions["RegionResolver"]
        steam["SteamStoreClient"]
        cheap["CheapSharkClient"]
        nbu["NbuRatesClient"]
        ua["UaMarketplaceScraper"]
        text["UiText"]
    end

    subgraph Data["Persistence"]
        db["AppDbContext"]
        sqlite["SQLite app.db"]
    end

    subgraph External["External systems"]
        rawgApi["RAWG API"]
        steamApi["Steam Store API"]
        cheapApi["CheapShark API"]
        nbuApi["NBU exchange rates"]
        uaSites["Prom / OLX / UA sources"]
    end

    browser --> indexPage
    browser --> detailsPage

    indexPage --> rawg

    detailsPage --> rawg
    detailsPage --> offers
    detailsPage --> fx
    detailsPage --> text

    offers --> db
    db --> sqlite

    offers --> regions
    offers --> steam
    offers --> cheap
    offers --> ua

    fx --> nbu

    rawg --> rawgApi
    steam --> steamApi
    cheap --> cheapApi
    nbu --> nbuApi
    ua --> uaSites
```

## Notes from code

- Service registration and wiring are defined in `GamesInfoSys/GamesInfoSys/Program.cs`.
- The main page models are `Pages/Games/Index.cshtml.cs` and `Pages/Games/Details.cshtml.cs`.
- The persistence layer is `AppDbContext` over SQLite `app.db`.

## А.6 Діаграма послідовності

Джерело моделі: `diagrams/03_sequence_search_and_details.md`.

# Sequence diagram for search and details flow

```mermaid
sequenceDiagram
    actor User
    participant IndexPage as Games Index Page
    participant DetailsPage as Game Details Page
    participant Rawg as RawgClient
    participant Offers as OfferAggregator
    participant Steam as SteamStoreClient
    participant Cheap as CheapSharkClient
    participant FX as CurrencyConverter
    participant NBU as NbuRatesClient
    participant DB as AppDbContext

    User->>IndexPage: Open /Games?q=...
    IndexPage->>Rawg: SearchGamesAsync(query)
    Rawg-->>IndexPage: GameSummary[]
    IndexPage-->>User: Catalog page

    User->>DetailsPage: Open /Games/{id}
    DetailsPage->>Rawg: GetGameAsync(id)
    Rawg-->>DetailsPage: GameDetails

    DetailsPage->>Offers: SyncOffersForRawgGameAsync(id, name, details)
    Offers->>DB: Load/create TrackedGame
    Offers->>Steam: GetAppPriceAsync(appId, region)
    Steam-->>Offers: SteamPriceResult
    Offers->>Cheap: GetDealsBySteamAppIdAsync(appId)
    Cheap-->>Offers: CheapSharkDeal[]
    Offers->>DB: Save StoreOffer and OfferPricePoint

    DetailsPage->>Offers: GetOffersForRawgGameAsync(id)
    Offers-->>DetailsPage: StoreOffer[]

    loop each offer
        DetailsPage->>FX: ToUahAsync(amount, currency)
        FX->>NBU: GetRatesToUahAsync()
        NBU-->>FX: Currency rates
        FX-->>DetailsPage: UAH value
    end

    DetailsPage-->>User: Details page with offers
```

## Notes from code

- Search flow: `GamesInfoSys/GamesInfoSys/Pages/Games/Index.cshtml.cs`
- Details flow: `GamesInfoSys/GamesInfoSys/Pages/Games/Details.cshtml.cs`
- Offer synchronization: `GamesInfoSys/GamesInfoSys/Services/OfferAggregator.cs`

## А.7 Модель даних

Джерело моделі: `diagrams/04_data_model.md`.

# Data model diagram for persistent entities

```mermaid
classDiagram
    class TrackedGame {
        +long Id
        +int? RawgGameId
        +string? Name
        +string? SteamAppId
        +string? PsnProductId
        +string? XboxProductId
        +string? NintendoProductId
        +string? EpicOfferId
        +string? GogProductId
        +DateTime CreatedUtc
    }

    class StoreOffer {
        +long Id
        +long TrackedGameId
        +string Store
        +string Platform
        +string Region
        +string ExternalId
        +string Title
        +string Url
        +string? Currency
        +long? PriceMinor
        +long? OriginalPriceMinor
        +DateTime LastSeenUtc
        +DateTime LastUpdatedUtc
    }

    class OfferPricePoint {
        +long Id
        +long StoreOfferId
        +DateTime AtUtc
        +string Currency
        +long PriceMinor
        +long? OriginalPriceMinor
    }

    TrackedGame "1" --> "many" StoreOffer : has
    StoreOffer "1" --> "many" OfferPricePoint : stores history
```

## Notes from code

- Entities are defined in:
  - `GamesInfoSys/GamesInfoSys/Data/Entities/TrackedGame.cs`
  - `GamesInfoSys/GamesInfoSys/Data/Entities/StoreOffer.cs`
  - `GamesInfoSys/GamesInfoSys/Data/Entities/OfferPricePoint.cs`
- EF Core configuration is defined in `GamesInfoSys/GamesInfoSys/Data/AppDbContext.cs`.

## А.8 Діаграма діяльності

Джерело моделі: `diagrams/05_offer_sync_activity.md`.

# Activity diagram for offer synchronization

```mermaid
flowchart TD
    A[Open game details page] --> B[Load or create TrackedGame]
    B --> C{Steam App ID exists?}

    C -- No --> D{Can infer from RAWG external links?}
    D -- Yes --> E[Save inferred Steam App ID]
    D -- No --> F[Skip Steam and CheapShark sync]
    E --> G[Sync Steam price]
    C -- Yes --> G

    G --> H[Upsert Steam StoreOffer]
    H --> I[Append OfferPricePoint]
    I --> J[Sync CheapShark deals]
    J --> K[Upsert CheapShark offers]
    K --> L{Name exists?}

    F --> L
    L -- Yes --> M[Search UA marketplaces for Xbox]
    M --> N[Search UA marketplaces for Switch]
    N --> O[Upsert UA marketplace offers]
    L -- No --> P[Return current offers]
    O --> P
```

## Notes from code

- Main method: `OfferAggregator.SyncOffersForRawgGameAsync(...)`
- Steam sync: `SyncSteamAsync(...)`
- CheapShark sync: `SyncCheapSharkAsync(...)`
- UA marketplace sync: `SyncUaMarketplacesAsync(...)`

