# typesafe-di

[![CircleCI](https://circleci.com/gh/m3dev/typesafe-di.svg?style=svg)](https://circleci.com/gh/m3dev/typesafe-di)

A zero-dependency simple DI library to create DI containers in typesafe way.

# installation

```
yarn add typesafe-di
```

# Getting Started

First of all, build your design of an object dependency graph. `Design` is an immutable blueprint of an object graph which knows how to build each object.

```typescript
const pureDesign = Design
    // `.bind` registers an object factory.
    .bind('name', () => 'jooohn')

    // You can register async functions.
    .bind('futureAge', async () => 31);
```

In order to register a function which depends on other objects, your factory should take an argument like the following.

```typescript
const dependencyDesign = Design.bind(
    'age',
    // Under the food, `Injector<{ birthday: Date }>` is just an alias of `Injector<{ birthday: Promise<Date> }>`.
    async (injector: Injector<{ birthday: Date }>): Promise<number> => {
        // To get the value of `injector.birthday`, you have to await.
        const birthday = await injector.birthday;
        return calculateAgeFromBirthday(birthday);
    },
);
```

Call `.resolve` with missing dependencies to instanciate the object graph.

```typescript
const userDesign = Design.bind('age', () => 30)
    // `{ age: number }` is bound to this design already, so `name` is the only missing dependency.
    .bind('user', async (injector: Injector<{ name: string; age: number }>) => ({
        name: await injector.name,
        age: await injector.age,
    }));

const { container } = await userDesign.resolve({ name: 'jooohn' });
console.log(container.user); // { name: 'jooohn', age: 30 }
```

The TypeScript compiler detects missing dependencies.

```typescript
const userDesign = ...

// Compile error since `name` is required but not given.
const { container } = await design.resolve({});
```

# Design creation

```typescript
// An empty design
const empty = Design.empty;

// From an existing mapping
const pure = Design.pure({
    name: 'jooohn',
    age: 30,
});
```

# Design composition

```typescript
type HasUserRepository = { userRepository: UserRepository };
const useCaseDesign = Design.bind('changeName', async (injector: Injector<HasUserRepository>) => {
    const userRepository = await injector.userRepository;
    return async (id: string, newName: string) => {
        const user = await userRepository.find(id);
        await userRepository.save({ ...user, name: newName });
    };
});

type HasDBConfig = { dbConfig: DBConfig };
const productionAdapterDesign = Design.bind('userRepository', async (injector: Injector<HasDBConfig>) => {
    const dbConfig = await injector.dbConfig;
    return new DBUserRepository(dbConfig);
});

const productionConfigDesign = Design.bind('dbConfig', () => ({
    user: 'dbuser',
    password: 'xxx',
}));

// Merge two design
const productionUseCaseDesign = useCaseDesign.merge(productionAdapterDesign).merge(productionConfigDesign);
```

# Resource management

One of the typical use cases of DI container is to manage the lifecycle of created objects. You can register a function to finalize a resource as the third argument of the `.bind` method.

```typescript
const resourcesDesign = Design.bind(
    'resource1',
    async () => {
        const resource1 = new Resource1();
        console.log('initializing resource 1');
        await resource1.initialize();
        return resource1;
    },
    async resource1 => {
        console.log('closing resource 1');
        await resource1.close();
    },
).bind(
    'resource2',
    async (injector: Injector<{ resource1: Resource1 }>) => {
        const resource1 = await injector.resource1;
        const resource2 = new Resource2({ underlying: resource1 });
        console.log('initializing resource 2');
        await resource2.initialize();
        return resource2;
    },
    async resource2 => {
        console.log('closing resource 2');
        await resource2.close();
    },
);
```

In that case, it is recommended to call `.use` method instead of `.resolve` to let `typesafe-di` clean up the created resources.

```typescript
// `.use` automatically calls registered finalizers in reverse order of its creation.
const result = await resourcesDesign.use({})(async ({ resource1, resource2 }) => {
    // ...
    console.log('do something with resource 1 and resource 2');
    // ...
    return 'done';
});
console.log(result);
```

The example above will write console.log in the following order.

```
initializing resource 1
initializing resource 2
do something with resource 1 and resource 2
closing resource 2
closing resource 1
```

You can control when to call finalizers if you instantiate the container by `.resolve`.

```typescript
const { container, finalize } = await resourcesDesign.resolve({});

...

process.on('SIGINT', () => {
  finalize().catch(console.error);
});
```
