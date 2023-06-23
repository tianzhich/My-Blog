---
title: "JavaScript的面向对象，你真的懂了吗？是否一定需要使用”类“来编程？"
description: "JavaScript的面向对象，你真的懂了吗？是否一定需要使用”类“来编程？"
date: 2020-04-25T00:00:00+08:00
draft: false
author: "Chris"
authorLink: "https://tianzhich.com"
tags: ["JavaScript", "面向对象", "类", "行为委托模式"]
categories: ["编程"]
featuredImage: "featured-image.webp"
---

JavaScript的面向对象，你真的懂了吗？是否一定需要使用”类“来编程？

<!--more-->

## 前言

提到 JavaScript，就不得不提那强大的原型链（prototype）。但是近些年的 JSers，我想真正在使用原型进行编程的应该很少。为什么？因为 JavaScript 处处是对象，面向对象设计似乎能与它天然结合。各大流行库例如 React，都包含着面向对象设计的思想。

自从 ES6 'class' 语法糖的出现，以及 Typescript 的一些增强语法（例如 public, private 关键字），更是为我们蒙了一层面纱，今天就是要扒开这层面纱，和大家聊聊 JavaScript 的面向对象设计，以及另外一种可替代的编程模式-行为委托模式。

![](https://user-gold-cdn.xitu.io/2020/4/25/171b08767d56f412?w=2078&h=1488&f=png&s=270871)

## 面向过程编程 vs 面向对象编程

### 面向过程编程

在 C 这类语言中，没有类和对象的概念，可以将完成某个功能的重复代码块定义为函数，将具有一类功能的函数声明在一个头文件中，不同类型的函数声明在不同的头文件中，以便对函数进行更好的管理和调用。

![](https://user-gold-cdn.xitu.io/2020/4/25/171af0704dcdaf06?w=1126&h=484&f=png&s=74700)

之所以叫做**面向过程编程（Procedure Oriented Programming）**，是因为编程的过程就是一步步函数调用的过程，C 语言有一个主函数，主函数调用其他函数，以此类推实现程序功能。

### 面向对象编程

因为 Java、C++等语言都支持类和对象，所以使用这些语言编程也叫做**面向对象编程（Object Oriented Programming）**，这些语言也被叫做面向对象的变成语言。我们可以使用类创建和维护对象，组织这一数据结构。

在 Java 中，可以将完成某个功能的代码块定义为方法，将具有相似功能的方法定义在一个类中，也就是定义在一个源文件中(因为一个源文件只能包含一个公共的类)，多 个源文件可以位于一个文件夹，这个文件夹有特定的称呼，叫做包。

![](https://user-gold-cdn.xitu.io/2020/4/25/171aefbdda8b729b?w=1070&h=536&f=png&s=75625)

面向对象编程在软件执行效率上绝对没有任何优势，它的主要目的是方便程序员组织和管理代码，快速梳理编程思路，带来编程思想上的革新。

面向对象编程强调“封装（Encapsulation）”，“继承（Inheritance）“和“多态（Polymorphism）”，这三者被称为面向对象的三大特性。

数据和与数据相关的操作被包装成对象（严格的说是“类”），每一种对象是相对完整和独立的。对象可以有派生的类型（**继承**），派生的类型可以覆盖（或重载）原本已有的操作（**多态**），从而达成更好的内聚性，即一种对象做好一件（或者一类相关的）事情，对象内部的细节外面世界不关心也看不到（**封装**）。以及降低耦合性，即不同种类的对象之间相互的依赖尽可能降低。而所有的这些，都有助于达成一个崇高的目标，就是可复用性。

### 类是结构体的升级

可以将类看做是结构体的升级，因为 C 语言晚辈们看到了结构体的不足，尝试加以改善，继承了结构体的思想，并进行了升级，让程序员在开发或扩展大中型项目时更容易。

接下来我们来看看结构体和类。

结构体是一种构造数据类型，可以包含不同成员（变量），每个成员的数据类型可以不一样；可以通过结构体来定义结构体变量，每个变量拥有相同的性质。例如如下 C 语言代码：

```c
#include <stdio.h>

int main(){
    // 定义结构体 Student
    struct Student {
        // 结构体包含的变量
        char *name;
        int age;
        float score;
    };
    // 通过结构体来定义变量
    struct Student stu1;
    // 操作结构体的成员
    stu1.name = "小明";
    stu1.age = 15;
    stu1.score = 92.5;
    // 运行结果：小明的年龄是 15，成绩是 92.500000
    printf("%s的年龄是 %d，成绩是 %f\n", stu1.name, stu1.age, stu1.score);
    return 0;
}
```

Java 中的类也是一种构造数据类型，但是进行了一些扩展，类的成员不但可以是变量，还可以是函数，例如 Student 定义：

```java
// 通过class关键字类定义类
pubic class Student {
    // 类包含的变量
    String name;
    int age;
    float score;
    // 类包含的函数
    void say() {
        System.out.println(name + "的年龄是 " + age + "，成绩是 " + score);
    }
}
```

通过类定义出来的变量也有特定的称呼，叫做“对象”。如 StudentTest 类定义：

```java
public class StudentTest {
    public static void main(String[] args) {
        Student stu1 = new Student(); // 必须使用new关键字
        stu1.name = "小明";
        stu1.age = 15;
        stu1.score = 92.5f;
        stu1.say(); // 运行结果：小明的年龄是 15，成绩是 92.500000
    }
}
```

在 C 语言中，通过结构体名称就可以完成结构体变量的定义，并分配内存空间；但是在 Java 中，仅仅通过类来定义变量不会分配内存空间，必须使用 new 关键字来完成内存空间的分配。

## JavaScript 实现面向对象编程

介绍了原始概念，接下来我们看看 JavaScript 如何进行面向对象编程。

### ES6 以前

在 ES6 以前，并没有“类”这一概念，但 JavaScript 处处是对象（Object），这些对象通过原型（prototype）链接在一起。聪明的 JSers 想到了很多办法进行面向对象编程。这里我们来看看 JavaScript 是如何实现面向对象的三大特性：“封装”、“继承”和“多态”的。

#### 封装的实现

我们知道，JavaScript 对象上的属性都是可访问，所以并不存在“私有”的概念。我们在看一些开源代码时，会发现一些带下划线的属性命名，来表示他是“私有”的，但这是不安全的，这种表示只是基于社区的约定，而不是 JavaScript 规范。

那么 ES6 以前的 JavaScript 如何实现真正的私有属性呢？使用闭包！

```javascript
function Person(name) {
  let _name = name;
  this.getName = function () {
    return "My name is " + _name;
  };
}

var p = new Person("tianzhich");
console.log(p.getName()); // My name is tianzhich
```

#### 继承的实现

JS 继承对象有很多种方法，我们来看看最常使用，也算是比较完美的一种继承，小红书上叫做：**寄生组合式继承（Parasitic Combination Inheritance）**。

我们先来拆解概念：组合继承和寄生继承。

组合继承组合的是纯原型链继承和借用构造函数继承。纯原型链继承是指子类原型指向父类的一个实例对象；而借用构造函数继承是指子类借用父类的构造函数来创建对象。因此我们不难发现组合继承有如下缺点：

1. 会调用两次父类的构造函数
2. 子类原型对象会包含父类实例对象的所有属性，我们只是在必要时进行了重写

再来看看寄生式继承。寄生式继承只用于继承单个对象字面量，而不是构造函数。其思想是借助一个工厂函数，封装继承的过程。在函数中使用`Object.create(parent)`创建子类对象，同时为其添加一些特有属性。他有如下缺点：

1. 只用于继承单个对象字面量
2. 每次创建一个新的子类对象而调用工厂函数时，无法做到函数复用

最后将他俩组合起来，实现寄生组合式继承：

```javascript
// 寄生式继承优点：工厂函数封装继承过程
// 同时避免组合式继承缺点：只继承prototype而不是实例
function inherit(subType, superType) {
  var prototype = Object.create(superType.prototype);
  prototype.constructor = subType;
  subType.prototype = prototype;
}
function Person(name) {
  this.name = name;
}
Person.prototype.showName = function () {
  return this.name;
};
function Boy(name, age) {
  // 组合式继承优点：借用构造函数
  Person.call(this, name);
  this.age = age;
}

inherit(Boy, SuperType);
// 避免寄生式继承缺点：避免函数重复声明
Boy.prototype.showAge = function () {
  return this.age;
};
```

#### 多态的实现

根据我们上面的概念，多态是派生的类型可以覆盖（或重载）原类型的方法。对于覆盖方法很好实现，因为通过原型链，底层的派生对象属性，可以“遮盖”高层的属性，实现覆盖。但是 JavaScript 并不能支持重载，需要我们额外实现：

```javascript
// 还是使用上面的例子，我们增加一个sayHi的方法
Person.prototype.sayHi = function () {
  console.log("Hi, " + "my name is " + this.name);
};
// 在子类中"重载"这个方法，增加参数toPerson和参数showAge
Boy.prototype.sayHi = function (toPerson, showAge) {
  if (toPerson) {
    console.log(
      "Hi, " + toPerson.showName() + "my name is " + this.name + showAge
        ? " and I am " + this.age + " years old"
        : ""
    );
  } else {
    // 伪多态
    Person.sayHi.call(this);
    if (showAge) {
      console.log("and I am " + this.age + "years old");
    }
  }
};
```

可以看到，要实现复杂的“重载”方法，我们需要做一些额外的参数判断。同时，如果在子类中调用父类方法，也只能是显式调用，这是一种伪多态。而在接下来的"class"语法糖中，我们能看到一种相对多态，使用`super`关键字。

### ES6 'class' 语法糖

我们先来看看定义：

> JavaScript classes, introduced in ECMAScript 2015, are primarily syntactical sugar over JavaScript's existing prototype-based inheritance. The class syntax does not introduce a new object-oriented inheritance model to JavaScript.

> Classes are in fact "special functions", and just as you can define function expressions and function declarations, the class syntax has two components: class expressions and class declarations.

可以得知，'class'只是 ES6 推出的语法糖，他本质还是我们上面介绍的基于原型的继承实现，并没有引入新的面向对象继承模型。同时，Class 就是特殊的函数，也有函数“类”声明和“类”表达式两种使用方法。不过值得注意的是：**类声明并不会提升，不管是声明还是表达式，都需要在其之后使用，否则会报错：_ReferenceError_。**

接下来我们用“class”来实现上述的继承模型：

```javascript
class Person {
  constructor(name) {
    this.name = name;
  }
  showName() {
    return this.name;
  }
  sayHi() {
    console.log("Hi, " + "my name is " + this.name);
  }
}

class Boy extends Person {
  constructor(name, age) {
    super(name);
    this.age = age;
  }
  showAge() {
    return this.age;
  }
  sayHi(toPerson, showAge) {
    if (toPerson == undefined && showAge == undefined) {
      super.sayHi();
    } else if (toPerson) {
      console.log(
        "Hi, " + toPerson.showName() + "my name is " + this.name + showAge
          ? " and I am " + this.age + " years old"
          : ""
      );
    } else {
      super.sayHi();
      console.log("and I am " + this.age + "years old");
    }
  }
}
```

上面的例子没有提到**封装**，事实上，在未来，我们可能会在类的属性和方法名前加上“#”来表示这个属性或方法是私有的。这一提案目前处于[stage3 阶段](https://github.com/tc39/proposal-class-fields)。

如果你使用 Typescript 中，则可以像 Java 一样使用*public*、*private*关键字来达到同样的目的。

**'class'语法糖为我们做了很多封装，虽然本质还是原型。但是我们能够用更优雅的语法来实现它。**

### JavaScript 是面向对象编程语言吗？

这个问题其实不好问答，我们还是先参考[MDN 上对 JavaScript 的定义](https://developer.mozilla.org/en-US/docs/Web/JavaScript/About_JavaScript)：

> JavaScript can function as both a procedural and an object oriented language. Objects are created programmatically in JavaScript, by attaching methods and properties to otherwise empty objects at run time, as opposed to the syntactic class definitions common in compiled languages like C++ and Java. Once an object has been constructed it can be used as a blueprint (or prototype) for creating similar objects.

我们得知，JavaScript 既可以实现面向对象的设计，又可以实现面向过程的设计。我觉得这要看你怎么使用函数，如果使用构造函数来创建对象，你可以实现面向对象的设计；但是如果像 C 语言那样编写函数调用，又可以实现面向过程设计。

JavaScript 是一门很灵活的语言，他可以在程序运行时动态地为对象添加属性和方法，也可以作为蓝图来创建多个对象。但是我们也要认识到，这和提供了“class”句法支持的 C++和 Java 语言不同，后者一旦定义了 class 则无法动态更改。

可以说，如果你理解 JavaScript 实现面向对象设计的本质，你就可以把它当成面向对象的语言。但是如果你不了解其本质，只是单纯认为他有“class”的概念就把他认为是面向对象语言，我觉得这还不够。

## 行为委托模式

看完了 JavaScript 实现继承，我们也了解到其本质是原型链。那么为什么我们要那么执着的一定要使用继承呢？前面我们说寄生式继承的时候提到过**对象字面量（Object Literal）**，也就是直接创建一个 JavaScript 对象。我们能不能直接基于一个普通对象来进行编程设计与开发呢？

下面我将提供一个设计模式，能够让你实现和上面一样的效果，这个模式叫做**行为委托（Behavior Delegation）**，这一模式出自[_You Don't Know JS: this & Object Prototypes_](https://github.com/getify/You-Dont-Know-JS/blob/1st-ed/this%20%26%20object%20prototypes/ch6.md)。

```javascript
var Person = {
  setName: function (name) {
    this.name = name;
  },
  showName: function () {
    return this.name;
  },
  sayHi: function () {
    console.log("Hi, " + "my name is " + this.name);
  },
};

var Boy = Object.create(Person);
Boy.init = function (name, age) {
  this.setName(name);
  this.age = age;
};
Boy.showAge = function () {
  return this.age;
};
Boy.sayHiToPerson = function (toPerson, showAge) {
  console.log(
    "Hi, " + toPerson.showName() + "my name is " + this.name + showAge
      ? " and I am " + this.age + " years old"
      : ""
  );
};
Boy.sayHiWithAge = function () {
  this.sayHi();
  console.log("and I am " + this.age + "years old");
};
```

行为委托模式具有以下特点（注意我这里使用底层和顶层的说法来代替子类和父类）：

1. 强调的编码方式是“委托”，是一种 **OLOO (Object Link to Other Object)** 方式，而实现这种连接的，正是原型链 _[[prototype]]_。底层想实现一个功能，可以“委托”顶层帮忙实现
2. 数据属性直接在底层上，顶层只是“委托”赋值的方法属性
3. 委托模式建议使用更加精确的命名，而且为了防止底层方法“遮盖”顶层方法，两者方法不能同名。因此这里无法使用多态，但是更加精确的命名也有助于开发者了解代码意图
4. 委托模式强调直接使用对象字面量，因此无法使用函数闭包来实现真正的“私有”属性，只能依托社区约定，也就是我们上面使用的下划线方式，例如：_\_variable_

这种模式虽然无法实现面向对象的三大特性，但很多时候，特别是 ES6 以前，我们也许并不需要实现复杂的继承，我们可以用行为委托来进行设计和编码，而且这种模式能够让你更加深刻地理解 JavaScript 的强大原型生态。你也可以在设计阶段更好的定义你的代码。

最后我们来看看两者在原型链结构上的比较：

1. 'class'继承

![](https://user-gold-cdn.xitu.io/2020/4/25/171b063248137b3f?w=659&h=577&f=png&s=41534)

2. behavior delegation

![](https://user-gold-cdn.xitu.io/2020/4/25/171b063bacb0d6f5?w=520&h=547&f=png&s=21321)

是不是简洁了很多呢？

## 总结

我们说过，一些软件设计和思想，在软件执行效率上可能并没有优势。但是针对软件开发者而不是机器来说，我们要做的是在开发阶段能够快速梳理编程思路，同时考虑代码的维护成本。这样来看，挑一个你喜欢的开发模式，进行编码吧！

## 参考

1. [Classes - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)
2. [About JavaScript - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/About_JavaScript)
3. [You Don't Know JS - 1st Edition](https://github.com/getify/You-Dont-Know-JS/tree/1st-ed)
4. [Encapsulation in JavaScript - Eric Elliott](https://medium.com/javascript-scene/encapsulation-in-javascript-26be60e325b4)
5. [oop - Is JavaScript object-oriented? - Stack Overflow](https://stackoverflow.com/questions/107464/is-javascript-object-oriented)
