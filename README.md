# 订阅管理系统后端架构方案文档

## 概述

本项目旨在为现有的纯前端订阅管理系统提供一个健壮的后端支持，实现用户账户管理、订阅数据的持久化存储与管理、批量操作以及重要的数据备份与恢复功能。通过引入后端服务，系统将不再依赖浏览器本地存储，提高数据的安全性和可用性，并支持跨设备访问。

选定技术栈:
*   后端语言: Java
*   后端框架: Spring Boot
*   数据库: MySQL

系统将采用微服务或单体应用结构，初期以单体应用快速实现核心功能，为未来扩展预留可能。

## API 设计

系统将采用 RESTful API 风格，通过标准的 HTTP 方法（GET, POST, PUT, DELETE）对资源进行操作。核心资源主要包括用户（Users）和订阅（Subscriptions）。

核心资源:
*   `users`: 用户账户信息管理。
*   `subscriptions`: 用户订阅记录管理。

关键 API 端点示例:

| 功能                 | HTTP 方法 | 端点                               | 描述                           |
| :------------------- | :-------- | :--------------------------------- | :----------------------------- |
| 用户注册             | `POST`    | `/api/users/register`              | 创建新用户账户                 |
| 用户登录             | `POST`    | `/api/users/login`                 | 用户身份验证并返回认证令牌     |
| 获取当前用户订阅列表 | `GET`     | `/api/subscriptions`               | 获取当前登录用户的所有订阅     |
| 添加单个订阅         | `POST`    | `/api/subscriptions`               | 为当前登录用户添加一个新订阅   |
| 更新单个订阅         | `PUT`     | `/api/subscriptions/{id}`          | 更新指定订阅的信息             |
| 删除单个订阅         | `DELETE`  | `/api/subscriptions/{id}`          | 删除指定订阅                   |
| 批量添加订阅         | `POST`    | `/api/subscriptions/batch`         | 为当前登录用户批量添加订阅     |
| 批量删除订阅         | `DELETE`  | `/api/subscriptions/batch`         | 为当前登录用户批量删除指定订阅 |
| 设置订阅意向（批量） | `PUT`     | `/api/subscriptions/batch/intention` | 批量设置订阅的续订/取消意向    |
| 数据备份（手动触发） | `POST`    | `/api/data/backup`                 | 手动触发数据备份               |
| 数据恢复（手动触发） | `POST`    | `/api/data/restore`                | 手动触发数据恢复               |

*API 版本控制 (例如 `/api/v1/`) 将在后续详细设计中考虑。*

## 数据库设计

采用 MySQL 关系型数据库存储用户和订阅数据。

实体关系图 (概念)

```mermaid
erDiagram
    users {
        INT id PK
        VARCHAR username UK
        VARCHAR password_hash
        VARCHAR email
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    subscriptions {
        INT id PK
        INT user_id FK
        VARCHAR name
        VARCHAR category
        VARCHAR billing_cycle
        DATE start_date
        DECIMAL amount
        VARCHAR currency
        VARCHAR notes
        VARCHAR intention ENUM("renew", "cancel")
        JSON associated_accounts
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    users ||--o{ subscriptions : "has"
```

*注: 上图是一个概念性的ER图，表示用户与订阅之间的一对多关系。一个用户可以有多个订阅，一个订阅只属于一个用户。*

表结构描述:

1.  `users` 表:
    存储系统用户的基本信息。

    | 字段          | 数据类型    | 约束                  | 描述             |
    | :------------ | :---------- | :-------------------- | :--------------- |
    | `id`          | `BIGINT`    | `PRIMARY KEY`, `AUTO_INCREMENT` | 用户唯一标识符     |
    | `username`    | `VARCHAR(50)` | `NOT NULL`, `UNIQUE`  | 用户名，用于登录 |
    | `password_hash` | `VARCHAR(255)`| `NOT NULL`            | 存储用户密码的哈希值 |
    | `email`       | `VARCHAR(100)`| `UNIQUE` (可选)       | 用户邮箱，可选或用于找回密码 |
    | `created_at`  | `TIMESTAMP` | `NOT NULL`, `DEFAULT CURRENT_TIMESTAMP` | 用户创建时间     |
    | `updated_at`  | `TIMESTAMP` | `NOT NULL`, `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` | 用户信息最后更新时间 |

2.  `subscriptions` 表:
    存储用户的订阅详细信息。字段设计参考了前端的数据结构，并添加了用户关联和意向字段。

    | 字段              | 数据类型      | 约束                      | 描述                 |
    | :---------------- | :------------ | :------------------------ | :------------------- |
    | `id`              | `BIGINT`      | `PRIMARY KEY`, `AUTO_INCREMENT` | 订阅记录唯一标识符     |
    | `user_id`         | `BIGINT`      | `NOT NULL`, `FOREIGN KEY` | 关联的用户 ID        |
    | `name`            | `VARCHAR(100)`| `NOT NULL`                | 订阅名称（如：Netflix） |
    | `category`        | `VARCHAR(50)` |                           | 订阅类别（如：娱乐）   |
    | `billing_cycle`   | `VARCHAR(20)` |                           | 计费周期（如：月付, 年付） |
    | `start_date`      | `DATE`        |                           | 订阅开始日期         |
    | `amount`          | `DECIMAL(10, 2)` | `NOT NULL` (如果收费)     | 订阅金额             |
    | `currency`        | `VARCHAR(10)` |                           | 货币单位（如：USD, CNY） |
    | `notes`           | `TEXT`        |                           | 订阅备注或说明       |
    | `intention`       | `ENUM('renew', 'cancel')` | `DEFAULT NULL`        | 用户对该订阅的意向     |
    | `associated_accounts` | `JSON`        |                           | 关联账户信息 (存储为 JSON 字符串) |
    | `created_at`      | `TIMESTAMP`   | `NOT NULL`, `DEFAULT CURRENT_TIMESTAMP` | 订阅创建时间         |
    | `updated_at`      | `TIMESTAMP`   | `NOT NULL`, `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` | 订阅信息最后更新时间 |

*注: `associated_accounts` 字段在前端允许动态增删账户，后端可以考虑将其存储为 JSON 类型，或者如果结构复杂且标准化，则考虑单独的关联表。此处选择 JSON 简化初期设计。*

## 模块划分

后端服务可以初步划分为以下几个核心模块：

1.  用户认证模块 (Authentication Module):
    *   职责：处理用户注册、登录、密码管理等。
    *   关键功能：用户数据持久化 (`users` 表交互)、密码哈希与验证、生成和验证认证令牌 (如 JWT)。
    *   技术：Spring Security (用于认证授权) 是合适的选型。

    ```svg
    <svg width="400" height="150" xmlns="http://www.w3.org/2000/svg">
      <rect x="10" y="10" rx="10" ry="10" width="180" height="130" style="fill:#e0f2f7;stroke:#01579b;stroke-width:2;"/>
      <text x="100" y="35" font-family="Arial" font-size="16" fill="#01579b" text-anchor="middle" font-weight="bold">用户认证模块</text>
      <line x1="10" y1="50" x2="190" y2="50" style="stroke:#01579b;stroke-width:1;"/>
      <text x="20" y="70" font-family="Arial" font-size="12" fill="#01579b">- 用户注册/登录</text>
      <text x="20" y="90" font-family="Arial" font-size="12" fill="#01579b">- 密码管理</text>
      <text x="20" y="110" font-family="Arial" font-size="12" fill="#01579b">- 认证/授权</text>
      <text x="20" y="130" font-family="Arial" font-size="12" fill="#01579b">- JWT 管理</text>

      <rect x="210" y="10" rx="10" ry="10" width="180" height="130" style="fill:#fff3e0;stroke:#e65100;stroke-width:2;"/>
       <text x="300" y="35" font-family="Arial" font-size="16" fill="#e65100" text-anchor="middle" font-weight="bold">数据库</text>
       <line x1="210" y1="50" x2="390" y2="50" style="stroke:#e65100;stroke-width:1;"/>
       <text x="220" y="75" font-family="Arial" font-size="12" fill="#e65100">- users 表</text>
       <text x="220" y="95" font-family="Arial" font-size="12" fill="#e65100">- subscriptions 表</text>
       <path d="M190 70 L210 70" stroke="#01579b" stroke-width="1" marker-end="url(#arrow)"/>
       <defs>
         <marker id="arrow" markerWidth="10" markerHeight="10" refX="5" refY="5" orient="auto">
           <path d="M0 0 L10 5 L0 10 Z" fill="#01579b" />
         </marker>
       </defs>
    </svg>
    ```

2.  订阅管理模块 (Subscription Management Module):
    *   职责：处理订阅数据的增删改查、批量操作。
    *   关键功能：订阅数据持久化 (`subscriptions` 表交互)、数据验证、批量操作逻辑实现、数据过滤与分页。
    *   技术：Spring Data JPA (简化数据库操作)。

    ```svg
    <svg width="400" height="150" xmlns="http://www.w3.org/2000/svg">
      <rect x="10" y="10" rx="10" ry="10" width="180" height="130" style="fill:#e8f5e9;stroke:#1b5e20;stroke-width:2;"/>
      <text x="100" y="35" font-family="Arial" font-size="16" fill="#1b5e20" text-anchor="middle" font-weight="bold">订阅管理模块</text>
      <line x1="10" y1="50" x2="190" y2="50" style="stroke:#1b5e20;stroke-width:1;"/>
      <text x="20" y="70" font-family="Arial" font-size="12" fill="#1b5e20">- 订阅 CRUD</text>
      <text x="20" y="90" font-family="Arial" font-size="12" fill="#1b5e20">- 批量操作</text>
      <text x="20" y="110" font-family="Arial" font-size="12" fill="#1b5e20">- 数据验证</text>
      <text x="20" y="130" font-family="Arial" font-size="12" fill="#1b5e20">- 过滤/分页</text>

      <rect x="210" y="10" rx="10" ry="10" width="180" height="130" style="fill:#fff3e0;stroke:#e65100;stroke-width:2;"/>
       <text x="300" y="35" font-family="Arial" font-size="16" fill="#e65100" text-anchor="middle" font-weight="bold">数据库</text>
       <line x1="210" y1="50" x2="390" y2="50" style="stroke:#e65100;stroke-width:1;"/>
       <text x="220" y="75" font-family="Arial" font-size="12" fill="#e65100">- subscriptions 表</text>
       <path d="M190 70 L210 70" stroke="#1b5e20" stroke-width="1" marker-end="url(#arrow_green)"/>
       <defs>
         <marker id="arrow_green" markerWidth="10" markerHeight="10" refX="5" refY="5" orient="auto">
           <path d="M0 0 L10 5 L0 10 Z" fill="#1b5e20" />
         </marker>
       </defs>
    </svg>
    ```

3.  数据备份与恢复模块 (Data Backup & Recovery Module):
    *   职责：实现数据的自动备份和手动触发的备份/恢复功能。
    *   关键功能：定期导出数据库数据（例如为 SQL 文件或 JSON 文件），存储到安全位置；根据备份文件恢复数据库。可以考虑增量备份和全量备份策略。
    *   技术：可以利用数据库自带的工具 (如 `mysqldump`) 或通过编程方式读取数据并导出。需要考虑文件存储位置（本地文件系统、云存储等）和安全性。

    ```svg
    <svg width="400" height="150" xmlns="http://www.w3.org/2000/svg">
      <rect x="10" y="10" rx="10" ry="10" width="180" height="130" style="fill:#e1f5fe;stroke:#0277bd;stroke-width:2;"/>
      <text x="100" y="35" font-family="Arial" font-size="16" fill="#0277bd" text-anchor="middle" font-weight="bold">备份恢复模块</text>
      <line x1="10" y1="50" x2="190" y2="50" style="stroke:#0277bd;stroke-width:1;"/>
      <text x="20" y="70" font-family="Arial" font-size="12" fill="#0277bd">- 自动定时备份</text>
      <text x="20" y="90" font-family="Arial" font-size="12" fill="#0277bd">- 手动备份/恢复</text>
      <text x="20" y="110" font-family="Arial" font-size="12" fill="#0277bd">- 数据导出/导入</text>

      <rect x="210" y="10" rx="10" ry="10" width="180" height="130" style="fill:#fff3e0;stroke:#e65100;stroke-width:2;"/>
       <text x="300" y="35" font-family="Arial" font-size="16" fill="#e65100" text-anchor="middle" font-weight="bold">数据库</text>
       <line x1="210" y1="50" x2="390" y2="50" style="stroke:#e65100;stroke-width:1;"/>
       <text x="220" y="75" font-family="Arial" font-size="12" fill="#e65100">- 所有数据</text>
       <path d="M190 70 L210 70" stroke="#0277bd" stroke-width="1" marker-end="url(#arrow_blue)"/>
       <defs>
         <marker id="arrow_blue" markerWidth="10" markerHeight="10" refX="5" refY="5" orient="auto">
           <path d="M0 0 L10 5 L0 10 Z" fill="#0277bd" />
         </marker>
       </defs>
    </svg>
    ```

## 技术选型理由

*   Java:
    *   成熟稳定: Java 语言及生态系统成熟稳定，适合构建企业级应用。
    *   跨平台性: JVM 的特性保证了应用程序在不同操作系统上的兼容性。
    *   丰富的库和社区支持: 拥有庞大的开发者社区和海量第三方库，解决问题资源丰富。
    *   性能良好: JVM 经过多年优化，运行时性能优异。

*   Spring Boot:
    *   简化开发: 大大简化了 Spring 应用程序的搭建和开发过程，通过“约定大于配置”的原则，快速构建可独立的、生产级的应用程序。
    *   集成能力强: 易于集成 Spring 生态系统中的其他项目 (如 Spring Security, Spring Data JPA) 和第三方库。
    *   内嵌服务器: 可以直接打包成可执行 JAR 文件，包含内嵌的 Tomcat/Jetty/Undertow 服务器，简化部署。
    *   微服务友好: 虽然本项目初期是单体，但 Spring Boot 天然支持构建微服务，为未来架构演进提供便利。

*   MySQL:
    *   普及度高: 是最流行的关系型数据库之一，资料丰富，易于部署和管理。
    *   稳定可靠: 适用于大多数Web应用的存储需求。
    *   性能不错: 对于中小型应用，MySQL 提供了良好的读写性能。
    *   成熟的驱动和集成: 与 Java 和 Spring Boot 的集成非常成熟稳定，使用方便。
    *   符合关系型数据特性: 用户和订阅之间的数据结构天然适合用关系型数据库建模。

这些技术栈组合能够快速有效地实现用户认证、订阅管理等核心后端功能，同时保证系统的稳定性和未来的可扩展性。Spring Boot 的简化开发特性将有助于加速开发进度，而 MySQL 提供了可靠的数据持久化能力。数据备份与恢复功能可以通过结合操作系统层面的数据库工具和应用程序逻辑来实现。
