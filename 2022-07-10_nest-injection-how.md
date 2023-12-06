# Nest.js는 실제로 어떻게 의존성을 주입해줄까?

이 글은 Nest.js v8.4.7을 기준으로 합니다.

[Nest.js](https://nestjs.com/)(이하 네스트)를 한 번이라도 써보셨다면, 네스트의 강력한 의존성 주입 기능을 사용해보셨을 겁니다! 데코레이터 몇 개만 달아줬다고 뚝딱 원하는 의존성을 넣어주는데요.

일단 주입해주니 받아서 잘 쓰긴 했는데.. 네스트는 어떻게 알고 이렇게 주입을 해주는걸까요?
자. 삽을 들어봅시다.

당연하겠지만, 과정 자체가 상당히 복잡하고 내용이 많습니다. 따라서 아래는 적당히 가지치기를 한 글이기 때문에, 대부분의 메서드는 내부적으로 어떻게 동작하는지만 설명하고 실제 코드를 보여드리진 않습니다.

실제 코드가 궁금하신 분은, [네스트 리포지토리](https://github.com/nestjs/nest)에서 찾아보시면 감사하겠습니다.

## 과정

아래의 글은 "네스트는 어떻게 의존성을 주입해주는가?"라는 질문의 대답에 도달하기까지의 과정을 서술하고 있습니다. 결론을 원하신다면 가장 아래의 '결론' 부분을 참고하시면 될 것 같아요!

### 기준이 되는 코드

네스트의 내용이 너무 방대하기 때문에, 기준을 세워두지 않으면 너무나도 거대한 삽질이 되어버립니다. 그렇기 때문에, 먼저 기준이 되는 코드를 보겠습니다.

```shell
$ nest new
```

위의 명령어를 통해 새로운 네스트 프로젝트를 만들 수 있습니다. 만들어지는 파일들 중에서 테스트 파일, 즉 `.spec.ts`로 끝나는 파일들은 제외하고, `main.ts`와 모듈, 컨트롤러, 서비스 파일만 보겠습니다.

```typescript
// main.ts
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

```typescript
// app.module.ts
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";

@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

```typescript
// app.controller.ts
import { Controller, Get } from "@nestjs/common";
import { AppService } from "./app.service";

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}
```

```typescript
// app.service.ts
import { Injectable } from "@nestjs/common";

@Injectable()
export class AppService {
  getHello(): string {
    return "Hello World!";
  }
}
```

이렇게 네 개의 파일이 준비되었습니다. 그럼 시작해볼까요?

### 공식문서의 설명

아무래도 네스트의 중심이 되는 기능이다보니, [공식문서](https://docs.nestjs.com/fundamentals/custom-providers#di-fundamentals)를 찾아보면 관련 설명이 존재합니다.

> 1. In cats.service.ts, the @Injectable() decorator declares the CatsService class as a class that can be managed by the Nest IoC container.
> 2. In cats.controller.ts, CatsController declares a dependency on the CatsService token with constructor injection:
>    `constructor(private catsService: CatsService)`
> 3. In app.module.ts, we associate the token CatsService with the class CatsService from the cats.service.ts file. We'll see below exactly how this association (also called registration) occurs.

간단하게 정리해보자면,

1. 주입할 프로바이더에 `@Injectable` 데코레이터를 붙이면 Nest IoC 컨테이너가 해당 프로바이더를 관리할 수 있다고 선언합니다.
2. 주입 받을 곳에는 주입 받을 프로바이더의 토큰을 생성자 주입 방식으로 선언합니다.
3. 모듈에 둘을 등록하면, 둘을 연결시켜줍니다.

이것 그 이상의 설명을 공식문서에서는 찾을 수 없었습니다.
하지만.. 궁금하니 어쩌겠어요! 코드를 파봐야겠죠?!

### 근데... 어디부터 파야해요?

![죠르디와 앙몬드가 먼 산을 보고 있음](https://velog.velcdn.com/images/coalery/post/4180892c-a617-45cc-800e-3a3a84aeaa44/image.png)

일단 삽은 들었는데, 어디부터 찔러봐야 하는걸까요?

흠... 먼저 모듈 파일부터 시작해봅시다.

```typescript
// packages/common/decorators/modules/module.decorator.ts
export function Module(metadata: ModuleMetadata): ClassDecorator {
  const propsKeys = Object.keys(metadata);
  validateModuleKeys(propsKeys);

  return (target: Function) => {
    // 클래스 데코레이터
    for (const property in metadata) {
      if (metadata.hasOwnProperty(property)) {
        // imports, controllers, providers, exports 만 이 안에 들어옵니다.
        // 프로퍼티를 각각의 이름을 키로 클래스의 메타데이터에 정의합니다.

        // @Module({
        //   providers: [SomeProvider]
        // })
        // export class SomeModule
        // 위와 같이 정의되었다면, SomeModule 클래스에 `providers`를 키로,
        // `[SomeProvider]`를 값으로 하는 메타데이터가 정의되는 것입니다.
        Reflect.defineMetadata(property, (metadata as any)[property], target);
      }
    }
  };
}
```

생각했던 것 보다는 그렇게 복잡하진 않네요. 사실상 모듈 클래스에 메타데이터를 정의하는 것 외에는 하는 게 없습니다.

### 모듈 다음은..?

`@Module` 데코레이터는 크게 의미가 있는 로직은 없었습니다. 다음은 `@Injectable` 데코레이터를 볼까요?

```typescript
// packages/common/decorators/core/injectable.decorator.ts
export function Injectable(options?: InjectableOptions): ClassDecorator {
  return (target: object) => {
    Reflect.defineMetadata(INJECTABLE_WATERMARK, true, target);
    Reflect.defineMetadata(SCOPE_OPTIONS_METADATA, options, target);
  };
}
```

모듈 데코레이터보다 훨씬 더 간단합니다. 그냥 데코레이터 두 개 붙이는 게 끝이네요.
여기도 크게 중요한 로직이 있는 것 같아 보이진 않아요.

### 컨트롤러는 어떤가요

```typescript
// packages/common/decorators/core/controller.decorator.ts
export function Controller(
  prefixOrOptions?: string | string[] | ControllerOptions
): ClassDecorator {
  const defaultPath = "/";

  const [path, host, scopeOptions, versionOptions] = isUndefined(
    prefixOrOptions
  )
    ? [defaultPath, undefined, undefined, undefined]
    : isString(prefixOrOptions) || Array.isArray(prefixOrOptions)
    ? [prefixOrOptions, undefined, undefined, undefined]
    : [
        prefixOrOptions.path || defaultPath,
        prefixOrOptions.host,
        { scope: prefixOrOptions.scope },
        Array.isArray(prefixOrOptions.version)
          ? Array.from(new Set(prefixOrOptions.version))
          : prefixOrOptions.version,
      ];

  return (target: object) => {
    Reflect.defineMetadata(CONTROLLER_WATERMARK, true, target);
    Reflect.defineMetadata(PATH_METADATA, path, target);
    Reflect.defineMetadata(HOST_METADATA, host, target);
    Reflect.defineMetadata(SCOPE_OPTIONS_METADATA, scopeOptions, target);
    Reflect.defineMetadata(VERSION_METADATA, versionOptions, target);
  };
}
```

좀 더 로직이 많아졌지만, 경로를 처리하는 것 말고는 크게 눈에 띄는 로직은 없네요.
대신, 등록하는 메타데이터가 엄청 많아보이는 건 있습니다.

요기도 넘어가도 될 거 같아요.

### 남은 건...

`main.ts` 파일 밖에 없어요. 그렇다면, 이건 여기서 의존성 처리가 완료된다는 뜻이에요! 이건 이것대로 가슴이 뛰네요!

눈에 띄는건 `NestFactory.create`에요. 해당 메서드를 조금 더 파볼게요.

```typescript
// packages/core/nest-factory.ts
public async create<T extends INestApplication = INestApplication>(
  module: any,
  serverOrOptions?: AbstractHttpAdapter | NestApplicationOptions,
  options?: NestApplicationOptions,
): Promise<T> {
  const [httpServer, appOptions] = this.isHttpServer(serverOrOptions)
    ? [serverOrOptions, options]
    : [this.createHttpAdapter(), serverOrOptions];
  // isHttpServer에서 false를 반환하므로, 뒤의 배열을 사용합니다.

  const applicationConfig = new ApplicationConfig();
  const container = new NestContainer(applicationConfig); // @1
  this.setAbortOnError(serverOrOptions, options);
  this.registerLoggerConfiguration(appOptions);

  await this.initialize(module, container, applicationConfig, httpServer); // @2

  const instance = new NestApplication(
    container,
    httpServer,
    applicationConfig,
    appOptions,
  );
  const target = this.createNestInstance(instance);
  return this.createAdapterProxy<T>(target, httpServer);
}

// packages/core/nest-factory.ts L257
private isHttpServer(
  serverOrOptions: AbstractHttpAdapter | NestApplicationOptions,
): serverOrOptions is AbstractHttpAdapter {
  return !!(
    serverOrOptions && (serverOrOptions as AbstractHttpAdapter).patch
  );
}
```

여기서는 `NestContainer`를 만드는 @1과, 무언가 초기화를 하는 것 같아 보이는 @2가 눈에 띄네요.

#### NestContainer

먼저 `NestContainer`를 살펴보겠습니다. [깃헙](https://github.com/nestjs/nest/blob/master/packages/core/injector/container.ts)

```typescript
// packages/core/injector/container.ts
export class NestContainer {
  private readonly globalModules = new Set<Module>();
  private readonly moduleTokenFactory = new ModuleTokenFactory();
  private readonly moduleCompiler = new ModuleCompiler(this.moduleTokenFactory);
  private readonly modules = new ModulesContainer();
  private readonly dynamicModulesMetadata = new Map<
    string,
    Partial<DynamicModule>
  >();
  private readonly internalProvidersStorage = new InternalProvidersStorage();
  private internalCoreModule: Module;

  // ...

  public async addModule(
    metatype: Type<any> | DynamicModule | Promise<DynamicModule>,
    scope: Type<any>[]
  ): Promise<Module | undefined> {
    /* ... */
  }

  public addProvider(
    provider: Provider,
    token: string
  ): string | symbol | Function {
    /* ... */
  }

  public addInjectable(
    injectable: Provider,
    token: string,
    host?: Type<Injectable>
  ) {
    /* ... */
  }

  public addController(controller: Type<any>, token: string) {
    /* ... */
  }

  // ...
}
```

프로퍼티들이나 메서드들의 이름을 봤을 때, 모듈을 관리하는 클래스라고 볼 수 있을 거 같아요.

#### initialize

이제 `initialize` 메서드를 볼께요.

```typescript
// packages/core/nest-factory.ts
private async initialize(
  module: any,
  container: NestContainer,
  config = new ApplicationConfig(),
  httpServer: HttpServer = null,
) {
  const instanceLoader = new InstanceLoader(container);
  const metadataScanner = new MetadataScanner();
  const dependenciesScanner = new DependenciesScanner(
    container,
    metadataScanner,
    config,
  );
  container.setHttpAdapter(httpServer);

  const teardown = this.abortOnError === false ? rethrow : undefined;
  await httpServer?.init();
  try {
    this.logger.log(MESSAGES.APPLICATION_START);

    await ExceptionsZone.asyncRun(
      async () => {
        await dependenciesScanner.scan(module); // @1
        await instanceLoader.createInstancesOfDependencies(); // @2
        dependenciesScanner.applyApplicationProviders(); // @3
      },
      teardown,
      this.autoFlushLogs,
    );
  } catch (e) {
    this.handleInitializationError(e);
  }
}
```

여기서는 세 줄이 눈에 띄어요! 이름만 봤을 때 각각이 하는 일을 예상해보자면,

1. 모든 의존성을 스캔하는 역할
2. 스캔한 의존성들을 인스턴스화하는 역할
3. 인스턴스화한 의존성들의 인스턴스를 적용하는 역할

아직까지는 예상 단계에요. 대충 그렇겠구나, 라고 생각하면서 이제 실제로 어떻게 동작하는지 더 파고 들어가보자구요!

### 네스트 초기화 단계 서론

본론에 들어가기 전에, 위의 세 메서드들이 속한 `InstanceLoader`와 `DependenciesScanner`이 무엇인지 알아야 이해할 수 있을 거 같아요.

먼저 `InstanceLoader`부터 볼게요.

#### InstanceLoader

이름만 봤을 때는 무슨 생각이 드시나요? 너무 직관적이어서, 사실 역할에 대해서는 큰 설명이 필요할 거 같진 않습니다. 내부의 프로퍼티나 메서드만 살짝 보고 넘어가볼게요.

```typescript
// packages/core/injector/instance-loader.ts
export class InstanceLoader {
  protected readonly injector = new Injector();
  constructor(
    protected readonly container: NestContainer // ...
  ) {}

  public async createInstancesOfDependencies(
    modules: Map<string, Module> = this.container.getModules()
  ) {
    /* ... */
  }
}
```

프로퍼티에서는 `Injector`라는 걸 새로 만들어주네요. 의존성을 주입하는 역할을 하는 거 같아요! `Injector`는 본론 들어갔을 때 다시 볼게요.

주목할 만한 점은, 메서드 중에서 `createInstancesOfDependencies`를 제외하고는 모두 `private` 메서드에요. 이것도 본론 들어갔을 때 다시 볼게요!

지금은 대충 이런 형태구나 정도만 느끼시면 될 거 같습니다.

#### MetadataScanner

`DependenciesScanner`를 알아보기 전에, 생성자의 매개변수로 들어가는 `MetadataScanner`를 먼저 알아보겠습니다.

```typescript
// packages/core/metadata-scanner.ts
export class MetadataScanner {
  public scanFromPrototype<T extends Injectable, R = any>(
    instance: T,
    prototype: object,
    callback: (name: string) => R
  ): R[] {
    // 모든 메서드들의 이름
    const methodNames = new Set(this.getAllFilteredMethodNames(prototype));

    return iterate(methodNames)
      .map(callback)
      .filter((metadata) => !isNil(metadata))
      .toArray();
  }

  *getAllFilteredMethodNames(prototype: object): IterableIterator<string> {
    // 메서드인가를 판단
    const isMethod = (prop: string) => {
      const descriptor = Object.getOwnPropertyDescriptor(prototype, prop);
      // setter나 getter가 있으면 메서드가 아님.
      if (descriptor.set || descriptor.get) {
        return false;
      }
      // 생성자가 아니고, 함수면 메서드이다.
      return !isConstructor(prop) && isFunction(prototype[prop]);
    };

    do {
      yield* iterate(Object.getOwnPropertyNames(prototype)) // 모든 프로퍼티 중에서
        .filter(isMethod) // 메서드인 것만 골라서
        .toArray(); // 배열로 반환
    } while (
      (prototype = Reflect.getPrototypeOf(prototype)) &&
      prototype !== Object.prototype // 상속 받았다면, 가장 위의 부모 클래스까지 반복
    );
  }
}
```

특정 프로토타입 객체를 넘겨주면 가장 위의 부모 클래스, 즉 부모가 `Object`인 클래스까지 프로토타입 체인을 타고 올라가면서 모든 메서드들의 이름을 가져옵니다.

각각의 메서드 이름에 대해 `callback`을 호출한 결과를 배열로 반환하는 역할을 합니다.

저런 기법은 처음 보는 것들이라 꽤 재밌네요! 계속 가봅시다!

#### DependenciesScanner

요 클래스가 속해있는 파일도 상당히 커서, 나중에 본론으로 들어갔을 때 제대로 살펴보겠습니다. 간단히 프로퍼티나 메서드만 살펴볼게요!

```typescript
// packages/core/scanner.ts
export class DependenciesScanner {
  // ...
  private readonly applicationProvidersApplyMap: ApplicationProviderWrapper[] =
    [];

  constructor(
    private readonly container: NestContainer,
    private readonly metadataScanner: MetadataScanner,
    private readonly applicationConfig = new ApplicationConfig()
  ) {}

  public async scan(module: Type<any>) {
    /* ... */
  }

  public async scanForModules(
    moduleDefinition:
      | ForwardReference
      | Type<unknown>
      | DynamicModule
      | Promise<DynamicModule>,
    scope: Type<unknown>[] = [],
    ctxRegistry: (ForwardReference | DynamicModule | Type<unknown>)[] = []
  ): Promise<Module[]> {
    /* ... */
  }

  public applyApplicationProviders() {
    /* ... */
  }

  // ...
}
```

여기서는 크게 눈에 띄는 건 없네요. 실제 코드가 없어서 그런 거 같아요.

이제 본론으로 들어가볼까요!

### 네스트 초기화 단계 본론

필요한 건 다 돌아본 것 같으니 시작해봅시다. 점점 어려워지는 것 같아서 무섭지만.. 계속 가봐야겠죠?

다시 `initialize` 메서드로 돌아와서,

```typescript
// packages/core/nest-factory.ts
private async initialize(
  module: any,
  container: NestContainer,
  config = new ApplicationConfig(),
  httpServer: HttpServer = null,
) {
  const instanceLoader = new InstanceLoader(container);
  const metadataScanner = new MetadataScanner();
  const dependenciesScanner = new DependenciesScanner(
    container,
    metadataScanner,
    config,
  );
  container.setHttpAdapter(httpServer);

  const teardown = this.abortOnError === false ? rethrow : undefined;
  await httpServer?.init();
  try {
    this.logger.log(MESSAGES.APPLICATION_START);

    await ExceptionsZone.asyncRun(
      async () => {
        await dependenciesScanner.scan(module); // @1
        await instanceLoader.createInstancesOfDependencies(); // @2
        dependenciesScanner.applyApplicationProviders(); // @3
      },
      teardown,
      this.autoFlushLogs,
    );
  } catch (e) {
    this.handleInitializationError(e);
  }
}
```

다른 부분은 제외하고, @1, @2, @3 부분을 중점적으로 살펴볼 거에요. 사실 다른 부분은 크게 주입이랑은 관련이 없어보이기도 하구요. 우리의 목적은 Nest.js가 전체적으로 어떻게 동작하는지를 알아보는게 아니라, 의존성을 어떻게 주입해주는지 알아보는 것이니 필요 없는 것들은 이렇게 가지치기를 하면서 진행할게요!

#### DependenciesScanner.scan

@1, @2, @3 중에서 가장 먼저 호출되는 @1인 scan 메서드는 어떻게 생겼는지 봅시다.

```typescript
// packages/core/scanner.ts
public async scan(module: Type<any>) {
  await this.registerCoreModule();
  await this.scanForModules(module);
  await this.scanModulesForDependencies();
  this.calculateModulesDistance();

  this.addScopedEnhancersMetadata();
  this.container.bindGlobalScope();
}
```

여러 메서드들을.. 참 많이 호출하고 있네요.
먼저 `registerCoreModule`부터 봅시다.

```typescript
// packages/core/scanner.ts
public async registerCoreModule() {
  const moduleDefinition = InternalCoreModuleFactory.create(
    this.container,
    this,
    this.container.getModuleCompiler(),
    this.container.getHttpAdapterHostRef(),
  );
  const [instance] = await this.scanForModules(moduleDefinition);
  this.container.registerCoreModuleRef(instance);
}
```

코어 모듈을 팩토리로부터 만든 뒤에, 모듈을 스캔해서 인스턴스를 받아오고 이걸 컨테이너에 등록하는 거 같아요. 요 부분은 느낌상 주입과 관련이 없어보이니 넘어가요!

다음은 `scanForModules`에요. 방금 위에서 나왔던 `registerCoreModule` 메서드에서도 잠시 등장했는데, 어떻게 생겼는지 한 번 볼까요?

```typescript
// packages/core/scanner.ts
public async scanForModules(
  moduleDefinition:
    | ForwardReference
    | Type<unknown>
    | DynamicModule
    | Promise<DynamicModule>,
  scope: Type<unknown>[] = [],
  ctxRegistry: (ForwardReference | DynamicModule | Type<unknown>)[] = [],
): Promise<Module[]> {
  // scope = []
  // ctxRegistry = []

  const moduleInstance = await this.insertModule(moduleDefinition, scope);
  // 1. moduleDefinition이 ForwardRef 라면 .forwardRef 메서드 호출
  // 2. 모듈이 Injectable이거나, 컨트롤러거나, 예외 필터면 경고 로그 출력
  // 3. 모듈 컴파일 -> 타입, 동적 메타데이터, 토큰을 가져옴
  // 4. 이를 기반으로 새로운 모듈 객체 생성
  // 5. 컨테이너의 modules(ModulesContainer)에 토큰과 모듈을 등록
  // 6. 만들어진 모듈 객체를 반환

  moduleDefinition =
    moduleDefinition instanceof Promise
      ? await moduleDefinition
      : moduleDefinition;
  // 비동기적으로 모듈이 정의되었을 때 await를 적용합니다.

  ctxRegistry.push(moduleDefinition);
  // ctxRegistry = [moduleDefinition]

  if (this.isForwardReference(moduleDefinition)) {
    moduleDefinition = (moduleDefinition as ForwardReference).forwardRef();
  }
  // 모듈이 ForwardRef라면, 메서드를 실행하여 실제 모듈로 변환합니다.

  const modules = !this.isDynamicModule(
    moduleDefinition as Type<any> | DynamicModule,
  ) // 동적 모듈이 아니면
    ? this.reflectMetadata(
        moduleDefinition as Type<any>,
        MODULE_METADATA.IMPORTS,
      ) // 모듈의 imports에 들어가있는 배열을 가져옵니다.
    : [
        ...this.reflectMetadata(
          (moduleDefinition as DynamicModule).module,
          MODULE_METADATA.IMPORTS, // = 'imports'
        ),
        ...((moduleDefinition as DynamicModule).imports || []),
      ];
  // 즉, 주어진 모듈의 imports 배열에 들어가있는 모듈들입니다.
  // 제일 위에서 Module 데코레이터의 내용을 봤었습니다.
  // 기억이 안 나신다면 다시 위로 가서 보고 와주세요!

  let registeredModuleRefs = [];
  // 등록된 모듈들의 배열

  for (const [index, innerModule] of modules.entries()) {
    // imports 배열에 있는 모듈들에 대해 반복합니다.

    // In case of a circular dependency (ES module system),
    // JavaScript will resolve the type to `undefined`.
    // 요 위의 영어 주석은 원래부터 코드에 있는 주석입니다.
    // 순환 참조가 발생하면, innerModule이 undefined이 된다고 하네요.
    if (innerModule === undefined) {
      throw new UndefinedModuleException(moduleDefinition, index, scope);
    }
    if (!innerModule) {
      throw new InvalidModuleException(moduleDefinition, index, scope);
    }
    if (ctxRegistry.includes(innerModule)) {
      continue;
    } // 이미 등록된 모듈이라면 넘어갑니다.

    const moduleRefs = await this.scanForModules(
      innerModule,
      [].concat(scope, moduleDefinition),
      ctxRegistry,
    );
    // 재귀..!! 를 돌고 있네요.
    // 이런 형태라면, scope는 현재 모듈의 부모 모듈들이 루트(AppModule)부터 순서대로 들어있겠네요!
    // ctxRegistry는 현재 등록된 모듈들에 대한 정보인 것 같아요.

    registeredModuleRefs = registeredModuleRefs.concat(moduleRefs);
    // 자식 모듈의 자식 모듈들까지 모두 스캔을 완료했다면, 등록 완료!
  }
  if (!moduleInstance) {
    return registeredModuleRefs;
  }

  return [moduleInstance].concat(registeredModuleRefs);
  // 현재 모듈과 등록된 자식 모듈들의 배열을 반환합니다.
}
```

엄청난.. 양이에요! 아마 주석을 일일이 읽어보시면 어느정도 이해가 될 거라 생각해요.
여기서 주목해볼만한 부분은, 재귀를 통해서 단 한 번의 호출로 해당 모듈의 자식 모듈, 그 자식 모듈의 자식 모듈까지 모두 불러온다는 점인데요.

즉, 모듈을 스캔(로딩)할 때는 **DFS**로 불러옵니다.

이제 `scanModulesForDependencies` 메서드를 봅시다.

```typescript
// packages/core/scanner.ts
public async scanModulesForDependencies(
  modules: Map<string, Module> = this.container.getModules(),
  // 컨테이너에 등록되어 있는 모듈들을 모두 불러옵니다.
) {
  for (const [token, { metatype }] of modules) {
    await this.reflectImports(metatype, token, metatype.name);
    // 1. 해당 모듈(A 모듈)의 imports 배열에 들어있는 모듈들을 가져옵니다.
    // 2. ForwardRef 모듈은 .forwardRef를 통해 실제 모듈을 가져옵니다.
    // 3. 구한 모듈을 컨테이너 내, A 모듈의 _imports(Set<Module>)에 추가합니다.
    // 아래도 거의 동일합니다.

    this.reflectProviders(metatype, token);
    this.reflectControllers(metatype, token);
    this.reflectExports(metatype, token);
  }
}
```

스캔이라고 해서 상당히 긴 게 나올거라 생각하고 살짝 무서워하면서 들어왔는데... 다행히 생각보다 그렇게 길지는 않네요.

4개의 reflect 메서드들은 실제로 인스턴스를 만드는게 아니라, 관련된 메타데이터들을 모듈에 등록하여 추후 실제 인스턴스와 연결할 준비를 하는 것 같습니다. 실제 구현을 보시려면 [여기](https://github.com/nestjs/nest/blob/34cb6001c30b3e2d2a9570bc87c6bd78a8c49524/packages/core/scanner.ts#L158)를 참고해주세요!

그 외의 세 메서드는 간단하게 보고 넘어가겠습니다.

- [calculateModulesDistance](https://github.com/nestjs/nest/blob/34cb6001c30b3e2d2a9570bc87c6bd78a8c49524/packages/core/scanner.ts#L301): 루트 모듈에서 각각의 모듈이 얼마나 떨어져있는지 계산합니다. 루트 모듈은 거리가 1이고, 루트 모듈의 imports 배열에 있는 모듈들은 2가 되는 형식입니다.
- [addScopedEnhancersMetadata](https://github.com/nestjs/nest/blob/34cb6001c30b3e2d2a9570bc87c6bd78a8c49524/packages/core/scanner.ts#L433): [프로바이더 스코프](https://docs.nestjs.com/fundamentals/injection-scopes#provider-scope)가 `REQUEST`이거나 `TRANSIENT`인 프로바이더들을, 해당 프로바이더가 속해있는 모듈의 컨트롤러에 `EnhancerMetadata`로 추가합니다. (그게 무엇인지는 아직 알아내지 못했습니다..)
- [bindGlobalScope](https://github.com/nestjs/nest/blob/34cb6001c30b3e2d2a9570bc87c6bd78a8c49524/packages/core/injector/container.ts#L203): 모든 글로벌 모듈들을 `_imports`에 추가합니다. 이에 따라, 전역 모듈은 우리가 추가하지 않아도 **모든 곳에서** 사용할 수 있게 되는 겁니다.

#### InstanceLoader.createInstancesOfDependencies

이름에서 풍겨져오는 느낌이, 아무래도 가장 중요한, 우리가 도달할 목표에 가장 가까운 메서드인 것 같아요. 힘내서 가보자구요!!

```typescript
// packages/core/injector/instance-loader.ts
public async createInstancesOfDependencies(
  modules: Map<string, Module> = this.container.getModules(),
) {
  this.createPrototypes(modules);
  await this.createInstances(modules);
}
```

시작은 제법 간단합니다. 이제 여기서 지옥(?)이 펼쳐질 예정이에요.
먼저 `createPrototypes`입니다.

```typescript
// packages/core/injector/instance-loader.ts
private createPrototypes(modules: Map<string, Module>) {
  modules.forEach(moduleRef => {
    this.createPrototypesOfProviders(moduleRef);
    this.createPrototypesOfInjectables(moduleRef);
    this.createPrototypesOfControllers(moduleRef);
  });
}

private createPrototypesOfProviders(moduleRef: Module) {
  const { providers } = moduleRef;
  providers.forEach(wrapper =>
    this.injector.loadPrototype<Injectable>(wrapper, providers),
  );
}

private createPrototypesOfControllers(moduleRef: Module) {
  const { controllers } = moduleRef;
  controllers.forEach(wrapper =>
    this.injector.loadPrototype<Controller>(wrapper, controllers),
  );
}

private createPrototypesOfInjectables(moduleRef: Module) {
  const { injectables } = moduleRef;
  injectables.forEach(wrapper =>
    this.injector.loadPrototype(wrapper, injectables),
  );
}
```

이름이 참 직관적이에요. 공통적으로 `loadPrototype` 메서드를 호출하니, 요 메서드를 한 번 살펴보면 될 거 같아요!

```typescript
// packages/core/injector/injector.ts
public loadPrototype<T>(
  { token }: InstanceWrapper<T>,
  collection: Map<InstanceToken, InstanceWrapper<T>>,
  contextId = STATIC_CONTEXT,
) {
  // contextId = STATIC_CONTEXT
  if (!collection) {
    return;
  }
  const target = collection.get(token);
  // 토큰으로 메타데이터를 불러옵니다.

  const instance = target.createPrototype(contextId);
  // 프로토타입을 생성해요. 만약

  if (instance) { // 만약 새로 생성되었다면,
    const wrapper = new InstanceWrapper({
      ...target,
      instance,
    }); // InstanceWrapper로 한 번 감싼 뒤에,
    collection.set(token, wrapper); // 주어진 collection에 등록합니다.
    // 이때, collection은 providers, controllers, injectables 중 하나에요.
  }
}
```

다른 건 이해 됐는데, `createPrototype`은 처음 봐요! 요기를 한 번 더 파볼게요.

```typescript
// packages/core/injector/instance-wrapper.ts
public createPrototype(contextId: ContextId) {
  // contextId = STATIC_CONTEXT
  const host = this.getInstanceByContextId(contextId);
  if (!this.isNewable() || host.isResolved) {
    // 이미 해당 맥락에서의 인스턴스가 만들어졌다면, 아무것도 반환하지 않습니다.
    return;
  }
  // 없다면, 새로 만들어서 반환해요.
  return Object.create(this.metatype.prototype);
}

public getInstanceByContextId(
  contextId: ContextId,
  inquirerId?: string,
): InstancePerContext<T> {
  // contextId = STATIC_CONTEXT

  // 프로바이더 스코프가 TRANSIENT이고, inquirerId가 있을 때
  if (this.scope === Scope.TRANSIENT && inquirerId) {
    return this.getInstanceByInquirerId(contextId, inquirerId);
  }

  // 해당 맥락에서의 인스턴스를 가져옵니다.
  const instancePerContext = this.values.get(contextId);
  return instancePerContext
    ? instancePerContext
    // 이미 해당 맥락 내에서 인스턴스가 존재한다면 그대로 반환합니다. @1
    : this.cloneStaticInstance(contextId);
    // 존재하지 않으면, 정적 인스턴스로부터 복사해서 만들어냅니다.
}
```

이렇게 각각의 프로토타입을 생성하고, 토큰과 함께 등록합니다.
@1 부분 때문에 [프로바이더 스코프](https://docs.nestjs.com/fundamentals/injection-scopes#provider-scope)가 `DEFAULT`일 때 공유될 수 있는 것이라 예상해요. 정확히 알아보려면 더 코드를 뜯어봐야하지만, 지금은 여기까지만 해도 괜찮을 거 같아요!

이제, 대망의 `createInstances` 입니다.

```typescript
// packages/core/injector/instance-loader.ts
private async createInstances(modules: Map<string, Module>) {
  await Promise.all(
    [...modules.values()].map(async moduleRef => { // 모든 모듈에 대해서
      await this.createInstancesOfProviders(moduleRef);
      // 프로바이더의 인스턴스들을 생성하고,

      await this.createInstancesOfInjectables(moduleRef);
      // Injectable의 인스턴스들을 생성하고,

      await this.createInstancesOfControllers(moduleRef);
      // 마지막으로 컨트롤러들의 인스턴스들을 생성합니다.

      const { name } = moduleRef.metatype;
      this.isModuleWhitelisted(name) &&
        this.logger.log(MODULE_INIT_MESSAGE`${name}`);
      // 모듈이 화이트리스트에 있다면, 모듈이 초기화 되었다고 알려주는 로그를 출력합니다.
      // 기본적으로, InternalCoreModule이 아니라면 화이트리스트에 있습니다.
      // 즉, InternalCoreModule 빼고 모든 모듈에 대해 로그를 출력합니다.
      // 근데 이럴거면 블랙리스트라고 하는게 맞지 않나 싶네요.
    }),
  );
}

private async createInstancesOfProviders(moduleRef: Module) {
  const { providers } = moduleRef;
  // 모듈 내에서, 프로바이더들만 가져옵니다.

  const wrappers = [...providers.values()];
  await Promise.all(
    wrappers.map(item => this.injector.loadProvider(item, moduleRef)),
    // 각각의 프로바이더를 로딩합니다.
  );
}

private async createInstancesOfInjectables(moduleRef: Module) {
  const { injectables } = moduleRef;
  const wrappers = [...injectables.values()];
  await Promise.all(
    wrappers.map(item => this.injector.loadInjectable(item, moduleRef)),
  );
}

private async createInstancesOfControllers(moduleRef: Module) {
  const { controllers } = moduleRef;
  const wrappers = [...controllers.values()];
  await Promise.all(
    wrappers.map(item => this.injector.loadController(item, moduleRef)),
  );
}
```

여기선 각각의 프로바이더, Injectable, 컨트롤러들을 불러오네요!
`loadProvider`, `loadInjectable`, `loadController` 모두 내부 내용은 거의 동일하기 때문에, 대표적으로 `loadProvider`만 살펴보겠습니다.

```typescript
// packages/core/injector/injector.ts
public async loadProvider(
  wrapper: InstanceWrapper<Injectable>,
  moduleRef: Module,
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper, // = undefined
) {
  const providers = moduleRef.providers;
  await this.loadInstance<Injectable>(
    wrapper,
    providers,
    moduleRef,
    contextId,
    inquirer,
  ); // 인스턴스를 로딩한다!?
  await this.loadEnhancersPerContext(wrapper, contextId, wrapper);
}
```

`loadInstance` 메서드가 눈에 띄네요! 저 메서드로 들어가봅시다.

```typescript
// packages/core/injector/injector.ts
public async loadInstance<T>(
  wrapper: InstanceWrapper<T>,
  collection: Map<InstanceToken, InstanceWrapper>, // = providers
  moduleRef: Module,
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper, // = undefined
) {
  const inquirerId = this.getInquirerId(inquirer); // = undefined
  const instanceHost = wrapper.getInstanceByContextId(contextId, inquirerId);
  // 해당 맥락에서의 인스턴스를 가져옵니다.
  // inquirerId가 undefined이므로 정적 인스턴스를 가져오게 됩니다.

  if (instanceHost.isPending) { // 이미 해당 인스턴스에 대해 처리 중이라면
    return instanceHost.donePromise;
  }

  const done = this.applyDoneHook(instanceHost);
  // instanceHost의 상태를 pending 으로 바꾸고,
  // instanceHost의 donePromise에 새로운 Promise를 만들어 넣습니다.
  // 그리고, 새로 만든 Promise의 resolve 함수를 반환합니다.
  // 즉, done을 호출하면 Promise가 종료됩니다.

  const token = wrapper.token || wrapper.name;
  // 설정된 토큰이 없다면, 이름을 토큰으로 선택합니다.

  const { inject } = wrapper;

  const targetWrapper = collection.get(token);
  if (isUndefined(targetWrapper)) {
    throw new RuntimeException();
  }
  // 존재하지 않는 토큰이면, 오류를 발생시킵니다.

  if (instanceHost.isResolved) { // 이미 처리된 인스턴스라면
    return done(); // 끝냅니다.
  }

  const callback = async (instances: unknown[]) => {
    const properties = await this.resolveProperties(
      wrapper,
      moduleRef,
      inject as InjectionToken[],
      contextId,
      wrapper,
      inquirer,
    );
    // 어딘가의 매개변수가 아니라, 프로퍼티에 대한 의존성을 처리합니다.
    // 사용하는 기준 코드에서는 프로퍼티에 무언가를 주입받지 않으므로,
    // 해당 메서드는 살펴보지 않습니다.

    const instance = await this.instantiateClass(
      instances,
      wrapper,
      targetWrapper,
      contextId,
      inquirer,
    );
    // 인스턴스를..! 생성합니다!!!

    this.applyProperties(instance, properties);
    // 방금 전에 처리된 의존성을 만든 인스턴스에 적용시킵니다.
    // 마찬가지로 살펴보지 않습니다.

    done();
    // 처리 끝!
  };

  await this.resolveConstructorParams<T>(
    wrapper,
    moduleRef,
    inject as InjectionToken[],
    callback,
    contextId,
    wrapper,
    inquirer,
  );
  // 생성자의 매개변수에 대한 의존성을 처리합니다.
}
```

엄청나게 기네요... 하지만 여기가 가장 중요한 부분인 거 같아요. 마음을 다잡고, 가봅시다.

위의 내용에서 가장 눈에 띄는 메서드는 `instantiateClass`에요. 저기서.. 실제로 인스턴스화가 진행될 거 같아요!! 그런데 해당 메서드는 콜백 내에서 호출되니, 콜백을 호출하는 곳으로 찾아가봐야 할 거 같아요. `resolveConstructorParams`부터 살펴볼까요?

```typescript
// packages/core/injector/injector.ts
public async resolveConstructorParams<T>(
  wrapper: InstanceWrapper<T>,
  moduleRef: Module,
  inject: InjectorDependency[],
  callback: (args: unknown[]) => void | Promise<void>,
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper, // = undefined
  parentInquirer?: InstanceWrapper, // = undefined
) {
  let inquirerId = this.getInquirerId(inquirer); // = undefined
  const metadata = wrapper.getCtorMetadata();

  // 메타데이터가 존재하는데, 프로바이더 스코프가 DEFAULT가 아닌 경우
  if (metadata && contextId !== STATIC_CONTEXT) {
    // 새로 인스턴스들을 다시 불러와서
    const deps = await this.loadCtorMetadata(
      metadata,
      contextId,
      inquirer,
      parentInquirer,
    );
    // 콜백을 호출합니다.
    return callback(deps);
  }

  const isFactoryProvider = !isNil(inject);
  const [dependencies, optionalDependenciesIds] = isFactoryProvider
    ? this.getFactoryProviderDependencies(wrapper)
    : this.getClassDependencies(wrapper);
  // 기준 코드에서는 팩토리 프로바이더를 사용하지 않으므로, 후자의 메서드를 호출합니다.
  // 전자의 dependencies는 reflectConstructorParams<T>의 결과를 가져오며,
  // 해당 메서드는 PARAMTYPES_METADATA(='design:paramtypes') 메타데이터의 값(배열)에
  // SELF_DECLARED_DEPS_METADATA(='self:paramtypes') 메타데이터 값(배열)을 덮어씌운 결과를 반환합니다.
  // 즉, PARAMTYPES_METADATA가 [1, 2, 3, 4, 5]고 SELF_DECLARED_DEPS_METADATA가
  // [8, 3, 7]이라 하면, 반환하는 값은 [8, 3, 7, 4, 5]가 됩니다.

  let isResolved = true;
  const resolveParam = async (param: unknown, index: number) => {
    try {
      if (this.isInquirer(param, parentInquirer)) {
        return parentInquirer && parentInquirer.instance;
      }
      if (inquirer?.isTransient && parentInquirer) {
        inquirer = parentInquirer;
        inquirerId = this.getInquirerId(parentInquirer);
      }
      // Inquirer는 아직 이해하지 못했습니다.. ㅠ

      const paramWrapper = await this.resolveSingleParam<T>(
        wrapper,
        param,
        { index, dependencies },
        moduleRef,
        contextId,
        inquirer,
        index,
      );
      // 매개변수 하나의 의존성을 해결한 뒤, 그 결과로 매개변수 래퍼를 반환해요.

      const instanceHost = paramWrapper.getInstanceByContextId(
        contextId,
        inquirerId,
      );
      // 매개변수 래퍼의 현재 맥락의 인스턴스를 가져와요.

      if (!instanceHost.isResolved && !paramWrapper.forwardRef) {
        // 의존성 처리가 제대로 완료되지 않았는데, 해당 매개변수의 의존성이
        // ForwardRef도 아니라면, 의존성 처리가 정상적으로 처리되지 않은 것!
        isResolved = false;
      }
      return instanceHost?.instance;
      // 의존성 처리가 제대로 완료되었다면, 해당 의존성의 인스턴스를 반환합니다.
    } catch (err) {
      const isOptional = optionalDependenciesIds.includes(index);
      if (!isOptional) {
        throw err;
      }
      return undefined;
    }
  };
  const instances = await Promise.all(dependencies.map(resolveParam));
  isResolved && (await callback(instances));
  // 지정된 의존성들의 인스턴스를 만들고, 정상적으로 처리되었다면 콜백을 호출합니다.
}
```

위의 코드에서는, 처음 보는 메타데이터 두 개와 눈에 띄는 메서드 하나가 있어요.

먼저 메타데이터부터 살펴볼게요. `PARAMTYPES_METADATA`와 `SELF_DECLARED_DEPS_METADATA`입니다. 두 메타데이터 중 `PARAMTYPES_METADATA` 메타데이터는 그 이름보다는 그 값에 더 집중을 해야하는데요! 바로, `design:paramtypes`입니다.

`design:paramtypes` 메타데이터는 네스트가 만든 것이 아니라, 타입스크립트와 `reflect-metadata` 라이브러리에서 제공하는 메타데이터에요. 아래의 코드를 봅시다.

```typescript
import "reflect-metadata";

function SomeDecorator() {
  // 의미 없는 클래스 데코레이터
  return (v: any) => {};
}

@SomeDecorator()
class Hello {
  constructor(someA: string, someB: number, someC: SomeClass) {}
}

console.log(Reflect.getMetadata("design:paramtypes", Hello));
// [
//   function String() { [native code] },
//   function Number() { [native code] },
//   class SomeClass { }
// ]
```

`tsconfig.json`에서 `experimentalDecorators`와 `emitDecoratorMetadata` 두 옵션에 `true`를 주어야 저 세 개의 메타데이터를 가져올 수 있어요.

각각은 위의 코드를 보면 알 수 있다시피, `design:paramtypes` 메타데이터는 생성자 함수의 매개변수 타입들을 가져올 수 있어요. 대신, 어떠한 형태로든 데코레이터가 붙어야 가져올 수 있고, 아무 데코레이터도 붙어있지 않다면 `undefined`을 반환합니다. 보통 `@Injectable()`이나 `@Controller()` 등의 데코레이터가 붙기 때문에, 메타데이터를 가져올 수 있어요.

이렇게 타입을 들고 올 수 있기 때문에, 우리가 굳이 주입 받을 타입을 명시하지 않아도 무엇을 주입 받을지 네스트가 알 수 있는 것입니다.

또, `SELF_DECLARED_DEPS_METADATA` 메타데이터는 위의 방식을 유사하게 네스트에서 만든 메타데이터입니다. `@Inject()` 데코레이터를 한 번 쯤 들어보셨을 거라 생각해요. `@Inject()` 데코레이터는 주입 받을 프로바이더의 토큰을 명시할 수 있어요. 해당 메타데이터는 `@Inject()` 데코레이터에서 저장하는데, 이건 코드를 보면 바로 이해하실 수 있을 거에요!

```typescript
// packages/common/decorators/core/inject.decorator.ts
export function Inject<T = any>(token?: T) {
  return (target: object, key: string | symbol, index?: number) => {
    const type = token || Reflect.getMetadata("design:type", target, key);

    if (!isUndefined(index)) {
      // index가 undefined이 아니라는 건, @Inject 데코레이터를 생성자의 매개변수에
      // 사용했다는 뜻이에요.

      let dependencies =
        Reflect.getMetadata(SELF_DECLARED_DEPS_METADATA, target) || [];
      // 현재까지 SELF_DECLARED_DEPS_METADATA 메타데이터에 저장된 값들을 불러와요.

      dependencies = [...dependencies, { index, param: type }];
      // 메타데이터에 현재의 토큰을 추가한 뒤에,
      Reflect.defineMetadata(SELF_DECLARED_DEPS_METADATA, dependencies, target);
      // 다시 배열을 메타데이터에 저장해요.
      return;
    }
    let properties =
      Reflect.getMetadata(PROPERTY_DEPS_METADATA, target.constructor) || [];
    // index가 undefined라는 건, @Inject 데코레이터를 프로퍼티(필드)에
    // 사용했다는 뜻이에요. 기준 코드에서는 프로퍼티에 @Inject 데코레이터를 사용하지
    // 않으므로, 이 아래는 무시하도록 할게요!

    properties = [...properties, { key, type }];
    Reflect.defineMetadata(
      PROPERTY_DEPS_METADATA,
      properties,
      target.constructor
    );
  };
}
```

이걸 보면 왜 위에서 `SELF_DECLARED_DEPS_METADATA` 메타데이터의 값으로 `PARAMTYPES_METADATA` 메타데이터의 값을 덮어씌우는지 알 수 있겠죠? `PARAMTYPES_METADATA` 메타데이터의 값을 통해 현재 생성자의 매개변수들의 타입을 가져올 수 있지만, `@Inject()` 데코레이터는 명시적으로 가져올 프로바이더를 지정하기 때문에, 네스트에서 찾아온 타입을 무시하고 `@Inject()` 데코레이터의 토큰을 사용하는 것입니다.

어쨌든, 이러한 과정을 통해 `dependencies` 배열에는 각각의 위치에 맞는 토큰이나 타입이 들어가 있을 거에요. 이 데이터를 통해 각각의 의존성을 처리하는 것이 `resolveConstructorParams` 메서드 안에 있는 `resolveParam` 함수입니다.

이제 `resolveSingleParam`을 볼 차례입니다.

```typescript
// packages/core/injector/injector.ts
public async resolveSingleParam<T>(
  wrapper: InstanceWrapper<T>,
  param: Type<any> | string | symbol | any,
  // 현재 처리할 의존성 정보 (토큰이나 타입)
  dependencyContext: InjectorDependencyContext,
  // 현재 처리할 의존성의 인덱스와, 생성자 내의 모든 의존성 정보를 갖는 배열
  moduleRef: Module,
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper, // = undefined
  keyOrIndex?: symbol | string | number, // 현재 처리할 의존성의 인덱스
) {
  // 타입이 undefined이라는 건, 순환 참조나 의존성이 존재하지 않는 거에요.
  if (isUndefined(param)) {
    this.logger.log(
      'Nest encountered an undefined dependency. This may be due to a circular import or a missing dependency declaration.',
    );
    throw new UndefinedDependencyException(
      wrapper.name,
      dependencyContext,
      moduleRef,
    );
  }

  // 의존성 정보(토큰)를 가져와서
  const token = this.resolveParamToken(wrapper, param);

  // 의존성 처리!
  return this.resolveComponentInstance<T>(
    moduleRef,
    token,
    dependencyContext,
    wrapper,
    contextId,
    inquirer,
    keyOrIndex,
  );
}
```

위에서는 `resolveParamToken`과 `resolveComponentInstance`를 봐야겠네요!
먼저 `resolveParamToken`입니다.

```typescript
// packages/core/injector/injector.ts
public resolveParamToken<T>(
  wrapper: InstanceWrapper<T>,
  param: Type<any> | string | symbol | any,
) {
  if (!param.forwardRef) { // ForwardRef 상태가 아니라면
    return param; // 들어온 의존성 정보를 그대로 반환합니다.
  }

  // 매개변수가 ForwardRef 상태이므로, 해당 매개변수가 포함된 생성자의 클래스도
  // ForwardRef 상태로 바꿉니다.
  wrapper.forwardRef = true;

  // 그리고 forwardRef 메서드를 통해 실제 값을 가져옵니다.
  return param.forwardRef();
}
```

기준 코드에서는 매개변수가 ForwardRef 상태가 아니므로, 의존성 정보를 그대로 반환해요. 이제 `resolveComponentInstance`를 살펴볼까요?.

```typescript
// packages/core/injector/injector.ts
public async resolveComponentInstance<T>(
  moduleRef: Module,
  token: InstanceToken,
  dependencyContext: InjectorDependencyContext,
  wrapper: InstanceWrapper<T>,
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper,
  keyOrIndex?: symbol | string | number,
): Promise<InstanceWrapper> {
  this.printResolvingDependenciesLog(token, inquirer);
  this.printLookingForProviderLog(token, moduleRef);

  // 모듈의 모든 프로바이더들을 가져와서
  const providers = moduleRef.providers;

  // 의존성을 찾아봅니다.
  // 1. 현재 모듈의 프로바이더들 중에, 현재 처리할 의존성의 토큰과 같은 토큰을 갖는
  //    프로바이더가 있다면, 해당 정보를 가져와 저장하고 반환합니다.
  // 2. 없다면 부모 모듈, 즉 imports 배열에 있는 모듈들에서 찾아보고, 찾을 때까지
  //    재귀적으로 반복합니다.
  const instanceWrapper = await this.lookupComponent(
    providers,
    moduleRef,
    { ...dependencyContext, name: token },
    wrapper,
    contextId,
    inquirer,
    keyOrIndex,
  );

   return this.resolveComponentHost(
    moduleRef,
    instanceWrapper,
    contextId,
    inquirer,
  );
}
```

이제 `resolveComponentHost`로 가봅시다!

```typescript
// packages/core/injector/injector.ts
public async resolveComponentHost<T>(
  moduleRef: Module, // 현재 모듈
  instanceWrapper: InstanceWrapper<T | Promise<T>>, // 처리할 의존성
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper, // = undefined
): Promise<InstanceWrapper> {
  const inquirerId = this.getInquirerId(inquirer);

  // 현재 맥락에서, 처리해야하는 의존성의 정보를 가져옵니다.
  const instanceHost = instanceWrapper.getInstanceByContextId(
    contextId,
    inquirerId,
  );

  // 처리되지 않았고, ForwardRef도 아니라면 아직 해당 의존성이 처리 전이라는 뜻이므로,
  if (!instanceHost.isResolved && !instanceWrapper.forwardRef) {
    // 해당 의존성을 먼저 처리합니다.
    await this.loadProvider(
      instanceWrapper,
      instanceWrapper.host ?? moduleRef,
      contextId,
      inquirer,
    );
  }
  // 의존성이 처리되지 않았는데, ForwardRef 상태이고, 프로바이더 스코프가
  // STATIC_CONTEXT가 아니거나 inquirerId가 비어있는 값이 아닐 때,
  // 현재는 contextId가 STATIC_CONTEXT이고, inquirerId도 비어있으므로,
  // 해당 분기로는 가지 않아요. 따라서 뛰어 넘을게요!
  else if (
    !instanceHost.isResolved &&
    instanceWrapper.forwardRef &&
    (contextId !== STATIC_CONTEXT || !!inquirerId)
  ) {
    /**
     * When circular dependency has been detected between
     * either request/transient providers, we have to asynchronously
     * resolve instance host for a specific contextId or inquirer, to ensure
     * that eventual lazily created instance will be merged with the prototype
     * instantiated beforehand.
     */
    instanceHost.donePromise &&
      instanceHost.donePromise.then(() =>
        this.loadProvider(instanceWrapper, moduleRef, contextId, inquirer),
      );
  }
  // 위의 분기에서 오류가 발생하지 않았다면, 의존성이 정상적으로 처리되었다는 뜻이에요.

  if (instanceWrapper.async) {
    const host = instanceWrapper.getInstanceByContextId(
      contextId,
      inquirerId,
    );
    host.instance = await host.instance;
    instanceWrapper.setInstanceByContextId(contextId, host, inquirerId);
  }
  return instanceWrapper;
}
```

메서드 안에서 `loadProvider`를 호출하고 있어요. 이 자체가 거대한 재귀 호출이었던 거죠. 결국 모듈은 트리 형태를 띄기 때문에, 이것도 의존성 트리라고 생각해보면 이해가 편할 거에요.
아무 의존성 없이 처리할 수 있는 리프 노드의 프로바이더부터 인스턴스화를 진행하고 저장해둔 뒤에, 해당 리프 노드 프로바이더이 필요한 프로바이더는 저장한 인스턴스를 가져와서 인스턴스화를 하고. 그렇게 의존성을 처리하는 거에요!

그럼 인스턴스화는 어디서 진행할까요? 🤔
계속 가보자구요!

요 메서드에서는 더 이상 다른 메서드를 호출하지 않아요. 이제 다시 저어어어 위로 돌아가봅시다!

```typescript
// packages/core/injector/injector.ts
public async resolveConstructorParams<T>(
  wrapper: InstanceWrapper<T>,
  moduleRef: Module,
  inject: InjectorDependency[],
  callback: (args: unknown[]) => void | Promise<void>,
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper, // = undefined
  parentInquirer?: InstanceWrapper, // = undefined
) {
  // ...
  let isResolved = true;
  const resolveParam = async (param: unknown, index: number) => {
    try {
    // ...

      const paramWrapper = await this.resolveSingleParam<T>(
        wrapper,
        param,
        { index, dependencies },
        moduleRef,
        contextId,
        inquirer,
        index,
      );
      // 이제 해당 매개변수의 의존성은 해결이 됐어요.

      const instanceHost = paramWrapper.getInstanceByContextId(
        contextId,
        inquirerId,
      );
      // 매개변수 래퍼의 현재 맥락의 인스턴스를 가져와요.

      if (!instanceHost.isResolved && !paramWrapper.forwardRef) {
        // 의존성 처리가 제대로 완료되지 않았는데, 해당 매개변수의 의존성이
        // ForwardRef도 아니라면, 의존성 처리가 정상적으로 처리되지 않은 것!
        isResolved = false;
      }
      return instanceHost?.instance;
      // 여기서 인스턴스를 잘 반환하겠죠?
    } catch (err) {
      const isOptional = optionalDependenciesIds.includes(index);
      if (!isOptional) {
        throw err;
      }
      return undefined;
    }
  };
  const instances = await Promise.all(dependencies.map(resolveParam));
  isResolved && (await callback(instances));
  // 해결된 의존성들을 모두 가져와서, 콜백을 호출해요.
}
```

드디어 끝이 보이기 시작해요!! 다시 콜백의 내용을 보러 `loadInstance` 메서드로 가보죠!

```typescript
public async loadInstance<T>(
  wrapper: InstanceWrapper<T>,
  collection: Map<InstanceToken, InstanceWrapper>,
  moduleRef: Module,
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper,
) {
  // ...
  const callback = async (instances: unknown[]) => {
    // 이제 처리된 의존성들의 배열이 여기로 들어오겠죠!

    // 프로퍼티에 대한 의존성을 처리해요. 기준 코드에서는 프로퍼티에 대한 의존성이
    // 존재하지 않기 때문에, 넘어가도록 할게요.
    const properties = await this.resolveProperties(
      wrapper,
      moduleRef,
      inject as InjectionToken[],
      contextId,
      wrapper,
      inquirer,
    );

    const instance = await this.instantiateClass(
      instances,
      wrapper,
      targetWrapper,
      contextId,
      inquirer,
    );

    // 위에서 처리한 의존성을 만들어진 인스턴스에 적용하는 메서드에요.
    this.applyProperties(instance, properties);

    // 끝!
    done();
  };
  // ...
}
```

대망의 `instantiateClass` 메서드가 남았어요! 이름부터 실제로 인스턴스를 만들어낼 거 같은 메서드네요.

```typescript
// packages/core/injector/injector.ts
public async instantiateClass<T = any>(
  instances: any[], // 처리된 인스턴스들
  wrapper: InstanceWrapper, // 토큰 정보 등을 담고 있음
  targetMetatype: InstanceWrapper, // 만들어낼 인스턴스
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper, // = undefined
): Promise<T> {
  const { metatype, inject } = wrapper;
  const inquirerId = this.getInquirerId(inquirer);

  // 현재 맥락에서 필요한 인스턴스
  const instanceHost = targetMetatype.getInstanceByContextId(
    contextId,
    inquirerId,
  );

  const isInContext =
    wrapper.isStatic(contextId, inquirer) ||
    wrapper.isInRequestScope(contextId, inquirer) ||
    wrapper.isLazyTransient(contextId, inquirer) ||
    wrapper.isExplicitlyRequested(contextId, inquirer);

  if (isNil(inject) && isInContext) {
    // 팩토리 프로바이더가 아니라면,
    instanceHost.instance = wrapper.forwardRef
      ? Object.assign(
          instanceHost.instance,
          new (metatype as Type<any>)(...instances),
        )
    // 인스턴스를 만들어서 저장합니다.
      : new (metatype as Type<any>)(...instances);

  } else if (isInContext) {
    const factoryReturnValue = (targetMetatype.metatype as any as Function)(
      ...instances,
    );
    instanceHost.instance = await factoryReturnValue;
  }

  // 해당 의존성이 처리되었다고 명시합니다.
  instanceHost.isResolved = true;

  // 만들어진 인스턴스를 반환합니다.
  return instanceHost.instance;
}
```

드디어! 실제로 인스턴스가 만들어지는 바로 그 곳입니다!!! 여기서 의존성을 만들어서 저장해두는 겁니다. 이제 `loadInstance` 메서드도 종료가 됩니다.

엄청나게 멀리 왔고, 이제 결론을 봤어요. 다시 돌아가볼까요?

돌아가는 중...
`loadInstance` > `loadProvider` > `createInstancesOfProviders` > `createInstances` > `createInstancesOfDependencies` > `initialize`

```typescript
// packages/core/nest-factory.ts
private async initialize(
  module: any,
  container: NestContainer,
  config = new ApplicationConfig(),
  httpServer: HttpServer = null,
) {
  // ...
  try {
    this.logger.log(MESSAGES.APPLICATION_START);

    await ExceptionsZone.asyncRun(
      async () => {
        await dependenciesScanner.scan(module);
        await instanceLoader.createInstancesOfDependencies();

        // 전역 프로바이더들(APP_PIPE, APP_INTERCEPTOR 등)을 등록합니다.
        dependenciesScanner.applyApplicationProviders();
      },
      teardown,
      this.autoFlushLogs,
    );
  } catch (e) {
    this.handleInitializationError(e);
  }
}
```

이제 정말 끝이 났어요. 인스턴스는 초기화 돼서 `instanceHost`에 저장되고, 나중에 필요할 때마다 꺼내서 사용하겠죠? 해당 부분은 지금 다루기는 어렵고, 추후 다루는 걸로 할께요.

## 결론

길고도 긴 과정이었지만, 결론적으로 간단하게 설명하자면 DFS로 모듈을 찾아 토큰 정보를 찾고, 토큰 정보를 기반으로 실제 인스턴스를 만들어내서 `instanceHost.instance`에 저장해요.

결론은 참으로 간단하지만, 긴 삽질기였습니다.

글이 길어 담지 못했던 이야기도 있습니다. [기준 코드를 기반으로 위 코드들을 다시 봅니다.](https://velog.io/@coalery/analyze-base-code) 일종의 부록이죠.

다음 글은 언제가 될진 모르겠지만, 요청이 들어왔을 때 어떻게 알맞은 메서드를 호출하는지 알아보도록 할게요.

잘못된 내용이나 오타에 대한 지적은 언제나 환영합니다.

긴 글 읽어주셔서 감사합니다.

---

글: https://velog.io/@coalery/nest-injection-how
