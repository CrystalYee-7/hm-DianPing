# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**hm-dianping** (黑马点评) is a Spring Boot learning project simulating a local shop review and flash-sale platform (similar to Dianping). It uses Spring Boot 2.3.12, MyBatis-Plus 3.4.3, MySQL, and Redis with Java 8.

## Build & Run Commands

```bash
# Build
mvn clean compile

# Run the application (starts on port 8081)
mvn spring-boot:run

# Run all tests
mvn test

# Run a single test class
mvn test -Dtest=HmDianPingApplicationTests
```

## Local Infrastructure

- **MySQL**: port 3307, database `hmdp`, credentials root/299415 (schema at `src/main/resources/db/hmdp.sql`)
- **Redis**: port 6379, password 299415
- **App**: port 8081

## Architecture

### Layered Structure (standard Spring Boot)

```
controller → service (interface) → service/impl → mapper (MyBatis-Plus BaseMapper)
```

### Key Patterns

**Redis caching**: The project uses `StringRedisTemplate` throughout for caching. Key services (Shop, ShopType) implement cache-aside: check Redis first, on miss query DB and populate cache. Null values are cached (with short TTL) to prevent cache penetration. Cache keys/prefixes are centralized in `RedisConstants`.

**Authentication**: Token-based via Redis. A `RefreshTokenInterceptor` (runs on all paths, order 0) reads the `authorization` header, looks up the token in Redis as a hash, hydrates a `UserDTO`, and stores it in `UserHolder` (a ThreadLocal). The `LoginInterceptor` (order 1) guards protected paths by checking if `UserHolder.getUser()` is null, returning 401 if so. Public paths are excluded in `MvcConfig`.

**Response format**: All API responses are wrapped in `Result` (success, errorMsg, data, total). Static factory methods `Result.ok()` and `Result.fail()` are used.

**Pagination**: MyBatis-Plus pagination plugin is configured (`MybatisConfig`). Controllers use `Page` objects with `SystemConstants.DEFAULT_PAGE_SIZE` (5) or `MAX_PAGE_SIZE` (10).

**Custom SQL**: When MyBatis-Plus query wrappers aren't enough, custom SQL goes in `src/main/resources/mapper/*.xml` (example: `VoucherMapper.xml` for a JOIN query).

**Global exception handling**: `WebExceptionAdvice` catches all `RuntimeException` and returns `Result.fail("服务器异常")`.

### Dependencies Worth Knowing

- **Hutool** (`cn.hutool`): Used heavily for JSON (`JSONUtil`), bean copying (`BeanUtil`), string validation (`StrUtil`), and random generation
- **Lombok**: `@Data`, `@Slf4j`, `@NoArgsConstructor`, `@AllArgsConstructor`
- **MyBatis-Plus**: Services extend `ServiceImpl<M, E>`, mappers extend `BaseMapper<E>`. Query via `query().eq(...).page(...)` style fluent API

### Domain Entities

- **Shop / ShopType**: Shop listing and categorization
- **Voucher / SeckillVoucher**: Coupons and flash-sale vouchers (linked via voucher_id)
- **VoucherOrder**: Flash-sale orders
- **Blog / BlogComments**: User-posted content and comments
- **User / UserInfo**: User accounts (phone-based login) and profiles
- **Follow**: User follow relationships

### Redis Key Namespace

All keys use constants from `RedisConstants`:
- `login:code:{phone}` — SMS verification code (TTL: 2 min)
- `login:token:{token}` — User session hash (TTL: 36000 min)
- `cache:shop:{id}` — Shop cache (TTL: 30 min, null cache: 2 min)
- `cache:shoptype:list:` — Shop type list cache
- `seckill:stock:{voucherId}` — Flash-sale stock counter
- `blog:liked:{blogId}` — Blog like tracking
- `feed:{userId}` — User feed
- `shop:geo:{shopId}` — Shop geo coordinates
- `sign:{userId}` — User sign-in tracking
