### 正确使用 esm 和 cjs

esm 支持两种导入方式和三种导出方式：

```
// export
export default 'name';  // default export
export const name = 'hello esm'; // named export

// import
import lib from './lib';  // default import
import * as lib from './lib';
import { methodA, methodB } from './lib'; // 解构
```

cjs 只支持一种导出和导出方式

```
// export
module.exports = {
    a: 1,
    b: 2,
}

// 和上面等价，算同一种
exports.a = 1;
exports.b = 2;

// import
const lib = require('./lib');
console.log(lib.a);
console.log(lib.b);
```

### esm 中不支持的用法

```
// lib.js
export default {
    a: 1,
    b: 2,
}

// main.js
import { a, b } from './lib';
console.log(a);
console.log(b);
```

上面的用法是错误的, 此时 a,b 的打印结果是 undefined, undefined。export default 时，不支持使用分模块导出的方式。
其实 esm 的标准是希望我们进行按需导入，如果我们使用 export default ，其实是全局导入了，所以为了避免上面的问题，我们应该尽量采取下面的导出方式：

```
// lib.js
// 导出方式 1
const a = 1;
const b = 2;
export {
   a,b
}

// 导出方式 2
export const a = 1;
export const b = 2;
```
main.js

```
// 导入方式 1
import { a, b } from './lib';
console.log(a);
console.log(b);

// 导入方式 2，导入所有模块
import * as lib from './lib';
console.log(lib.a);
console.log(lib.b);
```

那如果我们一定要使用 export default ，应该咋办呢，下面方式也是正确的，如下：

```
// lib.js
export default {
    a:1,
    b:2,
}

import lib from './lib';
console.log(lib.a);
console.log(lib.b);

const { a, b } from './lib';
console.log(a);
console.log(b);
```

### 总结

使用 export default xxx 的导出，使用 import xxx from xxx 进行导入，对于 named import 的导出，使用 import * as lib from './lib' 和 import { a, b, c } from './lib' 进行导入

### 参考文献

https://zhuanlan.zhihu.com/p/40733281

https://zhuanlan.zhihu.com/p/97335917

