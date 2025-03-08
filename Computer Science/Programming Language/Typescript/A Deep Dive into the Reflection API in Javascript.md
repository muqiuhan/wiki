#javascript #typescript

Before talking about Reflection API in JavaScript, let’s review some general concepts to understand better.

# What is Reflection API in programming?

The Reflection API in programming refers to a set of built-in tools and capabilities within a programming language that allows a program to inspect and manipulate its own structure, behavior, and metadata during runtime. Reflection enables the examination of classes, interfaces, methods, and fields at runtime without knowing their names at compile time.

Key functionalities provided by Reflection API typically include:

1. Inspecting Classes: Reflection allows you to examine the structure of classes, including fields, methods, constructors, and annotations associated with them.
2. Instantiating Classes Dynamically: With reflection, you can create instances of classes dynamically at runtime, even if you don’t know the class name at compile time.
3. Invoking Methods Dynamically: You can invoke methods on objects dynamically without knowing their names at compile time.
4. Accessing and Modifying Fields: Reflection provides the ability to access and modify fields of classes, including private fields, which are otherwise not accessible.
5. Examining Annotations: You can inspect annotations associated with classes, methods, fields, etc., and take appropriate actions based on their presence or values. And even more!

# Some examples in JavaScript

Now let’s see some examples in JavaScript:

1. Inspecting Object Properties: You can use `for...in` loop or `Object.keys()` to iterate over the properties of an object dynamically:

```javascript
const obj = {  
    name: 'John',  
    age: 30,  
    city: 'New York'  
};  
  
for (const key in obj) {  
    console.log(`${key}: ${obj[key]}`);  
}
```

2. Accessing and Modifying Object Properties Dynamically: You can access and modify object properties dynamically using square bracket notation:

```javascript
const obj = { name: 'John' };  
  
// Accessing property dynamically  
const propertyName = 'name';  
console.log(obj[propertyName]); // Output: John  
  
// Modifying property dynamically  
const newValue = 'Jane';  
obj[propertyName] = newValue;  
console.log(obj.name); // Output: Jane

3. Invoking Methods Dynamically: JavaScript allows you to invoke object methods dynamically using square bracket notation:

const obj = {  
    greet: function() {  
        console.log('Hello!');  
    }  
};  
  
const methodName = 'greet';  
obj[methodName](); // Output: Hello!
```

# Limitations of the Reflection API in JavaScript

In JavaScript, compared to languages like Java or C#, there are certain reflection capabilities that are not directly available or are more limited:

1. Strict Typing: JavaScript is dynamically typed, which means types are determined at runtime. In Java and C#, reflection can be used to inspect and manipulate types at compile time due to their static typing nature. This means that in JavaScript, you may not have as much compile-time safety when using reflection-like features.
2. Type Metadata: Java and C# have extensive metadata available at runtime due to their static typing nature and rich type systems. In JavaScript, since types are determined dynamically, there is less type metadata available at runtime.
3. Reflection Emit: In languages like C#, you can use reflection emit to generate IL (Intermediate Language) code dynamically at runtime and execute it. JavaScript does not have a direct equivalent to IL code or the ability to emit and execute code dynamically at runtime in the same manner.
4. Annotations and Attributes: Java and C# support annotations and attributes, respectively, which can be used to add metadata to types, methods, and fields. While JavaScript has something similar in the form of decorators (with TypeScript or Babel), they are not as deeply integrated into the language and runtime as annotations in Java or attributes in C#.

Overall, while JavaScript provides some level of reflection-like capabilities, its dynamic nature and design differences mean that certain features available in statically-typed languages like Java or C# are not directly possible or are more limited in JavaScript.

# Reflect-Metadata in JS / TS (Problem)

There are some packages that solves some of the limitations. `Reflect-Metadata` is one of them.

Before seeing what Reflect-Metadata is, let’s see what the problem is!

## The problem:

Let’s say we want to disregard users who don’t have permission to a certain API. We can implement in this way:

```javascript
function adminOnlyMethod() {  
  if (currentUser.role === roles.ADMIN) {  
    console.log("This method is only for admins.");  
    // Perform admin-specific actions  
  } else {  
    console.log("You don't have permission to perform this action.");  
    // Optionally, handle lack of permissions  
  }  
}

But what if we have one hundred APIs? Do we want to implement this logic each time? The answer is no. But still, we can accomplish acceptable ways without the need for Reflection API. To achieve a more declarative approach and avoid checking roles in every function, you can use a higher-order function or a middleware-like pattern to wrap your methods with role-based access control logic. Here’s an example using a higher-order function:

// Higher-order function to create role-restricted methods  
function restrictToRole(role, method) {  
  return function (...args) {  
    if (currentUser.role === role) {  
      return method.apply(this, args); // Execute the original method  
    } else {  
      console.log("You don't have permission to perform this action.");  
      // Optionally, handle lack of permissions  
    }  
  };  
}  
  
// Example method  
function adminOnlyMethod() {  
  console.log("This method is only for admins.");  
  // Perform admin-specific actions  
}  
  
// Wrap the method with role-based access control  
const restrictedAdminMethod = restrictToRole(roles.ADMIN, adminOnlyMethod);
```

This is not too bad! But **still not the best option**! Here is where Reflect-Metadata comes into place.

# How does Reflect-Metadata work?

`reflect-metadata` is a library in JavaScript and TypeScript that provides reflection metadata API for runtime reflection capabilities. It enables developers to add and read metadata to/from class declarations and members at runtime.

Here’s a brief overview of how `reflect-metadata` works:

1. Adding Metadata: Developers can use `Reflect.metadata()` to add metadata to class declarations, methods, or properties. Metadata is added using a metadata key and a metadata value.
2. Accessing Metadata: The library provides methods such as `Reflect.getMetadata()` to retrieve metadata associated with class declarations, methods, or properties at runtime.
3. Usage in TypeScript: `reflect-metadata` is often used in conjunction with TypeScript decorators. Decorators allow developers to add metadata to classes and members in a more intuitive and declarative way.
4. Runtime Reflection: With the help of `reflect-metadata`, developers can perform runtime reflection tasks such as dependency injection, serialization, validation, and more.

Here’s a simple example of using `reflect-metadata` with TypeScript decorators:

```typescript
import 'reflect-metadata';  
  
// Define a decorator function  
function MyDecorator(target: any, key: string) {  
  // Add metadata to the target (class) with the given key  
  Reflect.defineMetadata('customMetadataKey', 'someValue', target, key);  
}  
  
class MyClass {  
  @MyDecorator  
  myMethod() {}  
}  
  
// Retrieve metadata at runtime  
const metadataValue = Reflect.getMetadata('customMetadataKey', MyClass.prototype, 'myMethod');  
console.log(metadataValue); // Output: 'someValue'
```

# Solve the issue with Reflect-Metadata

What if you solve the problem with a decorator?

```typescript
@Post('login')  
@Public()  
async login(@Request() req: UserRequest): Promise<LoginResponseDto> {  
  return this.authService.login(req.user);  
}
```

We define a `@Public` decorator as follows (I did it in NestJS which abstracts the same lib):

```typescript
import { SetMetadata } from '@nestjs/common';  
  
export const IS_PUBLIC_KEY = 'isPublic';  
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

Now in my Guard, I can check if the current API is decorated with the given metadata key or not:

@Injectable()  
export class JwtAuthGuard extends AuthGuard('jwt') {  
  constructor(private reflector: Reflector) {  
    super();  
  }  
  
  canActivate(context: ExecutionContext) {  
    // we pass IS_PUBLIC_KEY as metadata key to see  
    // if the method is decorated with that  
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [  
      context.getHandler(),  
      context.getClass(),  
    ]);  
  
    if (isPublic) {  
      return true;  
    }  
  
    return super.canActivate(context);  
  }  
}
```

This is a more declarative way and reduces code duplication!