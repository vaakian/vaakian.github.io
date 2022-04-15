---
title: "TypeScript: Covariance & Contravariance"
date: 2022-04-15T13:36:51+08:00
draft: false
categories: ["TypeScript"]
---

> 个人复习用，不做科普。

创建两个类，`Dog`是`Animal`的子类。
```ts
class Animal {}

class Dog extends Animal {
    bark() {}
}

```

实现一个判断类型的`Type`，判断`T`是否是`U`的子类。
```ts
type Assignable<T, U> = T extends U ? true : false
```


测试，符合结果。
```ts
type R4 = Assignable<Dog, Animal> // true
```

但将同样的问题放到函数的`parameters`上，就发生了与第一直觉不一样的情况。

```ts

type ConsumeAnimal = (target: Animal) => Animal
type ConsumeDog = (target: Dog) => Dog

type R1 = Assignable<Son, Father> // false
type R2 = Assignable<Father, Son> // true

type R3 = Assignable<Parameters<Son>, Parameters<Father>> // true
```

## 问题出在哪里？
通过Google，找到两个关键词：`covariance`和`contravariance`

<!-- 1. 什么是covariance？

- co => together
- variance => change

所以意为，一起发生变化，A降则B降，A升则B升，反之亦然。

2. 什么是contravariance？

- contra => opposite
- variance => change

所以与covariance相反，A升则B降，A降则B升，反之亦然。 -->

![](images/20220415135421.png) 

### contravariance

将函数`ConsumeAnimal`看作为`Consumer`，它消费一个类`Animal`，保证参数`target: Animal`存在Animal的所有属性和方法，该函数绝对访问不会超过`Animal`之外的属性。
而ConsumeDog可能会访问到`Animal`的属性，**但也可能访问到`Dog`的属性**，这时候已经超过了`ConsumeAnimal`的范围，因为它拿到的只能是一个`Animal`!

而反过来看，`ConsumeDog`消费一个`Dog`类，保证参数`target: Dog`存在`Dog`的所有属性和方法。
如果`ConsumeAnimal`所做的行为，同样可以应用到类型`Dog`身上，因为`Dog`类包含`Animal`的所有属性和方法。


用以下代码解释上面的问题：

```ts
type ConsumeAnimal = (target: Animal) => Animal
type ConsumeDog = (target: Dog) => Dog

declare const consumeAnimal: ConsumeAnimal
declare const consumeDog: ConsumeDog

consumeAnimal = consumeDog
^^^^^^^^^^^^
// Property 'bark' is missing in type 'Animal' but required in type 'Dog'
```

`consumeAnimal`函数能传入`Animal`和`Dog`，因为`Dog`也能提供所有`Animal`的属性。\
而将`consumeDog`赋值给`consumeAnimal`，\
当向其传入`Animal`时，`consumeDog`有可能会访问`Animal`上所不存在的属性`bark`，超过了`Animal`的范围。

而以下代码就没有问题
```ts
declare const consumeAnimal: ConsumeAnimal
declare const consumeDog: ConsumeDog
consumeDog = consumeAnimal
```
`ConsumeDog`可以传入`Dog`，但不可以传入`Animal`，因为`Animal`不能保证提供所有`Dog`的属性。\
所以实际的消费函数consumeAnimal可以非常安全的消费一只`Dog`，所以`consumeAnimal`可以赋值给`consumeDog`。


### covariance

但对于对象实例来讲，它们可以作为一个`Provider`。和以上函数的情况是相反的。

```ts
declare let animal: Animal
declare let dog: Dog
animal = dog
dog = animal
^^^
// Property 'bark' is missing in type 'Animal' but required in type 'Dog'
```
因为`dog`属于`animal`的子类，所以`dog`可以很安全的赋值给一个`animal`类型。因为`dog`能过提供所有`animal`的属性。
反之，`animal`不能安全的赋值给`dog`，它可能不包含某些`dog`的属性。

## 总结
上面一堆绕话，是为了理清思路。这里尝试总结一下，供以后回顾。

`A ≼ B`表示A是B的子类\
`T → U`表示一个函数参数类型为T，返回类型为U\
那那：
- 对于类「实例」A ≼ B，是covariance。
- 对于函数「返回值」来讲 (T → A) ≼ (T → B)，是covariance。
- 对于函数「参数」 (B → T) ≼ (A → T)，是contravariance。

## 其它

还有关于一个`List<T>`的`invariant`情况（该作者的解释简单明了，所以直接引用过来了）：

Could `List<Dog>` be a subtype of `List<Animal>`?

The answer is a little nuanced. If lists are immutable, then it’s safe to say yes. But if lists are mutable, then definitely not!

Why? Suppose I need a `List<Animal>` and you pass me a `List<Dog>`. Since I think I have a `List<Animal>`, I might try to insert a Cat into it. Now your `List<Dog>` has a Cat in it! The type system should not allow this.
Formally: we can allow the type of immutable lists to be covariant in its type parameter, but the type of mutable lists must be invariant (neither covariant nor contravariant) in its type parameter.
## 参考资料

https://www.stephanboyer.com/post/132/what-are-covariance-and-contravariance
https://dmitripavlutin.com/typescript-covariance-contravariance/
https://dev.to/codeoz/how-i-understand-covariance-contravariance-in-typescript-2766
