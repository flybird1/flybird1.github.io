---
title: js中令人匪夷所思的this
date: 2017-03-14 17:16:59
tags: javascript this
---

为何说是匪夷所思？因为我以前是做java的，this在java中含义是明确且具体的，即指向当前对象，在编译期绑定;而JavaScript 中this 在运行期进行绑定的，这是JavaScript中this关键字具备多重含义的本质原因。
JavaScript 中，普通的函数调用方式有三种：直接调用、方法调用和 new 调用。除此之外，还有一些特殊的调用方式，比如通过bind() 将函数绑定到对象之后再进行调用、通过 call()、apply() 进行调用等。而 es6 引入了箭头函数之后，箭头函数调用时，其 this 指向又有所不同。下面就来分析这些情况下的 this 指向。

##直接调用
直接调用，就是通过 函数名(...) 这种方式调用。这时候，函数内部的 this 指向全局对象，在浏览器中全局对象是 window，在 NodeJs 中全局对象是 global。

来看一个例子：

```javascript
// 简单兼容浏览器和 NodeJs 的全局对象
const _global = typeof window === "undefined" ? global : window;
 
function test() {
    console.log(this === _global);    // true
}
 
test();    // 直接调用
```
这里需要注意的一点是，直接调用并不是指在全局作用域下进行调用，在任何作用域下，直接通过 函数名(...) 来对函数进行调用的方式，都称为直接调用。比如下面这个例子也是直接调用

```javascript

function foo(){console.info(z)}
var z=10;
(function(a){
var z=20; a();
})(foo);//10
```

##bind() 对直接调用的影响

还有一点需要注意的是 bind() 的影响。Function.prototype.bind() 的作用是将当前函数与指定的对象绑定，并返回一个新函数，这个新函数无论以什么样的方式调用，其 this 始终指向绑定的对象。还是来看例子：

```javascript
const obj = {};
 
function test() {
    console.log(this === obj);
}
 
const testObj = test.bind(obj);
test();     // false
testObj();  // true
```

##call 和 apply 对 this 的影响
它们的第一个参数都是指定函数运行时其中的 this 指向。
不过使用 apply 和 call 的时候仍然需要注意，如果目录函数本身是一个绑定了 this 对象的函数，那 apply 和 call 不会像预期那样执行，比如

```javascript
const obj = {};
 
function test() {
    console.log(this === obj);
}
 
// 绑定到一个新对象，而不是 obj
const testObj = test.bind({});
test.apply(obj);    // true
 
// 期望 this 是 obj，即输出 true
// 但是因为 testObj 绑定了不是 obj 的对象，所以会输出 false
testObj.apply(obj); // false
```

由此可见，bind() 对函数的影响是深远的，慎用！
##new调用函数
this指向新创建的对象
##方法调用
指通过对象来调用方法函数，即xx.(...)。
例如，

```javascript
const obj = {
    // 第一种方式，定义对象的时候定义其方法
    test() {
        console.log(this === obj);
    }
};
 
// 第二种方式，对象定义好之后为其附加一个方法(函数表达式)
obj.test2 = function() {
    console.log(this === obj);
};
 
// 第三种方式和第二种方式原理相同
// 是对象定义好之后为其附加一个方法(函数定义)
function t() {
    console.log(this === obj);
}
obj.test3 = t;
 
// 这也是为对象附加一个方法函数
// 但是这个函数绑定了一个不是 obj 的其它对象
obj.test4 = (function() {
    console.log(this === obj);
}).bind({});
 
obj.test();     // true
obj.test2();    // true
obj.test3();    // true
 
// 受 bind() 影响，test4 中的 this 指向不是 obj
obj.test4();    // false
```

这里需要注意的是，后三种方式都是预定定义函数，再将其附加给 obj 对象作为其方法。再次强调，函数内部的 this 指向与定义无关，受调用方式的影响。

语法细微变化都会意外改变this，下面再看一个例子
var name = 'the window';
var obj = {
name:'my obj',
getName:function(){
	return this.name;
	}
};
obj.getName();//'my obj'
(obj.getName)();//'my obj'
当你需要把这个对象的方法赋值给另一个对象的时候，比如
var ss = {name:'ssName'};
ss.getName = obj.getName;
ss.getName();//'ssName'
换种语法执行：
(ss.getName =  obj.getName)();//'the window'

##箭头函数中的 this
箭头函数没有自己的 this 绑定，this指向的其实是直接包含它的那个函数或函数表达式中的 this。比如

```javascript
const obj = {
    test() {
        const arrow = () => {
            // 这里的 this 是 test() 中的 this，
            // 由 test() 的调用方式决定
            console.log(this === obj);
        };
        arrow();
    },
 
    getArrow() {
        return () => {
            // 这里的 this 是 getArrow() 中的 this，
            // 由 getArrow() 的调用方式决定
            console.log(this === obj);
        };
    }
};
 
obj.test();     // true
 
const arrow = obj.getArrow();
arrow();        // true
```

示例中的两个 this 都是由箭头函数的直接外层函数(方法)决定的，而方法函数中的 this 是由其调用方式决定的。上例的调用方式都是方法调用，所以 this 都指向方法调用的对象，即 obj。

箭头函数让大家在使用闭包的时候不需要太纠结 this，不需要通过像 _this 这样的局部变量来临时引用 this 给闭包函数使用。来看一段 Babel 对箭头函数的转译可能能加深理解：

```javascript

// ES6
const obj = {
    getArrow() {
        return () => {
            console.log(this === obj);
        };
    }
}

// ES5，由 Babel 转译
var obj = {
    getArrow: function getArrow() {
        var _this = this;
        return function () {
            console.log(_this === obj);
        };
    }
};
```

另外需要注意的是，箭头函数不能用 new 调用，不能 bind() 到某个对象(虽然 bind() 方法调用没问题，但是不会产生预期效果)。不管在什么情况下使用箭头函数，它本身是没有绑定 this 的，它用的是直接外层函数(即包含它的最近的一层函数或函数表达式)绑定的 this。

