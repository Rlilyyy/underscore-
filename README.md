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
