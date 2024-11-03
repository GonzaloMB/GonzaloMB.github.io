---
title: "Understanding Inheritance (JS Fundamentals)"
date: 2024-11-03
author: GonzaloMB
tags: ["js", "inheritance", "oop"]
---

## Definition and Importance of Inheritance in Programming

Inheritance is a key concept in object-oriented programming (OOP) that allows a class to inherit properties and behaviors from another class. This hierarchy enables code reuse, reduces redundancy, and simplifies the management of related classes. Inheritance is crucial for modeling real-world relationships, allowing for clearer organization and easier-to-maintain code structures.

Inheritance not only promotes the reuse of existing code but also establishes a natural hierarchy between classes. By deriving new classes from existing ones, developers can create more specific implementations while retaining common functionality. This mechanism supports the creation of a more modular and organized codebase, where changes to the base class automatically propagate to derived classes, enhancing consistency and reducing maintenance efforts.

## Benefits of Using Inheritance for Code Reuse and Organization

Using inheritance allows developers to reuse code by creating hierarchies where classes can share properties and methods. This approach avoids duplication and ensures consistency of functionalities among classes, improving maintainability and scalability in larger applications.

In larger projects, managing similar code across multiple classes can become cumbersome. Inheritance addresses this by centralizing shared logic in a base class. Subclasses inherit this logic, meaning that any updates or bug fixes in the base class automatically reflect in all subclasses. This not only saves time but also ensures uniform behavior across different parts of the application.

# Classes in JavaScript

## Introduction to ES6 Classes

Introduced in ES6 (ECMAScript 2015), classes in JavaScript provide a simpler and more organized syntax for defining objects and managing inheritance. Although classes may resemble those in classic OOP languages, they are syntactic sugar over JavaScript's prototype-based inheritance model.

Prior to ES6, JavaScript developers relied on constructor functions and the prototype chain to simulate class-like behavior. The introduction of the `class` syntax made it easier to create and extend objects, bringing JavaScript closer to the OOP paradigms found in languages like Java and C++. However, it's important to understand that under the hood, JavaScript still uses prototypes for inheritance.

## Syntax and Usage

The class syntax in JavaScript is straightforward and easy to understand. It allows defining a basic structure for objects, and classes can be extended using the `extends` keyword, enabling inheritance from a parent class.

Classes can include constructors, methods, getters, setters, and static methods. They provide a clear structure for object creation and encapsulate related properties and behaviors.

## Constructors and Instance Creation

A constructor is a special method that initializes new instances of a class. It sets up any required properties and prepares the object for use. Constructors are key in object creation, providing a way to set the initial state of each class instance.

When creating a new instance of a class using the `new` keyword, the constructor method is automatically called. Constructors can accept parameters to customize the new object.

```javascript
// Inheritance using Classes
class Person {
  talk() {
    return "Talking";
  }
}

const me = new Person();
const you = new Person();
console.log(me.talk()); // Output: Talking
console.log(you.talk()); // Output: Talking

// To update the function for both instances you only have to do it once:
Person.prototype.talk = function () {
  return "New and improved Talking";
};

console.log(me.talk()); // Output: New and improved Talking
console.log(you.talk()); // Output: New and improved Talking
```

# Extending Classes

## Using the `extends` Keyword to Create Subclasses

The `extends` keyword allows creating subclasses that inherit from a parent class. This enables the subclass to inherit methods and properties from its parent class and build upon them.

This inheritance mechanism is fundamental for creating specialized classes from general ones. It promotes code reuse and logical structuring of related classes, making the codebase more maintainable and scalable.

```javascript
// Extending a Class using 'extends'
class Person {
  talk() {
    return "Talking";
  }
}

class SuperHuman extends Person {
  fly() {
    return "Flying";
  }
}

const me = new Person();
console.log(me.talk()); // Output: Talking
console.log(me.fly); // Output: undefined

const you = new SuperHuman();
console.log(you.fly()); // Output: Flying
console.log(you.talk()); // Output: Talking
```

# Objects and Prototypal Inheritance

## Explanation of JavaScript's Prototype-Based Inheritance Model

JavaScript uses a prototype-based inheritance model, where objects inherit properties and methods directly from other objects. Every object has a hidden `[[Prototype]]` property that links to another object, forming a prototype chain that allows shared behaviors without classical inheritance structures.

This prototype chain means that if a property or method isn't found on an object, JavaScript looks up the chain to the object's prototype, and so on, until it finds the property or reaches the end of the chain (`null`). This allows for dynamic inheritance and is fundamental to how JavaScript objects work.

```javascript
// Inheritance using a Constructor Function
function Person() {}
Person.prototype.talk = function () {
  return "Talking";
};

const me = new Person();
const you = new Person();
console.log(me.talk()); // Output: Talking
console.log(you.talk()); // Output: Talking
```

## How Objects Inherit Properties and Methods

An object can access properties and methods from its prototype, making inheritance dynamic and flexible. This model allows for reuse and extension of behaviors without the need for rigid structures.

You can create an object that inherits from another using `Object.create()`. This method creates a new object with the specified prototype.

```javascript
// Inheritance using pure objects with Object.create()
const person = {
  talk() {
    return "Talking";
  },
};

const me = Object.create(person);
console.log(me.talk()); // Output: Talking
```

Another way to establish prototypal inheritance is by using `Object.setPrototypeOf()`, which sets the prototype of an existing object.

```javascript
// Inheritance using pure objects with Object.setPrototypeOf()
const person = {
  talk() {
    return "Talking";
  },
};

const me = {};
Object.setPrototypeOf(me, person);
console.log(me.talk()); // Output: Talking
```

**Explanation:**

- **Defining the Prototype Object:** The methods and properties are defined directly on the prototype object.
- **Setting the Prototype:** Using `Object.setPrototypeOf()` sets one object as the prototype of another.
- **Accessing Inherited Methods:** The inheriting object can access methods and properties from its prototype.

**Note:** While `Object.setPrototypeOf()` allows you to set an object's prototype after creation, it's generally recommended to use `Object.create()` when possible, as setting the prototype of an existing object can have performance implications.

## Differences Between Prototypal and Classical Inheritance

In prototypal inheritance, objects inherit directly from other objects, creating a flexible hierarchy. Classical inheritance, as in languages like Java, defines rigid classes. JavaScript's model is more adaptable, though it requires understanding of prototype chains.

Classical inheritance is based on classes and instances, with a clear distinction between the two. Prototypal inheritance blurs this line, as objects can serve as prototypes for other objects. This allows for more flexible and dynamic object structures but can be less intuitive for those accustomed to classical OOP.

# Properties vs. Methods

## Distinction Between Properties (Data) and Methods (Functions) in Objects

In JavaScript, properties store data related to an object, while methods are functions associated with the object. Properties define the state of an object, whereas methods define its behaviors.

Understanding the difference is crucial for effective object manipulation. Properties can be thought of as nouns (e.g., `name`, `age`), while methods are verbs (e.g., `run`, `speak`). This distinction helps in designing objects that accurately represent entities in your application.

### Examples:

```javascript
const Person = {
  name: "Gon", // Property
  age: 28,      // Property
  talk() {
    // Method
    console.log("Hello! My name is " + this.name + "and I'm " + this.age);
  },
  run() {
    // Method
    console.log("I'm running!");
  },
};

console.log(Person.name); // Output: Gon
console.log(Person.age);  // Output: 28
Person.talk();            // Output: Hello! My name is Gon and I'm 28
Person.run();             // Output: I'm running!

```

In an object representing a car:

- **Properties:** `brand` and `model` store data about the car.
- **Methods:** `startEngine` and `drive` define actions the car can perform.

### Using Properties and Methods Together

```javascript
const rectangle = {
  width: 10, // Property
  height: 5, // Property
  area() {
    // Method
    return this.width * this.height;
  },
  perimeter() {
    // Method
    return 2 * (this.width + this.height);
  },
};

console.log(rectangle.area()); // Output: 50
console.log(rectangle.perimeter()); // Output: 30
```

In an object representing a rectangle:

- **Properties:** `width` and `height` define the dimensions.
- **Methods:** `area` and `perimeter` calculate values based on the properties.

### Modifying Properties and Methods

Properties can be modified directly, while methods can change the object's state by modifying its properties.

```javascript
const bankAccount = {
  balance: 1000, // Property
  deposit(amount) {
    // Method
    this.balance += amount;
    console.log(`Deposited $${amount}. New balance: $${this.balance}`);
  },
  withdraw(amount) {
    // Method
    if (amount <= this.balance) {
      this.balance -= amount;
      console.log(`Withdrew $${amount}. New balance: $${this.balance}`);
    } else {
      console.log("Insufficient funds");
    }
  },
};

bankAccount.deposit(500); // Output: Deposited $500. New balance: $1500
bankAccount.withdraw(200); // Output: Withdrew $200. New balance: $1300
console.log(bankAccount.balance); // Output: 1300
```

In an object representing a bank account:

- The `balance` property represents the state of the bank account.
- The `deposit` and `withdraw` methods modify the `balance` property.

### Accessor Properties (Getters and Setters)

Accessor properties allow you to define methods that are accessed like properties. For example:

```javascript
const person = {
  firstName: "John", // Property
  lastName: "Doe", // Property
  get fullName() {
    // Getter Method
    return `${this.firstName} ${this.lastName}`;
  },
  set fullName(name) {
    // Setter Method
    [this.firstName, this.lastName] = name.split(" ");
  },
};

console.log(person.fullName); // Output: John Doe
person.fullName = "Jane Smith";
console.log(person.firstName); // Output: Jane
console.log(person.lastName); // Output: Smith
```

- A `fullName` getter method that returns a combination of `firstName` and `lastName`.
- A `fullName` setter method that splits a provided string and updates `firstName` and `lastName`.

## How Inheritance Affects Properties and Methods

Inherited properties and methods are accessible to subclasses but can be overridden. This flexibility allows customizing behaviors while reusing shared functionality.

Overriding allows a subclass to provide a specific implementation of a method that's already defined in its superclass. This is essential for tailoring inherited behaviors to fit the specific needs of the subclass while still maintaining a common interface.

### Example of Overriding Methods

```javascript
class Animal {
  constructor(name) {
    this.name = name; // Property
  }

  makeSound() {
    // Method
    console.log(`${this.name} makes a sound.`);
  }
}

class Dog extends Animal {
  makeSound() {
    // Overridden Method
    console.log(`${this.name} barks.`);
  }
}

const animal = new Animal("Generic Animal");
animal.makeSound(); // Output: Generic Animal makes a sound.

const dog = new Dog("Rex");
dog.makeSound(); // Output: Rex barks.
```

In an `Animal` class hierarchy:

- The base `Animal` class has a `makeSound` method.
- The `Dog` subclass overrides `makeSound` to provide a specific implementation.

### Inheriting Properties and Adding New Ones

```javascript
class Employee {
  constructor(name) {
    this.name = name; // Property
  }

  work() {
    // Method
    console.log(`${this.name} is working.`);
  }
}

class Manager extends Employee {
  constructor(name, department) {
    super(name);
    this.department = department; // New Property
  }

  work() {
    console.log(`${this.name} is managing the ${this.department} department.`);
  }
}

const manager = new Manager("Alice", "Sales");
manager.work(); // Output: Alice is managing the Sales department.
```

In an `Employee` class hierarchy:

- The `Manager` subclass inherits the `name` property and `work` method from `Employee`.
- It adds a new property `department` and overrides the `work` method.

---
