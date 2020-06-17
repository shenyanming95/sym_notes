# 1.MVVM模式

后端有`MVC`模式，前端有`MVVM`模式，何为`MVVM`模式？简单地来说，M还是指数据，V即视图，VM则是数据与视图之间的调度器。

![](./images/MVVM模式.png)

VM与Model双向绑定，既可以将数据写入Model也可以从Model读出数据，然后将数据展示到View上。起初使用jQuery，假如需要操作一个DOM的数据，需要是$()先选取，再调用适当的方法改变它的数据；而Vue抛弃这一操作，它采用MVVM模式，用一个调度器来控制视图和数据，使用者只需要关心“在哪里（where） 做什么（what），而不需要直接操作DOM，留更多精力关注业务逻辑。

# 2.初识vue

## 2.1.简介

Vue (读音 /vjuː/，类似于 view) 是一套用于构建用户界面的渐进式框架。与其它大型框架不同的是，Vue 被设计为可以自底向上逐层应用。Vue 的核心库只关注视图层，不仅易于上手，还便于与第三方库或既有项目整合。另一方面，当与现代化的工具链以及各种支持类库结合使用时，Vue 也完全能够为复杂的单页应用提供驱动

## 2.2.入门程序

当从本地引入vue.min.js后，vue就会被注册成一个全局变量。使用`new Vue()`就可以创建一个Vue实例，而这个实例就是上节所说的VM即数据和视图的调度器。在视图使用“{{...}}”(差值表达式)就可以标记一个变量名，使用“el”属性告诉Vue操作哪片区域，使用“data”属性即表示数据：

```html
<div id="app">
  <p>{{msg}}</p>
</div>
<script src="../../lib/vue.js"></script>
<script>
	new Vue({
    el:"#app",
    data:{
      msg:"vue!hello world!"
    }
  });
</script>
```

# 3.vue默认指令

## 3.1.v-cloak

在使用`{{..}}`渲染数据时，会出现渲染变量闪烁现象，比如入门程序中页面出现时，会显示{{msg}}，过了几秒后，才会渲染数据，解决这一现象，可以使用`v-cloak`指令：

1. 在“el”关联的标签上添加“v-cloak”属性

2. 在样式表加上:[v-cloak]{display:none}，[v-cloak]意思是会选择带有此属性的标签，如：

```html
<style>
  /*防止渲染数据时出现闪烁现象*/
  [v-closk] {
    display: none;
  }
</style>
<div id="app" v-cloak>
  <p>{{msg}}</p>
</div>
```

## 3.2.v-text

`v-text`指令与差值表达式`{{}}`效果是一样的，都是用来标识变量名；但是与差值表达式有两个区别：

1. `v-text`不会有变量闪烁问题，而`{{}}`会有变量闪烁问题

2. `v-text`会覆盖原有标签的所有内容，而`{{}}`是属于嵌套内容，不会覆盖

```html
<div id="app">
  <h3>使用v-text属性渲染数据</h3>
  <h2 v-text="msg">内容会被覆盖</h2>
</div>
<script>
	new Vue({
    el:"#app",
    data:{
      msg:"v-text渲染数据, 会覆盖掉原先的数据"
    }
  });
</script>
```

## 3.3.v-html

差值表达式`{{}}`和`v-text`都会将数据当成字符串渲染，如果需要将数据解析成html格式，则要使用`v-html`指令，需要注意的是，`v-html`和`v-text`一样都会覆盖掉原有标签的所有内容：

```html
<div id="app">
  <h3>使用v-html解析html标签</h3>
  <div v-html="msg">内容也会被覆盖</div>
</div>
<script>
	new Vue({
    el:"#app",
    data:{
      msg:"<h1>新功能！</h1>" //会将这个值解析成HTML, 而不是字符串
    }
  });
</script>
```

## 3.4.v-bind

`v-bind`指定可以绑定标签的属性，将它的属性值标记成一个Vue的变量名。从而让Vue能将变量的值赋给属性的值。(`v-bind`的缩写是`:`，例如：title="msg4")下面例子可以将按钮的名称通过vue的变量名指定：

```html
<div id="app">
  <h3>使用v-bind绑定标签属性，标识标签属性的值为变量名</h3>
  <input type="button" value="v-bind" v-bind:title="msg"/>
</div>
<script>
	new Vue({
    el:"#app",
    data:{
      msg:"每天进步一点点"
    }
  });
</script>
```

`v-bind`不仅可以绑定变量名，还可以拼接字符串，只要满足JS表达式。Vue会将变量值拼接上字符串一起展示到页面：

```html
<div id="app">
  <!--属性值可以拼接字符串，只要满足JS表达式-->
  <input type="text" v-bind:value="msg4+' JS表达式'"/>
</div>
<script>
	new Vue({
    el:"#app",
    data:{
      msg:"每天进步一点点"
    }
  });
</script>
```

如果嫌弃`v-bind`太麻烦，直接使用`：`便可以表示：

```html
<input type="button" value="v-bind" :title="msg">
```

## 3.5.v-on

如果说`v-bind`是绑定标签的属性，那么`v-on`就是绑定事件，比如：`v-on:click`可以绑定元素的点击事件，这时候需要用Vue实例的另一个属性：methods。（`v-on`的缩写是@，比如：`@click='cli'`）

```html
<div id="app">
  <h3>5、使用v-on绑定点击事件</h3>
  <input type="button" value="点击" v-on:click="cli"/>
</div>
<script>
	new Vue({
    el:"#app",
    data:{
    },
    methods:{
      cli: function () {
        alert("努力+奋斗");
      }
    }
  });
</script>
```

## 3.6.v-model

`v-model`可以双向绑定数据，但仅限于表单元素。何为双向绑定？在Vue实例中修改会同时表现在页面，在页面中修改可以同时改变Vue实例中的数据。使用方式：（跟`v-html`、`v-text`一样，指定data域的变量名即可）

```html
<div id="app">
  <h3>使用v-model可以实现双向绑定数据。仅限于表单元素</h3>
  <input type="text" v-model="msg" style="width: 50%"/>
</div>
<script>
	new Vue({
    el:"#app",
    data:{
      msg:"表单元素与v-model结合可以实现数据的双向绑定"
    }
  });
</script>
```

**验证方式：**

使用`var vm = new Vue()`的方式创建Vue实例，会将创建完的对象赋值给vm变量，这个变量是属于window级（就是全局变量）。在console窗口使用`window.vm`就可以查看该实例；

- 改变输入框的值，查看vm中的数据变化；
- 改变vm实例的数据，查看输入框的变化。

## 3.7.v-for

`v-for`指令可以遍历元素，它可以遍历：普通数组、对象数组、对象、数字。`v-for`的通用格式为: `v-for:"(临时变量名,索引值) in [变量名]"`，当然在遍历对象时，格式为：` v-for:"(属性值,属性名,索引值) in [变量名]"`。索引值可要可不要，其它必须存在。**被v-for修饰的元素会循环自己**

```html
<div id="app">
  <h3>7、使用v-for可以循环迭代：普通数组、对象数组、对象、数字</h3>
  <!--item表示临时变量,index表示遍历是的索引-->
  普通数组:<span v-for="(item,index) in commonArr">{{index}}--{{ item }}、</span>
  对象数组:<span v-for="(i,idx) in objArr">{{idx}}--{id:{{i.id}},name:{{i.name}}}、</span>
  对象:<span v-for="(val,key,idx) in obj">{{idx}}--{{key}}:{{val}}、</span>
  数字:<span v-for="i in 10">{{i}}、< /span>
</div>
<script>
    new Vue({
      el:"#app",
      data:{
        commonArr: ['我', '忍', '才', '能', '成', '大', '事'],//普通数组
        objArr: [{id: 1, name: 'sym'}, {id: 2, name: 'sxq'}],//对象数组
        obj: {id: 3, name: 'sxq', age: 23}
      }
    });
</script>
```

## 3.8.v-if和v-show

`v-if`和`v-show`的效果一样（原理不一样），通过语义很明显可以知道是控制元素的变化（更详细地说是控制元素的出现和消失）。`v-if`的原理是通过元素的创建和销毁，`v-show`的原理是通过元素的display样式。`v-if`和`v-show`是支持动态绑定的，值改变，元素的变化也会跟着变。

```html
<div id="app">
  	<!--v-if通过元素的创建和销毁来实现对元素的控制-->
    <p v-if="flag">这个元素是被v-if控制的</p>
    <!--v-show是通过元素的display样式实现对元素的控制-->
    <p v-show="flag">这个元素是被v-show控制的</p>
</div>
<script>
	new Vue({
    el:"#app",
    data:{
      flag: true
    }
  });
</script>
```

特别的，`v-if`还可以搭配使用`v-else-if`和`v-else`

```html
<p v-if="flag=='1'">111111111</p>
<p v-else-if="flag=='2'">22222222</p>
<p v-else="flag=='3'">3333333</p>
```

## 3.9.小结

### 3.9.1.使用this访问vue的data域

在new Vue()实例中，如果想要获取data上的数据，或者是想调用methods中的方法，必须通过this.数据变量名或this.方法名来进行访问；这里的this就表示new出来的Vue实例。

```html
<script>
	new Vue({
    el:"#app",
    data:{
      msg: "猥琐发育"
    },
    methods:{
      test(){
        console.log(this.msg);
      }
    }
  });
</script>
```

### 3.9.2.v-on事件修饰符

①**事件修饰符简介**

- `.stop`：阻止冒泡事件

- `.prevent`：阻止默认行为

- `.capture`：使用捕获模式监听事件

- `.self`：仅在元素自身触发事件

- `.once`：事件只触发一次

②**事件修饰符使用方式**

`	@click.stop `或者 `v-on:click.stop`

③**补充内容**

​	JS的的冒泡过程和捕获过程，执行顺序是相反的，可以用下图表示：

​	![](./images/js冒泡过程和捕获过程.png)

- 冒泡过程：在一个对象上触发某类事件（比如单击onclick事件），如果此对象定义了此事件的处理程序，那么此事件就会调用这个处理程序，如果没有定义此事件处理程序或者事件返回true，那么这个事件会向这个对象的父级对象传播，从里到外，直至它被处理（父级对象所有同类事件都将被激活），或者它到达了对象层次的最顶层，即document对象（有些浏览器是window）。

- 捕获过程：事件将沿着DOM树向下传送，经过目标节点的每一个祖先节点，直至目标节点。例如，用户单击了一个超链接，则该单击事件将从document节点转送到html元素、body元素以及包含该链接的p元素。目标节点就是触发事件的DOM节点。

### 3.9.3.v-on事件名

v-on绑定事件的名称为原生JS事件名称去掉前面两个字符“on”。例如：

- JS原生点击事件onclick，给vue绑定为v-on:click或@click

- JS原生内容改变事件onchange，换成vue为v-on:change或@change

- JS原生获取焦点事件onfocus，换成vue为v-on:focus或@focus

...依次类推

### 3.9.4.v-on事件传参

在vue中需要为元素绑定事件，如点击事件，可以这样写：`v-on:click="cli"`，也可以这样写：`v-on:click="cli()"`加了小括号的好处是可以传参了。

### 3.9.5.vue键盘修饰符

键盘修饰符的用法跟事件修饰符类似，只不过事件修饰符可以满足所有的事件，而键盘修饰符只适用于keyup和keydown事件，一般用于文本框等表单元素。例如：当用户输入完信息后，单击回车键就可以执行登陆。实际上，在JS中，每个按键都有对应的键码，具体可见[https://www.jb51.net/article/33434.htm](https://www.jb51.net/article/33434.htm)，Vue就是通过这些键码得知用户按下了哪个键，但为了简单易记，都会使用别名，Vue内置别名有：

![](./images/vue内置键盘别名.png)

如果需要额外的按键修饰符，可以通过全局`config.keyCodes`对象定义，如：需要F2键：`Vue.config.keyCodes.F2 = 113`(113是F2的键码)

### 3.9.6.Vue设置css样式

可以使用v-bind指令绑定元素的class属性，然后通过：

1. 数组的方式：在[]中，加了''就表示样式的类名，不加则表示vue的变量名

   ```html
   <h1 :class="['fontColor','fontSize']">通过vue来设置绑定元素的class样式</h1>
   ```

2. 依然是数组的方式，但表达式变成了三元表达式：（注意flag是vue的变量）当变量flag为true使用类名为'fontStyle'的样式，否则不使用

   ```html
   <h1 :class="['fontColor','fontSize',flag?'fontStyle':'']">通过vue来设置绑定元素的class样式</h1>
   ```

3. 对象的方式，也就是直接绑定vue中的一个对象，如:class="obj",然后在vue中定义一个对象obj,对象保存“类名：布尔值”形式的数据

   ```html
   <h1 :class="obj">通过vue来设置绑定元素的class样式</h1>
   <script>
       var vm = new Vue({
           el: "#app",
           data: {
               flag: true,
               obj: {
                   fontSize: true,
                   fontColor: true,
                   fontStyle: true,
                   acttor: true
               }
           }
       });
   </script>
   ```

### 3.9.7.v-for迭代方法

v-for指令不仅可以迭代vue实例中的data域，还可以迭代vue实例中的methods域，只要方法有返回数据即可：

```html
<!-- v-for不仅可以迭代data里面的数据，也可以迭代methods里函数返回的数据
         下面的'search(content)'意思就是执行search方法将其返回的数据进行迭代，content的意思就是
         将data域的content传递给search()方法-->
<tr class="active" v-for="item in search(content)" :key="item.id">
  <td>{{item.id}}</td>
  <td>{{item.name}}</td>
  <td>{{item.time}}</td>
  <td>
    <button type="button" class="btn btn-primary" @click="del(item.id)">删除</button>
  </td>
</tr>
```

如果在v-for表达式中传递了参数，比如上图中的content。就会将vue实例data域中的content变量的值传给方法search()。请记住,v-for的字符串是一个表达式，可以解析的

### 3.9.8.v-for的key的注意事项

借用官方的介绍：[https://cn.vuejs.org/v2/guide/list.html#key](https://cn.vuejs.org/v2/guide/list.html#key)。当 Vue.js 用 v-for 正在更新已渲染过的元素列表时，它默认用“就地复用”策略。如果数据项的顺序被改变，Vue 将不会移动 DOM 元素来匹配数据项的顺序， 而是简单复用此处每个元素，并且确保它在特定索引下显示已被渲染过的每个元素。啥意思？举个例子：用v-for遍历数组时，如果加了复选框，如下所示：

![](./images/v-for注意事项-1.png)

0-表示索引，李世民-表示数组的内容，当勾选0-李世民后，如果动态改变了该数组。比如在数组开头添加了新对象：

![](./images/vue-for注意事项-2.png)

则选中的会是单雄飞，而不是李世民，因为vue只记住索引，不会记住具体的数据项，正规的原因呢就如上面所说的。为了给 Vue 一个提示，以便它能跟踪每个节点的身份，从而重用和重新排序现有元素，你需要为每项提供一个唯一 key 属性。理想的 key 值是每项都有的且唯一的 id。但它的工作方式类似于一个属性，所以你需要用 `v-bind `来绑定动态值：

```html
<p v-for="(item,idx) in list" :key="item.id">
  <input type="checkbox">
  {{idx}}---{{item.name}}
</p>
```

### 3.9.9.devtools调试vue

谷歌浏览器插件Vue.js devtools可以帮助调试vue的开发程序。翻墙搜索直接安装即可，安装完必须勾选下面的方框

![](./images/devtools.png)

直接打开vue应用程序，点击F12进入开发者模式，单击最后面的【Vue】选项就可以进入到插件，前提：项目不能使用vue.min.js压缩文件，需要使用开发文件:vue.js

![](./images/devtools调试vue.png)

# 4.vue的过滤器

## 4.1.全局过滤器

Vue允许自定义过滤器，可以用作一些常见的文本格式化，过滤器只能用在两个地方：①差值表达式、②v-bind指令。全局过滤器是所有vue实例共享。注意：全局过滤器需要写在new Vue()实例前面。

1. 定义过滤器

   `Vue.filter(<过滤器名>,function(){...});`第一个参数是过滤器名，第二个参数是函数，函数的第一个参数被绑定成差值表达式待格式化的变量，函数需要返回格式化后的字符串。

   ```html
   <script>
     Vue.filter('filter1', function (msg) {
       // return msg.replace('曾经','<至少>');->如果是这种写法，只会替换掉第一个而已
       
       // 如果使用正则表达式（/../）替换内容写到双斜线里面，跟参数g（表示全局）
       return msg.replace(/曾经/g, '<至少>');
     });
   </script>
   ```

2. 使用过滤器

   如果使用差值表达式：`{{ msg | filter }}`，使用管道符`|`隔开变量和过滤器名字；如果使用`v-bind`指令：`v-bind:value="{{ msg | filter }}"`，同样也是使用管道符`|`
   
   ```html
   <p>{{msg | filter1}}</p>
   <input type="text" :value="msg | filter1"></input>
   ```

### 4.1.1.最简过滤器

最简过滤器，只需要在function中传入待格式化的变量名即可，当然这是必须要传递的值

```javascript
Vue.filter('filter1', function (msg) {
  // return msg.replace('曾经','<至少>');->如果是这种写法，只会替换掉第一个而已
  // 如果使用正则表达式（/../）替换内容写到双斜线里面，跟参数g（表示全局）
  return msg.replace(/曾经/g, '<至少>');
});
```

### 4.1.2.带参过滤器

直接在过滤器加上变量值即可

```html
<p>{{msg | filter2('<以前>')}}</p>
<script>
  Vue.filter('filter2',function (msg,opt='66') {//当opt不传参时默认是66
    return msg.replace(/曾经/g,opt);
  });
</script>
```

### 4.1.3.连用过滤器

连用过滤器就是在过滤器后面再跟上““|”，在跟上过滤器名字即可，Vue会将执行完前一个过滤器得到的值用作后一个过滤器的参数，以此类推

```html
<p>{{msg | filter3 | filter4}}</p>
```

## 4.2.私有过滤器

全局过滤器是全部的Vue实例共享的，而私有过滤器仅仅是单个Vue实例独有，它是定义在Vue实例里面，使用filters属性来定义。在filters中，以`<filter-name>:function(arg){..}`定义私有过滤器。跟之前全局过滤器语法一样，可以带参！但处理函数的首个参数肯定是待处理的变量！

```javascript
new Vue({
  el:'#exe',
  data:{
    msg:new Date()
  },
  methods:{},
  filters:{
    /**
      * 私有过滤器定义在每个Vue实例内部，用属性filters表示,语法为：<过滤器名字>:<处理函数>
      * 如果有私有过滤器和全局过滤器重名，采用就近原则，优先使用私有的过滤器
      */
    dateFormate:function (msg) {
      var date = new Date(msg);
      var y = date.getFullYear();
      // padStart是es6新出的用于填充字符串的函数
      var m = (date.getMonth()+1).toString().padStart(2,"0");
      //参数1表示填充到的临界点，2表示填充到字符串大小为2即可；参数2表示填充的内容
      var d = date.getDate().toString().padStart(2,'0');
      return `${y}-${m}-${d}`;
    }
  }
});
```

# 5.vue自定义指令

之前学习的`v-bind`、`v-on`都是Vue内置的指令，Vue也支持自定义，同样自定义指令也分为全局指令和私有指令。这里还会涉及到一个“钩子函数”的概念，现在可以理解成：绑定事件的函数，类似于回调函数，但是钩子函数在捕获消息的第一时间就会执行，而回调函数在整个捕获过程结束时，最后一个被执行

## 5.1.全局自定义指令

全局过滤器使用`filter`关键字，全局指令使用`directive`关键字，语法为：`Vue.directive('',{})`，第一个参数是指令的名称，元素使用自定义指令需要加上`v-`前缀，而在这里定义的时候不需要前缀`v-`；第二个参数是一个对象，该对象包含了5个钩子函数，对应着从元素与指令绑定到元素与指令解绑的一系列过程

### 5.1.1.钩子函数

一个指令定义的对象，就是上面所说的第二个参数，存在5个钩子函数，每个钩子函数代表着绑定过程中的一个瞬间：

- `bind`：只调用一次，指令第一次绑定到元素时调用。在这里可以进行一次性的初始化设置；
- `inserted`：被绑定元素插入父节点时调用 (仅保证父节点存在，但不一定已被插入文档中)；
- `update`：所在组件的 VNode 更新时调用，**但是可能发生在其子 VNode 更新之前**。指令的值可能发生了改变，也可能没有。但是可以通过比较更新前后的值来忽略不必要的模板更新；

- `componentUpdated`：指令所在组件的 VNode **及其子 VNode** 全部更新后调用；
- `unbind`：只调用一次，指令与元素解绑时调用。

 这里会多出来一个vnode的概念？ Vue.js将Dom抽象成一个以javascript对象为节点的虚拟Dom树，以VNode节点模拟真实Dom，可以对这颗抽象树进行创建节点、删除节点以及修改节点等操作。每个钩子都存在4个参数，第一个参数是必须的，代表与指令绑定元素的DOM：

- `el`：指令所绑定的元素，可以用来直接操作 DOM；
- `binding`：一个对象，包含以下 property：
  - `name`：指令名，不包括 `v-` 前缀。
  - `value`：指令的绑定值，例如：`v-my-directive="1 + 1"` 中，绑定值为 `2`。
  - `oldValue`：指令绑定的前一个值，仅在 `update` 和 `componentUpdated` 钩子中可用。无论值是否改变都可用。
  - `expression`：字符串形式的指令表达式。例如 `v-my-directive="1 + 1"` 中，表达式为 `"1 + 1"`。
  - `arg`：传给指令的参数，可选。例如 `v-my-directive:foo` 中，参数为 `"foo"`。
  - `modifiers`：一个包含修饰符的对象。例如：`v-my-directive.foo.bar` 中，修饰符对象为 `{ foo: true, bar: true }`。
- `vnode`：Vue 编译生成的虚拟节点。移步 [VNode API](https://cn.vuejs.org/v2/api/#VNode-接口) 来了解更多详情。
- `oldVnode`：上一个虚拟节点，仅在 `update` 和 `componentUpdated` 钩子中可用。

除了 `el` 之外，其它参数都应该是只读的，切勿进行修改。如果需要在钩子之间共享数据，建议通过元素的 [`dataset`](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/dataset) 来进行。

### 5.1.2.使用方法

1. 先定义全局的指令，用`Vue.directive()`注册：inserted含义是将元素插入到父节点的时候执行，el参数是该元素的DOM对象，通过el.value将元素的值修改为666

   ```javascript
   Vue.directive('add', {
       /*以下的5个函数：bind、inserted、update、componentUpdated、unbind被称为钩子函数*/
       /*钩子函数有4个参数：el、binding、vnode、oldVnode*/
   
       bind : function (el,binding,vnode,oldVnode) {
         // 只调用一次，在指令第一次绑定到元素时调用
       },
       inserted:function (el,binding,vnode,oldVnode) {
         // 被绑定元素插入父节点时调用（仅保证父节点存在，而不一定被插入到文档中）
         el.value='666';
       },
       update : function (el,binding,vnode,oldVnode) {
         // 指令所在组件的 VNode 更新时调用
       },
       componentUpdated : function (el,binding,vnode,oldVnode) {
         // 指令所在组件的 VNode 及其子 VNode 全部更新后调用
       },
       unbind : function (el,binding,vnode,oldVnode) {
         // 只调用一次，指令与元素解绑时调用
       }
   });
   ```

2. 元素使用自定义指令，需要加上“v-”前缀：

   ```html
   <input type="text" class="form-control" v-add/>
   ```

## 5.2.私有自定义指令

私有指令的定义是在vue实例中完成，实例接收一个directives参数（与全局指令相比，少了一个s）每个自定义指令以——`'<指令名>':{<对象>}`定义。在对象里面，与上面说的5个钩子函数一样，钩子函数参数也一样

```javascript
new Vue({
    el:'#app',
    directives:{
      self:{
        inserted:function (el,binding) {
          el.focus();
          console.log(typeof binding.arg);
        }
      }
    }
});
```

# 6.Vue的生命周期

## 6.1.生命周期图示

Vue 实例从“创建→运行→销毁”这一整个过程就是Vue的生命周期。这一期间有相应的生命周期钩子函数执行，是Vue提供给用户添加自己代码的机会。官网对Vue生命周期的图示：[https://cn.vuejs.org/v2/guide/instance.html](https://cn.vuejs.org/v2/guide/instance.html)借助官网的图示，尝试来理解Vue生命周期间的事件（钩子函数）

![](./images/生命周期-1.png)

![](./images/生命周期-2.png)

![](./images/生命周期-3.png)

![](./images/生命周期-4.png)