# Rate Limiting (NestJS)

**Source:** https://docs.nestjs.com/security/rate-limiting

Brute-force attack থেকে application কে protect করার জন্য একটা common technique হলো **rate-limiting**। এর জন্য `@nestjs/throttler` package লাগবে।

---

## 1. Installation

```bash
npm i --save @nestjs/throttler
```

---

## 2. Basic Configuration

Installation শেষ হলে অন্য যেকোনো Nest package এর মতোই `ThrottlerModule` কে `forRoot` অথবা `forRootAsync` method দিয়ে configure করা যায়।

```typescript
@Module({
  imports: [
    ThrottlerModule.forRoot({
      throttlers: [
        {
          ttl: 60000,
          limit: 10,
        },
      ],
    }),
  ],
})
export class AppModule {}
```

উপরের configuration টা পুরো application এর guarded routes এর জন্য global option সেট করে দেয়:

- **`ttl`** → time to live, milliseconds এ।
- **`limit`** → ওই `ttl` এর মধ্যে সর্বোচ্চ কতগুলো request allowed হবে।

Module import করার পর `ThrottlerGuard` কীভাবে bind করবে সেটা তোমার choice — [guards](https://docs.nestjs.com/guards) section এ যেভাবে বলা আছে, সেভাবেই bind করা যায়। Globally bind করতে চাইলে যেকোনো module এ এই provider টা add করো:

```typescript
{
  provide: APP_GUARD,
  useClass: ThrottlerGuard,
}
```

---

## 3. Multiple Throttler Definitions

কখনো কখনো একসাথে একাধিক throttling rule দরকার হতে পারে — যেমন, প্রতি সেকেন্ডে ৩টার বেশি call না, ১০ সেকেন্ডে ২০টার বেশি না, আর ১ মিনিটে ১০০টার বেশি না। এই ধরনের ক্ষেত্রে named options দিয়ে array এ definition গুলো সেট করা যায়, যেগুলো পরে `@SkipThrottle()` এবং `@Throttle()` decorator এ reference করা যাবে।

```typescript
@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,
        limit: 3,
      },
      {
        name: 'medium',
        ttl: 10000,
        limit: 20,
      },
      {
        name: 'long',
        ttl: 60000,
        limit: 100,
      },
    ]),
  ],
})
export class AppModule {}
```

---

## 4. Customization

### 4.1 `@SkipThrottle()`

Guard কে যদি controller অথবা globally bind করা থাকে, কিন্তু কোনো একটা বা একাধিক endpoint এর জন্য rate limiting disable করতে চাও, তাহলে `@SkipThrottle()` decorator ব্যবহার করতে হবে — পুরো class এর জন্য অথবা একটা single route এর জন্য।

`@SkipThrottle()` একটা object ও নিতে পারে (string key + boolean value) — যদি controller এর বেশিরভাগ route skip করতে চাও কিন্তু সবগুলো না, এবং একাধিক throttler set থাকলে per-throttler configure করতে চাও। Object না দিলে default হবে `{ default: true }`।

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {}
```

এই decorator দিয়ে একটা route/class skip করা যায়, অথবা একটা skipped class এর মধ্যে specific route এর skip টাকে negate করা যায়:

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {
  // এই route এ rate limiting apply হবে।
  @SkipThrottle({ default: false })
  dontSkip() {
    return 'List users work with Rate limiting.';
  }

  // এই route rate limiting skip করবে।
  doSkip() {
    return 'List users work without Rate limiting.';
  }
}
```

### 4.2 `@Throttle()`

`@Throttle()` decorator দিয়ে global module এ সেট করা `limit` আর `ttl` কে override করা যায় — tighter অথবা looser security option দেওয়ার জন্য। এটা class অথবা function, দুই জায়গাতেই ব্যবহার করা যায়।

Version 5 থেকে এই decorator একটা object নেয়, যেখানে key হলো throttler set এর নাম (নাম না দিলে `default` ব্যবহার করতে হবে), আর value হলো `limit` ও `ttl` key সহ একটা object:

```typescript
// Rate limiting এবং duration এর জন্য default configuration override করা হচ্ছে।
@Throttle({ default: { limit: 3, ttl: 60000 } })
@Get()
findAll() {
  return "List users works with custom rate limiting.";
}
```

---

## 5. Proxies

Application যদি একটা proxy server এর পেছনে চলে, তাহলে HTTP adapter কে সেই proxy trust করতে configure করা জরুরি। [Express](http://expressjs.com/en/guide/behind-proxies.html) আর [Fastify](https://www.fastify.io/docs/latest/Reference/Server/#trustproxy) এর নিজস্ব adapter option আছে `trust proxy` set করার জন্য।

Express adapter এ `trust proxy` enable করার example:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';
import { NestExpressApplication } from '@nestjs/platform-express';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);
  app.set('trust proxy', 'loopback'); // loopback address থেকে আসা request trust করবে
  await app.listen(3000);
}

await bootstrap();
```

`trust proxy` enable করলে `X-Forwarded-For` header থেকে original IP address পাওয়া যায়। চাইলে `getTracker()` method override করে সরাসরি এই header থেকে IP extract করার behavior customize করা যায়:

```typescript
import { ThrottlerGuard } from '@nestjs/throttler';
import { Injectable } from '@nestjs/common';

@Injectable()
export class ThrottlerBehindProxyGuard extends ThrottlerGuard {
  protected async getTracker(req: Record<string, any>): Promise<string> {
    return req.ips.length ? req.ips[0] : req.ip; // নিজের প্রয়োজন অনুযায়ী IP extraction customize করো
  }
}
```

> **Hint:** Express এর জন্য `req` Request object এর API [এখানে](https://expressjs.com/en/api.html#req.ips), আর Fastify এর জন্য [এখানে](https://www.fastify.io/docs/latest/Reference/Request/) পাবে।

---

## 6. WebSockets

এই module websocket এর সাথেও কাজ করে, তবে কিছু class extension লাগবে। `ThrottlerGuard` extend করে `handleRequest` method override করতে হবে:

```typescript
@Injectable()
export class WsThrottlerGuard extends ThrottlerGuard {
  async handleRequest(requestProps: ThrottlerRequest): Promise<boolean> {
    const {
      context,
      limit,
      ttl,
      throttler,
      blockDuration,
      getTracker,
      generateKey,
    } = requestProps;

    const client = context.switchToWs().getClient();
    const tracker = client._socket.remoteAddress;
    const key = generateKey(context, tracker, throttler.name);
    const { totalHits, timeToExpire, isBlocked, timeToBlockExpire } =
      await this.storageService.increment(
        key,
        ttl,
        limit,
        blockDuration,
        throttler.name,
      );

    const getThrottlerSuffix = (name: string) =>
      name === 'default' ? '' : `-${name}`;

    // User যদি তার limit এ পৌঁছে যায় তাহলে error throw করবে।
    if (isBlocked) {
      await this.throwThrottlingException(context, {
        limit,
        ttl,
        key,
        tracker,
        totalHits,
        timeToExpire,
        isBlocked,
        timeToBlockExpire,
      });
    }

    return true;
  }
}
```

> **Hint:** `ws` ব্যবহার করলে `_socket` এর জায়গায় `conn` ব্যবহার করতে হবে।
> `@nestjs/platform-ws` package ব্যবহার করলে `client._socket.remoteAddress` ব্যবহার করা যাবে।

**WebSocket নিয়ে কাজ করার সময় মনে রাখার বিষয়:**

- Guard কে `APP_GUARD` অথবা `app.useGlobalGuards()` দিয়ে register করা যাবে না।
- Limit reach হলে Nest একটা `exception` event emit করবে, তাই এর জন্য একটা listener রেডি রাখতে হবে।
- একাধিক throttler definition configure করা থাকলে, প্রতিটা throttler set এর জন্য `handleRequest()` আলাদাভাবে run হয়। তাই storage key generate করার সময় এবং `ThrottlerLimitDetail` report করার সময় `ThrottlerRequest` থেকে পাওয়া `throttler.name` ব্যবহার করতে হবে, যাতে প্রতিটা named throttler তার নিজের limit track করতে পারে।

---

## 7. GraphQL

`ThrottlerGuard` কে GraphQL request এর সাথেও কাজ করানো যায়। এবার guard extend করে `getRequestResponse` method override করতে হবে:

```typescript
@Injectable()
export class GqlThrottlerGuard extends ThrottlerGuard {
  getRequestResponse(context: ExecutionContext) {
    const gqlCtx = GqlExecutionContext.create(context);
    const ctx = gqlCtx.getContext();
    return { req: ctx.req, res: ctx.res };
  }
}
```

---

## 8. Configuration Options

`ThrottlerModule` এর options array এ যেসব field পাঠানো যায়:

| Option | বিবরণ |
|---|---|
| `name` | কোন throttler set ব্যবহার হচ্ছে সেটা internally track করার নাম। না দিলে default হয় `default`। |
| `ttl` | প্রতিটা request storage এ কতক্ষণ (milliseconds) থাকবে। |
| `limit` | ওই TTL এর মধ্যে সর্বোচ্চ কতগুলো request allowed। |
| `blockDuration` | Limit ছাড়িয়ে গেলে request কতক্ষণ (milliseconds) block থাকবে। |
| `ignoreUserAgents` | কোন কোন user-agent throttling থেকে বাদ দেওয়া হবে, তার regular expression array। |
| `skipIf` | `ExecutionContext` নিয়ে একটা `boolean` return করে এমন function — `@SkipThrottle()` এর মতোই কাজ করে, কিন্তু request-based। |

উপরের option গুলো global ভাবে প্রতিটা throttler set এ apply করতে চাইলে, বা custom storage সেট করতে চাইলে, এগুলো `throttlers` option key এর ভেতরে পাঠাতে হবে:

| Option | বিবরণ |
|---|---|
| `storage` | Throttling এর হিসাব কোথায় track হবে তার custom storage service। ([এখানে দেখো](#9-storage)) |
| `ignoreUserAgents` | কোন কোন user-agent throttling থেকে বাদ দেওয়া হবে, তার regular expression array। |
| `skipIf` | `ExecutionContext` নিয়ে একটা `boolean` return করে এমন function। |
| `throttlers` | উপরের table অনুযায়ী define করা throttler set গুলোর array। |
| `errorMessage` | Default throttler error message override করার জন্য একটা `string`, অথবা `ExecutionContext` ও `ThrottlerLimitDetail` নিয়ে একটা `string` return করে এমন function। |
| `getTracker` | `Request` নিয়ে একটা `string` return করে এমন function — default `getTracker` method এর logic override করার জন্য। |
| `generateKey` | `ExecutionContext`, tracker `string`, আর throttler name (`string`) নিয়ে একটা `string` return করে এমন function — rate limit value store করার জন্য যে final key ব্যবহার হবে, সেটা override করার জন্য। এটা default `generateKey` method এর logic override করে। |

---

## 9. Async Configuration

Rate-limiting configuration যদি synchronously না নিয়ে asynchronously নিতে চাও, তাহলে `forRootAsync()` method ব্যবহার করা যায় — এতে dependency injection এবং `async` method দুটোই সাপোর্ট করে।

### Factory function দিয়ে:

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => [
        {
          ttl: config.get('THROTTLE_TTL'),
          limit: config.get('THROTTLE_LIMIT'),
        },
      ],
    }),
  ],
})
export class AppModule {}
```

### `useClass` দিয়ে:

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      useClass: ThrottlerConfigService,
    }),
  ],
})
export class AppModule {}
```

এটা কাজ করবে যতক্ষণ `ThrottlerConfigService` ক্লাসটা `ThrottlerOptionsFactory` interface implement করছে।

---

## 10. Storage

Built-in storage হলো একটা in-memory cache, যেটা request গুলো track করে রাখে যতক্ষণ না global options এ সেট করা TTL পার হয়ে যায়। `ThrottlerModule` এর `storage` option এ নিজের storage বসানো যায়, যদি সেই class টা `ThrottlerStorage` interface implement করে।

Distributed server এর জন্য single source of truth রাখতে [Redis এর community storage provider](https://github.com/jmcdo29/nest-lab/tree/main/packages/throttler-storage-redis) ব্যবহার করা যায়।

> **Note:** `ThrottlerStorage` কে `@nestjs/throttler` থেকে import করা যায়।

---

## 11. Time Helpers

Timing গুলো readable করার জন্য কয়েকটা helper method আছে, direct millisecond লেখার বদলে। `@nestjs/throttler` package থেকে ৫টা helper export হয়: `seconds`, `minutes`, `hours`, `days`, আর `weeks`। এগুলো ব্যবহার করতে শুধু `seconds(5)` (বা অন্য যেকোনো helper) call করলেই সঠিক millisecond value পেয়ে যাবে।

---

## 12. Migration Guide (পুরোনো version থেকে)

- বেশিরভাগ ক্ষেত্রে তোমার options গুলোকে একটা array এর মধ্যে wrap করলেই যথেষ্ট।
- Custom storage ব্যবহার করলে, `ttl` আর `limit` কে array তে wrap করে `throttlers` property তে assign করতে হবে।
- `@SkipThrottle()` decorator এখনো specific route বা method এর জন্য throttling bypass করতে ব্যবহার করা যায়। এটা একটা optional boolean parameter নেয়, যেটার default value `true`।
- `@Throttle()` decorator এখন string key সহ একটা object নেয় — key গুলো throttler context এর নাম (নাম না দিলে `'default'`), আর value গুলো `limit` ও `ttl` key সহ object।

> **⚠️ গুরুত্বপূর্ণ:** এখন `ttl` **milliseconds** এ দিতে হয়। Readability এর জন্য seconds এ রাখতে চাইলে এই package এর `seconds` helper ব্যবহার করো — এটা শুধু `ttl` কে ১০০০ দিয়ে multiply করে millisecond এ convert করে দেয়।

আরো জানার জন্য দেখো: [Changelog](https://github.com/nestjs/throttler/blob/master/CHANGELOG.md#500)
