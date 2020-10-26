# 一.数据驱动

## 1.前沿：Vue 与模板

使用步骤:

1. 编写 页面 模板
   1. 直接在 HTML 标签中写 标签
   2. 使用 template
   3. 使用 单文件 ( <template /> )
2. 创建 Vue 的实例
   1. 在 Vue 的构造函数中提供: data, methods, computed, watcher, props, ...
3. 将 Vue 挂载到 页面中 ( mount )

```js
new Vue({
  el: "#app",
  data: {
    message: "liu",
  },
});
//vuejs会自动的将其挂载到页面中
```

## 2.数据驱动模型

### Vue 中数据驱动模型的执行流程

1. 获得模板: 模板中有 "坑"
2. 利用 Vue 构造函数中所提供的数据来 "填坑", 得到可以在页面中显示的 "标签"
3. 将标签替换页面中原来有坑的标签

Vue 利用 我们提供的数据 和 页面中 模板 生成了 一个新的 HTML 标签 ( node 元素 ),
替换到了 页面中 放置模板的位置.

### 数据驱动模型的实现：

#### 简单实现：

```js
// 用于替换的正则表达式规则
let regExp = /\{\{(.+?)\}\}/g;
// 1.拿到模板：
let template = document.querySelector("#root");
// 2.拿到数据：
let data = {
  name: "liu",
  age: 21,
};
// 3.将数据和模板结合，得到一个新的DOM元素：
function compiler(node, data) {
  // 3.1获取所有子节点
  let childNodes = node.childNodes;
  for (var i = 0; i < childNodes.length; i++) {
    let type = childNodes[i].nodeType;
    if (type == 3) {
      // 3.2文本节点才应用正则表达式进行替换
      var text = childNodes[i].nodeValue.replace(regExp, function (match, $1) {
        return data[$1.trim()]; //trim()的作用是清除掉模板字符前后的空格，比如{{   name   }}
      });
      // 3.2.1将该文本节点的内容重新设置为最新替换的内容
      childNodes[i].nodeValue = text;
    } else if (type == 1) {
      // 3.3对于元素节点，需要进行递归
      compiler(childNodes[i], data);
    }
  }
}
// 4.为了不改变原来的模板，需要克隆出一个新的dom元素：
var generateNode = template.cloneNode(true);
compiler(generateNode, data);
// 5.将新的DOM元素替换掉原来的dom元素：
template.parentNode.replaceChild(generateNode, template);
console.log(root);
```



#### 封装为一个Vue构造函数：

```js
let regExp = /\{\{(.+?)\}\}/g;
function compiler(node, data) {
    let childNodes = node.childNodes;
    for (var i = 0; i < childNodes.length; i++) {
        let type = childNodes[i].nodeType;
        if (type == 3) {
            var text = childNodes[i].nodeValue.replace(regExp, function (
                                                       match,
                                                        $1
                                                       ) {
                return data[$1.trim()];
            });
            childNodes[i].nodeValue = text;
        } else if (type == 1) {
            compiler(childNodes[i], data);
        }
    }
}
function VueLiu(options) {
    // 内部的数据使用下划线 开头, 只读数据使用 $ 开头
    this._data = options.data;
    this._el = options.el;
    this.$el = this._templateDOM = document.querySelector(this._el);
    // 渲染工作：
    this.render();
}
//   将数据与模板结合，然后再添加到页面中
VueLiu.prototype.render = function () {
    this.compiler();
};
//   将数据与模板结合
VueLiu.prototype.compiler = function () {
    let realHTMLDOM = this._templateDOM.cloneNode(true);
    compiler(realHTMLDOM, this._data);
    this.update(realHTMLDOM);
};
//   将DOM元素放在页面上
VueLiu.prototype.update = function (realHTMLDOM) {
    this._templateDOM.parentNode.replaceChild(
        realHTMLDOM,
        this._templateDOM
    );
};
new VueLiu({
    el: "#app",
    data: {
        name: "liu",
        age: 21,
    },
});
```



#### 处理带有层级的data数据：

```js
<h1> {{info.name}} </h1>
// 根据字符串路径获取对象中的某个数据
// 核心：一层一层的获取，直到最后一层
function getValueByPath(data, path) {
    //   path: info.name
    let paths = path.split("."); //[info,name]
    let value = data;
    while (paths.length) {
        value = value[paths.shift()];
    }
    return value;
}
let data={
    info:{
        name:'liu'
    }
}
getValueByPath(data,'info.name')//liu
```



#### 创建虚拟DOM：

##### **1.什么是虚拟DOM：**

在内存中通过js对象来描述的一个dom结构

##### **2.虚拟DOM的一个简单的数据结构：**

```js
class VNode {
    constructor(tag, data, value, type) {
        this.tag = tag && tag.toLowerCase(); //当前节点的标签名
        this.date = data; //当前节点的属性
        this.value = value; //文本节点的值
        this.type = type; //当前节点的类型
        this.children = []; //当前节点的子节点
    }
    // 添加子节点
    appendChild(vnode) {
        this.children.push(vnode);
    }
}
```

##### **3.将真正的DOM转化为虚拟DOM：**

```js
//   使用递归的方式生成虚拟DOM：
//   Vue实际是采用栈+递归来实现的
//   实际上是将dom元素生成一个字符串，然后再得到抽象语法树，再将其转换为一棵虚拟DOM？？？？
function generateVNode(node) {
    // 获取节点的类型：
    let nodeType = node.nodeType;
    let _vnode = null; //根据实际dom生成的虚拟DOM
    if (nodeType == 1) {
        // 元素节点：
        var nodeName = node.nodeName;
        var attributes = node.attributes; //获取节点所有的属性节点
        var _attributes = {}; //以为键值对的形式存储所有的属性
        for (var i = 0; i < attributes.length; i++) {
            _attributes[attributes[i].nodeName] = attributes[i].nodeValue;
        }
        _vnode = new VNode(nodeName, _attributes, undefined, nodeType);
        //   获取当前节点的子节点
        var children = node.childNodes;
        //   以此将每一个子节点转换为虚拟的DOM，并添加在父节点的children属性中
        for (var i = 0; i < children.length; i++) {
            let childVnode = generateVNode(children[i]);
            _vnode.appendChild(childVnode);
        }
    } else if (nodeType == 3) {
        // 文本节点：
        _vnode = new VNode(undefined, undefined, node.nodeValue, nodeType);
    }
    return _vnode;
}
```

##### **4.通过虚拟DOM转化为真正的DOM：**

```js
//   将虚拟DOM转换为真正的DOM：
function parseVNode(vnode) {
    let node = null;
    if (vnode.type == 1) {
        // 创建元素节点
        node = document.createElement(vnode.tag);
        // 设置当前节点的属性
        for (var i in vnode.VNode) {
            if (data.hasOwnProperty(i)) {
                node.setAttribute(i, data[i]);
            }
        }
        //   设置当前节点的子节点：
        let children = vnode.children;
        for (var i = 0; i < children.length; i++) {
            var childNode = parseVNode(children[i]);
            node.appendChild(childNode);
        }
    } else if (vnode.type == 3) {
        // 文本节点：
        node = document.createTextNode(vnode.value);
    }
    return node;
}
```





#### (优化)函数柯里化：

**柯里化在vue中的应用1：**

```js
/* 
	什么是柯里化：
		一个函数原本有多个参数, 每次只传入一个参数, 生成一个新函数, 由新函数接收剩下的参数来运行得到结构.
    函数柯里化的目的：
        缓存一些内容（利用闭包？），减少冗余代码的解析。
        这样在频繁的调用时可以提高一些性能。            
*/
// 实际情况下，path的值始终是不变的，改变的是data中的数据，所以可以将path的值缓存起来，
function createGetValueByPath(path) {
    let paths = path.split(".");
    return function getValueByPath(data) {
        let paths = path.split(".");
        let value = data;
        while (paths.length) {
            value = value[paths.shift()];
        }
        return value;
    };
}
let getValueByPath = createGetValueByPath("info.name");
let data = {
    info: {
        name: "liu",
    },
};
let rs = getValueByPath(data);
alert(rs); //liu
//   当data中的数据改变时，只需要做对应的处理
rs = getValueByPath({
    info: {
        name: "liu2",
    },
});
alert(rs); //liu2

```



**判断一个标签是否为内置的标签：**

```js
// 原始方法： 时间复杂度：O(n)
//   let tags = "div,p,a,img,ul,li".split(",");
//   function isHTMLTag(tagName) {
//     tagName = tagName.toLowerCase();
//     if (tags.indexOf(tagName) > -1) return true;
//     return false;
//   }

//   利用柯里化进行优化：时间复杂度：O(1)
let tags = "div,p,a,img,ul,li".split(",");
function makeMap(keys) {
    var set = {};
    tags.forEach((key) => {
        set[key] = true;
    });
    return function (tagName) {
        return !!set[tagName]; //这里!!的作用时可以将undefined转换为false
    };
}
let isHTMLTag = makeMap(tags);
console.log(isHTMLTag("user")); //false
console.log(isHTMLTag("img")); //true
```

