# Handlebars

why
---
最近一直执着于自动化的代码生成，主要原因还是写很多项目的时候，总是会习惯性的维护一个框架，而维护框架时，就必然会有一组重复的编码工作（当然，可能是能力有限，没办法做到完全的把重复编码取消掉-_-），再加上框架本身也在不断的完善，所以每隔一段时间就会有一大段重复代码会被统一重构一轮，这部分工作其实是非常的浪费精力（虽然是有价值的）。

所以，身为一个懒人的我，很自然的想到通过一组配置文件来生成一组代码，这样一个框架只需要维护一套模板脚本就可以很方便的重构维护了（HoHo~~）。

以前也写过类似的东西，譬如几年前做的UI编辑器，就是通过格式化的PSD文件生成程序框架，在端游时代是直接生成c++、lua的代码，页游时代生成as3的代码，这样美术统一一套制作流程，可以适应不同的开发环境了。

当然，也还有一部分网络框架、静态表这类的代码生成，简单的说就是用脚本来生成代码，前面这些的实现方式大都是脚本硬写的，折腾过几次以后，希望用一些更现代化的方式来处理，于是开始找现成的模板引擎库（其实中间还折腾过一段时间的语法分析器，那个其实很有用的，等后面再来聊吧）。

好了，终于接近主题——**模板引擎**了。

因为近来很多工作都倾向于轻量化的脚本方式，一组脚本语言中尝试下来，最终选择了js，考虑到跨平台和方便性方面，又在js里面选择了nodejs。

js里面，其实有很多现成的模板引擎，毕竟做网页的多，但我想要的并不是html的模板引擎，我需要通用的模板引擎，于是就找到了今天的主角—— **[Handlebars](http://handlebarsjs.com/)** 。

Handlebars
---
**[Handlebars](http://handlebarsjs.com/)** 是一个弱逻辑（logic-less templeate）的js模板库，语法上兼容 **[Mustache](https://mustache.github.io/)** 。

基本语法
---
Handlebars表达式 是 Handlebars 模板中最基本的单元，使用方法是加两个花括号``{{value}}``, Handlebars 模板会自动匹配相应的数值。 

例如：

```
// 源数据
var source = '{{firstName}} {{lastName}} say: {{hi}}';
// 编译生成模板
var template = Handlebars.compile(source);

// 构造context
var context = {firstName: "Carl", lastName: "Lerche", hi: "Hi!"};
// 使用前面生成的模板生成最终输出
var output = template(context);
```

这时，output应该是``Carl Lerche say: Hi!``。

context也可以是一个复杂的对象。

```
const MODULENAME = '{{curmod.name_lc}}';

exports.Module_{{curmod.name}} = Module_{{curmod.name}};
```

> **注意，默认情况下，Handlebars 会处理转义（要支持HTML模板导出啊），但我们可以使用3个``{````}``(也就是``{{{value}}}```)来强制不转义的输出。**

Handlebars块表达式
---
在 Handlebars 里，``#``用来标识块表达式，块表达式需要配合``/``来结束，譬如：

```
<ul>
{{#each people}}
  <li>{{firstName}} {{lastName}}</li>
{{/each}}
</ul>
```

配合下面的context：

```
{
  people: [
    {firstName: "Yehuda", lastName: "Katz"},
    {firstName: "Carl", lastName: "Lerche"},
    {firstName: "Alan", lastName: "Johnson"}
  ]
}
```

可以得到：

```
<ul>
  <li>Yehuda Katz</li>
  <li>Carl Lerche</li>
  <li>Alan Johnson</li>
</ul>
```

这里的 ``each`` 是遍历数组或对象，遍历数组时，可以通过``{{@index}}``取到索引，遍历对象时，可以通过``{{@key}}``取到属性名。

Handlebars表达式的检索路径
---
支持常用的``.``和``../``，类似目录结构，``../``是到父节点，一般用在``each``里面，还有``this``表示当前层级。

```
  {{#each router_module}}
  {"name": "{{../projname_lc}}/mod/{{name}}", "src": "", "type": "dir", "jscode": "params.curmodrouter = params.router_module[{{@index}}]"},
  {{#each lstmodule}}
  {"name": "{{../../projname_lc}}/mod/{{../name_lc}}/{{name_lc}}.js", "src": "/main/mod/mod/mod.js", "type": "file", "jscode": "params.curmod = params.curmodrouter.lstmodule[{{@index}}]"},
  {{/each}}
  {{/each}}
```

能有这样的逻辑关系基本上就足够了。

> **注意：``{{@../index}}``是父节点的索引，这个是有效的。**

内置块表达式——``if``
---
这里的``if``其实功能很弱，只能判断对象的值，如果是``false``、``undefined``、``null``、``""``、``0``、``[]``、``{}``都算失败。

```
<div class="entry">
  {{#if author}}
    <h1>{{firstName}} {{lastName}}</h1>
  {{/if}}
</div>
```

如果``author``是``undefined``或者``[]``的时候，都会产生以下的输出：

```
<div class="entry">
</div>
```

我们还可以使用``else``：

```
<div class="entry">
  {{#if author}}
    <h1>{{firstName}} {{lastName}}</h1>
  {{else}}
    <h1>Unknown Author</h1>
  {{/if}}
</div>
```

内置块表达式——``unless``
---
``unless``就是``if``的反向表达式。

```
<div class="entry">
  {{#unless license}}
  <h3 class="warning">WARNING: This entry does not have a license!</h3>
  {{/unless}}
</div>
```

内置块表达式——``each``
---
遍历数组的例子上面有了，下面看个遍历对象的吧

```
{{#each dbmgr}}
exports.{{@key}}_host = '{{host}}';
exports.{{@key}}_user = '{{user}}';
exports.{{@key}}_pwd = '{{password}}';
exports.{{@key}}_name = '{{database}}';
{{/each}}
```

就我个人的使用来说，其实到这一步基本上就够了，当然，我还会配合js脚本来实现一些特殊的功能。

> **注意：``{{@../key}}``也是有效的。**

内置块表达式——``with``
---
``with``估计要在很复杂的情况下才有用，反正我是没用上，不过还是简单的看一下吧，这样一段模板：

```
<div class="entry">
  <h1>{{title}}</h1>

  {{#with author}}
  <h2>By {{firstName}} {{lastName}}</h2>
  {{/with}}
</div>
```

配合一个context：

```
{
  title: "My first post!",
  author: {
    firstName: "Charles",
    lastName: "Jolley"
  }
}
```

结果如下：

```
<div class="entry">
  <h1>My first post!</h1>

  <h2>By Charles Jolley</h2>
</div>
```

``with``还可以配合``else``来工作，譬如：

```
{{#with author}}
  <p>{{name}}</p>
{{else}}
  <p class="empty">No content</p>
{{/with}}
```

自定义的块表达式
---
这个我就跟没用上了，其实也就是给你一个机会实现类似``each``这样的效果。

感兴趣的同学自己看官方文档吧。

写在最后
---
一开始也说了，我折腾 **Handlebars** 的目的是自动化代码生成，专门为这个需求，我实现了 **[zHandlebars](https://github.com/zhs007/zhandlebars)** ，下次再详细的聊聊这个项目吧。

