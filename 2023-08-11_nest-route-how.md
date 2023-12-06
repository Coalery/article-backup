# Nest.js는 실제로 어떻게 라우트를 처리할까?

안녕하세요!

NestJS 관련해서는 거의 1년만에 글을 쓰는 것 같은데, 이번에는 라우트를 어떻게 처리하는지 알아보면서 이모저모 정리해두고 싶어서 글을 시작합니다.

기준은 글을 작성하는 현재 기준 가장 최신 버전인 `v10.1.3`을 활용하고, 기본적으로 제공해주는 ExpressJS Adapter를 활용합니다.

이번에도 그렇게 체계적으로 찾아보는 방법 대신 삽질...으로 알아볼 예정인데요!

음. 시작해볼까요?

**아래의 모든 글은 증명되지 않은, 말그대로 코드만 보고 제 나름대로 분석한 글이기 때문에 사실과 다를 수도 있습니다. 참고 부탁드려요!**

---

# 그래서. 어디서부터?

저번에는 `NestFactory.create`부터 시작해서 파봤습니다. 그럼 이번에도 어디서 시작할지 정하고 가야 더 효율적으로 알아볼 수 있을 것 같아요.

그래서 이번에 선택한 방법은,

> **컨트롤러에서 에러 발생시키기!**

입니다. 빠밤밤.

### 에러를 일으킬 프로젝트 구조

이게 메인은 아니라서, 간단하게만 잡아봤어요.

```typescript
import { Controller, Get, UseFilters } from "@nestjs/common";
import { SomeExceptionFilter } from "./exception.filter";

@Controller()
export class AppController {
  @Get()
  @UseFilters(new SomeExceptionFilter())
  getHello(): string {
    throw Error();
  }
}
```

컨트롤러에서는 이렇게 에러를 던집니다.

그러면 요걸 Exception Filter가 받을텐데요!

```typescript
import { ArgumentsHost, Catch, ExceptionFilter } from "@nestjs/common";
import { Response } from "express";

@Catch()
export class SomeExceptionFilter implements ExceptionFilter {
  catch(exception: Error, host: ArgumentsHost) {
    console.log(exception.stack);
    host.switchToHttp().getResponse<Response>().status(500).json({});
  }
}
```

여기서 에러의 stack trace를 출력해줍니다.

응답은 나가야 하니까 빈 객체를 반환해줬어요. 모듈은 컨트롤러만 있어서 생략할게요.

그리고 요청을 보내주면, 다음과 같이 trace가 나와요.

```
Error:
    at AppController.getHello (/Users/lery/abc/abc/src/app.controller.ts:9:11)
    at /Users/lery/abc/abc/node_modules/@nestjs/core/router/router-execution-context.js:38:29
    at InterceptorsConsumer.intercept (/Users/lery/abc/abc/node_modules/@nestjs/core/interceptors/interceptors-consumer.js:12:20)
    at /Users/lery/abc/abc/node_modules/@nestjs/core/router/router-execution-context.js:46:60
    at /Users/lery/abc/abc/node_modules/@nestjs/core/router/router-proxy.js:9:23
    at Layer.handle [as handle_request] (/Users/lery/abc/abc/node_modules/express/lib/router/layer.js:95:5)
    at next (/Users/lery/abc/abc/node_modules/express/lib/router/route.js:144:13)
    at Route.dispatch (/Users/lery/abc/abc/node_modules/express/lib/router/route.js:114:3)
    at Layer.handle [as handle_request] (/Users/lery/abc/abc/node_modules/express/lib/router/layer.js:95:5)
    at /Users/lery/abc/abc/node_modules/express/lib/router/index.js:284:15
```

우리의 목적은 ExpressJS가 아니라, ExpressJS가 받은 요청을 어떻게 NestJS가 잘 지지고 볶아서 컨트롤러로 보내주는지 알아보는 것이기 때문에, trace가 `.../node_modules/express/...`에서 `.../node_modules/@nestjs/...`로 바뀌는 부분부터 컨트롤러까지 올라와보면 될 거 같아요.

그러면, 처음 볼 곳이 정해진 거 같네요.

바로, `@nestjs/core` 패키지 안에 있는 `router-proxy` 파일이에요!

# 이번에도 역시, 시작해봅시다.

## 첫 번째 trace, router-proxy

먼저 위의 경로는 `router-proxy.js`이므로 타입스크립트 파일의 라인 넘버와는 다를 수 있어요. 따라서 먼저 JS 파일을 살펴보고, 이걸 타입스크립트 파일과 비교해보면서 진행하겠습니다.

```javascript
// node_modules/@nestjs/core/router/router-proxy.js
// ...
class RouterProxy {
  createProxy(targetCallback, exceptionsHandler) {
    return async (req, res, next) => {
      try {
        await targetCallback(req, res, next);
        //    ^ 여기!
      } catch (e) {
        // ...
      }
    };
  }
  // ...
}
exports.RouterProxy = RouterProxy;
```

위의 코드에서 "여기!"라고 표현한 곳이 `router-proxy.js:9:23` 위치에요.

이를 타입스크립트 코드와 비교해봤을 때, 다음과 같이 생겼어요.

```typescript
// @nestjs/core/router/router-proxy.ts
export class RouterProxy {
  public createProxy(
    targetCallback: RouterProxyCallback,
    exceptionsHandler: ExceptionsHandler
  ) {
    return async <TRequest, TResponse>(
      req: TRequest,
      res: TResponse,
      next: () => void
    ) => {
      try {
        await targetCallback(req, res, next);
      } catch (e) {
        const host = new ExecutionContextHost([req, res, next]);
        exceptionsHandler.next(e, host);
        return res;
      }
    };
  }
}
```

즉, 요청이 들어오면 `targetCallback`을 호출해준다는 뜻인데요. 이 코드에서는 크게 의미를 찾을 순 없을 거 같아요. 그냥 try-catch를 함수로 감싸고 있는 형태인 거 같아요.

그나저나 `createProxy` 메서드가 반환하는 비동기 익명 함수의 형태가 어디서 많이 본 형태같지 않나요?
바로 ExpressJS에서 미들웨어를 선언하는 방법이에요.

NestJS는 ExpressJS 어댑터와 Fastify 어댑터를 둘 다 지원할텐데 이렇게 ExpressJS스러운 코드를 작성해도 되나.. 싶었는데, 요거 관련해서 찾아보니 Fastify에서도 [hook](https://fastify.dev/docs/latest/Reference/Hooks/)이라는 이름으로 동일한 형태의 기능을 제공해주고 있어요. (물론 조금 다른 기능이긴 합니다!)

아무튼, 여기서는 크게 얻을 수 있는게 없어보이니 다른 곳으로 가볼까요?

일단 먼저 `RouterProxy.createProxy`를 호출하는 곳으로 가본 다음, ExpressJS에서 어떻게 요청을 넘겨주는지를 확인한 후에 다음 stack trace로 가볼게요.

해당 메서드는 총 3곳에서 호출되고 있어요.

- [`@nestjs/core/middleware/middleware-module.ts#L300` in createProxy method](https://github.com/nestjs/nest/blob/26e2886e6b11d35bcb685fe997629614a08fe1db/packages/core/middleware/middleware-module.ts#L300)
- [`@nestjs/core/router/router-explorer.ts#L352` in createCallbackProxy method](https://github.com/nestjs/nest/blob/26e2886e6b11d35bcb685fe997629614a08fe1db/packages/core/router/router-explorer.ts#L352)
- [`@nestjs/core/router/rooutes-resolver.ts#L150` in registerNotFoundHandler method](https://github.com/nestjs/nest/blob/26e2886e6b11d35bcb685fe997629614a08fe1db/packages/core/router/routes-resolver.ts#L150)

세 번째 메서드는 이름만 봐도 이번 주제와는 크게 관계가 없어보이고, 첫 번째 혹은 두 번째에서 골라가면 될 거 같아요. 그런데 해당 파일들이 속해있는 디렉토리의 이름이나 파일의 이름을 봤을 때, 미들웨어보다는 직접적으로 `router`라는 단어를 쓴 두 번째가 더 끌리는 거 같아요.

두 번째 파일로 가볼게요.

```typescript
// @nestjs/core/router/router-explorer.ts
private createCallbackProxy(
  instance: Controller,
  callback: RouterProxyCallback,
  methodName: string,
  moduleRef: string,
  requestMethod: RequestMethod,
  contextId = STATIC_CONTEXT,
  inquirerId?: string,
) {
  const executionContext = this.executionContextCreator.create(
    instance,
    callback,
    methodName,
    moduleRef,
    requestMethod,
    contextId,
    inquirerId,
  );
  const exceptionFilter = this.exceptionsFilter.create(
    instance,
    callback,
    moduleRef,
    contextId,
    inquirerId,
  );
  return this.routerProxy.createProxy(executionContext, exceptionFilter);
}
```

실제로 `createProxy` 메서드를 호출하는 곳이에요. `exceptionFilter`는 굳이 살펴볼 필요는 없을 거 같고, `executionContext`만 살펴보면 될 거 같아요.

그 이유는, route-proxy에서 호출했던 `targetCallback`이 바로 `executionContext`기 때문이에요. 그럼 `executionContext`를 따라가보면 컨트롤러에 가까워질 수 있을 거 같아요. 마침 다음 stack trace의 위치가 아래와 같이 `router-execution-context.js` 파일이거든요!

```
at /Users/lery/abc/abc/node_modules/@nestjs/core/router/router-execution-context.js:46:60
```

일단 해당 메서드가 `createProxy`의 반환 값을 그대로 반환해주고 있으므로, 일종의 라우트 핸들러 프록시를 만들어 반환해주는 것으로 보입니다. 따라서 해당 핸들러가 저장되는 곳을 살펴보면 어떻게 ExpressJS에서 NestJS로 요청 데이터가 넘어오는지 살펴볼 수 있을 거 같아요.

`createCallbackProxy`를 호출하는 곳을 살펴볼게요. 해당 메서드가 `private`이니, 같은 파일 안에 있겠죠? 해당 메서드를 호출하는 곳은 총 두 곳입니다.

- [`@nestjs/core/router/router-explorer.ts#L184-L190` in applyCallbackToRouter method](https://github.com/nestjs/nest/blob/26e2886e6b11d35bcb685fe997629614a08fe1db/packages/core/router/router-explorer.ts#L184-L190)
- [`@nestjs/core/router/router-explorer.ts#L380-L388` in createRequestScopedHandler method](https://github.com/nestjs/nest/blob/26e2886e6b11d35bcb685fe997629614a08fe1db/packages/core/router/router-explorer.ts#L380-L388)

메서드 이름을 봤을 때, 두 번째보다는 첫 번째가 우리가 원하는 목적에 가까울 거 같아요.

먼저 두 번째 쪽 메서드 이름에 Request-scoped가 들어가있고, 첫 번째 메서드에서 두 번째 메서드를 호출하는 형태라서요!

이때, Request-scoped는 요청이 올 때마다 인스턴스를 새로 만들어내는 걸 말해요.
참고: [NestJS docs: Injection scopes](https://docs.nestjs.com/fundamentals/injection-scopes)

따라서 첫 번째 메서드로 가보는게 좋을 거 같아요.

```typescript
private applyCallbackToRouter<T extends HttpServer>(
  router: T,
  routeDefinition: RouteDefinition,
  instanceWrapper: InstanceWrapper,
  moduleKey: string,
  routePathMetadata: RoutePathMetadata,
  host: string | RegExp | Array<string | RegExp>,
) {
  const {
    path: paths,
    requestMethod,
    targetCallback,
    methodName,
  } = routeDefinition;

  const { instance } = instanceWrapper;
  const routerMethodRef = this.routerMethodFactory
    .get(router, requestMethod)
    .bind(router);

  const isRequestScoped = !instanceWrapper.isDependencyTreeStatic();
  const proxy = isRequestScoped
  ? this.createRequestScopedHandler(
      instanceWrapper,
      requestMethod,
      this.container.getModuleByKey(moduleKey),
      moduleKey,
      methodName,
    )
  : this.createCallbackProxy(
      instance,
      targetCallback,
      methodName,
      moduleKey,
      requestMethod,
    );

  const isVersioned =
        (routePathMetadata.methodVersion ||
         routePathMetadata.controllerVersion) &&
        routePathMetadata.versioningOptions;
  let routeHandler = this.applyHostFilter(host, proxy);

  paths.forEach(path => {
    if (
      isVersioned &&
      routePathMetadata.versioningOptions.type !== VersioningType.URI
    ) {
      // All versioning (except for URI Versioning) is done via the "Version Filter"
      routeHandler = this.applyVersionFilter(
        router,
        routePathMetadata,
        routeHandler,
      );
    }

    routePathMetadata.methodPath = path;
    const pathsToRegister = this.routePathFactory.create(
      routePathMetadata,
      requestMethod,
    );
    pathsToRegister.forEach(path => {
      const entrypointDefinition: Entrypoint<HttpEntrypointMetadata> = {
        type: 'http-endpoint',
        methodName,
        className: instanceWrapper.name,
        classNodeId: instanceWrapper.id,
        metadata: {
          key: path,
          path,
          requestMethod: RequestMethod[
            requestMethod
          ] as keyof typeof RequestMethod,
          methodVersion: routePathMetadata.methodVersion as VersionValue,
          controllerVersion:
          routePathMetadata.controllerVersion as VersionValue,
        },
      };

      routerMethodRef(path, routeHandler);

      this.graphInspector.insertEntrypointDefinition<HttpEntrypointMetadata>(
        entrypointDefinition,
        instanceWrapper.id,
      );
    });

    const pathsToLog = this.routePathFactory.create(
      {
        ...routePathMetadata,
        versioningOptions: undefined,
      },
      requestMethod,
    );
    pathsToLog.forEach(path => {
      if (isVersioned) {
        const version = this.routePathFactory.getVersion(routePathMetadata);
        this.logger.log(
          VERSIONED_ROUTE_MAPPED_MESSAGE(path, requestMethod, version),
        );
      } else {
        this.logger.log(ROUTE_MAPPED_MESSAGE(path, requestMethod));
      }
    });
  });
}
```

뭔가... 뭔가에요.. 많이 길어요..

일단 몇 개의 부분으로 나눠서 살펴볼게요.

```typescript
const {
  path: paths,
  requestMethod,
  targetCallback,
  methodName,
} = routeDefinition;

const { instance } = instanceWrapper;
const routerMethodRef = this.routerMethodFactory
  .get(router, requestMethod)
  .bind(router);
```

`routeDefinition`에는 실제 path와 http method(GET, POST, ...), 그리고 `targetCallback`과 메서드 이름이 들어왔어요. 이때 `targetCallback`은 컨트롤러 내에 `@Get`, `@Post` 등의 데코레이터가 달린 메서드, 즉 라우트 핸들러에요.

또, `instanceWrapper`가 갖고 있는 `instance`는 컨트롤러의 인스턴스입니다.

그리고 `RouteMethodFactory`는 아래와 같이 생겼어요.

```typescript
// @nestjs/core/helpers/router-method-factory.ts
export class RouterMethodFactory {
  public get(target: HttpServer, requestMethod: RequestMethod): Function {
    switch (requestMethod) {
      case RequestMethod.POST:
        return target.post;
      case RequestMethod.ALL:
        return target.all;
      case RequestMethod.DELETE:
        return target.delete;
      case RequestMethod.PUT:
        return target.put;
      case RequestMethod.PATCH:
        return target.patch;
      case RequestMethod.OPTIONS:
        return target.options;
      case RequestMethod.HEAD:
        return target.head;
      case RequestMethod.GET:
        return target.get;
      default: {
        return target.use;
      }
    }
  }
}
```

각 http method에 따라 `HttpServer` 인터페이스의 각 해당하는 메서드들을 반환해주고 있어요.

이때 `HttpServer` 인터페이스는 여기부터 엄청나게 거슬러 올라가서, `NestApplication`에 있는 `httpAdapter`까지 올라가요. 해당 값은 `NestFactory.create`에서 `NestApplication` 객체를 만들 때 주입되며, `NestFactory`의 `createHttpAdapter`라는 메서드를 통해 객체가 생성됩니다.

여기서 `NestFactory.create`는 우리가 `main.ts`에서 호출해주는 그 메서드가 맞습니다!

해당 메서드는 아래와 같이 생겼어요.

```typescript
// @nestjs/core/nest-factory.ts
private createHttpAdapter<T = any>(httpServer?: T): AbstractHttpAdapter {
  const { ExpressAdapter } = loadAdapter(
    '@nestjs/platform-express',
    'HTTP',
    () => require('@nestjs/platform-express'),
  );
  return new ExpressAdapter(httpServer);
}
```

네! ExpressJS와 연결되는 ExpressAdapter에요.

잠시 `platform-express` 패키지를 보자면..

```typescript
// @nestjs/platform-express/adapters/express-adapter.ts

// ...
import * as express from "express";
// ...

export class ExpressAdapter extends AbstractHttpAdapter<
  http.Server | https.Server
> {
  private readonly routerMethodFactory = new RouterMethodFactory();
  private readonly logger = new Logger(ExpressAdapter.name);
  private readonly openConnections = new Set<Duplex>();

  constructor(instance?: any) {
    super(instance || express());
  }

  // ...
}
```

`express()`를 통해 Express 객체를 생성해서 부모 클래스인 `AbstractHttpAdapter`로 넘겨주고 있어요. 요 클래스는 `core` 패키지에 있어요!

```typescript
// @nestjs/core/adapters/http-adapter.ts
export abstract class AbstractHttpAdapter<
  TServer = any,
  TRequest = any,
  TResponse = any
> implements HttpServer<TRequest, TResponse>
{
  protected httpServer: TServer;

  constructor(protected instance?: any) {}

  // ...

  public get(handler: RequestHandler);
  public get(path: any, handler: RequestHandler);
  public get(...args: any[]) {
    return this.instance.get(...args);
  }

  // ...
}
```

여기까지 보고, 잠시 expressjs에서 라우트를 등록하는 방법을 알아볼까요?

```typescript
import * as express from "express";

const app = express();

app.get("/", (req, res) => {
  res.send("Hello World");
});
```

여기서 `app = express()` 부분이 `AbstractHttpAdapter` 생성자의 매개변수인 `instance?`와 동일합니다. 이를 통해 `AbstractHttpAdapter.get`는 위의 `app.get`을 호출하는 것과 동일하다는 걸 알 수 있습니다.

즉, `RouterMethodFactory.get`에서 반환되는 함수를 받아서 매개변수로 핸들러와 함께 넘겨주면, **express 어플리케이션에 라우트를 등록하는 효과**를 보여준다는 뜻이 됩니다.

다시 원래 코드로 돌아가볼게요.

```typescript
const routerMethodRef = this.routerMethodFactory
  .get(router, requestMethod)
  .bind(router);
```

여기서 `this.routerMethodFactory.get(router, requestMethod)`까지가 각 method에 맞는 라우트를 등록할 수 있는 함수를 반환하고, `bind`를 통해 올바르게 처리될 수 있도록 `router`를 바인딩 합니다.

이제 다음 코드!

```typescript
const isRequestScoped = !instanceWrapper.isDependencyTreeStatic();
const proxy = isRequestScoped
  ? this.createRequestScopedHandler(
      instanceWrapper,
      requestMethod,
      this.container.getModuleByKey(moduleKey),
      moduleKey,
      methodName
    )
  : this.createCallbackProxy(
      instance,
      targetCallback,
      methodName,
      moduleKey,
      requestMethod
    );
```

request-scoped가 아니기 때문에 위에서 살펴봤던 `createCallbackProxy`를 호출합니다. 따라서 `proxy`는 라우트 핸들러 프록시를 갖고 있어요.

다음!

```typescript
const isVersioned =
  (routePathMetadata.methodVersion || routePathMetadata.controllerVersion) &&
  routePathMetadata.versioningOptions;
let routeHandler = this.applyHostFilter(host, proxy);
```

메서드나 컨트롤러가 현재는 [버저닝 처리](https://docs.nestjs.com/techniques/versioning)가 되어 있지 않기 때문에 `isVersioned = false`가 될 것이고, `routeHandler`는 `applyHostFilter`를 살펴봐야 해요.

그 전에, `host`는 `@Controller` 데코레이터의 `host` 값을 가져오게 되며 따로 설정은 안 해줬기 때문에 비어 있습니다.

```typescript
private applyHostFilter(
  host: string | RegExp | Array<string | RegExp>,
  handler: Function,
) {
  if (!host) {
    return handler;
  }

  const httpAdapterRef = this.container.getHttpAdapterRef();
  const hosts = Array.isArray(host) ? host : [host];
  const hostRegExps = hosts.map((host: string | RegExp) => {
    const keys = [];
    const regexp = pathToRegexp(host, keys);
    return { regexp, keys };
  });

  const unsupportedFilteringErrorMessage = Array.isArray(host)
  ? `HTTP adapter does not support filtering on hosts: ["${host.join(
    '", "',
  )}"]`
  : `HTTP adapter does not support filtering on host: "${host}"`;

  return <TRequest extends Record<string, any> = any, TResponse = any>(
    req: TRequest,
    res: TResponse,
    next: () => void,
    ) => {
      (req as Record<string, any>).hosts = {};
      const hostname = httpAdapterRef.getRequestHostname(req) || '';

      for (const exp of hostRegExps) {
        const match = hostname.match(exp.regexp);
        if (match) {
          if (exp.keys.length > 0) {
            exp.keys.forEach((key, i) => (req.hosts[key.name] = match[i + 1]));
          } else if (exp.regexp && match.groups) {
            for (const groupName in match.groups) {
              req.hosts[groupName] = match.groups[groupName];
            }
          }
          return handler(req, res, next);
        }
      }
      if (!next) {
        throw new InternalServerErrorException(
          unsupportedFilteringErrorMessage,
        );
      }
      return next();
    };
}
```

`host`가 없으면 받은 핸들러를 그대로 반환해주고, 만약 있다면 이번 요청의 `host`가 패턴과 일치하는지를 확인합니다. 일치하는 패턴이 있는 경우 `handler`를 호출하고, 없는 경우 `next()`를 호출해서 컨트롤러 호출 없이 넘어가는 필터를 추가해주고 있어요.

메서드 이름 그대로 host filter로 한 번 handler를 감싸는 모습을 볼 수 있습니다. 재밌는 패턴인 거 같아요!

다시 원래 코드로 돌아와서 다음으로 넘어가볼게요.

```typescript
paths.forEach((path) => {
  // ... 버전 관련 코드 생략
  routePathMetadata.methodPath = path;
  const pathsToRegister = this.routePathFactory.create(
    routePathMetadata,
    requestMethod
  );
  pathsToRegister.forEach((path) => {
    const entrypointDefinition: Entrypoint<HttpEntrypointMetadata> = {
      type: "http-endpoint",
      methodName,
      className: instanceWrapper.name,
      classNodeId: instanceWrapper.id,
      metadata: {
        key: path,
        path,
        requestMethod: RequestMethod[
          requestMethod
        ] as keyof typeof RequestMethod,
        methodVersion: routePathMetadata.methodVersion as VersionValue,
        controllerVersion: routePathMetadata.controllerVersion as VersionValue,
      },
    };

    routerMethodRef(path, routeHandler);

    this.graphInspector.insertEntrypointDefinition<HttpEntrypointMetadata>(
      entrypointDefinition,
      instanceWrapper.id
    );
  });

  const pathsToLog = this.routePathFactory.create(
    {
      ...routePathMetadata,
      versioningOptions: undefined,
    },
    requestMethod
  );
  pathsToLog.forEach((path) => {
    // 버전 관련 코드 생략
    this.logger.log(ROUTE_MAPPED_MESSAGE(path, requestMethod));
  });
});
```

여기도 조금씩 나눠서 볼게요.

```typescript
const pathsToRegister = this.routePathFactory.create(
  routePathMetadata,
  requestMethod
);
```

음. `RoutePathFactory`를 살펴보죠!

```typescript
export class RoutePathFactory {
  constructor(private readonly applicationConfig: ApplicationConfig) {}

  public create(
    metadata: RoutePathMetadata,
    requestMethod?: RequestMethod
  ): string[] {
    let paths = [""];

    // ... 버전 관련 코드 생략

    paths = this.appendToAllIfDefined(paths, metadata.modulePath);
    paths = this.appendToAllIfDefined(paths, metadata.ctrlPath);
    paths = this.appendToAllIfDefined(paths, metadata.methodPath);

    if (metadata.globalPrefix) {
      paths = paths.map((path) => {
        if (
          this.isExcludedFromGlobalPrefix(
            path,
            requestMethod,
            versionOrVersions,
            metadata.versioningOptions
          )
        ) {
          return path;
        }
        return stripEndSlash(metadata.globalPrefix || "") + path;
      });
    }

    return paths
      .map((path) => addLeadingSlash(path || "/"))
      .map((path) => (path !== "/" ? stripEndSlash(path) : path));
  }

  // ...
}
```

이름 그대로 path를 만들어주는 역할을 해요. 이렇게 변환해주는 프로세스가 없으면 경로가 하나의 패턴으로 정해지지 않는 문제가 발생할 수 있기 때문에 이러한 작업을 해주는 것으로 보여요. 깊이는 안 들어갈게요! 아무튼 경로를 만들어준다!

버저닝 관련 설정을 안 했다면 결과적으로 이쁘게 만들어진 경로 문자열 하나만 들어있는 배열이 반환될 것으로 보여요.

다음!

```typescript
pathsToRegister.forEach((path) => {
  const entrypointDefinition: Entrypoint<HttpEntrypointMetadata> = {
    type: "http-endpoint",
    methodName,
    className: instanceWrapper.name,
    classNodeId: instanceWrapper.id,
    metadata: {
      key: path,
      path,
      requestMethod: RequestMethod[requestMethod] as keyof typeof RequestMethod,
      methodVersion: routePathMetadata.methodVersion as VersionValue,
      controllerVersion: routePathMetadata.controllerVersion as VersionValue,
    },
  };

  routerMethodRef(path, routeHandler);

  this.graphInspector.insertEntrypointDefinition<HttpEntrypointMetadata>(
    entrypointDefinition,
    instanceWrapper.id
  );
});
```

뭔가 거창하게 많은데, Graph inspector ([아마 요거](https://docs.nestjs.com/devtools/overview)) 내용을 제외하면, 아래와 같이 아주 간단해져요.

```typescript
pathsToRegister.forEach((path) => {
  routerMethodRef(path, routeHandler);
});
```

여기서 `routerMethodRef`는 expressjs에 핸들러를 등록하는 역할을 하는 함수였는데, 이걸 드디어 호출했어요!

이제 ExpressJS에서 `path`로 요청이 들어오면, `routeHandler`를 호출해준다는 것까지 알아냈어요.

마지막으로 남은 코드까지 봅시다.

```typescript
const pathsToLog = this.routePathFactory.create(
  {
    ...routePathMetadata,
    versioningOptions: undefined,
  },
  requestMethod
);
pathsToLog.forEach((path) => {
  if (isVersioned) {
    const version = this.routePathFactory.getVersion(routePathMetadata);
    this.logger.log(
      VERSIONED_ROUTE_MAPPED_MESSAGE(path, requestMethod, version)
    );
  } else {
    this.logger.log(ROUTE_MAPPED_MESSAGE(path, requestMethod));
  }
});
```

이름 그대로 로그 찍어주는 코드에요. 위의 코드에 따르면 `ROUTE_MAPPED_MESSAGE` 함수를 호출하게 되는데, 이게 아래와 같이 생겼어요.

```typescript
// @nestjs/core/helpers/message.ts
export const ROUTE_MAPPED_MESSAGE = (path: string, method: string | number) =>
  `Mapped {${path}, ${RequestMethod[method]}} route`;
```

이거 어디서 많이 보지 않았나요? 네 맞아요! Nest 어플리케이션이 켜질 때 나와요!

![Mapped log](https://velog.velcdn.com/images/coalery/post/2522c7c5-b421-43f4-8087-21a10267b9a4/image.png)

`applyCallbackToRouter` 메서드는 이제 끝까지 알아봤어요. 여기까지가 우리의 목적이었지만, 조금만 더 가볼까요?

`applyCallbackToRouter` 메서드는 `applyPathsToRouterProxy` 메서드에서 다음과 같이 각 라우트에 대해 반복 호출돼요.

```typescript
public applyPathsToRouterProxy<T extends HttpServer>(
  router: T,
  routeDefinitions: RouteDefinition[],
  instanceWrapper: InstanceWrapper,
  moduleKey: string,
  routePathMetadata: RoutePathMetadata,
  host: string | RegExp | Array<string | RegExp>,
) {
  (routeDefinitions || []).forEach(routeDefinition => {
    const { version: methodVersion } = routeDefinition;
    routePathMetadata.methodVersion = methodVersion;

    this.applyCallbackToRouter(
      router,
      routeDefinition,
      instanceWrapper,
      moduleKey,
      routePathMetadata,
      host,
    );
  });
}
```

그럼, `routeDefinitions`는 어디서 올까요?

더 거슬러 올라가보면, `applyPathsToRouterProxy` 메서드는 `explore` 메서드에서 호출한다는 걸 알 수 있어요.

```typescript
public explore<T extends HttpServer = any>(
  instanceWrapper: InstanceWrapper,
  moduleKey: string,
  applicationRef: T,
  host: string | RegExp | Array<string | RegExp>,
  routePathMetadata: RoutePathMetadata,
) {
  const { instance } = instanceWrapper; // = 컨트롤러 인스턴스
  const routerPaths = this.pathsExplorer.scanForPaths(instance);
  this.applyPathsToRouterProxy(
    applicationRef,
    routerPaths,
    instanceWrapper,
    moduleKey,
    routePathMetadata,
    host,
  );
}
```

여기서 `routerPaths` = `routeDefinitions`에요. 그럼 `PathsExplorer`를 안 살펴볼 수 없겠죠!

```typescript
// @nestjs/core/router/paths-explorer.ts
public scanForPaths(
  instance: Controller,
  prototype?: object,
): RouteDefinition[] {
  const instancePrototype = isUndefined(prototype)
    ? Object.getPrototypeOf(instance)
    : prototype;

  return this.metadataScanner
    .getAllMethodNames(instancePrototype)
    .reduce((acc, method) => {
      const route = this.exploreMethodMetadata(
        instance,
        instancePrototype,
        method,
      );

      if (route) {
        acc.push(route);
      }

      return acc;
    }, []);
}
```

`instancePrototype`은 `prototype`이 `undefined`이므로 `instance`의 프로토타입을 가져오게 되며, 즉 컨트롤러 클래스의 프로토타입을 가져옵니다.

`getAllMethodNames`는 이름에서도 "주어진 프로토타입의 모든 메서드 이름 조회"라는 기능을 유추할 수 있으므로 코드까지 보는건 생략할게요.

그러면 컨트롤러 내에 있는 모든 메서드의 이름을 가져왔고, `exploreMethodMetadata`에서 라우트 핸들러인 메서드의 정보만 가져오는 것으로 보여요.

```typescript
public exploreMethodMetadata(
  instance: Controller,
  prototype: object,
  methodName: string,
): RouteDefinition | null {
  const instanceCallback = instance[methodName];
  const prototypeCallback = prototype[methodName];
  const routePath = Reflect.getMetadata(PATH_METADATA, prototypeCallback);
  if (isUndefined(routePath)) {
    return null;
  }
  const requestMethod: RequestMethod = Reflect.getMetadata(
    METHOD_METADATA,
    prototypeCallback,
  );
  const version: VersionValue | undefined = Reflect.getMetadata(
    VERSION_METADATA,
    prototypeCallback,
  );
  const path = isString(routePath)
  ? [addLeadingSlash(routePath)]
  : routePath.map((p: string) => addLeadingSlash(p));

  return {
    path,
    requestMethod,
    targetCallback: instanceCallback,
    methodName,
    version,
  };
}
```

`PATH_METADATA = 'path'` 메타데이터를 조회해서, 존재하지 않으면 라우트 핸들러가 아니기 때문에 `null`을 반환합니다.

라우트 핸들러라고 한다면 해당 라우트의 http method와 버전 정보를 가져와서 `RouteDefinition`으로 반환하게 됩니다.

이때, `PATH_METADATA`, `METHOD_METADATA`, `VERSION_METADATA`는 `@Get`, `@Post` 등의 데코레이터를 통해 붙게 되며, `path`를 생략하더라도 기본 값으로 `'/'`이 들어가게 됩니다.

예를 들어 `@Get`을 볼게요.

```typescript
// @nestjs/common/decorators/http/request-mapping.decorator.ts
const defaultMetadata = {
  [PATH_METADATA]: "/",
  [METHOD_METADATA]: RequestMethod.GET,
};

export const RequestMapping = (
  metadata: RequestMappingMetadata = defaultMetadata
): MethodDecorator => {
  const pathMetadata = metadata[PATH_METADATA];
  const path = pathMetadata && pathMetadata.length ? pathMetadata : "/";
  const requestMethod = metadata[METHOD_METADATA] || RequestMethod.GET;

  return (
    target: object,
    key: string | symbol,
    descriptor: TypedPropertyDescriptor<any>
  ) => {
    Reflect.defineMetadata(PATH_METADATA, path, descriptor.value);
    Reflect.defineMetadata(METHOD_METADATA, requestMethod, descriptor.value);
    return descriptor;
  };
};

const createMappingDecorator =
  (method: RequestMethod) =>
  (path?: string | string[]): MethodDecorator => {
    return RequestMapping({
      [PATH_METADATA]: path,
      [METHOD_METADATA]: method,
    });
  };

export const Get = createMappingDecorator(RequestMethod.GET);
```

위와 같이 `defaultMetadata`에 기본적으로 `'/'`가 들어가있고, 각 값이 메타데이터에 들어가게 됩니다.

여기까지! 이렇게 한 컨트롤러에 있는 모든 라우트 핸들러 정보를 불러와서 Express에 등록하는 것을 봤습니다.

이후는 간단하게만 살펴보자면,

1. `RouteExplorer#explore`는 [`RoutesResolver#registerRouters`](https://github.com/nestjs/nest/blob/26e2886e6b11d35bcb685fe997629614a08fe1db/packages/core/router/routes-resolver.ts#L131-L137)에서 호출됩니다.
2. `registerRouters` 메서드는 [`RoutesResolver#resolve`](https://github.com/nestjs/nest/blob/26e2886e6b11d35bcb685fe997629614a08fe1db/packages/core/router/routes-resolver.ts#L74-L80) 메서드에서 호출됩니다.
3. `resolve` 메서드는 [`NestApplication#registerRouter`](https://github.com/nestjs/nest/blob/26e2886e6b11d35bcb685fe997629614a08fe1db/packages/core/nest-application.ts#L213) 메서드에서 호출됩니다.
4. `registerRouter` 메서드는 [`NestApplication#init`](https://github.com/nestjs/nest/blob/26e2886e6b11d35bcb685fe997629614a08fe1db/packages/core/nest-application.ts#L192) 메서드에서 호출됩니다.
5. `init` 메서드는 [`NestApplication#listen`](https://github.com/nestjs/nest/blob/26e2886e6b11d35bcb685fe997629614a08fe1db/packages/core/nest-application.ts#L295) 메서드에서 호출됩니다.
6. `listen` 메서드는 우리가 `NestFactory.create`를 통해 만든 `INestApplication`으로 `main.ts` 파일에서 호출합니다.

조금 의외였던 부분은 `NestFactory.create` 부분에서 라우트 등록도 할 줄 알았는데, `listen` 메서드에서 한다는 거였네요.

정말 먼 길을 돌아왔어요. 이제 다시 돌아갈 때에요.

저어 위에서 봤었던 `route-explorer.ts` 파일로 다시 돌아가볼게요.

```typescript
// @nestjs/core/router/router-explorer.ts
private createCallbackProxy(
  instance: Controller,
  callback: RouterProxyCallback,
  methodName: string,
  moduleRef: string,
  requestMethod: RequestMethod,
  contextId = STATIC_CONTEXT,
  inquirerId?: string,
) {
  const executionContext = this.executionContextCreator.create(
    instance,
    callback,
    methodName,
    moduleRef,
    requestMethod,
    contextId,
    inquirerId,
  );
  const exceptionFilter = this.exceptionsFilter.create(
    instance,
    callback,
    moduleRef,
    contextId,
    inquirerId,
  );
  return this.routerProxy.createProxy(executionContext, exceptionFilter);
}
```

우리는 여기서 `createCallbackProxy`가 어디서 호출되는지 따라가다가 결국 끝까지 가버렸어요.

일단 여기까지 정리해볼게요.

- `createCallbackProxy`의 두 번째 매개변수인 `callback`을 호출하면 컨트롤러의 코드를 호출해요.
- `routerProxy.createProxy`는 `executionContext`(얘 함수에요!)를 호출하는 곳을 감싼 익명 함수를 반환해요.

그럼 자연스럽게 `executionContext`에서 `callback`을 호출하는 부분이 있을 것이라 생각할 수 있어요. 자, 이제 `executionContext`가 어떻게 만들어지는지 보러 갈 차례에요.

## 두 번째 trace, router-execution-context

```typescript
// @nestjs/core/router/router-execution-context.ts
public create(
  instance: Controller,
  callback: (...args: any[]) => unknown,
  methodName: string,
  moduleKey: string,
  requestMethod: RequestMethod,
  contextId = STATIC_CONTEXT,
  inquirerId?: string,
) {
  // ...
  const handler =
    <TRequest, TResponse>(
      args: any[],
      req: TRequest,
      res: TResponse,
      next: Function,
    ) =>
    async () => {
      // @1
      fnApplyPipes && (await fnApplyPipes(args, req, res, next));
      return callback.apply(instance, args);
    };

  return async <TRequest, TResponse>(
    req: TRequest,
    res: TResponse,
    next: Function,
  ) => {
    const args = this.contextUtils.createNullArray(argsLength);

    // @2
    fnCanActivate && (await fnCanActivate([req, res, next]));

    // ...

    // @3
    const result = await this.interceptorsConsumer.intercept(
      interceptors,
      [req, res, next],
      instance,
      callback,
      handler(args, req, res, next),
      contextType,
    );
    // ...
  };
}
```

여기는 해당 부분의 코드를 생략했지만 파이프, 가드, 인터셉터를 실제로 적용하는 부분으로 보여요.

저기까지 살펴보기에는 이미 너무 많은 길을 왔기 때문에, 해당 부분은 다음 글을 통해 알아볼게요. 실제 코드를 보고 싶으신 분들은 [여기](https://github.com/nestjs/nest/blob/26e2886e6b11d35bcb685fe997629614a08fe1db/packages/core/router/router-execution-context.ts#L80-L175)를 참고해주세요.

아무튼, 위 코드에서 `@1` 표시해둔 부분이 파이프를 실제로 적용하는 것으로 보이고, `@2`가 가드, `@3`이 인터셉터를 적용하는 것으로 보여요.

참고로, stack trace에서 router-proxy.js 바로 위에 있던 router-execution-context.js 호출하는 부분이 바로 저 `interceptorsConsuemr.intercept` 부분입니다.

이제 다음 행선지로 떠나볼까요? 다시 stack trace를 볼게요. ExpressJS 관련 부분은 제외했어요.

```
Error:
    at AppController.getHello (/Users/lery/abc/abc/src/app.controller.ts:9:11)
    at /Users/lery/abc/abc/node_modules/@nestjs/core/router/router-execution-context.js:38:29
    at InterceptorsConsumer.intercept (/Users/lery/abc/abc/node_modules/@nestjs/core/interceptors/interceptors-consumer.js:12:20)
    at /Users/lery/abc/abc/node_modules/@nestjs/core/router/router-execution-context.js:46:60
    at /Users/lery/abc/abc/node_modules/@nestjs/core/router/router-proxy.js:9:23
```

이제 `interceptors-consumer.js` 부분을 살펴보러 가야 할 거 같네요.

## 세 번째 trace: interceptors-consumer

```typescript
// @nestjs/core/interceptors/interceptors-consumer.ts
public async intercept<TContext extends string = ContextType>(
  interceptors: NestInterceptor[],
  args: unknown[],
  instance: Controller,
  callback: (...args: unknown[]) => unknown,
  next: () => Promise<unknown>,
  type?: TContext,
): Promise<unknown> {
  if (isEmpty(interceptors)) {
    return next();
  }
  const context = this.createContext(args, instance, callback);
  context.setType<TContext>(type);

  const nextFn = async (i = 0) => {
    if (i >= interceptors.length) {
      return defer(AsyncResource.bind(() => this.transformDeferred(next)));
    }
    const handler: CallHandler = {
      handle: () => fromPromise(nextFn(i + 1)).pipe(mergeAll()),
    };
    return interceptors[i].intercept(context, handler);
  };
  return defer(() => nextFn()).pipe(mergeAll());
}
```

사실 에러를 일으킨 코드에서 인터셉터를 사용하고 있지 않기 떄문에, 바로 `next()`를 호출하게 돼요. 이때 `next`로 들어오는 함수가 바로 파이프, 가드가 적용된 그 핸들러에요.

그래도 들어오긴 했으니 안 살펴볼 순 없겠죠? 여긴 그래도 간단?해요!

```typescript
const context = this.createContext(args, instance, callback);
context.setType<TContext>(type);
```

`context`를 생성하고 `type`을 설정해주는데, 이때 `type`은 `http`입니다.

또, `createContext`는 `ExecutionContextHost` 객체를 생성하는데 해당 클래스는 `ExecutionContext` 인터페이스의 구현체에요.

네, 맞아요! 인터셉터 구현할 때 첫 번째 매개변수로 들어오는 그 변수가 바로 여기서 생성돼요.

하나하나 퍼즐이 맞춰가는 기분이 들지 않나요!

그 외에는 rxjs의 처리를 위한 코드가 대부분이고, 주목해봐야 할 코드는 요거 밖에 없는 거 같아요.

```typescript
return interceptors[i].intercept(context, handler);
```

여기서 우리가 만든 인터셉터들을 실제로 호출해주게 됩니다.

대충 살펴본 거 같으니, 다음으로 넘어가볼게요.

## 네 번째 trace, 다시 router-execution-context

```typescript
// @nestjs/core/router/router-execution-context.ts
public create(
  instance: Controller,
  callback: (...args: any[]) => unknown,
  methodName: string,
  moduleKey: string,
  requestMethod: RequestMethod,
  contextId = STATIC_CONTEXT,
  inquirerId?: string,
) {
  // ...
  const handler =
    <TRequest, TResponse>(
      args: any[],
      req: TRequest,
      res: TResponse,
      next: Function,
    ) =>
    async () => {
      // @1
      fnApplyPipes && (await fnApplyPipes(args, req, res, next));
      return callback.apply(instance, args);
    };

  return async <TRequest, TResponse>(
    req: TRequest,
    res: TResponse,
    next: Function,
  ) => {
    const args = this.contextUtils.createNullArray(argsLength);
    fnCanActivate && (await fnCanActivate([req, res, next]));

    // ...

    const result = await this.interceptorsConsumer.intercept(
      interceptors,
      [req, res, next],
      instance,
      callback,
      handler(args, req, res, next),
      contextType,
    );
    // ...
  };
}
```

아까 위에서 살펴봤던 코드 있죠? 거기서 `handler`라고 선언된 함수의 `callback.apply`에서 에러가 발생해요. 이때 `callback`은 `Function` 타입이고, `Function.apply`는 첫 번째 매개변수를 `this`로, 그리고 두 번째 매개변수를 해당 함수의 매개변수로서 함수를 호출해요. [참고(MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply)

즉, 이렇게저렇게 지지고 볶아졌던 핸들러가 실제로 호출되는 부분이에요. 그러면 프록시를 타고 컨트롤러가 실제로 호출되게 됩니다.

## 5번째 trace, 우리의 코드

여기부터는 우리 코드에요! 바로 우리가 고쳐야 하는 버그기도 하죠 (..)

---

# 마치며.

이번에도 정말 긴 글이 하나 뚝딱 나왔어요.

실제로 trace를 따라가보는 글은 크게 길지 않았지만, ExpressJS에서 NestJS로 연결되는 부분을 알아보는 데에 정말 많은 양의 글이 작성됐어요.

저번에 의존성을 어떻게 주입해주는지 알아볼 때도 그렇고, 이번에 라우트를 처리하는 방법을 알아볼 때도 그렇고, 결국 개발에 마법은 없다는 걸 항상 느껴요.

누군가가 정의한 규칙에 의해 코드가 작성되고, 그 코드에 의해 우리의 규칙을 반영하는 코드들이 맞물려 돌아간다는 거죠. 이런 것들을 하나하나 찾아갈 때마다 큰 재미를 느끼는 것 같아요.

아무튼 다음 글에서는 위에서 생략했던 파이프, 가드, 인터셉터들이 실제로 어떻게 적용되는지를 알아보려 해요. 저번 글과 비교해서 1년 정도의 시간이 흘렀는데, 다음 글은 또 언제 나올지 모르겠네요.

여러분들이 NestJS를 더 잘 이해하면서 사용하는 데에 도움이 되었으면 좋을 것 같아요.

긴 글 읽어주셔서 감사합니다.

---

글: https://velog.io/@coalery/nest-route-how
