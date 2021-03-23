# @rotcare/codegen

把 TypeScript 项目分解为多个 Git 仓库来编写，并提供 TypeScript => JavaScript 的全套构建工具链。

## 拆分的意义是什么？

为了控制依赖，避免产生所有人依赖所有人的糟糕结果。

![monolith](./monolith.drawio.svg)

我们有必要把代码拆分成多个 Git 仓库，并通过限制 Git 仓库之间的依赖关系，来促使代码往高内聚低耦合的方向发展。

![motherboard+plugin](./motherboard-plugin.drawio.svg)

比如上图中，插件 Git 仓库与插件 Git 仓库之间不能有依赖关系。这样我们就迫使所有的集成代码必须下沉到主板 Git 仓库里去表达。

然而我们在把业务逻辑拆分成多个 Git 仓库来编写的时候，就遇到了代码如何整合回来的问题。

* 主板和插件怎么整合到一起？
* 顶层的那个项目只起到聚合代码的作用，那是不是可以代码生成出来？

`@rotcare/codegen` 所演示的能力，就是利用编译期的工具，在 TypeScript => JavaScript 的过程中，把拆分的代码文件整合回一个文件。这种整合代码的方式，是在所有限制管控依赖关系的技术里，实现代价最低的方案。

## 主板 + 插件是如何粘合的？

在主板中定义一个 class，把其中的一些 method 用注释标记为 virtual 的。然后在插件的同位置文件中，定义一个同名的 class，写一个同名的 method 用注释标记为 override 的。这样就实现了主板和插件的对应关系。在 TypeScript => JavaScript 的构建过程里，会把 override 的 method 顶替掉 virtual 的 method。

例如在主板的 [Home/Private/SomeCommand.ts](https://github.com/rotcare/demo/blob/main/demos/demo-composite-motherboard/Home/Private/SomeCommand.ts) 中我们定义了

```ts
export class SomeCommand {
    public async run() {
        await this.step1();
        await this.step2();
    }
    /**
     * @virtual
     */
    protected async step1() {
    }
    /**
     * @virtual
     */
    protected async step2() {
    }
}
```

我们写了一个插件1，其同位置  [Home/Private/SomeCommand.impl.ts](https://github.com/rotcare/demo/blob/main/demos/demo-composite-plugin1/Home/Private/SomeCommand.impl.ts) 中我们定义了同名的 class：

```ts
import { SomeCommand as INF } from '@motherboard/Home/Private/SomeCommand';

export class SomeCommand extends INF {
    /**
     * @override
     */
    protected async step1() {
        // 演示 error 的 stack trace 能够正确被 source map
        console.log('step1 provided by plugin1', new Error());
    }
}
```

注意到插件中覆盖主板的文件最好命名为 .impl.ts 的后缀，以清晰的显示这个文件是用来实现主板定义的抽象接口的。同时，为了让 TypeScript 文件在 IDE 中能够有正确的代码提示，`SomeCommand extends INF` 继承了主板定义的 SomeCommand。这个继承关系仅仅是为了让 IDE 能够出正确的代码提示，并不会真正有这么一个继承关系。其中 `from '@motherboad/Home/Private/SomeCommand` 引用了 `@motherboard` 这个别名，其定义于 [tsconfig.json](https://github.com/rotcare/demo/blob/main/demos/demo-composite-plugin1/tsconfig.json) 中：

```json
{
    "compilerOptions": {
        "target": "es2018",
        "strict": true,
        "moduleResolution": "node",
        "experimentalDecorators": true,
        "module": "CommonJs",
        "strictPropertyInitialization": false,
        "jsx": "react",
        "noEmit": true,
        "paths": {
            "@motherboard/*": ["../demo-composite-motherboard/*"]
        },
        "baseUrl": "."
    }
}
```

类似于插件1，我们还可以写一个[插件2](https://github.com/rotcare/demo/blob/main/demos/demo-composite-plugin2/Home/Private/SomeCommand.impl.ts)：

```ts
import { SomeCommand as INF } from '@motherboard/Home/Private/SomeCommand';

export class SomeCommand extends INF {
    /**
     * @override
     */
    protected async step2() {
        console.log('step2 provided by plugin2');
    }
}
```

然后可以用过一条命令 `yarn rotcare-show Home/Private/SomeCommand` 得知粘合之后的 JavaScript 代码是什么样子的：

```bash
cd demos/demo-composite-project
yarn rotcare-show Home/Private/SomeCommand
```

注意需要进入 demo-composite-project 执行，要不然 rotcare-show 不知道要粘合哪些插件和主板。输出的 JavaScript 是

```js
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports.SomeCommand = void 0;

class SomeCommand {
  async run() {
    await this.step1();
    await this.step2();
  }

  async step1() {
    // 演示 error 的 stack trace 能够正确被 source map
    console.log('step1 provided by plugin1', new Error());
  }

  async step2() {
    console.log('step2 provided by plugin2');
  }

}

exports.SomeCommand = SomeCommand;
```

可以看到所谓粘合，就是类似文本替换的效果。

## 界面如何粘合？

前面演示了服务端的一个命令的两个步骤，是如何拆分成两个插件来实现。那么如果一个界面上有多个区块，要拆分为多个插件来实现，怎么弄呢？

也是一样的实现方案。所谓界面，就是一个 render 函数。比如我们定义这样的一个 React 组件在主板的 [Home/Ui/HomePage.tsx](https://github.com/rotcare/demo/blob/main/demos/demo-composite-motherboard/Home/Ui/HomePage.tsx)：

```tsx
import * as React from 'react';

export class HomePage {
    public render() {
        return (
            <div>
                {this.renderLine1()}
                <hr />
                {this.renderLine2()}
                <div>
                <button
                    onClick={() => {
                        fetch('http://localhost:3000/SomeCommand', { method: 'POST' });
                    }}
                >
                    call SomeCommand
                </button>
                </div>
            </div>
        );
    }
    /**
     * @virtual
     */
    protected renderLine1() {
        return <></>;
    }
    /**
     * @virtual
     */
    protected renderLine2() {
        return <></>;
    }
}
```

这里就是一个标准的 React 函数组件。定义在一个 HomePage class 里，仅仅是为了让插件可以 impl 其中的 renderLine1 和 renderLine2 这两个 method。并不是什么 class component。然后我们定义 renderLine1 的实现[在插件1中](https://github.com/rotcare/demo/blob/main/demos/demo-composite-plugin1/Home/Ui/HomePage.impl.tsx)：

```tsx
import { HomePage as INF } from '@motherboard/Home/Ui/HomePage';
import * as React from 'react';

export class HomePage extends INF {
    /**
     * @override
     */
    protected renderLine1() {
        // 演示 error 的 stack trace 能够正确被 source map
        console.log(new Error());
        return <span>line 1 provided by plugin1</span>
    }
}
```

这样就可以了。

## 聚合项目怎么用代码生成来搞？

聚合项目需要把插件和主板粘合到一起。这个通过定义 demo-composite-project 的 [package.json](https://github.com/rotcare/demo/blob/main/demos/demo-composite-project/package.json) 声明一下，构建时候知道要粘合什么了：

```json
{
    "name": "@rotcare/demo-composite-project",
    "version": "0.1.0",
    "license": "MIT",
    "rotcare": {
        "project": "composite"
    },
    "dependencies": {
        "@rotcare/demo-composite-motherboard": "*",
        "@rotcare/demo-composite-plugin1": "*",
        "@rotcare/demo-composite-plugin2": "*"
    }
}
```

因为声明了 rotcare.project 为 composite，所以就知道要把 dependencies 里的 package 都用粘合的方式弄到一起去构建。

除此之外，对外还需要提供一个完整的功能入口。比如说，监听 3000 端口，并响应 http 请求。那么，聚合项目里势必是要写路由派发的代码，就需要有人来维护这个聚合项目。但这样的样板代码是枯燥无味的，我们可以用代码生成来解决。比如在 demo-composite-project 中定义 [backend.ts](https://github.com/rotcare/demo/blob/main/demos/demo-composite-project/backend.ts) 文件：

```ts
import * as http from 'http';
import { codegen, Model } from '@rotcare/codegen';

// 用代码生成的方式生成路由代码，避免在这里手工 require 每一个后端的 RPC 方法
const routes = codegen<Record<string, () => void>>((...models: Model[]) => {
    const lines = [`const routes={}`];
    for (const model of models) {
        if (model.qualifiedName.includes('/Private/') || model.qualifiedName.includes('/Public/')) {
            lines.push(`routes['/${model.tableName}'] = () => new (require('@motherboard/${model.qualifiedName}').${model.tableName})().run()`);
        }
    }
    lines.push('return routes')
    return lines.join('\n') as any;
})

new http.Server((req, resp) => {
    resp.setHeader('Access-Control-Allow-Origin', '*');
    // CORS 有一个预检
    if (req.method === 'OPTIONS') {
        resp.setHeader('Access-Control-Allow-Headers', '*');
        resp.end('');
        return;
    }
    routes[req.url!]();
    resp.end();
    return;
}).listen(3000);
```

这里使用了 `@rotcare/codegen` 提供的 `codegen` 函数。这个对 codegen 的调用，就类似于 C/C++ 里的宏。在 TypeScript => JavaScript 的构建过程中，会对 codegen 的回调进行求值，并把求值得到的 JavaScript 源代码粘贴到 codegen 的调用处。我们可以用前面用过的 `yarn rotcare-show backend` 来查看一下代码生成的结果：

```js
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports.routes = void 0;

var http = _interopRequireWildcard(require("http"));

function _getRequireWildcardCache() { if (typeof WeakMap !== "function") return null; var cache = new WeakMap(); _getRequireWildcardCache = function () { return cache; }; return cache; }

function _interopRequireWildcard(obj) { if (obj && obj.__esModule) { return obj; } if (obj === null || typeof obj !== "object" && typeof obj !== "function") { return { default: obj }; } var cache = _getRequireWildcardCache(); if (cache && cache.has(obj)) { return cache.get(obj); } var newObj = {}; var hasPropertyDescriptor = Object.defineProperty && Object.getOwnPropertyDescriptor; for (var key in obj) { if (Object.prototype.hasOwnProperty.call(obj, key)) { var desc = hasPropertyDescriptor ? Object.getOwnPropertyDescriptor(obj, key) : null; if (desc && (desc.get || desc.set)) { Object.defineProperty(newObj, key, desc); } else { newObj[key] = obj[key]; } } } newObj.default = obj; if (cache) { cache.set(obj, newObj); } return newObj; }

const routes = (() => {
  const routes = {};

  routes['/SomeCommand'] = () => new (require('@motherboard/Home/Private/SomeCommand').SomeCommand)().run();

  return routes;
})();

exports.routes = routes;
new http.Server((req, resp) => {
  resp.setHeader('Access-Control-Allow-Origin', '*'); // CORS 有一个预检

  if (req.method === 'OPTIONS') {
    resp.setHeader('Access-Control-Allow-Headers', '*');
    resp.end('');
    return;
  }

  routes[req.url]();
  resp.end();
  return;
}).listen(3000);
```

其中 `const routes` 就代表了代码生成之后的内容，其余部分的代码可以用 routes 来引用生成的代码。在命令行启动 http 服务时，我们使用：

```bash
node -r @rotcare/register backend.ts
```

这里的 -r 参数就完成了 TypeScript => JavaScript 的构建工作，用起来就和写 JavaScript 的体验是差不多的。

利用上述的两种机制，我们就可以做到 demo-composite-project 项目是自动生成的，而不需要频繁修改。值得注意的是，上面的例子里，界面用的就是 React，也可以换成 Vue 等任意前端框架。后端代码里用的就是 node.js 的 http，不和任何的 RPC 框架耦合，可以随意使用自己中意的框架。codegen 的机制是在 TypeScript 源代码层面工作的，可以和任意的库，任意的框架进行集成。比如我们可以用 codegen 生成 react-router 的页面路由入口。

## codegen 的其他用途

codegen 是通用的编译期代码生成，当然可以用来做表单生成之类的事情。比如根据表单的字段定义

```ts
export class EnrollmentForm {
    public studentName: string;
    public studentAge: number;
    public course: string;
}
```

生成这个表单的 React 组件

```ts
import { codegen, Model } from '@rotcare/codegen';
import { generateFormEditor } from '@rotcare/demo-codegen-form';
import { EnrollmentForm } from './EnrollmentForm';

// 渲染一个表单，把用户提交的值绑定到表单对象上
export const EnrollmentFormEditor = codegen(
    (enrollment: Model<EnrollmentForm>) => {
        return generateFormEditor(enrollment, {
            studentAge: '学生年龄',
            studentName: '学生姓名',
            course: '课程',
        });
    }
);
```

然后渲染出来

```tsx
import { EnrollmentFormEditor } from '../../WidgetCodegen/Ui/EnrollmentFormEditor';
import * as React from 'react';
import { EnrollmentForm } from '../../WidgetCodegen/Ui/EnrollmentForm';

export function HomePage() {
    const [form, _] = React.useState(() => new EnrollmentForm());
    return (
        <div>
            <EnrollmentFormEditor form={form}/>
            <button onClick={() => {
                alert(JSON.stringify(form, undefined, '  '));
            }}>提交</button>
        </div>
    );
}
```

完整的代码 https://github.com/rotcare/demo/tree/main/demos/demo-codegen-form

可以把一些不太方便抽函数的规律和模式，用抽 generateFormEditor 的方式来实现沉淀。当然，要留心三点：

* 如果能够用简单的抽函数的地方，应该优先用抽函数来实现。
* 以 UI 一致性规范等共识为主。不要研发自己来倒推规范，UI 设计师和产品经理可能不是这么想的。
* 如果一个 Model 要被多个生成器使用，或者用于很多用途。不要把所有的生成器的参数都以 decorator 的形式往 Class 定义上加。这样相当于把函数的参数，改成用全局变量来实现。

比如如果后端有一个 Enrollment 的表，那么是不是可以根据 Enrollment 表来生成 EnrollmentForm 和 EnrollmentFormEditor 呢？可能可以，可能不可以。取决于你的业务需求。

## 相关的包

* [`@rotcare/codegen`](https://github.com/rotcare/codegen)：提供 codegen 的 API，可以业务代码里引用。会在构建的时候做代码生成，并把生成的代码插入。
* [`@rotcare/project`](https://github.com/rotcare/project)：提供 TypeScript => JavaScript 的构建工具，在构建脚本中使用。
* [`@rotcare/register`](https://github.com/rotcare/register)：包装了 `@rotcare/project` 提供的构建能力，和 Node.js 做好了对接，用 `node -r @rotcare/register` 就可以在执行的时候即时完成构建转译。
* [`@rotcare/project-esbuild`](https://github.com/rotcare/project-esbuild)：包装了 `@rotcare/project` 提供的构建能力，和 esbuild 做好了对接，提供了 esbuildPlugin 这个插件，插入到 esbuild 的构建体系里。

