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

### Vue 的执行流程

1. 获得模板: 模板中有 "坑"
2. 利用 Vue 构造函数中所提供的数据来 "填坑", 得到可以在页面中显示的 "标签"
3. 将标签替换页面中原来有坑的标签

Vue 利用 我们提供的数据 和 页面中 模板 生成了 一个新的 HTML 标签 ( node 元素 ),
替换到了 页面中 放置模板的位置.

### 具体实现步骤：

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

# 简单的模板渲染

# 虚拟 DOM

目标:

1. 怎么将真正的 DOM 转换为 虚拟 DOM
2. 怎么将虚拟 DOM 转换为 真正的 DOM

思路与深拷贝类似
