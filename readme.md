## 📘 Описание UML-диаграмм

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

    actor Пользователь as User
    participant Client as "Frontend / Client"
    participant Controller as "auth.controller.ts"
    participant Service as "auth.service.ts"
    participant DB as "База данных"

    %% Проверка доступа с access token
    User->>Client: HTTP ONLY COOKIE: access token
    Client->>Controller: запрос к защищённому эндпоинту
    Controller->>Service: проверка валидности access token
    Service->>Service: проверка срока действия токена

    alt access token истёк
        Service-->>Controller: ошибка ИСТЕК_АКСЕСС_ТОКЕН
        Controller-->>Client: status code = ИСТЕК_АКСЕСС_ТОКЕН
        Client->>Controller: запрос на /refresh
    else access token валиден
        Service-->>Controller: токен действителен
        Controller-->>Client: status code = OK
    end

    %% Обновление токенов (/refresh)
    Client->>Controller: POST /refresh\nHTTP ONLY COOKIE: refresh token
    Controller->>Service: проверка refresh token (валиден? не истёк?)
    Service->>DB: поиск refresh token
    DB-->>Service: найден / не найден

    alt refresh token валиден
        Service->>Service: генерация новых refresh + access токенов\n(обновлённые даты)
        Service-->>Controller: новые токены
        Controller-->>Client: Cookies: refreshToken + accessToken
        Note right of Controller: Всё ок — клиент получает новые токены
    else refresh token не найден или просрочен
        Service-->>Controller: ошибка авторизации
        Controller-->>Client: перенаправление на sign-in / sign-up
    end

    %% Повторная проверка
    Client->>Controller: повторный запрос с новым access token
    Controller->>Service: проверка токена
    Service-->>Controller: статус OK
    Controller-->>Client: статус OK, запрос выполнен
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
  %% КЛАССЫ ДОМЕНА
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

  class CryptoAsset {
    +id: number
    +ticker: string
    +name: string
    +description?: string
    +slug?: string
    +logoUrl?: string
    +websiteUrl?: string
    +sector?: string
    +source?: string
    +isActive?: boolean
    +priceUsd?: string
    +marketCapUsd?: string
    +fdvUsd?: string
    +dominance?: string
    +volume24hUsd?: string
    +volume24hNative?: string
    +circulatingSupply?: string
    +totalSupply?: string
    +maxSupply?: string
    +rank?: number
    +marketCapRank?: number
    +sparkline7d?: any
    +lastUpdatedAt?: Date
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
  %% ENUMЫ (как в TS)
  %% =========================
  class AssetType {
    <<enumeration>>
    CRYPTO = "Crypto"
    BOND   = "Bond"
    STOCK  = "Stock"
    FIAT   = "Fiat"
  }

  class PortfolioType {
    <<enumeration>>
    CRYPTO = "Crypto"
    STOCK  = "Stock"
    BOND   = "Bond"
  }

  class TransactionType {
    <<enumeration>>
    BUY  = "BUY"
    SELL = "SELL"
  }

  class CandleInterval {
    <<enumeration>>
    MIN1  = "1m"
    MIN5  = "5m"
    MIN15 = "15m"
    H1    = "1h"
    H4    = "4h"
    D1    = "1d"
    W1    = "1w"
    M1    = "1mo"
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

  CryptoAsset "1" o-- "0..*" CryptoCandle : candles

```
Диаграмма показывает структуру данных платформы аналитики портфелей: пользователей и их роли, портфели и состав портфелей (активы с количеством и средней ценой), а также историю транзакций. Типы активов/портфелей/транзакций вынесены в перечисления, чтобы явно фиксировать допустимые значения.