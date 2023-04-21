---
slug: namespace-and-module-in-typescript
title: "Re: TypeScript - Namespace? Module?"
authors: higan
tags: [ Typescript ]
---

TypeScript 是我最喜欢的脚本语言之一，静态类型的特性可以让 IDE 提供非常强大的 CodeLens 与 IntelliSense 功能，再加上 MS
出品的信仰加成，NodeJS 的方便快捷，简直是开发后端的不二之选。本文将会简单的介绍 TypeScript
中容易让人混淆的概念--命名空间（namespace）与模块（module）。
<!--truncate-->
# 文件与模块

在 TypeScript 中，每个源代码文件都可以理解为一个模块，例如：

**drinks.ts:**

```TypeScript
export class Cola { ... }

export class Sprite { ... }

export class Fanta { ... }
```

上述代码在一个 drinks.ts 文件中创建了 `Cola`，`Sprite`，`Fanta` 三个类，我们在其他文件中可以这样使用这些类。

**index.ts:**

```TypeScript
import * as Drinks from "./drinks";

let cola = new Drinks.Cola();
let sprite = new Drinks.Sprite();
let fanta = new Drinks.Fanta();
```

drinks.ts 作为一个模块被载入到了 "index.ts" 中。这就是 TypeScript 中的文件既模块的概念。

#全局与命名空间
由于 TypeScript 把每个文件都理解为独立的模块，文件之间除非 `import`，`export` 是没有任何关联的。作为一个被 C#
毒害很深的程序员，这种机制非常的反直觉，下面这种在 C# 中理所当然的现象也不会出现。

**./Animals/Wildlife.cs:**

```CSharp
namespace Animals {
    public class Wildlife { ... }
}
```

**./Animals/Serval.cs:**

```CSharp
namespace Animals {
    public class Serval:Wildlife { ... }
}
```

在 C# 里面我们习惯于将每个类都单独放在一个文件中，并使用命名空间将这些类组合在一起。而在 TypeScript 中 namespace 的概念与
C# 完全不一样，类似的概念就是放在全局空间 namespace了。

在 TypeScript 中下面的代码是合法的。
**./Animals/Wildlife.ts:**

```CSharp
namespace Animals {
    export class Wildlife { ... }
}
```

**./Animals/Serval.ts:**

```CSharp
namespace Animals {
    export class Serval:Wildlife { ... }
}
```

虽然 Wildlife.ts 与 Serval.ts 是两个不同的文件，但是 namespace 表示他们都是处于全局空间中的 Animals 命名空间中。这样看来，感觉和
C#
的用法差不多？其实不然，前面我们说过了，每个文件都是一个模块，直接使用全局命名空间的类型会导致运行时错误，所以我们需要使用 `--outFile`
编译选项，让所有在全局命名空间的类型编译到一个文件中。这样也算是符合 TypeScript 的文件既模块的概念。

```PowerShell
> tsc --outFile index.js Animals/Wildlife.ts Animals/Serval.ts
```

> 另外由于使用全局命名空间的时候，不能使用 import 导入其他模块，所以全局命名空间并不常用，可以放一些 Interface 的定义。
> # 在模块里的模块
> 之前我们提到了 TypeScript 中的文件既模块的思想，当一个文件包含一个模块的声明，那么这就是一个在模块中的模块。

**lunch.ts:**

```TypeScript
export module Drinks {
    export class Cola { ... }

    export class Sprite { ... }

    export class Fanta { ... }
}
export module Foods {
    export class Hamburger { ... }

    export class FrenchFries { ... }
}
```

在 lunch.ts 中包含了两个模块，两个模块里面又包含了各自的类型，当我们在其他文件导入这个模块的时候，可以有两种用法。

```TypeScript
// 将 lunch 导入当做 Lunch 的模块。
import * as Lunch from './lunch';

let food = new Lunch.Foods.Hamburger();

// 将 lunch 中的 Drinks，Foods 导入到当前语境。
import {Drinks, Foods} from './lunch';

let drink = new Drinks.Cola();
```

# export 与命名空间

上面我们讲到了命名空间是用于全局的一种方式，而当 namespace 遇到了 export 关键字，namespace 就变成了 module。
下面两段代码含义完全一致：

```TypeScript
export module Drinks {
    export class Cola { ... }

    export class Sprite { ... }

    export class Fanta { ... }
}
```

```TypeScript
export namespace Drinks {
    export class Cola { ... }

    export class Sprite { ... }

    export class Fanta { ... }
}
```

在之前的 TypeScript 版本中，namespace 被称作为内部模块，而 module 被称作外部模块，这也可以方便我们理解他们的关系。namespace
一般用于项目中不需要公开的部分，而 module 则可以被外部 import。当一个内部模块被 export 那么就说明这个内部模块变成了外部模块，那么
namespace 和 module 似乎就是一样的。

# 理解文件，模块与命名空间

之前在 SOF 中看到一个非常恰当的比喻 TypeScript 中文件，模块与命名空间概念。

## 三张桌子，三个盒子，三本书

我们看下面这三段代码。

**cola.ts:**

```TypeScript
export module Drinks {
    export class Cola { ... }
}
```

**sprite.ts:**

```TypeScript
export module Drinks {
    export class Sprite { ... }
}
```

**fanta.ts:**

```TypeScript
export module Drinks {
    export class Fanta { ... }
}
```

每个文件都是一张桌子，每个 module 都是一个盒子，每个 class 都是一本书。
这三段代码描述的就是，每个桌子上面都有一个盒子，每个盒子里面又都有一本书。
它们都是不一样的事物，每个桌子都是独特的，每个盒子也都是独特的，尽管它们可能长的一样，名字一样，但是它们仍然是独一无二的。

## 地板，盒子与三本书

再看下面三段代码。
**cola.ts:**

```TypeScript
namespace Drinks {
    export class Cola { ... }
}
```

**sprite.ts:**

```TypeScript
namespace Drinks {
    export class Sprite { ... }
}
```

**fanta.ts:**

```TypeScript
namespace Drinks {
    export class Fanta { ... }
}
```

全局空间就是地板，叫做 Drink 的命名空间是一个盒子，每个 class 都是一本书。
这三段代码描述的就是，在地板上摆放了一个叫做 Drink 的盒子，盒子里面放着三本书。
namespace 和 module 不一样，namespace 在全局空间中具有唯一性，也正是放在地板上，所以 namespace
才具有唯一性。桌子也是放在地板上，所以桌子也是具有唯一性（你不可能在同一个地方放同名的两个文件）。

## 三张桌子，三本书

接下来再看这三段代码。

**cola.ts:**

```TypeScript
export class Cola { ... }
```

**sprite.ts:**

```TypeScript
    export class Sprite { ... }
```

**fanta.ts:**

```TypeScript
export class Fanta { ... }
```

同样的，每个文件是一张桌子，每个 class 都是一本书。
这三段代码描述的就是，有三张桌子，每张桌子上面一本书。

# 适合大型 TypeScript 应用的项目结构

习惯了 C# 中每个类型一个文件，每个命名空间一个文件夹的代码结构，这样的结构对于大型应用来说，是十分直观明了的结构。TypeScript
也能这样做么？
答案是肯定的，在 Kanro 的编码过程中，为了让调用者方便，超过 3K 行的代码都放在同一个文件中，这对我后期的项目管理造成了很大的麻烦。
参考了很多开源项目，看了很多大家对大型项目的管理的方式，最后总结出了一个比较符合 C# 风格的项目结构。

```PowerShell
project/
└── src/
    ├── index.ts
    └── Drinks/
    │   ├── index.ts
    │   ├── Cola.ts
    │   ├── Sprite.ts
    │   └── Fanta.ts
    └── Foods/
        ├── index.ts
        ├── Hamburger.ts
        └── FrenchFries.ts
```

每个类型都用一个文件存放。
所有的 index.ts 都用于总结平级目录下的所有类型。

**project/src/Drinks/Cola.ts:**

```TypeScript
export class Cola { ... }
```

**project/src/Drinks/index.ts:**

```TypeScript
export * from "./Cola";
export * from "./Sprite";
export * from "./Fanta";
```

**project/src/index.ts:**

```TypeScript
import * as Drinks from "./Drinks";
import * as Foods from "./Foods";

export {Drinks, Foods};
```

这样的话，外部程序需要导入这个项目的时候只需要

```TypeScript
import * as Project from "project";

let drink = new Project.Drinks.Cola();
```

十分符合 C# 的编码习惯，也对开发者友好。
> 注意，平级的文件之间的引用只能通过显式指定文件的模式引用，而不能通过 index.ts 引用。通过 index.ts 引用可能会导致引用冲突。
> 例如： Sprite 需要引用 Cola 的时候只能使用 `import { Cola } from "./Cola"` 不能使用 `import { Cola } from "."` 的形式。