## 实现Vue数据双向绑定的一些心得
### 创建项目
本文github地址：[https://github.com/csuZipple/Vue_study](https://github.com/csuZipple/Vue_study)
由于项目采用ES6语法进行开发，我们需要构建一个基于webpack的项目
```
    1. npm init -y
    2. npm install -D webpack webpack-command
```
<!-- more -->
然后创建[代码结构](https://github.com/csuZipple/Vue_study),添加[webpack.config.js](https://www.webpackjs.com/)
```javascript
module.exports = {
    entry: "./app.js",
    mode:"development",
    output: {
        filename: "./bundle.js",
    },
    watch: true
};
```

### 数据双向绑定原理
参考链接：[https://github.com/DMQ/mvvm](https://github.com/DMQ/mvvm)

`vue.js` 是采用数据劫持结合发布者-订阅者模式的方式，通过`Object.defineProperty()`来劫持各个属性的setter，getter，在数据变动时发布消息给订阅者，触发相应的监听回调。
总的来说，就以下几个步骤：
1. 实现一个数据监听器Observer，能够对数据对象的所有属性进行监听，如有变动可拿到最新值并通知订阅者 
2. 实现一个指令解析器Compile，对每个元素节点的指令进行扫描和解析，根据指令模板替换数据，以及绑定相应的更新函数 
3. 实现一个Watcher，作为连接Observer和Compile的桥梁，能够订阅并收到每个属性变动的通知，执行指令绑定的相应回调函数，从而更新视图 
4. mvvm入口函数，整合以上三者

### 模块介绍 

[参考链接](http://www.php.cn/js-tutorial-390322.html)
#### Observer

>遍历data中的所有属性，同时通过递归的方式遍历这些属性的子属性，通过Object.defineProperty()将这些属性都定义为访问器属性，访问器属性自带get和set方法，通过get和set方法来监听数据变化。

#### Dep

>我们需要消息订阅器来收集所有的订阅者，也就是在这里维护一个列表，当属性的get方法被触发时，需要判断是否要添加订阅者，如果需要就在列表中增加一个订阅者，当set方法被触发时就通知列表中所有的订阅者来做出响应。

#### Watcher

>因为我们是在get函数中判断是否要添加订阅者的，要想把一个订阅者添加到列表中我们就需要在初始化这个订阅者时触发get函数，我们可以在Dep.target上缓存下订阅者，添加成功后再将其去掉就可以了

#### Compile

>compile负责初始化时的编译解析，遍历每一个结点，看哪些结点需要订阅者，也负责后续为订阅者绑定更新函数。 


### 实现Observer
```javascript
class Observer {
    constructor(obj){
        this.data = obj; 
        if (!this.data || typeof this.data !== 'object') return;
        Object.keys(this.data).forEach(key=>{
            console.log(key);
            this._listen(key,this.data[key]);//使用箭头函数绑定this
        })
    }
     _listen(key, val){
        new Observer(val);
        Object.defineProperty(this.data,key,{
            enumerable:true,
            configurable:false,
            /*   set:function (newval) {
                   console.log("the key "+key+" set new value:",newval);
                   val = newval;
               },
               get:function () {
                   return val;
               }*///以上设置特权函数的方法也是可以的，下面采用es6的写法
            set(newval){
                val = newval;
            },
            get(){
                return val;
            }
        });
    }
}

export default Observer;
```
就上面的思路中提到的发布-订阅者模式而言，上面的代码还需要定义一个对象来收集所有的订阅者。我们在`_listen`方法第二行可以初始化一个对象` let dep = new Dep();`当接收到set请求的时候，通知dep所有的订阅者调用自己的update方法.
那么在什么时候为dep添加订阅者呢？我们给订阅者起了一个名字叫Watcher，而Watcher中有一个update方法会调用data的属性，也就是会触发Observer中的get方法。同时Observer在get方法中添加订阅者。从架构层次上来讲，添加订阅者应该是在编译节点的时候添加。
```javascript
//Observer get方法片段
  if (Dep.target) dep.listen(Dep.target);

//Watcher 代码片段
    constructor(node,name,vm){
        this.node = node;
        this.name = name;
        this.vm = vm;

        Dep.target = this;
        this.update();//触发observer，添加订阅
        Dep.target = null;
    }
    update(){
        console.log("watch name ",this.name);
        this.node.nodeValue = this.vm[this.name];//触发data中key的get方法
    }

//Compiler中的compile方法片段
 if(REG.test(node.nodeValue)){
                    let name = RegExp.$1;//返回第一个匹配的子串
                    new Watcher(node,name.trim(),this.vm);
                }
```
<font color="red">注意：上面的代码必须是Dep</font>

### 实现Dep
```

class Dep {
    constructor(){
        this.subs=[];//维护的订阅者列表
    }

    listen(sub){
        this.subs.push(sub);
        console.log("dep添加sub成功,当前维护的subs length="+this.subs.length);
    }

    notify(){
        console.log("检测到属性修改,model修改触发view修改");
        this.subs.forEach(function (sub, index, array) {
            console.log(sub);
            sub.update();
        })
    }
}

Dep.prototype.target=null;//表示当前对象是否已监听,原型链上的对象是不共享的

export default Dep;
```

### 实现Watcher

实际上Watcher就是一个订阅者的类，里面包含了node,name(属性),和vm(this，也就是mvvm对象的上下文)。通过Dep.target缓存当前Watcher的上下文，而update方法会触发Observer的get方法，在其中会首先判断Dep.target是否为null。如果不是null表示当前watcher还没有被Dep添加到sub列表中。


```javascript
import Dep from "./dep"
class Watcher {
    constructor(node,name,vm){
        this.node = node;
        this.name = name;
        this.vm = vm;

        Dep.target = this;
        this.update();//触发observer，添加订阅
        Dep.target = null;
    }
    update(){
        console.log("watch name ",this.name);
        this.node.nodeValue = this.vm[this.name];//触发data中key的get方法
    }
}
export default Watcher;
```

### 实现compiler

而在compiler中，由于需要操作dom节点。这里我们采用fragment来进行优化，提升性能。同时从绑定的根节点出发，判断节点是否需要添加订阅。
```javascript
import Watcher from "./watcher"

const REG = /\{\{(.*)\}\}/;
class Compiler {
    constructor(el,vm){
        this.el = document.querySelector(el);
        this.vm = vm;

        this.frag = this.createFragment();//为了提高dom操作的性能
        this.el.appendChild(this.frag);
    }

    createFragment() {
        let frag = document.createDocumentFragment();
        let child;
        while(child = this.el.firstChild){
            this.compile(child);
            frag.appendChild(child);
        }
        return frag;
    }

    compile(node){
        switch(node.nodeType){
            case 1://node
                let attr = node.attributes;
                let self = this;
                if (attr.hasOwnProperty("v-model")){
                    let name = attr["v-model"].nodeValue;
                    node.addEventListener("input",function (e) {
                        self.vm[name] = e.target.value;//触发set事件 调用update
                    });
                    node.value = this.vm[name];
                }
                break;
            case 3://element
                if(REG.test(node.nodeValue)){
                    let name = RegExp.$1;//返回第一个匹配的子串，也就是获取到文本标签{{message}}中的message
                    new Watcher(node,name.trim(),this.vm);
                }
                break;
            default:
                console.log("compile node error .default nodeType.")
        }
    }
}
export default Compiler;
```

### 实现index

index就是mvvm框架的入口文件了，在这里初始化参数，设置根节点，监听的数据对象，以及其他一些设置。像vue的参数是这样的，
```javascript
new Vue({
    el: '#app',
    data: {
        message: "hello myVue"
    }
});
```
我们模拟实现一下这样的功能。
```javascript
import Observer from './Observer'
import Compiler from './Compiler'

class Mvvm {
    constructor(options){
        this.$options = options;
        this.$el = options.el;
        this.data = options.data;

        Object.keys(this.data).forEach(key=>{
            this.listen(key);
        });
        //对data属性进行监听
        new Observer(this.data);
        new Compiler(this.$el,this);
    }

    listen(key) {
        //监听options的属性,方便通过options.name==>options.data.name
        let self = this;
        Object.defineProperty(self,key,{
            get(){
                return self.data[key];
            },
            set(newval){
                self.data[key] = newval;//此处触发data的set方法
            }
        })
    }
}

export default Mvvm;
```
在上面的代码中可以看到我们对传进来的options参数通过listen方法实现了代理，比如options.name==>options.data.name。所以在Watcher中可以直接通过this.vm\[name\]来调用options中data的值。

做到这里其实已经基本上完成了mvvm框架的数据双向绑定了，哦，对了，还有html结构如下：
```html
    <div id="app">
        <input type="text" v-model="message">
        {{ message }}
    </div>
```

代码参考链接：[https://blog.csdn.net/ns2250225/article/details/79534656](https://blog.csdn.net/ns2250225/article/details/79534656)