#JavaScript 异步编程的4种方法

[原文链接](http://www.ruanyifeng.com/blog/2011/08/a_detailed_explanation_of_jquery_deferred_object.html)

你可能知道，Javascript语言的执行环境是"单线程"（single thread）。

所谓"单线程"，就是指一次只能完成一件任务。如果有多个任务，就必须排队，前面一个任务完成，再执行后面一个任务，以此类推。

这种模式的好处是实现起来比较简单，执行环境相对单纯。
**坏处** 是只要有一个任务耗时很长，后面的任务都必须排队等着，会拖延整个程序的执行，就是常说的 **js阻塞**。

为了解决这个问题，Javascript语言将任务的执行模式分成两种：同步（Synchronous）和异步（Asynchronous）。

### 一、回调函数
这是异步编程最基本的方法，也是我们熟悉使用的`ajax`里的`success`，`fail`等回调。

假设有两个函数`f1`和`f2`，后者`f2`等待`f1`的执行结果，代码如下：
```javascript
f1(f2);
f3();
```
采用这种方式，`f1`不会堵塞程序运行，所以`f3`的执行不会等待`f2`

回调函数的优点是**简单**、**容易理解**和**部署**，缺点是不利于代码的**阅读**和**维护**，各个部分之间**高度耦合**（Coupling），流程**混乱**，而且每个任务只能指定一个回调函数。

### 二、事件监听
另一种思路是采用事件驱动模式。任务的执行不取决于代码的顺序，而取决于某个事件是否发生。
还是以`f1`和`f2`为例。首先，为`f1`绑定一个事件
```javascript
f1.on('done', f2);
function f1(){
　　　　setTimeout(function () {
　　　　　　// f1的任务代码
　　　　　　f1.trigger('done');
　　　　}, 1000);
　　}
```

触发```done```事件，从而开始执行```f2```

>这种方法的 **优点** 是比较容易理解，可以绑定多个事件，每个事件可以指定多个回调函数，而且可以**"解耦合"**（`Decoupling`），有利于实现模块化。
**缺点** 是整个程序都要变成事件驱动型，运行流程会变得很不清晰。


### 三、发布/订阅
我们假定，存在一个 **"信号中心"** ，某个任务执行完成，就向信号中心 **"发布"** （`publish`）一个信号，其他任务可以向信号中心 **"订阅"** （`subscribe`）这个信号，从而知道什么时候自己可以开始执行。这就叫做**"发布/订阅模式"**（`publish-subscribe pattern`），又称 **"观察者模式"** （`observer pattern`）。

这个模式有多种实现，下面采用的是`Ben Alman`的`Tiny Pub/Sub`，这是`jQuery`的一个插件

首先，`f2`向**"信号中心"**`jQuery`订阅"`done`"信号
```javascript
jQuery.subscribe("done", f2);
function f1(){
　　　　setTimeout(function () {
　　　　　　// f1的任务代码
　　　　　　jQuery.publish("done");
　　　　}, 1000);
　　}
```
`f1`执行完成后，向**"信号中心"**`jQuery`发布"`done`"信号，从而引发`f2`的执行。
此外，`f2`完成执行后，也可以取消订阅（`unsubscribe`）。
```javascript
jQuery.unsubscribe("done", f2);
```

>这种方法的性质与**"事件监听"**类似，但是明显优于后者。因为我们可以通过查看**"消息中心"**，了解存在多少信号、每个信号有多少订阅者，从而监控程序的运行。

### 四、Promise对象
`Promises`对象是`CommonJS`工作组提出的一种规范，目的是为异步编程提供统一接口。

简单说，它的思想是，每一个异步任务返回一个`Promise`对象，该对象有一个`then`方法，允许指定回调函数。比如，`f1`的回调函数`f2`,可以写成：
```javascript
f1().then(f2);
```

`f1`要进行如下改写：
```javascript
function f1(){
　　var dfd = $.Deferred();
　　setTimeout(function () {
　　　　// f1的任务代码
　　　　dfd.resolve();
　　}, 500);
　　return dfd.promise;
}
```

> 这样写的**优点**在于，回调函数变成了链式写法，程序的流程可以看得很清楚，而且有一整套的配套方法，可以实现许多强大的功能。

比如，指定多个回调函数：
```javascript
f1().then(f2).then(f3);
```
再比如，指定发生错误时的回调函数：
```javascript
f1().then(f2).fail(f3);
```
> 这种方法的**缺点**就是编写和理解，都相对比较难。

### 五、jQuery的deferred对象详解
从`jQuery 1.5.0`版本开始引入的一个新功能 —— [deferred对象](http://api.jquery.com/category/deferred-object/)。
它彻底改变了如何在`jQuery`中使用`ajax`，`jQuery`的全部`ajax`代码都被重写了。

#### 1、什么是deferred对象？
在回调函数方面，jQuery的功能非常弱。为了改变这一点，jQuery开发团队就设计了deferred对象。**简单说，deferred对象就是jQuery的回调函数解决方案。**

它解决了如何处理耗时操作的问题，对那些操作提供了更好的控制，以及统一的编程接口。

#### 2、ajax操作的链式写法
首先，看一下`jQuery`的`ajax`操作的传统写法：
```javascript
    $.ajax({
　　　　url: "test.html",
　　　　success: function(){
　　　　　　alert("哈哈，成功了！");
　　　　},
　　　　error:function(){
　　　　　　alert("出错啦！");
　　　　}
　　});
```
> 在上面的代码中，`$.ajax()`接受一个对象参数，这个对象包含两个方法：`success`方法指定操作成功后的回调函数，`error`方法指定操作失败后的回调函数。

> `$.ajax()`操作完成后，如果使用的是低于`1.5.0`版本的`jQuery`，返回的是`XHR`对象，你没法进行链式操作；如果高于`1.5.0`版本，返回的是`deferred`对象，可以进行链式操作。

新的写法是这样的:
```javascript
$.ajax("test.html")
　　.done(function(){ alert("哈哈，成功了！"); })
　　.fail(function(){ alert("出错啦！"); });
```
> 可以看到，`done()`相当于`success`方法，`fail()`相当于`error`方法。采用链式写法以后，代码的可读性大大提高。

#### 3、指定同一操作的多个回调函数
`deferred`对象的一大好处，就是它允许你自由添加多个回调函数。

```javascript
$.ajax("test.html")
　　.done(function(){ alert("哈哈，成功了！");} )
　　.fail(function(){ alert("出错啦！"); } )
　　.done(function(){ alert("第二个回调函数！");} );
```
> 回调函数可以添加任意多个，它们按照添加顺序执行。

#### 4、为多个操作指定回调函数
`deferred`对象的另一大好处，就是它允许你为多个事件指定一个回调函数，这是传统写法做不到的。
请看代码，用到了一个新的方法`$.when()`
```javascript
$.when($.ajax("test1.html"), $.ajax("test2.html"))
　　.done(function(){ alert("哈哈，成功了！"); })
　　.fail(function(){ alert("出错啦！"); });
```
> 这段代码的意思是，先执行两个操作`$.ajax("test1.html")`和`$.ajax("test2.html")`，如果都成功了，就运行`done()`指定的回调函数；如果有一个失败或都失败了，就执行`fail()`指定的回调函数。

#### 5、普通操作的回调函数接口（上）
`deferred`对象的最大优点，就是它把这一套回调函数接口，从`ajax`操作扩展到了所有操作。也就是说，任何一个操作（不管是`ajax`操作还是本地操作，也不管是 **异步操作** 还是 **同步操作** ）都可以使用`deferred`对象的各种方法，指定回调函数。

我们来看一个具体的例子。假定有一个很耗时的操作`wait`：
```javascript
var wait = function(){
　　var tasks = function(){
　　　　alert("执行完毕！");
　　};
　　setTimeout(tasks,5000);
};
```
我们为它指定回调函数，很自然会想到，可以使用`$.when()`:
```javascript
$.when(wait())
　　.done(function(){ alert("哈哈，成功了！"); })
　　.fail(function(){ alert("出错啦！"); });
```
> 但是，这样写的话，`done()`方法会立即执行，起不到回调函数的作用。

原因在于`$.when()`的参数只能是`deferred`对象，所以必须对`wait()`进行改写：
```javascript
var dtd = $.Defferred();
function wait(dtd){

    var task = function(){
        alert("执行完毕！");
        dtd.resolve();    
    };
    
    setTimeout(tasks,5000);
    return dtd;
    
}
```
现在，`wait()`函数返回的是`deferred`对象，这就可以加上链式操作了。
```javascript
$.when(wait(dtd))
　　.done(function(){ alert("哈哈，成功了！"); })
　　.fail(function(){ alert("出错啦！"); });
```
`wait()`函数运行完，就会自动运行`done()`方法指定的回调函数。

#### 6、deferred.resolve()方法和deferred.reject()方法
我们上面的`wait()`函数中，用到了`dtd.resolve()`方法，要说这个问题，我们要先引入一个`执行状态`的概念。

 `jQuery`规定，`deferred`对象有三种执行状态——`未完成`，`已完成`和`已失败`。
- 如果执行状态是`"已完成"（resolved）`,调用`done()`方法指定的回调函数；
- 如果执行状态是`"已失败"（rejected）`，调用`fail()`方法指定的回调函数；
- 如果执行状态是`"未完成"`，则继续等待，或者调用`progress()`方法指定的回调函数（jQuery1.7版本添加）。

> 这两种方法可以用于程序员对`deferred对象`进行手动指定`执行状态`，从而触发`done()`,`fail()`方法。

#### 7、deferred.promise()方法
上面这种写法，还是有问题。那就是`dtd`是一个全局对象，所以它的执行状态可以从外部改变。
请看代码：
```javascript
var dtd = $.Deferred(); // 新建一个Deferred对象
var wait = function(dtd){
　　var tasks = function(){
　　　　alert("执行完毕！");
　　　　dtd.resolve(); // 改变Deferred对象的执行状态
　　};
　　setTimeout(tasks,5000);
　　return dtd;
};

$.when(wait(dtd))
    .done(function(){ alert("哈哈，成功了！"); })
    .fail(function(){ alert("出错啦！"); });
    
dtd.resolve();
```
以上代码在尾部加了一行`dtd.resolve();`，直接改变了`dtd`对象的执行状态，导致`done()`方法立刻执行，跳出**“哈哈，成功了！”**的提示框，等5秒之后再跳出**“执行完毕！”**的提示框。
> 为了避免这种情况，`jQuery`提供了`deferred.promise()`方法。

> 它的作用是:
- 在原来的`deferred对象`上返回另一个`deferred对象`。
- 只开放与改变执行状态无关的方法（比如`done()`方法和`fail()`方法）
- 屏蔽与改变执行状态有关的方法（比如`resolve()`方法和`reject()`方法）

> **从而使得执行状态不能被改变。**

请看代码：
```javascript
var dtd = $.Deferred(); // 新建一个Deferred对象
　　var wait = function(dtd){
　　　　var tasks = function(){
　　　　　　alert("执行完毕！");
　　　　　　dtd.resolve(); // 改变Deferred对象的执行状态
　　　　};

　　　　setTimeout(tasks,5000);
　　　　return dtd.promise(); // 返回promise对象
　　};
　　var d = wait(dtd); // 新建一个d对象，改为对这个对象进行操作
　　$.when(d)
　　.done(function(){ alert("哈哈，成功了！"); })
　　.fail(function(){ alert("出错啦！"); });
　　d.resolve(); // 此时，这个语句是无效的
```
> 在上面的这段代码中，`wait()`函数返回的是`promise对象`。然后，我们把回调函数绑定在这个对象上面，而不是原来的`deferred对象`上面。

> 这样的好处是，无法改变这个对象的执行状态，要想改变执行状态，只能操作原来的deferred对象。

但是，更好的写法应该是，将`dtd对象`变成`wait()函数`的内部对象：
```javascript
var wait = function(dtd){
var dtd = $.Deferred(); //在函数内部，新建一个Deferred对象
　　　　var tasks = function(){
　　　　　　alert("执行完毕！");
　　　　　　dtd.resolve(); // 改变Deferred对象的执行状态
　　　　};

　　　　setTimeout(tasks,5000);
　　　　return dtd.promise(); // 返回promise对象
　　};
　　$.when(wait())
　　.done(function(){ alert("哈哈，成功了！"); })
　　.fail(function(){ alert("出错啦！"); });
```

#### 8、普通操作的回调函数接口（中）
另一种防止执行状态被外部改变的方法，是使用`deferred对象`的建构函数`$.Deferred()`，直接把`wait()`传入：
```javascript
$.Deferred(wait)
　　.done(function(){ alert("哈哈，成功了！"); })
　　.fail(function(){ alert("出错啦！"); });
```
> `jQuery`规定，`$.Deferred()`可以接受一个函数名（注意，是函数名）作为参数，`$.Deferred()`所生成的`deferred对象`将作为这个函数的默认参数。

#### 9、普通操作的回调函数接口（下）
除了上面两种方法以外，我们还可以直接在`wait对象`上部署`deferred`接口：
```javascript
var dtd = $.Deferred(); // 生成Deferred对象
var wait = function(dtd){
　　var tasks = function(){
　　　　alert("执行完毕！");
　　　　dtd.resolve(); // 改变Deferred对象的执行状态
　　};
　　setTimeout(tasks,5000);
};
dtd.promise(wait);
wait.done(function(){ alert("哈哈，成功了！"); })
    .fail(function(){ alert("出错啦！"); });
wait(dtd);
```
> 这里的关键是`dtd.promise(wait)`这一行.

> 它的作用就是在`wait对象`上部署`Deferred接口`。所以才能直接在`wait`上面调用`done()`和`fail()`。

#### 10、小结：deferred对象的方法
- 生成一个`deferred`对象
- `deferred.done()` 指定操作`成功`时的回调函数
- `deferred.fail()` 指定操作`失败`时的回调函数
- `deferred.promise()` 
> - 没有参数时，返回一个新的`deferred对象`，该对象的运行状态无法被改变
> - 接受参数时，作用为在参数对象上部署`deferred接口`
- `deferred.resolve()` 手动改变`deferred对象`的运行状态为`"已完成"`，从而立即触发`done()`方法。
- `deferred.reject()` 手动改变`deferred对象`的运行状态为`"已失败"`，从而立即触发`fail()`方法。
-  `$.when()` 为多个操作指定回调函数。

    **除了这些方法以外，`deferred对象`还有二个重要方法：**

-  `deferred.then()`,可以把`done()`和`fail()`一起写：

```javascript
$.when($.ajax( "/main.php" ))
　　.then(successFunc, failureFunc );
```
> 如果`then()`有两个参数，那么第一个参数是`done()`方法的回调函数，第二个参数是`fail()`方法的回调方法。

> 如果`then()`只有一个参数，那么等同于`done()`。

- `deferred.always()`方法也是用来指定回调函数的，它的作用是，不管调用的是`deferred.resolve()`还是`deferred.reject()`，最后总是执行。

```javascript
$.ajax( "test.html" )
　　.always( function() { alert("已执行！");} );
```


