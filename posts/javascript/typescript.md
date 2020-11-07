# typescript

## typescript类的使用

```typescript
class Person {
  public name: string  // = init, 
  // age: number
  private age: number   // 访问修饰符,如private只能在类的内部使用, 默认为public
  // protected gender: boolen    // 只允许在继承后的子类中访问成员
  protected readonly gender: boolean    // 只读属性,可以在类型声明中通过等号初始化或构造器中初始化

  constructor (name: string, age: number) {
    this.name = name    // 直接访问会报错,使用之前要明确声明属性
  }
  say (msg: string): void {
    console.log(`${this.name}-${this.age}`)
  }
}

const Aron = new Person('Aron', 18)
console.log(Aron.age)   // 无法访问
```

`constructor`也可以添加修饰符,如果添加了private,则外部不能实例化,只能在类的内部通过静态方法创建实例

### 接口

一种规范,用来约定对象的结构,使用一个接口必须遵循这个接口的全部约定,在ts中,最直观的体现是约定一个对象的成员,成员的类型

```typescript
interface Post {
  // 使用逗号或分号分割,可省略
  title: string
  content: string
  // 可选成员
  subtitle?: string   // string或undefined
  // 只读成员
  readonly summay: string
  // 动态成员---动态的键值
  [key: string]: string // key只是一个格式,可以为任意名称
}

function printPost (post: Post) {

}
printPost({
  title: 'title',
  content: 'content'
})
```

typescript中的接口只是为了给有结构的数据做类型约定,实际运行阶段并没有意义

### 类与接口

比如以下示例,人和动物类有相同的特性,比如吃东西,但是行为仍然是不一样的,这时候抽象为一个公共的父类不合适,这里可以使用接口实现公共的能力

```typescript
// 其他语言的接口建议是让接口定义更加简单,细化,因此一个接口只约束一个能力
interface Eat {
  eat (food: string): void
}

interface Run {
  run (distance: number): void
}

class Person implements Eat, Run {
  eat (food: string): void {
    console.log('人吃东西')
  },
  run (distance: number) {
    console.log(`人走了${number}`)
  }
}

class Animal implements Eat, Run {
  eat (food: string): void {
    console.log('动物吃东西')
  },
 run (distance: number) {
    console.log(`动物爬了${number}`)
  }
}
```

### 抽象类

类似接口,约束子类必须要有哪些成员,不同于接口,抽象类可以包含具体的实现,接口只是成员的抽象

```typescript
// 只能被继承不能实例化
abstract class Animal {
  eat (food: string): void {
    console.log(`动物吃${food}`)
  }
  // 定义抽象方法,不需要方法体,子类必须实现
  abstract run (distance: number): void
}

class Dog extends Animal {
  run(distance: number): void {
    console.log(`狗爬了${distance}`)
  }
}
```

### 泛型

定义函数,接口,类的时候没有定义具体的类型,使用的时候再指定具体类型,方便复用代码

```typescript
// 调用函数时传递具体类型
function createArray<T> (length: number, value: T): T:[] {
  return Array<T>(length).fill(value)   // Array默认创建的是any类型
}

const res = createArray<string>(3, 'aa')
```

### 类型声明

```typescript
declare function camelCase (input: string): string
```