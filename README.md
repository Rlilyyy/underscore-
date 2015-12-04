# 浅析underscore
## 前端小菜鸟阅读underscore的小小心得，可能理解有错，多多包涵指正~
### 1.baseCreate(prototype)
### 作用:返回一个拥有prototype所有属性的对象
``` js
var baseCreate = function(prototype) {
  	// 如果prototype不是Object，返回一个空对象
    if (!_.isObject(prototype)) return {};
    // 如果ES5的Object.create()可以使用，那么使用该方法
    if (nativeCreate) return nativeCreate(prototype);
    // 否则使用以下方法
    // 而以下的这种方法，只继承了原型prototype的所有属性
    // 假设你传入的prototype的constructor拥有若干this.属性
    // 那么这些属性将不会被继承
    // 因为实际上new Ctor()的时候执行的构造函数是Ctor()
    // 并且复制了一份prototype给予result
    // 即只继承原型的属性，构造函数的属性将不会被继承
    
    // 可以想想继承的写法
    // 例如SubClass.prototype = new SuperClass()
    // 这是标准的原型链继承方式
    // 这里SubClass的实例为什么可以读取SuperClass的构造函数的属性
    // 原因就是这里的子类的prototype实际上是SuperClass的实例
    // new SuperClass()的时候，调用了SuperClass的构造函数
    // 相当于复制了一份构造函数的属性放到了子类的prototype里
    // 同时子类的prototype里会有一个__proto__属性指向父类的prototype
    Ctor.prototype = prototype;
    var result = new Ctor;
    // result的constructor将指向prototype.constructor
    // 因为Ctor的prototype是传入的prototype，被覆盖了
    // 这里将Ctor的prototype指向null,是为了释放内存
    Ctor.prototype = null;
    return result;
};
```

### 2.optimizeCb(func, context, argCount)
### 作用:根据argCount判断要返回何种迭代函数
``` js
var optimizeCb = function(func, context, argCount) {
	// func->需要执行的函数
	// context->执行函数的上下文
	// argCount->参数个数
	// 如果没有传入上下文context，则直接返回需要执行的func
  if (context === void 0) return func;
  // 
  switch (argCount == null ? 3 : argCount) {
  	// 参数1个，返回context.func(value)
    case 1: return function(value) {
      return func.call(context, value);
    };
    // The 2-parameter case has been omitted only because no current consumers
    // made use of it.
    // 参数2个的情况并没被使用到，可能以后会有

    // 参数3个，返回context.func(value, index, collection)
    // value->当前遍历的集合内容
    // index->当前索引
    // collection->当前的集合
    case 3: return function(value, index, collection) {
      return func.call(context, value, index, collection);
    };

    // 参数4个，返回context.func(accumulator, value, index, collection)
    case 4: return function(accumulator, value, index, collection) {
      return func.call(context, accumulator, value, index, collection);
    };
  }

  // 其实switch可以不要，因为下面的apply是可以完成上面的工作的，其实就是为了尽量不适用arguments，在V8引擎下无法优化
  return function() {
    return func.apply(context, arguments);
  };
};
```
### 3.cb(value, context, argCount)
### 作用:根据value的类型，对value做相对应的处理
``` js
// A mostly-internal function to generate callbacks that can be applied
// to each element in a collection, returning the desired result — either
// `identity`, an arbitrary callback, a property matcher, or a property accessor.
// 这里判断value的类型，进行返回
var cb = function(value, context, argCount) {
	if (value == null) return _.identity;
	if (_.isFunction(value)) return optimizeCb(value, context, argCount);
	if (_.isObject(value)) return _.matcher(value);
	return _.property(value);
};
```
### 4.restArgs(func, startIndex) 
### 作用:处理func，使func从startIndex开始后的所有参数都被视为剩余参数，然后包装成一个数组
``` js
var restArgs = function(func, startIndex) {
  	// 用.invoke的例子解释
  	// 首先会判断invoke需要restArgs的函数的参数个数
  	// 很明显，是3个，那么startIndex就是2
  	// 然后返回一个函数给invoke
  	// 此时invoke是没有明确声明形参的
  	// 那么根据你传入的参数个数，之前已经确定了有三个参数，那么第三个默认为剩余参数的集合
  	// arguments.length获取的是invoke()里的传入的参数的个数，这样判断需要获取多少个剩余参数加入集合
  	// 根据startIndex可以开始call
  	// 总而言之，这个函数的作用就是你传入一个函数，他帮你处理成一个默认你传入的函数里的最后一个形参为收集剩余参数的集合
  	// 并返回这个处理好的函数
  	// 其实返回的函数是复用的，就是将重复做的工作处理成模板
    startIndex = startIndex == null ? func.length - 1 : +startIndex;
    return function() {
    	// 返回的函数里有衡量好的startIndex，根据这个可以处理剩余参数
      var length = Math.max(arguments.length - startIndex, 0);
      var rest = Array(length);
      for (var index = 0; index < length; index++) {
        rest[index] = arguments[index + startIndex];
      }
      switch (startIndex) {
      	// 根据开始的索引，判断传入的数组的位置
        case 0: return func.call(this, rest);
        case 1: return func.call(this, arguments[0], rest);
        case 2: return func.call(this, arguments[0], arguments[1], rest);
      }
      // 若非剩余参数数量过多，则将直接使用apply接收包装好的非剩余参数和剩余参数数组
      var args = Array(startIndex + 1);
      for (index = 0; index < startIndex; index++) {
        args[index] = arguments[index];
      }
      args[startIndex] = rest;
      return func.apply(this, args);
    };
  };
```
### 5.property(key)
### 作用:返回能够获取obj里key属性的函数（eg：getLength(obj)）
``` js
// 返回能够获取obj的key属性的函数
  var property = function(key) {
    return function(obj) {
      return obj == null ? void 0 : obj[key];
    };
  };
```
### 6.isArrayLike(collention)
### 作用:判断collection是否类数组类型
``` js
var isArrayLike = function(collection) {
    // 获取collection的length属性
    var length = getLength(collection);
    // 如果length是一个数值并且大于0小于最大值，那么返回true
    return typeof length == 'number' && length >= 0 && length <= MAX_ARRAY_INDEX;
  };
```
### 6.each(obj, iteratee, context)/forEach(obj, iteratee, context)
### 作用:根据iteratee迭代obj的每个元素
``` js
_.each = _.forEach = function(obj, iteratee, context) {
    iteratee = optimizeCb(iteratee, context);
    var i, length;
    if (isArrayLike(obj)) {
      // 如果传入的obj类似数组
      for (i = 0, length = obj.length; i < length; i++) {
        iteratee(obj[i], i, obj);
      }
    } else {
      // 如果obj是对象
      // 那么现将obj转换成可迭代的数组
      var keys = _.keys(obj);
      for (i = 0, length = keys.length; i < length; i++) {
        iteratee(obj[keys[i]], keys[i], obj);
      }
    }
    // 最后返回原对象，方便链式调用
    return obj;
  };
```
