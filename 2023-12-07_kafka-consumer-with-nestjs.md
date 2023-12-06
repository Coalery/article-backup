# Nest.js를 카프카 컨슈머로 만들기! 그런데 많이 이상하게.

안녕하세요!

NestJS 글은 오랜만에 써보는 거 같아요. 이번에도 이상한 주제로 시작된 이상한 삽질기를 갖고 와봤는데요.

NestJS를 카프카 컨슈머로 만들기! 부터가 사실 재밌는 일인데, 많이 이상하게..라니.

사실 NestJS에서는 이미 `@nestjs/microservices`라는 패키지로 카프카 컨슈머를 지원하고 있어요. ([관련 문서](https://docs.nestjs.com/microservices/kafka))
하지만, 이번에 만들어볼 카프카 컨슈머는 추가적인 패키지 없이 `@nestjs/common`과 `@nestjs/core`로만 만들어 볼 거에요.

그럼 시작해볼까요!

---

## 기본적인 네스트 앱은.

기본적으로 우리가 네스트 앱을 만들면 `main.ts`에서 다음과 같은 코드를 작성합니다.

```typescript
import { NestFactory } from "@nestjs/core";

import { AppModule } from "@app/app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}

bootstrap();
```

많이 익숙한 코드죠? 이렇게 `NestFactory.create`에 아무 값 없이 모듈만 넘겨주면 기본적으로 `ExpressAdapter`를 사용하게 된다는 건 여러분도 잘 알고 계실 거 같아요.

원한다면 `Fastify`의 어댑터를 사용할 수도 있구요. 하지만 그 외에는 크게 해당 내용을 다루는 곳이 없어요.

## 어. 그런데 어댑터요?

어댑터. 그런데 어댑터란 말이죠? 그럼 다른 코드도 끼워넣을 수 있다는 이야기가 되는걸까요?

맞습니다. 이런저런 걸 다 끼워넣어 볼 수 있어요.

사실 이 아이디어는 NestJS Korea의 NestJS 밋업, ["야너두 NestJS!"](https://nestjs-korea.notion.site/3rd-NestJS-wrap-up-1bd4e5cf44f94797a0645316ac431f3b)의 세 번째 발표인 [`Lets NestJS Anything`](https://youtu.be/IsX1vgWYVpE?t=4183)에서 영감을 받았어요.

마침 당시 발표를 알게 되었을 때 [네스트가 어떻게 라우트를 처리하는지 알아보면서](https://velog.io/@coalery/nest-route-how) `ExpressAdapter`가 어떻게 동작하는지를 알게 됐었거든요.

발표에서는 코드 공개를 안 해주셔서.. 이거 해보면 될 거 같은데?? 하면서 처음 만든게 [네스트 디스코드 봇](https://github.com/Coalery/discord-bot-with-nestjs)이에요.

> 코드는 아래에서 자세히 다뤄볼게요! 지금은 이야기를 해볼거에요.

## 디스코드 봇 어댑터를 구현하다.

![](https://velog.velcdn.com/images/coalery/post/36c70f20-b910-4637-b1ab-0fd64543de6f/image.png)

저 아이디어를 생각해내고 바로 만들어낸게 이 디스코드 봇 어댑터에요. 이 슬라이드는 회사 사내 테크톡에서 발표했던 자료인데, 사진을 잘 보시면 `@DiscordHandler`에서 `name` 파라미터에 넘겨준 `'hello'`가 명령어로 사용되고 있고, 그 응답으로 `AppService#getHello`의 반환값이 디스코드 채팅으로 나가는 모습을 볼 수 있어요.

![](https://velog.velcdn.com/images/coalery/post/a9e3f745-b82a-4d55-8187-b6c45a82d294/image.png)

어댑터 레벨에서 디스코드 봇을 구현한 것이라서, 놀랍게도.. 우리가 네스트에서 HTTP 요청을 처리하는 것처럼 가드나 인터셉터, 미들웨어 등등 모두 사용이 가능해요!

### 하지만 한계가 있었으니..

```typescript
public get(handler: RequestHandler);
public get(path: any, handler: RequestHandler);
public get(rawPath: unknown, rawHandler?: unknown) {
  if (!rawHandler) {
    return;
  }

  const path = rawPath as string;
  const handler = rawHandler as RequestHandler;
  const command = this.removeLeadingSlash(path);
  this.commands[command] = handler;
}
```

이게 라우트 핸들러를 명령어에 매핑해주는 코드인데, 파라미터로 `path`랑 `handler` 밖에 안 온다는 문제가 있었어요. 이때 `handler`는 단순히 컨트롤러에 등록된 라우트 핸들러 메서드가 아니라, 해당 메서드에 가드나 인터셉터 등등이 감싸진 상태로 구현이 되어 있었어요.

```typescript
public create(/* ... */) {
  // ...
  return async <TRequest, TResponse>(
    req: TRequest,
    res: TResponse,
    next: Function,
  ) => {
    const args = this.contextUtils.createNullArray(argsLength);
    fnCanActivate && (await fnCanActivate([req, res, next]));

    this.responseController.setStatus(res, httpStatusCode);
    hasCustomHeaders &&
      this.responseController.setHeaders(res, responseHeaders);

    const result = await this.interceptorsConsumer.intercept(
      interceptors,
      [req, res, next],
      instance,
      callback,
      handler(args, req, res, next),
      contextType,
    );
    await (fnHandleResponse as HandlerResponseBasicFn)(result, res, req);
  };
}
```

위 코드는 실제로 여러 겹으로 감싸는 라우트 핸들러가 만들어지는 코드인데요. ([깃헙에서 보기](https://github.com/nestjs/nest/blob/v10.2.10/packages/core/router/router-execution-context.ts#L80-L175))

반환되는 비동기 익명 함수 안에서 `InterceptorsConsumer#intercept`의 4번째 **파라미터가 파이프가 적용된 핸들러**이며, 실질적으로 매핑할 때 받을 수 있는 핸들러는 저 반환되는 비동기 익명 함수였습니다.

즉, 컨트롤러의 라우트 핸들러 메서드를 가져올 수 없게 되고, 이에 따라 라우트 핸들러에 붙어있는 메타데이터 또한 가져올 수 없게 됩니다.

디스코드 명령어나, 명령어의 인자에 대한 정보를 담으려면 컨트롤러의 라우트 핸들러에 데코레이터를 붙이는 것이 기존 네스트 개발과 거의 동일한데, 이렇게 되어버리면 `path`에 있는 명령어 외에는 어댑터로 전달할 방법이 없어지게 된 것입니다.

### 일단 방법이 없는 건 아닌데.

실제로 명령어 처리를 시작하는 것은 `INestApplication#listen` 메서드를 호출한 뒤라는 사실을 활용하면 조금 tricky 하더라도 메타데이터 정보를 어댑터 안으로 가져올 수 있습니다.

```typescript
async function bootstrap() {
  const adapter = new DiscordBotAdapter({
    token: process.env.DISCORD_TOKEN,
    clientId: process.env.DISCORD_CLIENT_ID,
    guildId: process.env.DISCORD_GUILD_ID,
  });
  const app = await NestFactory.create(AppModule, adapter);
  adapter.registerApp(app);
  await app.listen(3000);
}
```

위 코드처럼 어댑터 객체를 미리 만들고 `NestFactory.create`에 넘겨준 다음에 만들어진 `app`을 다시 `SomeAdapter#registerApp` 등의 메서드를 활용하여 어댑터 내에서 `app`을 참조할 수 있게 되면, `ModulesContainer`를 활용하여 모든 컨트롤러를 순회할 수 있긴 합니다.

다만 네스트 앱이 전반적으로 어댑터를 의존하고 있는 상황에서, 어댑터가 다시 앱을 의존하게 되어버리는게 맞는건가..라는 생각이 자꾸 들어서 시도해보진 않았었습니다.

## 그냥 내가 순회하면 되는 거 아닌가?

그렇게 디스코드 봇 어댑터를 만들고 몇 달이 지난 뒤에도 문득 생각이 나면 계속 고민을 해봤어요.
그러던 찰나, 문득 아이디어가 머리를 스쳐지나갔습니다.

> 어? 이거 그냥 내가 순회하면 되는 거 아닌가?

아이디어를 증명할 겸, 회사 발표도 할 겸 해서, 회사 동료분들은 디스코드보다는 카프카가 잘 와닿을 것 같아서 카프카 컨슈머로 새로 만들어봤어요. 그게 이번에 본격적으로 소개할 카프카 컨슈머의 정체인데요!

코드 먼저 보고 오실 분들은 요기 아래 깃헙 링크로 가면 됩니다.
https://github.com/Coalery/kafka-consumer-in-nestjs

### 핵심 아이디어

이번에 만들어본 카프카 컨슈머는 아래 4가지 핵심 아이디어를 기반으로 작성됐어요.

1. `AbstractHttpAdapter#get`, `post`, `...`는 모듈에 등록된 **모든 컨트롤러를 순회하면서 path와 핸들러를 넘겨주는데**, 이들을 맵에 저장해준 뒤에 kafka message가 들어왔을 때 올바르게 핸들러를 호출해줍니다.
   - 이때, 핸들러는 이미 가드나 파이프 기타 등등이 모두 적용된 핸들러임을 알고 있어야 합니다.
   - `ExpressAdapter`를 사용할 때에는 `express`가 이 역할을 대신 해주고 있어요. 우리가 `express`에서 라우트를 어떻게 등록하는지를 생각해보면 이해하기 쉬울 거에요.
2. `AppModule`만 있으면 내부 모듈이 아닌 **모든 모듈에 접근할 수 있습니다**.
   - 우리가 카프카 컨슈머로 사용할 컨트롤러의 라우트 핸들러 메서드들은 무조건 우리가 등록하는 것들만 존재하며, 네스트 내부적으로는 절대 등록되지 않습니다.
   - 따라서 카프카 컨슈머의 핸들러들이 등록된 모든 컨트롤러는 우리가 만든 모듈에 등록되며, 이들은 루트 모듈인 `AppModule`에서 접근할 수 있다는 사실을 알 수 있습니다.
3. 컨트롤러 단에서 `@Req`, `@Res`로 받을 수 있는 `Request`, `Response` 객체를 **직접 만들어서** 넘겨줄 수 있습니다.
   - `ExpressAdapter`를 사용할 때에는 `@Req`와 `@Res` 데코레이터에서 각각 `express`의 요청 객체와 응답 객체인 `Request`와 `Response`를 받을 수 있습니다.
   - 하지만 우리는 `express`를 기반하지 않고 직접 어댑터를 만들기 때문에, 커스텀한 요청 및 응답 객체를 만들어서 넘겨줄 수 있습니다.
4. `AbstractHttpAdapter`의 대부분의 메서드는 **구현하지 않아도 어떻게든 굴러가므로**, 필요한 것들만 구현하면 됩니다.
   - 토이 프로젝트 레벨에서 간단하게만 만들어본 것이기 때문에 확실하지는 않습니다.
   - 커스텀 어댑터를 실제 고객에게 서빙하는 프로덕션에서 쓰는 케이스를 들어본 적은 없어서, 좀 더 많은 검증이 필요합니다.
   - 사실 굳이 쓸려고 할 거 같진 않습니다. (..)

### 불필요한 메서드들을 빈 함수로 구현

```typescript
// src/kafka-consumer-adapter/adapter/empty.http-server.ts
export class EmptyHttpServer {
  once() {}
  removeListener() {}
  address() {
    return "kafka-bot";
  }
}
```

```typescript
// src/kafka-consumer-adapter/adapter/empty.adapter.ts
export class EmptyAdapter extends AbstractHttpAdapter {
  constructor(private readonly type: string) {
    super();
  }

  initHttpServer(options: NestApplicationOptions) {
    this.httpServer = new EmptyHttpServer();
  }

  close() {}
  useStaticAssets(...args: any[]) {}
  setViewEngine(engine: string) {}
  // 그리고 수많은 비어있는 함수들..
}
```

http 어댑터를 어떻게든 다른 어댑터로 만드는 과정이기 때문에, http 어댑터로써 필요한 기능들은 모두 필요하지 않습니다. 따라서 모두 빈 함수로 구현하게 됩니다.

`EmptyHttpServer`에 구현되어 있는 세 함수가 없으면 켜지지 않기 때문에, 이들만 빈 함수로 구현했어요.

### 컨트롤러와 메시지 핸들러 데코레이터 구현

```typescript
// src/kafka-consumer-adapter/handler/kafka-controller.decorator.ts
export const KafkaController = (): ClassDecorator => (target: Function) => {
  Reflect.defineMetadata(KAFKA_CONTROLLER_METADATA, true, target);
  Controller()(target);
};

// src/kafka-consumer-adapter/handler/message-handler.decorator.ts
export const MessageHandler =
  (topic: string): MethodDecorator =>
  (
    target: any,
    propertyKey: string | symbol,
    descriptor: TypedPropertyDescriptor<any>
  ) => {
    const metadata: MessageHandlerMetadata = { topic };
    Reflect.defineMetadata(
      MESSAGE_HANDLER_METADATA,
      metadata,
      descriptor.value
    );

    return Get(topic)(target, propertyKey, descriptor);
  };
```

계속 언급하지만! (제일 중요해요!) 우리는 http 어댑터를 개조해서 카프카 컨슈머로 만드는 상황이에요.

따라서, 어댑터로 `path`와 `handler`를 받으려면, 일반적인 http 라우트 핸들러를 등록하는 것처럼 `@Controller` 데코레이터와 `@Get`, `@Post` 등 라우트 데코레이터를 적용해야 합니다.

여기서는 `topic`을 `path`로써 사용하게 구현했어요. 하지만 뭘 구현하느냐에 따라 다른 걸 넣어줄 수도 있겠죠? 디스코드 봇 만들 때는 저기에 명령어가 들어갔어요!

### 모든 메시지 핸들러들의 메타데이터 가져오기.

```typescript
type Module = Type<any>;

export class MessageHandlerMetadataExplorer {
  explore(module: Module): MessageHandlerMetadataMap {
    const result: MessageHandlerMetadataMap = {};

    this.exploreInternal(module, result);

    return result;
  }

  private exploreInternal(module: Module, map: MessageHandlerMetadataMap) {
    const imports: Module[] = Reflect.getMetadata("imports", module);
    imports.forEach((importedModule) =>
      this.exploreInternal(importedModule, map)
    );

    const controllers = Reflect.getMetadata("controllers", module);
    if (
      !controllers ||
      !Array.isArray(controllers) ||
      controllers.length === 0
    ) {
      return;
    }

    controllers
      .filter((controller) => this.isKafkaController(controller))
      .forEach((controller) => this.exploreController(controller, map));
  }

  private exploreController(
    controller: Type<any>,
    map: MessageHandlerMetadataMap
  ) {
    Object.values(Object.getOwnPropertyDescriptors(controller.prototype))
      .map((descriptor) =>
        Reflect.getMetadata(MESSAGE_HANDLER_METADATA, descriptor.value)
      )
      .filter((metadata): metadata is MessageHandlerMetadata => !!metadata)
      .forEach((metadata) => (map[metadata.topic] = metadata));
  }

  private isKafkaController(controller: Type<any>): boolean {
    return !!Reflect.getMetadata(KAFKA_CONTROLLER_METADATA, controller);
  }
}
```

요게 루트 모듈을 넣어주면 루트 모듈에 속한 모든 모듈들을 순회하면서 컨트롤러들을 가져와서, 각 핸들러들의 메타데이터를 저장한 맵을 만들어 반환하는 클래스에요.

1. `explore` 메서드를 루트 모듈과 함께 호출하면,
2. 결과 맵을 생성해서 `exploreInternal`을 호출해요.
3. `exploreInternal`은 재귀적으로 주어진 모듈의 `imports` 배열에 대해 DFS를 시행해요.
4. 주어진 모듈의 `controllers` 중 카프카 컨트롤러인 것들만 골라서, 모든 프로퍼티를 돌면서 메시지 핸들러를 찾아 메타데이터를 가져온 뒤 맵에 저장해요.
   - 위에서 구현한 `@KafkaController` 데코레이터가 붙여주는 메타데이터 존재 여부를 통해 카프카 컨트롤러인지를 판별해요.
   - 메시지 핸들러를 찾는 것도 `@MessageHandler` 데코레이터가 붙여주는 메타데이터의 존재 여부를 통해 확인해요.
5. 만들어진 맵을 반환해요.

눈치 채신 분들도 있지만, 위 코드는 모듈이 정적 모듈로 이루어진 트리 구조를 갖고 있다고 가정한 채 구현된 코드에요. 즉 forward ref나 dynamic module은 사용하지 못해요.

만약 모든 모듈 등록 방법을 지원하면 그때부터는 트리 구조가 아니라 그래프가 되기 때문에 방문 확인을 해줘야 하는데, 이번에 만든 코드는 그게 핵심은 아니라서 구현하지 않았어요.

### 대망의 어댑터 구현!

```typescript
type ListenFnCallback = (...args: unknown[]) => void;

export type KafkaRequest = {
  /* ... */
};

// 비동기 처리라서 응답 객체는 없음
export type KafkaResponse = object;

export class KafkaConsumerAdapter extends EmptyAdapter {
  private readonly kafkaClient: Kafka;
  private readonly kafkaConsumer: Consumer;
  private readonly config: KafkaConfig;

  private readonly metadataMap: MessageHandlerMetadataMap;
  private readonly handlers: Record<string, RequestHandler>;

  constructor(module: Type<any>, config: KafkaConfig) {
    /* ... */
  }

  get(handler: RequestHandler): void;
  get(path: any, handler: RequestHandler): void;
  get(rawPath: unknown, rawHandler?: unknown): void {
    /* ... */
  }

  async initKafkaConsumer(): Promise<void> {
    /* ... */
  }

  listen(port: string | number, callback?: () => void): any;
  listen(port: string | number, hostname: string, callback?: () => void): any;
  listen(
    port: unknown,
    hostname?: ListenFnCallback | string,
    rawCallback?: ListenFnCallback
  ): any {
    /* ... */
  }
  close() {
    /* ... */
  }

  private removeLeadingSlash(path: string): string {
    /* ... */
  }
}
```

코드로 보기엔 길지 않은데, 글로 보기엔 조금 길어서, 먼저 어떻게 생겼는지 전체적인 구조를 보고 시작할게요!

#### `constructor`

```typescript
constructor(module: Type<any>, config: KafkaConfig) {
  super('kafka');

  this.kafkaClient = new Kafka(config.client);
  this.kafkaConsumer = this.kafkaClient.consumer(config.consumer);

  const explorer = new MessageHandlerMetadataExplorer();
  this.metadataMap = explorer.explore(module);

  this.config = config;
  this.handlers = {};
}
```

카프카 클라이언트와 컨슈머를 만들고, 파라미터로 받은 `module`에 대해 메타데이터 맵을 만든 뒤에 저장해둡니다.

그리고 토픽을 키로, 핸들러를 값으로 갖는 맵(`handlers`)을 만들어서 넣어줘요.

#### `get`, `removeLeadingSlash`

```typescript
get(handler: RequestHandler): void;
get(path: any, handler: RequestHandler): void;
get(rawPath: unknown, rawHandler?: unknown): void {
  if (!rawHandler) {
    return;
  }

  const path = rawPath as string;
  const handler = rawHandler as RequestHandler;
  const topic = this.removeLeadingSlash(path);

  this.handlers[topic] = handler;
}

private removeLeadingSlash(path: string): string {
  return path[0] === '/' ? path.substring(1) : path;
}
```

여기는 핸들러를 실제로 매핑해주는 부분이에요. `rawHandler`가 없는 경우에는 `handler`만 들어온 경우인데, 이때는 처리해줄 수 없기 때문에 `path`와 `handler`가 있는 경우에만 매핑되도록 구현했어요.

`path`는 보통 맨 앞에 슬래시(/)를 포함하기 때문에, 이걸 제거해준 다음에 매핑해주고 있어요.

#### `initKafkaConsumer`

```typescript
async initKafkaConsumer(): Promise<void> {
  await this.kafkaConsumer.connect();
  await this.kafkaConsumer.subscribe(this.config.subscribe);
  await this.kafkaConsumer.run({
    eachMessage: async (payload) => {
      const handler = this.handlers[payload.topic];
      if (!handler) {
        return;
      }

      const kafkaRequest: KafkaRequest = {
        topic: payload.topic,
        partition: payload.partition,
        offset: payload.message.offset,
        key: payload.message.key,
        value: payload.message.value,
        timestamp: payload.message.timestamp,
      };
      const kafkaResponse: KafkaResponse = {};
      const next = () => {};
      await handler(kafkaRequest, kafkaResponse, next);
    },
  });
}
```

`listen`에서 콜백을 호출하기 전에 완료되어야 하는 작업으로써, 카프카 컨슈머를 초기화하는 역할을 하고 있어요.

`KafkaConsumer#run`의 `eachMessage`를 보면, 토픽에 맞는 핸들러를 가져와서 커스텀한 요청 객체와 응답 객체, 그리고 `next` 함수를 만들어준 뒤에 핸들러를 직접 호출해주는 모습을 볼 수 있어요. 이렇게 호출해주면 미들웨어를 타고, 가드를 타고, 이모저모를 탄 뒤에 컨트롤러의 핸들러를 호출해줄 수 있을 거에요.

저는 일단 [`eachMessage`](https://kafka.js.org/docs/consuming#a-name-each-message-a-eachmessage)에 대해서만 구현했는데, 필요에 따라 [`eachBatch`](https://kafka.js.org/docs/consuming#a-name-each-batch-a-eachbatch)에 대해서 구현하고, 이에 맞는 `KafkaRequest`를 커스텀하게 만들어 넘겨줄 수도 있겠죠!

#### `listen`

```typescript
listen(port: string | number, callback?: () => void): any;
listen(port: string | number, hostname: string, callback?: () => void): any;
listen(
  port: unknown,
  hostname?: ListenFnCallback | string,
  rawCallback?: ListenFnCallback,
): any {
  let callback: ListenFnCallback = () => {};

  if (typeof hostname === 'function') {
    callback = hostname;
  } else if (typeof rawCallback === 'function') {
    callback = rawCallback;
  }

  this.initKafkaConsumer()
    .then(() => callback())
    .catch((e) => callback(e));
}
```

어댑터의 `listen`은 우리가 `NestFactory.create`로 만든 `INestApplication`의 `listen`을 호출할 때 호출돼요.

이때, `NestFactory.create`는 `INestApplication` 인터페이스를 준수하는 `NestApplication` 객체를 생성해서 반환해주는데요. 반환된 `NestApplication` 객체의 `listen`을 호출하면, 어댑터의 `listen`을 호출하는 모습을 코드에서 볼 수 있어요.

실제 코드는 [여기](https://github.com/nestjs/nest/blob/v10.2.10/packages/core/nest-application.ts#L311)를 참고해주세요! 글 작성 시점 가장 최신 버전인 v10.2.10의 코드에요.

아무튼 다시 어댑터 코드로 돌아와서, 우리는 `listen`에서 `port`와 `hostname`을 넣어줄 필요가 없어요. 왜냐하면 해당 정보는 `KafkaClient`의 `brokers` config에 포함되어 있거든요!
따라서 `listen`에서는 콜백 함수를 파라미터로부터 추출해서, `initKafkaConsumer`의 결과에 따라 콜백 함수를 호출해주는 역할만 해요.

#### `close`

```typescript
close() {
  this.kafkaConsumer.disconnect();
}
```

마지막으로 `close`에요. 딱 봐도 보이지만, 카프카 컨슈머를 정리하는 역할을 해요.

`close` 메서드는 `NestApplication`의 [`dispose`](https://github.com/nestjs/nest/blob/v10.2.10/packages/core/nest-application.ts#L101)에서 호출되며, `dispose`는 `NestApplication`이 상속 받는 `NestApplicationContext`의 [`close`](https://github.com/nestjs/nest/blob/v10.2.10/packages/core/nest-application-context.ts#L251)와 [`listenToShutdownSignals`](https://github.com/nestjs/nest/blob/v10.2.10/packages/core/nest-application-context.ts#L338)의 `cleanup` 함수에서 호출돼요.

이때 `NestApplicationContext#close`는 우리가 `NestFactory.create`를 통해 만들어낸 `INestApplication` 객체의 `close`를 호출하면 호출이 되구요.

그리고 `listenToShutdownSignals`는 모든 shutdown signal을 돌면서 `cleanup` 함수를 콜백으로 등록해요([코드](https://github.com/nestjs/nest/blob/v10.2.10/packages/core/nest-application-context.ts#L353-L356)). 따라서 shutdown signal을 프로세스가 받으면 정리하는 역할을 합니다.

## 실제로 사용하면?

```
src
├ all-exception.filter.ts
├ app.controller.ts
├ app.module.ts
├ app.service.ts
├ json.guard.ts
├ key.decorator.ts
├ main.ts
└ user-created.dto.ts
```

어댑터 코드를 제외하고 프로젝트 구조는 위와 같아요. 예제 코드는 간단해서 가볍게 살펴볼 수 있을 거에요.

### `main.ts`

```typescript
async function bootstrap() {
  const app = await NestFactory.create(
    AppModule,
    new KafkaConsumerAdapter(AppModule, {
      client: { brokers: ["localhost:9093"] },
      consumer: { groupId: "groupId-1" },
      subscribe: { topics: ["user.created", "user.left"] },
    })
  );
  await app.listen(0);
}
bootstrap();
```

네스트 앱의 시작점이죠! 포트 번호는 중요하지 않기 때문에 `listen`에 `0`이라는 의미 없는 값을 넣어주었고, 카프카 컨슈머 어댑터를 넣어줬어요.

그러면! 짜잔! HTTP 어플리케이션이었던게 카프카 컨슈머로 바뀌어요!

### `app.module.ts`

```typescript
@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

모듈 파일이에요. 우리가 이때까지 써왔던 형태랑 전혀 다르지 않아요.

### `app.controller.ts`

```typescript
@KafkaController()
@UseFilters(AllExceptionFilter)
export class AppController {
  constructor(private readonly appService: AppService) {}

  @MessageHandler("user.created")
  @UseGuards(JsonGuard)
  userCreated(@Req() request: KafkaRequest, @Key() key: UserCreatedDto): void {
    console.log(JSON.stringify(request, null, 2));
    console.log(JSON.stringify(key, null, 2));
  }

  @MessageHandler("user.left")
  userLeft(@Req() request: KafkaRequest): void {
    console.log(this.appService.getHello());
  }
}
```

컨트롤러에요. 앞서 만들어줬던 `@KafkaController`와 `@MessageHandler`를 사용했고, 가드와 필터를 적용한 모습도 볼 수 있어요.

`@Req()` 데코레이터를 사용해서 어댑터에서 만들어준 `KafkaRequest`를 가져올 수 있고, `@Key()`를 통해 메시지의 키 페이로드를 가져올 수 있어요. 근데 `UserCreatedDto`는 값 페이로드에 있어야 하지 않나.. 하는 생각이 잠시 스쳐지나가네요.

> If you want to run validation for custom decorators as well, you can build another ValidationPipe based on the existing one. I believe that I explained the reason why it's not a default behavior in another issue already
> 원글: https://github.com/nestjs/nest/issues/2010#issuecomment-483625315

추가적으로 네스트에서 구현되어 있는 `ValidationPipe`를 사용할 수 있다면 좋겠지만, 아쉽게도 위 글에서 볼 수 있듯이 커스텀 데코레이터에서는 `ValidationPipe`를 직접 만들어서 적용하라는 모습을 볼 수가 있습니다.

### `app.service.ts`

```typescript
@Injectable()
export class AppService {
  getHello(): string {
    return "Hello World!";
  }
}
```

서비스 코드입니다. 간단하죠?

### `key.decorator.ts`

```typescript
export const Key = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request: KafkaRequest = ctx.getArgByIndex(0);
    return request.keyJson;
  }
);
```

`@Key()` 데코레이터에요. 이것도 간단하죠? 사실 네스트 공식문서에서 커스텀 데코레이터 만드는 걸 설명할 때 사용하는 코드랑 크게 다르지 않아요.

### `json.guard.ts`

```typescript
@Injectable()
export class JsonGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request: KafkaRequest = context.getArgByIndex(0);

    request.keyJson = request.key
      ? this.parseJsonBufferToObject(request.key)
      : null;
    request.valueJson = request.value
      ? this.parseJsonBufferToObject(request.value)
      : null;

    return request.keyJson && request.valueJson;
  }

  private parseJsonBufferToObject(value: Buffer): Record<string, any> | null {
    try {
      return JSON.parse(value.toString());
    } catch (e) {
      return null;
    }
  }
}
```

`JsonGuard`에요. 사실 크게 특별한 건 없어요. 그저 `express`의 `Request` 객체를 받는게 아니라 `KafkaRequest`를 받을 뿐이고, `key`와 `value` 모두 JSON일때만 통과하는 가드에요. 겸사겸사 `request` 객체에 파싱된 json도 넣어주고요!

### `all-exception.filter.ts`

```typescript
@Catch()
export class AllExceptionFilter implements ExceptionFilter {
  private readonly logger: ConsoleLogger;

  constructor() {
    this.logger = new ConsoleLogger(AllExceptionFilter.name);
  }

  catch(exception: any, host: ArgumentsHost) {
    this.logger.error(exception);
  }
}
```

예외 필터인데, 이것도 크게 특별한 점은 없죠? 그냥 로그 찍어주는 역할을 해요!

## 마무리.

이번에는 네스트의 커스텀 어댑터로 카프카 컨슈머를 어떻게 만드는지를 여러 코드들을 통해 알아봤어요.

오늘의 글에서 더 나아가서는, 좀 더 일반화된 커스텀 어댑터도 구현해볼 수 있을 것 같아요. abstract class로 세부 사항은 모두 구현해두고, 사용하는 곳에서는 상속 받아서 사용하기만 하는거죠! 이건 숙제로 남겨두도록 합시다.

참고로 [레포지토리](https://github.com/Coalery/kafka-consumer-in-nestjs)에 가면 위에서 살펴봤던 실행 가능한 예제를 도커 컴포즈로 구성해뒀으니 활용해주시면 좋을 것 같습니다.

사실 정상적으로 네스트를 활용하는 형태는 아닌 것 같아서(?) 실제 프로덕션 환경에서 사용할 일은 잘 없을 것 같긴 하나, 네스트의 요청 처리 플로우를 그대로 사용할 수 있고, 네스트가 제공해주는 DI 또한 모두 활용할 수 있다는 점에서 재미있는 경험이었던 것 같아요.

여러분도 제가 느낀 재미를 함께 느끼셨다면 기쁠 것 같네요!

읽어주셔서 감사합니다! :>

---

글: https://velog.io/@coalery/kafka-consumer-with-nestjs
