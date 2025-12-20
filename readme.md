## 📘 Описание UML-диаграмм

### Roadmap
```mermaid
gantt
  title Wealthify Roadmap (2025-09-01 – 2025-12-12)
  dateFormat YYYY-MM-DD
  axisFormat %d %b

  section Бэкенд: база (готово)
  Users/Auth/Roles (JWT)          :done, a1, 2025-09-01, 2025-10-10
  Assets/Portfolios/Transactions  :done, a2, 2025-09-15, 2025-10-20
  Crypto-data-worker (1 парсер)   :done, a3, 2025-10-10, 2025-11-05
  Sign-in/Sign-up (FE)            :done, a4, 2025-10-05, 2025-10-25

  section Микросервисы и связь
  Сплит — Identity/Assets/Portfolio-core :crit, active, b1, 2025-11-12, 2025-11-20
  Внедрение RabbitMQ (pub/sub, RPC)      :crit, b2, 2025-11-15, 2025-11-22
  Деприкация TCP + фоллбек               :b3, 2025-11-22, 2025-11-24

  section Парсинг и масштабирование
  Puppeteer-cluster PoC                  :c1, 2025-11-15, 2025-11-18
  Очереди — раздача ссылок воркерам      :c2, 2025-11-18, 2025-11-20
  Антибот/ретраи/таймауты                :c3, 2025-11-19, 2025-11-23
  Метрики и настройки конкаренси         :c4, 2025-11-23, 2025-11-24

  section Фронтенд (Next.js)
  Страница All Assets                    :crit, active, d1, 2025-11-12, 2025-11-18
  Страница Asset                         :crit, d2, 2025-11-18, 2025-11-22
  Страница Portfolios                    :crit, d3, 2025-11-19, 2025-11-25
  Страница Portfolio                     :crit, d4, 2025-11-25, 2025-11-28

  section Индексы и агрегатор
  Выбор источников/контракты данных      :e1, 2025-11-20, 2025-11-21
  Реализация парсеров индексов           :crit, e2, 2025-11-21, 2025-11-30
  Aggregation API + кэш                  :crit, e3, 2025-11-26, 2025-12-02
  Интеграция в All Assets                :crit, e4, 2025-12-02, 2025-12-03

  section Ops/QA/Безопасность
  CI/CD, Docker, Staging                 :active, f1, 2025-11-12, 2025-11-20
  E2E (auth, buy/sell флоу)              :f2, 2025-11-20, 2025-11-27
  Нагрузочные (API/воркеры)              :f3, 2025-11-27, 2025-11-30
  Security pass (RBAC/JWT/ratelimits)    :f4, 2025-11-30, 2025-12-05

  section Документы и релиз
  Документация/README/рунабук            :g1, 2025-12-06, 2025-12-08
  UAT и баг-фикс буфер                   :g2, 2025-12-08, 2025-12-10
  Code Freeze                            :milestone, g3, 2025-12-11, 1d
  Release v1                             :milestone, g4, 2025-12-12, 1d
```

### Diagrams
Представленные UML-диаграммы описывают ключевые процессы и структуру взаимодействия компонентов системы.

```mermaid
sequenceDiagram
    title Процесс регистрации
    actor User as Пользователь
    participant Controller as auth.controller.ts
    participant Service as auth.service.ts
    participant DB as База данных
    
    User->>Controller: POST /sign-up\nbody: {username, email, password}
    Controller->>Service: registerUser(username, email, password)
    Service->>Service: хеширование пароля
    Service->>DB: INSERT INTO users (сохраняет пользователя)
    DB-->>Service: OK / Ошибка
    Service->>Service: генерация refreshToken + accessToken
    Service-->>Controller: результат регистрации (успех / ошибка)
    Controller-->>User: response\nCookies: refreshToken + accessToken\nBody: {status: ok / ne ok}
```
Процесс регистрации — диаграмма последовательности, демонстрирующая шаги, выполняемые при создании нового пользователя: передача данных с клиента, обращение к контроллеру и сервису, хеширование пароля, сохранение записи в базе данных и генерация токенов доступа.
```mermaid
sequenceDiagram
    title Процесс авторизации и обновления токенов пользователя

    actor User as "Пользователь"
    participant Client as "Frontend / Client"
    participant Controller as "auth.controller.ts"
    participant Service as "auth.service.ts"
    participant DB as "База данных"

    User->>Client: Открывает защищённую страницу\n(HTTP-Only access token)
    Client->>Controller: запрос к защищённому эндпоинту
    Controller->>Service: проверка access token
    Service->>Service: проверка срока действия

    alt access token истёк
        Service-->>Controller: 401 ИСТЕК_АКСЕСС_ТОКЕН
        Controller-->>Client: 401 / требуется refresh
        Client-->>User: токен истёк, обновляем…
        Client->>Controller: POST /refresh\n(HTTP-Only refresh token)
    else access token валиден
        Service-->>Controller: OK
        Controller-->>Client: 200 OK + данные
        Client-->>User: данные отображены
    end

    Client->>Controller: POST /refresh\n(HTTP-Only refresh token)
    Controller->>Service: валидация refresh
    Service->>DB: найти refresh token
    DB-->>Service: найден / не найден

    alt refresh валиден
        Service->>Service: сгенерировать новые токены
        Service-->>Controller: refresh + access
        Controller-->>Client: Set-Cookie: refreshToken, accessToken
        Client-->>User: авторизация обновлена
    else refresh не найден/просрочен
        Service-->>Controller: ошибка авторизации
        Controller-->>Client: 401 / redirect
        Client-->>User: требуется вход (sign-in)
    end

    Client->>Controller: повторный запрос с новым access token
    Controller->>Service: проверка токена
    Service-->>Controller: OK
    Controller-->>Client: 200 OK + данные
    Client-->>User: ответ отображён
```
Процесс авторизации и обновления токенов — диаграмма последовательности, показывающая проверку access token при обращении к защищённым эндпоинтам, обработку ситуации с истекшим токеном, генерацию новых refresh и access токенов и повторную проверку авторизации.

```mermaid
classDiagram
    %% ==== МОДЕЛИ ====
    class Пользователи {
        + id : PK
        + username : string
        + email : string
        + password : string (hashed)
    }

    class Роли {
        + id : PK
        + name : string
    }

    class UserRoles {
        + id : PK
        + userId : FK
        + roleId : FK
        --
        "Таблица связи многие-ко-многим между пользователями и ролями"
    }

    class Портфели {
        + id : PK
        + name : string
        + type : string (Крипта, акции, облигации, фиат)
        + userId : FK
    }

    class Активы {
        + id : PK
        + name : string
        + ticker : string
        + type : string (Крипта, акция, облигация, фиат)
    }

    class PortfolioAssets {
        + id : PK
        + portfolioId : FK
        + assetId : FK
        + quantity : float
        + avgPurchasePrice : float
        --
        "Таблица связи многие-ко-многим между портфелями и активами"
    }

    class Транзакции {
        + id : PK
        + userId : FK
        + assetId : FK
        + portfolioId : FK
        + type : string
        + quantity : float
        + buyPrice : float
        + date : datetime
    }

    class RefreshToken {
        + id : PK
        + token : string
        + userId : FK
        + expiryDate : datetime
    }

    class ResetToken {
        + id : PK
        + token : string
        + userId : FK
        + expiryDate : datetime
    }

    %% ==== СВЯЗИ ====
    %% Пользователи и Роли — many-to-many
    Пользователи "1" -- "0..*" UserRoles : связывается >
    Роли "1" -- "0..*" UserRoles : связывается >
    note for UserRoles "Связь many-to-many между Users и Roles"

    %% Пользователь ↔ Портфели
    Пользователи "1" --> "0..*" Портфели : владеет >

    %% Пользователь ↔ Токены
    Пользователи "1" --> "0..*" RefreshToken : имеет >
    Пользователи "1" --> "0..*" ResetToken : может иметь >

    %% Пользователь ↔ Транзакции
    Пользователи "1" --> "0..*" Транзакции : совершает >

    %% Портфели ↔ Активы (через PortfolioAssets)
    Портфели "1" -- "0..*" PortfolioAssets : связывает >
    Активы "1" -- "0..*" PortfolioAssets : связывает >
    note for PortfolioAssets "Связь many-to-many между Portfolios и Assets"

    %% Портфели ↔ Транзакции
    Портфели "1" --> "0..*" Транзакции : имеет >

    %% Активы ↔ Транзакции
    Активы "1" --> "0..*" Транзакции : участвует >
```
Модели и их взаимодействие (User Portfolio System) — диаграмма классов, отражающая структуру данных приложения: пользователей, роли, портфели, активы, транзакции и токены.
В ней указаны первичные и внешние ключи, а также связи между сущностями: один-ко-многим (например, пользователь → портфели) и многие-ко-многим (через таблицы UserRoles и PortfolioAssets).

*Доступные действия пользователя*

![Диаграмма пользоватея](imgs/User_Capabilities_Account.png)
Диаграмма отображает все действия, доступные пользователю для управления аккаунтом. Эти действия включают регистрацию, авторизацию, обновление токенов, сброс пароля и изменение пароля. Часть функций доступна публично, но для большинства операций требуется авторизация через JwtAuthGuard.

![Диаграмма пользоватея](imgs/User_Capabilities_Portfolios.png)
Диаграмма отображает все действия, доступные пользователю для управления аккаунтом. Эти действия включают регистрацию, авторизацию, обновление токенов, сброс пароля и изменение пароля. Часть функций доступна публично, но для большинства операций требуется авторизация через JwtAuthGuard.


![Диаграмма пользоватея](imgs/User_Capabilities_Transactions.png)

Диаграмма отображает все действия, доступные пользователю для управления аккаунтом. Эти действия включают регистрацию, авторизацию, обновление токенов, сброс пароля и изменение пароля. Часть функций доступна публично, но для большинства операций требуется авторизация через JwtAuthGuard.


![Диаграмма пользоватея](imgs/Admin_Capabilities_Roles.png)

Диаграмма показывает действия, доступные администратору для управления пользователями и ролями. Администратор может получать информацию обо всех пользователях, добавлять и удалять роли, а также создавать новые роли и получать их по значению.


![Диаграмма пользоватея](imgs/Admin_Capabilities_Transactions.png)

Диаграмма показывает действия, доступные администратору для работы с активами и транзакциями. Администратор может создавать активы, удалять их по тикеру, а также просматривать все транзакции. 

Все эти действия также требуют наличия роли ADMIN и проверки прав через JwtAuthGuard и RolesGuard.

```mermaid
classDiagram
  %% =========================
  %% ОСНОВНЫЕ СУЩНОСТИ (Identity)
  %% =========================
  class User {
    +id: number
    +username: string
    +email: string
    +password: string
  }

  class Role {
    +id: number
    +value: string
    +description: string
  }

  class UserRoles {
    +id: number
    +userId: number
    +roleId: number
  }

  class RefreshToken {
    +id: number
    +token: string  %% hashed
    +userId: number %% unique
    +expiryDate: Date
  }

  class ResetToken {
    +id: number
    +token: string  %% hashed
    +userId: number %% unique
    +expiryDate: Date
  }

  class FavoriteAsset {
    +id: number
    +userId: number
    +assetId: number
    +createdAt: Date
  }

  %% =========================
  %% ПОРТФЕЛИ / ТРАНЗАКЦИИ
  %% =========================
  class Portfolio {
    +id: number
    +name: string
    +type: PortfolioType
    +userId: number
  }

  class PortfolioAssets {
    +id: number
    +portfolioId: number
    +assetId: number
    +quantity: number
    +averageBuyPrice: number
    +purchaseDate: Date?
  }

  class Transaction {
    +id: number
    +portfolioId: number
    +assetId: number
    +type: TransactionType
    +quantity: number
    +pricePerUnit: number
    +date: Date
  }

  %% =========================
  %% АКТИВЫ (общая таблица assets)
  %% =========================
  class Asset {
    +id: number
    +name: string
    +ticker: string
    +type: AssetType
  }

  %% =========================
  %% КАТЕГОРИИ АКТИВОВ (выделено в сущность)
  %% =========================
  class AssetCategory {
    +id: number
    +name: string
    +source?: string
  }

  class AssetCategoryLink {
    +id: number
    +assetId: number
    +categoryId: number
  }

  %% =========================
  %% КРИПТО-ДАННЫЕ (crypto-data-worker)
  %% =========================
  class CryptoAssetData {
    +id: number
    +ticker: string
    +name: string
    +description?: string
    +slug?: string
    +logoUrl?: string
    +logoUrlLocal?: string
    +source?: string
    +rank?: number
    +currentPriceUsd?: number
    +marketCapUsd?: number
    +fdvUsd?: number
    +circulatingSupply?: number
    +totalSupply?: number
    +maxSupply: string|null
    +contractAddress?: string|null
    +volume24HUsd?: number
    +change1HUsdPct?: number
    +change24HUsdPct?: number
    +change7DUsdPct?: number
    +change14DUsdPct?: number
    +change30DUsdPct?: number
    +change1YUsdPct?: number
    +sparkline7D?: any
    +lastUpdatedAt?: Date
    +assetId: number  %% FK -> assets.id (unique)
  }

  class CryptoChartsData {
    +id: number
    +assetDataId: number  %% FK -> crypto_assets.id (unique)
    +capturedAt?: Date
    +h24Stats?: Array<[number,number]>
    +h24Volumes?: Array<[number,number]>
    +d7Stats?: Array<[number,number]>
    +d7Volumes?: Array<[number,number]>
    +d30Stats?: Array<[number,number]>
    +d30Volumes?: Array<[number,number]>
    +d90Stats?: Array<[number,number]>
    +d90Volumes?: Array<[number,number]>
    +d365Stats?: Array<[number,number]>
    +d365Volumes?: Array<[number,number]>
    +maxStats?: Array<[number,number]>
    +maxVolumes?: Array<[number,number]>
  }

  class CryptoCandle {
    +id: number
    +assetId: number %% FK -> crypto_assets.id
    +interval: CandleInterval
    +openTime: Date
    +open: string
    +high: string
    +low: string
    +close: string
    +volume?: string
    +marketCapUsd?: string
  }

  class RecentSearch {
    +id: number
    +userId: number
    +assetId: number
    +createdAt: Date
  }

  %% =========================
  %% ENUM'Ы
  %% =========================
  class AssetType {
    <<enumeration>>
    CRYPTO
    STOCK
    BOND
    FIAT
  }

  class PortfolioType {
    <<enumeration>>
    CRYPTO
    STOCK
    BOND
  }

  class TransactionType {
    <<enumeration>>
    BUY
    SELL
  }

  class CandleInterval {
    <<enumeration>>
    MIN1
    MIN5
    MIN15
    H1
    H4
    D1
    W1
    M1
  }

  %% =========================
  %% СВЯЗИ
  %% =========================

  %% Users <-> Roles (many-to-many)
  User "1" -- "0..*" UserRoles : userRoles
  Role "1" -- "0..*" UserRoles : roleUsers

  %% Tokens (1 user -> 0..1 token row, так как userId unique)
  User "1" o-- "0..1" RefreshToken : refreshToken
  User "1" o-- "0..1" ResetToken : resetToken

  %% Favorites / Recent searches
  User "1" o-- "0..*" FavoriteAsset : favorites
  Asset "1" o-- "0..*" FavoriteAsset : favoredBy

  User "1" o-- "0..*" RecentSearch : recentSearches
  Asset "1" o-- "0..*" RecentSearch : searchedAssets

  %% Portfolios
  User "1" o-- "0..*" Portfolio : portfolios
  Portfolio "1" o-- "0..*" PortfolioAssets : holdings
  Asset "1" -- "0..*" PortfolioAssets : inPortfolios

  Portfolio "1" o-- "0..*" Transaction : transactions
  Asset "1" -- "0..*" Transaction : transactionAsset

  %% Asset <-> Crypto snapshot (1:1)
  Asset "1" o-- "0..1" CryptoAssetData : cryptoSnapshot

  %% Charts and candles
  CryptoAssetData "1" o-- "0..1" CryptoChartsData : charts
  CryptoAssetData "1" o-- "0..*" CryptoCandle : candles

  %% Categories (вынесены в отдельную сущность)
  Asset "1" -- "0..*" AssetCategoryLink : categoryLinks
  AssetCategory "1" -- "0..*" AssetCategoryLink : assets
```
Диаграмма показывает структуру данных платформы аналитики портфелей: пользователей и их роли, портфели и состав портфелей (активы с количеством и средней ценой), а также историю транзакций. Типы активов/портфелей/транзакций вынесены в перечисления, чтобы явно фиксировать допустимые значения.

```mermaid
classDiagram
    class Frontend {
        <<component>>
        +Запросы API
    }
    class APIGateway {
        <<component>>
        +Обрабатывает запросы
        +Маршрутизация
    }
    class AuthMicroservice {
        <<component>>
        +Аутентификация
        +Управление ролями
    }
    class PortfolioMicroservice {
        <<component>>
        +Управление портфелями
        +Транзакции
    }
    class CryptoDataWorker {
        <<component>>
        +Сбор данных криптовалют
    }
    class PostgreSQL {
        <<component>>
        +Хранение данных
    }
    class ExternalAPI {
        <<component>>
        +Данные о криптовалютах
    }

    Frontend --> APIGateway : Запросы
    APIGateway --> AuthMicroservice : Запросы на аутентификацию
    APIGateway --> PortfolioMicroservice : Запросы на управление портфелем
    APIGateway --> CryptoDataWorker : Запросы на сбор данных
    APIGateway --> PostgreSQL : Запросы на базу данных
    CryptoDataWorker --> ExternalAPI : Получение данных о криптовалютах
    PostgreSQL --> AuthMicroservice : Хранение данных пользователей
    PostgreSQL --> PortfolioMicroservice : Хранение портфелей

```
Диаграмма компонентов показывает ключевые компоненты системы "Wealthify" и их взаимодействие. Она включает компоненты фронтенда, API-шлюз, микросервисы для аутентификации, управления портфелями, сбора данных о криптовалютах и базу данных PostgreSQL. Взаимодействие между компонентами представлено через запросы API и данные, передаваемые между сервисами.

```mermaid
stateDiagram-v2
    [*] --> FillForm

    FillForm        : Заполнение формы
    Validate        : Валидация данных
    DataCorrect     : Данные корректны
    CreatePortfolio : Создание портфеля
    ShowError       : Отображение ошибки
    Success         : Успешное создание

    FillForm --> Validate
    Validate --> DataCorrect
    DataCorrect --> CreatePortfolio
    DataCorrect --> ShowError
    CreatePortfolio --> Success
    ShowError --> [*]
    Success --> [*]

```
Диаграмма активности описывает процесс создания инвестиционного портфеля. Сначала пользователь заполняет форму, затем система проверяет корректность введенных данных. В случае успешной валидации создается новый портфель, а при ошибке выводится сообщение об ошибке.

```mermaid
sequenceDiagram
    actor User
    participant Frontend
    participant APIGateway
    participant PortfolioService
    participant Database

    User->>Frontend: Заполнение формы
    Frontend->>APIGateway: POST /portfolio/create
    APIGateway->>PortfolioService: Создание портфеля
    PortfolioService->>Database: Обновление базы данных
    Database-->>PortfolioService: Успешная запись
    PortfolioService-->>APIGateway: Ответ об успехе
    APIGateway-->>Frontend: Уведомление об успехе
    Frontend-->>User: Портфель создан

```
Диаграмма последовательности демонстрирует взаимодействие между пользователем, фронтендом, API-шлюзом, сервисом портфелей и базой данных при создании портфеля. Каждый этап операции отображает запросы и ответы между компонентами, включая успешное завершение операции.

```mermaid
classDiagram
    class User {
        +id : PK
        +username : string
        +email : string
        +password : string
        +createdAt : Date
        +updatedAt : Date
    }

    class Portfolio {
        +id : PK
        +userId : FK
        +name : string
        +type : PortfolioType
        +createdAt : Date
        +updatedAt : Date
    }

    class Asset {
        +id : PK
        +name : string
        +ticker : string
        +type : AssetType
        +createdAt : Date
        +updatedAt : Date
    }

    class Transaction {
        +id : PK
        +portfolioId : FK
        +assetId : FK
        +quantity : float
        +pricePerUnit : float
        +type : TransactionType
        +date : Date
    }

    User "1" --> "0..*" Portfolio : имеет
    Portfolio "1" --> "0..*" Transaction : включает
    Portfolio "1" --> "0..*" Asset : содержит
    Asset "1" --> "0..*" Transaction : участвует в

```
Диаграмма классов описывает основные сущности системы: пользователи, портфели, активы и транзакции. Она показывает связи между объектами, такие как отношения один ко многим между пользователем и портфелем, а также многие ко многим между активами и портфелями через транзакции.

```mermaid
stateDiagram-v2
    [*] --> NotLoggedIn
    NotLoggedIn --> LoggedIn : Ввод правильных данных
    LoggedIn --> NotLoggedIn : Ввод неправильных данных
    LoggedIn --> LoggedOut : Выход
    LoggedOut --> NotLoggedIn : Вход с правильными данными
```

Диаграмма состояний представляет процесс авторизации пользователя в системе. Она описывает различные состояния пользователя (неавторизованный, авторизованный, вышедший) и переходы между ними, основанные на правильности введенных данных.

```mermaid
sequenceDiagram
    title CoinGecko парсинг — ультра-коротко (ключевые моменты)

    participant Cron as Cron
    participant Scraper as Scraper
    participant CG as CoinGecko
    participant Pool as WorkerPool\n(1 worker = 1 proxy)
    participant DB as Postgres\n(tx)

    Cron->>Scraper: tick
    alt isRunning
        Scraper-->>Cron: skip
    else run
        Scraper->>CG: collect links\n(+ load_more_coins)
        Scraper->>Scraper: batches = split(links,\nBATCH_SIZE = 2 * proxies)

        loop each batch
            Scraper->>Pool: dispatch chunks\n(~2 links per worker)
            Pool->>CG: parse in parallel\n(Puppeteer + proxy + retry)
            Pool-->>Scraper: parsed CryptoData[]
            Scraper->>DB: upsert batch\n(atomic transaction)
        end
    end
```
Парсинг CoinGecko — диаграмма последовательности, показывающая краткий поток работы `crypto-data-worker`: cron запускает сбор, сервис не допускает параллельных запусков через `isRunning`, затем собирает ссылки на активы (с догрузкой), режет их на батчи размера `2 * proxies`, раздаёт чанки воркерам (1 воркер = 1 proxy, ~2 ссылки на воркер), воркеры параллельно парсят страницы через Puppeteer с ретраями и возвращают `CryptoData[]`, после чего батч сохраняется в Postgres через upsert в транзакции, чтобы не было частичных обновлений данных.


```mermaid
sequenceDiagram
    title Логотипы CoinGecko — скачивание + раздача статики

    participant Cron as CryptoLogosCron\n(every 10m)
    participant Logos as CryptoLogosService
    participant DB as Postgres\n(CryptoAssetData)
    participant CG as CoinGecko CDN
    participant Store as LogoStorageService
    participant FS as FileSystem\n(storage/logos)
    participant Client as Frontend
    participant Static as Static server\n(/static/logos/*)

    Cron->>Logos: backfillMissingLogos(50)
    Logos->>DB: SELECT assets\nWHERE logoUrlLocal IS NULL\nAND logoUrl IS NOT NULL\nLIMIT 50 ORDER BY id ASC
    DB-->>Logos: CryptoAssetData[]

    loop for each assetData
        Logos->>Logos: ensureLocalLogo(assetData)

        alt logoUrlLocal уже есть
            Logos-->>Logos: return (skip)
        else logoUrl отсутствует
            Logos-->>Logos: return (skip)
        else нужно скачать
            Logos->>CG: GET assetData.logoUrl\n(fetch + AbortController 10s)

            alt HTTP 200..299
                CG-->>Logos: image bytes + content-type
                Logos->>Logos: detectExtension(content-type)
                Logos->>Store: saveCryptoAssetLogo(assetId, buffer, ext)
                Store->>FS: mkdir (if needed)\n+ writeFile(assetId.ext)
                FS-->>Store: OK
                Store-->>Logos: publicUrl\n(/static/logos/crypto/assetId.ext)
                Logos->>DB: UPDATE CryptoAssetData\nSET logoUrlLocal = publicUrl
                DB-->>Logos: OK
            else HTTP error / timeout
                CG-->>Logos: error
                Logos-->>Logos: warn + skip\n(logoUrlLocal остаётся NULL)
            end
        end
    end

    Note over Client,DB: В карточках активов клиент получает logoUrlLocal из БД\n(эти записи приходят от парсера, а cron только “докачивает” локальные файлы)

    Client->>Static: GET /static/logos/crypto/{assetId}.{ext}
    Static->>FS: readFile(assetId.ext)
    FS-->>Static: bytes
    Static-->>Client: 200 image/*
```
Cron раз в 10 минут выбирает из БД активы, у которых уже есть внешний logoUrl (пришёл от парсера), но ещё нет logoUrlLocal. Для каждого такого актива логотип скачивается с CoinGecko с таймаутом 10 секунд, сохраняется в файловое хранилище, а в БД записывается публичный путь /static/logos/.... Клиент затем запрашивает логотип по этому пути, и сервер отдаёт файл как статику.
