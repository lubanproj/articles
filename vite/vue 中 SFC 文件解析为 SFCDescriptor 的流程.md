我们平时写的 .vue 文件称为 SFC(Single File Components)。vue 会先对 .vue 文件进行解析，分成 template、script、styles、customBlocks 四个部分，称为 descriptor。之后，再对这四个部分分别进行编译最终得到可以在浏览器中执行的 .js 文件。本文介绍将 SFC 解析为 descriptor 这一过程。

SFCDescriptor，是表示 .vue 各个代码块的对象，为以下数据格式：

```
// an object format describing a single-file component.
declare type SFCDescriptor = {
    template: ?SFCBlock;
    script: ?SFCBlock;
    styles: Array<SFCBlock>;
    customBlocks: Array<SFCBlock>;
};
```

vue 提供了一个 compiler.parseComponent(file, [options])方法，来将 .vue 文件解析成一个 SFCDescriptor。

### 文件入口


解析 sfc 文件的源码入口在 src/sfc/parser.js 中，编译后的产出在 /packages/vue-template-compiler 和 /packages/vue-server-renderer 下的 build.js 中。

build.js 文件中直接 export 出了parseComponent方法。

首先我们来看看parseComponent方法都做了哪些事情。

##### parseComponent

```
/**
 * Parse a single-file component (*.vue) file into an SFC Descriptor Object.
 */
export function parseComponent (
  content: string,
  options?: Object = {}
): SFCDescriptor {
    const sfc: SFCDescriptor = {
        template: null,
        script: null,
        styles: [],
        customBlocks: []
    }
    let depth = 0
    let currentBlock: ?SFCBlock = null

    function start (tag: string, attrs: Array<Attribute>, unary: boolean, start: number, end: number) {}

    // ...

    function end (tag: string, start: number, end: number) {}

    // ...

    parseHTML(content, {
        start,
        end
    })

    return sfc
}

```
parseComponent方法中主要定义了start和end两个函数，之后调用了parseHTML方法来对 .vue 文件内容践行编译。start和end两个函数作为参数传给了parseHTML，我们等下再看。

先看下这个parseHTML方法是做啥的呢？

###### parseHTML

该方法看名字可以猜到是一个 html-parser。

parseHTML的代码细节较多，我们可以简单理解为：遍历解析查找文件中的各个标签，解析到每个起始标签时，调用 option 中的 start 方法进行处理；解析到每个结束标签时，调用 option 中的 end 方法进行处理。

对应到这里，就是分别调用parseComponent方法中定义的 start 和 end 函数进行处理。

由于我们这里只是想要找到第一层标签，也就是 template、script这些。因此可以在parseComponent中维护一个 depth 变量，在start中将depth++，在end中depth--。那么，每个depth === 1的标签就是我们需要获取的信息，包含 template、script、style 以及一些自定义标签。

接下来我们来看start和end中进行了哪些处理。

##### start

每当遇到一个起始标签时，执行start函数。

```
function start (
    tag: string,
    attrs: Array<Attribute>,
    unary: boolean,
    start: number,
    end: number
) {
    if (depth === 0) {
        currentBlock = {
            type: tag,
            content: '',
            start: end,
            attrs: attrs.reduce((cumulated, { name, value }) => {
            cumulated[name] = value || true
            return cumulated
            }, {})
        }
        if (isSpecialTag(tag)) {
            checkAttrs(currentBlock, attrs)
            if (tag === 'style') {
            sfc.styles.push(currentBlock)
            } else {
            sfc[tag] = currentBlock
            }
        } else { // custom blocks
            sfc.customBlocks.push(currentBlock)
        }
    }
    if (!unary) {
        depth++
    }
}

```

记录下 currentBlock。

每个 currentBlock 包含以下内容：

```
declare type SFCBlock = {
    type: string;
    content: string;
    start?: number;
    end?: number;
    lang?: string;
    src?: string;
    scoped?: boolean;
    module?: string | boolean;
};
```

根据 tag 名称，将 currentBlock 对象保存在在返回结果对象中。

返回结果对象定义为 sfc，如果tag不是 script,style,template 中的任一个，就放在 sfc.customBlocks 中。如果是style，就放在 sfc.styles 中。script 和 template 则直接放在 sfc 下。

```
if (isSpecialTag(tag)) {
    checkAttrs(currentBlock, attrs)
    if (tag === 'style') {
        sfc.styles.push(currentBlock)
    } else {
        sfc[tag] = currentBlock
    }
} else { // custom blocks
    sfc.customBlocks.push(currentBlock)
}
```
##### end

每当遇到一个结束标签时，执行end函数。

```
function end (tag: string, start: number, end: number) {
    if (depth === 1 && currentBlock) {
        currentBlock.end = start
        let text = deindent(content.slice(currentBlock.start, currentBlock.end))
        // pad content so that linters and pre-processors can output correct
        // line numbers in errors and warnings
        if (currentBlock.type !== 'template' && options.pad) {
            text = padContent(currentBlock, options.pad) + text
        }
        currentBlock.content = text
        currentBlock = null
    }
    depth--
}
```

如果当前是第一层标签(depth === 1)，并且 currentBlock 变量存在，那么取出这部分text，放在 currentBlock.content 中。

```
if (depth === 1 && currentBlock) {
  currentBlock.end = start
  let text = deindent(content.slice(currentBlock.start, currentBlock.end))
  // pad content so that linters and pre-processors can output correct
  // line numbers in errors and warnings
  if (currentBlock.type !== 'template' && options.pad) {
    text = padContent(currentBlock, options.pad) + text
  }
  currentBlock.content = text
  currentBlock = null
}

```

进行 depth--

在将 .vue 整个遍历一遍后，得到的 sfc 对象即为我们需要的 SFCDescriptor。

### 生成 .js

compiler.parseComponent(file, [options])得到的只是一个组件的 SFCDescriptor，最终编译成.js 文件是交给 vue-loader 等库来做的。

原文：https://meixg.cn/2018/04/23/vue-sfc-parser/

