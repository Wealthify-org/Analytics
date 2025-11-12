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
  %% ДОМЕННЫЕ КЛАССЫ
  %% =========================
  class User {
    +id: number
    +username: string
    +email: string
    +password: string
    +createdAt: Date
    +updatedAt: Date
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
    +userId: number
    +token: string
    +expiryDate: Date
  }

  class ResetToken {
    +id: number
    +userId: number
    +token: string
    +expiryDate: Date
  }

  class Portfolio {
    +id: number
    +userId: number
    +name: string
    +type: PortfolioType
    +createdAt: Date
    +updatedAt: Date
  }

  class Asset {
    +id: number
    +ticker: string
    +name: string
    +type: AssetType
    +createdAt: Date
    +updatedAt: Date
  }

  class PortfolioAssets {
    +id: number
    +portfolioId: number
    +assetId: number
    +quantity: number
    +averageBuyPrice: number
    +purchaseDate: Date
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

  %% ======== КРИПТО-ДАННЫЕ ========
  class CryptoAssetData {
    +id: number
    +assetId: number
    +ticker: string
    +name: string
    +description?: string
    +slug?: string
    +logoUrl?: string
    +categories?: string
    +websiteUrl?: string
    +source?: string
    +isActive?: boolean
    +priceUsd?: number|string
    +marketCapUsd?: number|string
    +fdvUsd?: number|string
    +dominance?: number|string
    +volume24HUsd?: number
    +circulatingSupply?: number
    +totalSupply?: number
    +maxSupply?: string|null
    +rank?: number
    +marketCapRank?: number
    +sparkline7d?: any
    +lastUpdatedAt?: Date
  }

  class CryptoChartsData {
    +id: number
    +assetDataId: number
    +capturedAt?: Date
    %% пары [timestamp, value]
    +h24Stats?: Array<[number,number]>
    +h24Volumes?: Array<[number,number]>
    +d7Stats?: Array<[number,number]>
    +d7Volumes?: Array<[number,number]>
    +d30Stats?: Array<[number,number]>
    +d30Volumes?: Array<[number,number]>
    +d90Stats?: Array<[number,number]>
    +d90Volumes?: Array<[number,number]>
    +d180Stats?: Array<[number,number]>
    +d180Volumes?: Array<[number,number]>
    +y1Stats?: Array<[number,number]>
    +y1Volumes?: Array<[number,number]>
    +maxStats?: Array<[number,number]>
    +maxVolumes?: Array<[number,number]>
  }

  class CryptoCandle {
    +id: number
    +assetId: number
    +interval: CandleInterval
    +openTime: Date
    +closeTime: Date
    +open: string
    +high: string
    +low: string
    +close: string
    +volume?: string
    +marketCapUsd?: string
  }

  %% =========================
  %% ENUM'Ы
  %% =========================
  class AssetType {
    <<enumeration>>
    CRYPTO
    BOND
    STOCK
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
  User "1" -- "0..*" UserRoles : userRoles
  Role "1" -- "0..*" UserRoles : roleUsers

  User "1" o-- "0..*" Portfolio : portfolios
  User "1" o-- "0..1" RefreshToken : refreshToken
  User "1" o-- "0..1" ResetToken : resetToken

  Portfolio "1" o-- "0..*" PortfolioAssets : holdings
  Asset "1" -- "0..*" PortfolioAssets : inPortfolios

  Portfolio "1" o-- "0..*" Transaction : transactions
  Asset "1" -- "0..*" Transaction : transactionAsset

  %% Asset ↔ данные рынка
  Asset "1" o-- "0..1" CryptoAssetData : assetData
  CryptoAssetData "1" o-- "0..1" CryptoChartsData : charts
  CryptoAssetData "1" o-- "0..*" CryptoCandle : candles
```
Диаграмма показывает структуру данных платформы аналитики портфелей: пользователей и их роли, портфели и состав портфелей (активы с количеством и средней ценой), а также историю транзакций. Типы активов/портфелей/транзакций вынесены в перечисления, чтобы явно фиксировать допустимые значения.