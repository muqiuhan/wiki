	#typescript 

In this post, we are going to implement a Dependency Injection Container also known as Inversion of Control Container (IoC container) from scratch with typescript.

# What is a DI Container?

Before going through the code, let’s see what exactly a DI container (or IoC container) is! Let’s say we have a user class that needs to work with the database class. The first idea could be:

```typescript
class Database {  
  constructor(user: string, pass: string) { /* connect */ }  
}  
  
class User {  
  constructor() {  
    const database = new Database('test_user', 123456);  
    database.query();  
  }  
}  
  
const user = new User();
```

In this case, we are violating the Inversion of Control principle. This has two **main issues**:

- Any change in the shape of the Database class will affect the User class (User is highly coupled to the Database class)
- It’s hard to write a unit test for the User class.

## How to Fix?

To fix this issue, we need to pass the Database class from outside. Let’s see how it works in the code:

```typescript
class Database {  
  constructor(user: string, pass: string) { /* connect */ }  
}  
  
class User {  
  constructor(private database: Database) {}  
}  
  
const database = new Database('test_user', 123456);  
const user = new User(database);  // ✅
```

Now, not only User class won’t change in the case of the Database changes, but also you can write unit tests easily by providing a fake database instance.

## Let’s go further!

Now we fixed our issue, but we still want to manage our classes in a better way so that we don’t need to manage them directly. Let’s say we have such a dependency:

User -> Database -> ORM -> EnvVariable

Then, whenever we want to instantiate a `User` class it would be really frustrating to start from `SomeOtherClass` and create an instance for each class manually, something like this:

new EnvVariable() -> new ORM(env) -> new Database(orm) -> new User(database)

# Implementation

Now it’s time to create our DI container from scratch.

Let’s first start with an API of the code. What should the result look like? For the sake of simplicity, I added some simple classes instead of our previous example ( `User -> Order-> Product`)

```typescript
@Injectable()  
class UserService {  
  constructor(private orderService: OrderService) {}  
  
  getUsers() {  
    console.log('getUsers runs!');  
    this.orderService.getOrders();  
  }  
}  
  
@Injectable()  
class OrderService {  
  constructor(private productService: ProductService) {}  
  
  getOrders() {  
    console.log('getting orders..!! 📦📦📦');  
    this.productService.getProducts();  
  }  
}  
  
@Injectable()  
class ProductService {  
  constructor() {}  
  
  getProducts() {  
    console.log('getting products..!! 🍊🍊🍊');  
  }  
}

And our container result would look like this.

const app = new Container().init([UserService]);  
const userService = app.get(UserService);  
  
userService.getUsers();
```

I am highly inspired by NestJS. This is what we have in the `main.ts` of a NestJS project:

```typescript
const app = await NestFactory.create(AppModule);  
const appConfigService = app.get<AppConfigService>(AppConfigService);
```

As you see, we don’t need to manually instantiate the classes and pass one to another in order to get the user object. Now let’s implement `Container` class. Before reading make sure you check my previous article here: [[A Deep Dive into the Reflection API in Javascript]]

Because I’ve used `Reflection API` in the implementation.

The next part is `@Injectable` decorator. We just define metadata on top of each class and set it `true`. (As you see in the class definition in the previous part)

```typescript
function Injectable() {  
  return function (target: any) {  
    Reflect.defineMetadata('injectable', true, target);  
  };  
}

And finally our `Container` :

class Container {  
  dependencies = [];  
  
  init(deps: any[]) {  
    deps.map((target) => {  
      const isInjectable = Reflect.getMetadata('injectable', target);  
      if (!isInjectable) return;  
  
      // get the typeof parameters of constructor  
      const paramTypes = Reflect.getMetadata('design:paramtypes', target) || [];  
  
      // resolve dependecies of current dependency  
      const childrenDep = paramTypes.map((paramType) => {  
        // recursively resolve all child dependencies:  
        this.init([paramType]);  
  
        if (!this.dependencies[paramType.name]) {  
          this.dependencies[paramType.name] = new paramType();  
          return this.dependencies[paramType.name];  
        }  
        return this.dependencies[paramType.name];  
      });  
  
      // resolve dependency by injection child classes that already resolved  
      if (!this.dependencies[target.name]) {  
        this.dependencies[target.name] = new target(...childrenDep);  
      }  
    });  
  
    return this;  
  }  
  
  public get<T extends new (...args: any[]) => any>(  
    serviceClass: T,  
  ): InstanceType<T> {  
    return this.dependencies[serviceClass.name];  
  }  
}

```

Let’s analyze it. As you see `init` method gets an array of classes. So we passed `User` class and we want to get an instance of that. We iterate over the input and make sure we resolve all the dependencies of the input class and also its dependencies recursively.

## Why Recursive?

Because we don’t know about the depth of dependencies. What if the product class has another dependency, And so on? That’s why we are calling `this.init()` inside the init function. This is how all the dependencies are being resolved in our example:

```
... -> new Product -> new Order -> new User
```
## Configuration of tsconfig

Last but not least, make sure you have these configurations in the `tsconfig` file:

```json
"emitDecoratorMetadata": true,  
"experimentalDecorators": true,
```

You can find the [source code here](https://github.com/vahidvdn/realworld-design-patterns/tree/master/app/dependency-injection). If you liked it feel free to give it a star 🌟. I hope you enjoyed it!