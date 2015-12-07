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
### 7.each(obj, iteratee, context)/forEach(obj, iteratee, context)
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
8.所有源代码+理解
``` js
//     Underscore.js 1.8.3
//     http://underscorejs.org
//     (c) 2009-2015 Jeremy Ashkenas, DocumentCloud and Investigative Reporters & Editors
//     Underscore may be freely distributed under the MIT license.

(function() {

  // Baseline setup
  // --------------

  // Establish the root object, `window` (`self`) in the browser, `global`
  // on the server, or `this` in some virtual machines. We use `self`
  // instead of `window` for `WebWorker` support.
  // 保存一个全局对象，命名为root
  var root = typeof self == 'object' && self.self === self && self ||
            typeof global == 'object' && global.global === global && global ||
            this;

  // Save the previous value of the `_` variable.
  // 保存全局变量root._，防止被覆盖（即命名冲突后，仍可以通过某些方法恢复）
  var previousUnderscore = root._;

  // Save bytes in the minified (but not gzipped) version:
  var ArrayProto = Array.prototype, ObjProto = Object.prototype;

  // Create quick reference variables for speed access to core prototypes.
  // 将一些原型的方法存储在闭包变量中
  var
    push = ArrayProto.push,
    slice = ArrayProto.slice,
    toString = ObjProto.toString,
    hasOwnProperty = ObjProto.hasOwnProperty;

  // All **ECMAScript 5** native function implementations that we hope to use
  // are declared here.
  // ES5方法，如果有则可以使用
  var
    nativeIsArray = Array.isArray,
    nativeKeys = Object.keys,
    nativeCreate = Object.create;

  // Naked function reference for surrogate-prototype-swapping.
  // 意义不明
  var Ctor = function(){};

  // Create a safe reference to the Underscore object for use below.
  // 创建_变量
  // 这里初始化一个_函数作为全局变量，也是underscore
  // 下面的所有函数都是_的构造函数的一部分
  // new _(obj)，则新建的_实例的原型就是使用mixin构造的，并不是下面使用_.methodName
  // underscore实例调用的方法实际都来自mixin创建的原型方法
  // 而最原始的_的方法是来自对象的方法,新建实例的时候这些方法不会被赋予新实例
  var _ = function(obj) {
    if (obj instanceof _) return obj;
    if (!(this instanceof _)) return new _(obj);
    this._wrapped = obj;
  };

  // Export the Underscore object for **Node.js**, with
  // backwards-compatibility for their old module API. If we're in
  // the browser, add `_` as a global object.
  // (`nodeType` is checked to ensure that `module`
  // and `exports` are not HTML elements.)
  // 根据宿主环境将_变量保存在不同的全局变量中
  if (typeof exports != 'undefined' && !exports.nodeType) {// 这里判断是否在Node.js的环境下
    if (typeof module != 'undefined' && !module.nodeType && module.exports) {
      exports = module.exports = _;
    }
    exports._ = _;
  } else {// 这里是普通JS环境，root之前已经保存了全局变量
    root._ = _;
  }

  // Current version.
  // 版本号保存
  _.VERSION = '1.8.3';

  // Internal function that returns an efficient (for current engines) version
  // of the passed-in callback, to be repeatedly applied in other Underscore
  // functions.
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

  _.iteratee = function(value, context) {
  	// 这里设置参数个数为无限个
    return cb(value, context, Infinity);
  };

  // Similar to ES6's rest param (http://ariya.ofilabs.com/2013/03/es6-and-rest-parameter.html)
  // This accumulates the arguments passed into an array, after a given index.
  //  _.invoke = restArgs(function(obj, method, args) {
  //   var isFunc = _.isFunction(method);
  //   return _.map(obj, function(value) {
  //     var func = isFunc ? method : value[method];
  //     return func == null ? func : func.apply(value, args);
  //   });
  // });
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
      var args = Array(startIndex + 1);
      for (index = 0; index < startIndex; index++) {
        args[index] = arguments[index];
      }
      args[startIndex] = rest;
      return func.apply(this, args);
    };
  };

  // An internal function for creating a new object that inherits from another.
  var baseCreate = function(prototype) {
  	// 如果原型不是Object，返回一个空对象
    if (!_.isObject(prototype)) return {};
    // 如果ES5的Object.create()可以使用，那么使用该方法
    if (nativeCreate) return nativeCreate(prototype);
    // 否则使用以下方法
    // 这里不要跟继承混淆
    // 原型prototype里包括constructor和原型的属性
    // 实际上跟result = new prototype.constructor差不多，只不过一个是Ctor的实例，一个是prototype的构造函数的实例
    // 使用此种方法实际上不会运行prototype的constructor的内容
    // 这里只会继承prototype的属性，构造函数的属性将无法寻找到
    // j继承方法中例如SubClass.prototype = new SuperClass();
    // 实际上，new SuperClass()执行了SuperClass的构造函数和prototype，那么其中的构造函数的属性相当于复制了一份给SubClass.prototype
    // 而以下的这种方法，只继承了原型prototype
    // 但是实际上执行的构造函数是Ctor(),而不是prototype.constructor
    // 即只继承原型的属性，构造函数的属性将不会被继承
    Ctor.prototype = prototype;
    var result = new Ctor;
    Ctor.prototype = null;
    return result;
  };

  // 返回一个传入obj获取之前预定好的key的值的函数
  var property = function(key) {
    return function(obj) {
      return obj == null ? void 0 : obj[key];
    };
  };

  // Helper for collection methods to determine whether a collection
  // should be iterated as an array or as an object.
  // Related: http://people.mozilla.org/~jorendorff/es6-draft.html#sec-tolength
  // Avoids a very nasty iOS 8 JIT bug on ARM-64. #2094
  var MAX_ARRAY_INDEX = Math.pow(2, 53) - 1;
  var getLength = property('length');
  var isArrayLike = function(collection) {
    // 获取collection的length属性
    var length = getLength(collection);
    // 如果length是一个数值并且大于0小于最大值，那么返回true
    return typeof length == 'number' && length >= 0 && length <= MAX_ARRAY_INDEX;
  };

  // Collection Functions
  // --------------------

  // The cornerstone, an `each` implementation, aka `forEach`.
  // Handles raw objects in addition to array-likes. Treats all
  // sparse array-likes as if they were dense.
  // 根据iteratee迭代obj的每个元素
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

  // Return the results of applying the iteratee to each element.
  // map与each的区别在于，map会保留obj执行iteratee后的结果并返回
  // each只是单纯地执行iteratee
  // 如果iteratee是对象，那么可以判断obj里的每个元素是否都是iteratee类型
  _.map = _.collect = function(obj, iteratee, context) {
    iteratee = cb(iteratee, context);
    var keys = !isArrayLike(obj) && _.keys(obj),
        length = (keys || obj).length,
        results = Array(length);
    for (var index = 0; index < length; index++) {
      var currentKey = keys ? keys[index] : index;
      results[index] = iteratee(obj[currentKey], currentKey, obj);
    }
    return results;
  };

  // Create a reducing function iterating left or right.
  var createReduce = function(dir) {
    // Optimized iterator function as using arguments.length
    // in the main function will deoptimize the, see #1991.
    var reducer = function(obj, iteratee, memo, initial) {
      var keys = !isArrayLike(obj) && _.keys(obj),
          length = (keys || obj).length,
          index = dir > 0 ? 0 : length - 1;
      if (!initial) {
        memo = obj[keys ? keys[index] : index];
        index += dir;
      }
      for (; index >= 0 && index < length; index += dir) {
        var currentKey = keys ? keys[index] : index;
        memo = iteratee(memo, obj[currentKey], currentKey, obj);
      }
      return memo;
    };

    return function(obj, iteratee, memo, context) {
      // 调用reduce的时候，iteratee一共可以接受4个参数，分别是当前归并值的结果、准备与第一个值合并的value、当前的index、合并的数组或对象
      // 如果没有传入memo，那么initial为false，那么reducer将会默认obj的第一个（或最后一个）数值作为初始memo
      var initial = arguments.length >= 3;
      return reducer(obj, optimizeCb(iteratee, context, 4), memo, initial);
    };
  };

  // **Reduce** builds up a single result from a list of values, aka `inject`,
  // or `foldl`.
  // 传入1表示从左往右开始合并
  _.reduce = _.foldl = _.inject = createReduce(1);

  // The right-associative version of reduce, also known as `foldr`.
  // 传入-1表示从右往左开始合并
  _.reduceRight = _.foldr = createReduce(-1);

  // Return the first value which passes a truth test. Aliased as `detect`.
  // 得到obj里第一个与predicate函数（或对象）匹配的值
  _.find = _.detect = function(obj, predicate, context) {
    var key;
    if (isArrayLike(obj)) {
      key = _.findIndex(obj, predicate, context);
    } else {
      key = _.findKey(obj, predicate, context);
    }
    if (key !== void 0 && key !== -1) return obj[key];
  };

  // Return all the elements that pass a truth test.
  // Aliased as `select`.
  // 这里与find不同的区别是，这里返回是一个匹配的结果集，并不是第一个匹配值
  // 如果predicate是对象，那么意思就是找出obj对象里跟predicate一样的对象
  // 如果predicate是函数，那么根据函数返回true的条件去得到结果集
  _.filter = _.select = function(obj, predicate, context) {
    var results = [];
    predicate = cb(predicate, context);
    _.each(obj, function(value, index, list) {
      if (predicate(value, index, list)) results.push(value);
    });
    return results;
  };

  // Return all the elements for which a truth test fails.
  // 与filter相反，这里是返回不符合predicate的结果集
  _.reject = function(obj, predicate, context) {
    return _.filter(obj, _.negate(cb(predicate)), context);
  };

  // Determine whether all of the elements match a truth test.
  // Aliased as `all`.
  // 判断obj里的所有值是否都符合predicate函数（或对象）
  _.every = _.all = function(obj, predicate, context) {
    predicate = cb(predicate, context);
    var keys = !isArrayLike(obj) && _.keys(obj),
        length = (keys || obj).length;
    for (var index = 0; index < length; index++) {
      var currentKey = keys ? keys[index] : index;
      if (!predicate(obj[currentKey], currentKey, obj)) return false;
    }
    return true;
  };

  // Determine if at least one element in the object matches a truth test.
  // Aliased as `any`.
  // 如果obj有一个值符合preidicate，那么返回true，若全部不符合，那么返回false
  _.some = _.any = function(obj, predicate, context) {
    predicate = cb(predicate, context);
    var keys = !isArrayLike(obj) && _.keys(obj),
        length = (keys || obj).length;
    for (var index = 0; index < length; index++) {
      var currentKey = keys ? keys[index] : index;
      if (predicate(obj[currentKey], currentKey, obj)) return true;
    }
    return false;
  };

  // Determine if the array or object contains a given item (using `===`).
  // Aliased as `includes` and `include`.
  // 判断obj里从fromIndex开始遍历,是否包含item
  _.contains = _.includes = _.include = function(obj, item, fromIndex, guard) {
    // 如果不是类数组对象,那么得到obj的所有值
    if (!isArrayLike(obj)) obj = _.values(obj);
    // 如果fromIndex不是数字或者guard为true,则开始查找的索引值为0
    if (typeof fromIndex != 'number' || guard) fromIndex = 0;
    return _.indexOf(obj, item, fromIndex) >= 0;
  };

  // Invoke a method (with arguments) on every item in a collection.
  // 返回obj调用method函数的结果集，args为收集剩余参数的变量
  _.invoke = restArgs(function(obj, method, args) {
    var isFunc = _.isFunction(method);
    return _.map(obj, function(value) {
      var func = isFunc ? method : value[method];
      return func == null ? func : func.apply(value, args);
    });
  });

  // Convenience version of a common use case of `map`: fetching a property.
  // map的简化版本，将obj中所有值的key属性的值提取出来
  _.pluck = function(obj, key) {
    return _.map(obj, _.property(key));
  };

  // Convenience version of a common use case of `filter`: selecting only objects
  // containing specific `key:value` pairs.
  // 返回obj里和attrs一致的结果集
  // 和filter类似，不过filter更多变，因为filter可以定义过滤的条件，而这里过滤的条件只是单一的对象相等
  // 只要obj里的值的属性拥有attrs的所有属性，即为结果之一
  _.where = function(obj, attrs) {
    return _.filter(obj, _.matcher(attrs));
  };

  // Convenience version of a common use case of `find`: getting the first object
  // containing specific `key:value` pairs.
  // 返回obj里和attrs一致的结果集里的第一个函数
  _.findWhere = function(obj, attrs) {
    return _.find(obj, _.matcher(attrs));
  };

  // Return the maximum element (or element-based computation).
  // 取最大值
  // iteratee用来提取需要比较的属性值
  _.max = function(obj, iteratee, context) {
    var result = -Infinity, lastComputed = -Infinity,
        value, computed;
    if (iteratee == null || (typeof iteratee == 'number' && typeof obj[0] != 'object') && obj != null) {
      // 如果iteratee为空或数值且obj的第一个值不是object类型且obj不为空
      obj = isArrayLike(obj) ? obj : _.values(obj);
      for (var i = 0, length = obj.length; i < length; i++) {
        value = obj[i];
        if (value != null && value > result) {
          result = value;
        }
      }
    } else {
      iteratee = cb(iteratee, context);
      _.each(obj, function(v, index, list) {
        computed = iteratee(v, index, list);
        if (computed > lastComputed || computed === -Infinity && result === -Infinity) {
          result = v;
          lastComputed = computed;
        }
      });
    }
    return result;
  };

  // Return the minimum element (or element-based computation).
  // 取最小值，类似max
  _.min = function(obj, iteratee, context) {
    var result = Infinity, lastComputed = Infinity,
        value, computed;
    if (iteratee == null || (typeof iteratee == 'number' && typeof obj[0] != 'object') && obj != null) {
      obj = isArrayLike(obj) ? obj : _.values(obj);
      for (var i = 0, length = obj.length; i < length; i++) {
        value = obj[i];
        if (value != null && value < result) {
          result = value;
        }
      }
    } else {
      iteratee = cb(iteratee, context);
      _.each(obj, function(v, index, list) {
        computed = iteratee(v, index, list);
        if (computed < lastComputed || computed === Infinity && result === Infinity) {
          result = v;
          lastComputed = computed;
        }
      });
    }
    return result;
  };

  // Shuffle a collection.
  // 返回obj的一个打乱序列
  _.shuffle = function(obj) {
    return _.sample(obj, Infinity);
  };

  // Sample **n** random values from a collection using the modern version of the
  // [Fisher-Yates shuffle](http://en.wikipedia.org/wiki/Fisher–Yates_shuffle).
  // If **n** is not specified, returns a single random element.
  // The internal `guard` argument allows it to work with `map`.
  // 对传入的obj进行返回一个打乱的obj
  // n为返回多少个打乱数值的obj
  // 从obj中产生一个随机样本。传递一个数字表示从obj中返回n个随机元素。否则将返回一个单一的随机项。
  // 若obj是对象则返回的是值的随机样本
  _.sample = function(obj, n, guard) {
    if (n == null || guard) {
      if (!isArrayLike(obj)) obj = _.values(obj);
      return obj[_.random(obj.length - 1)];
    }
    var sample = isArrayLike(obj) ? _.clone(obj) : _.values(obj);
    var length = getLength(sample);
    n = Math.max(Math.min(n, length), 0);
    var last = length - 1;
    for (var index = 0; index < n; index++) {
      // 返回index到last之间的一个随机数
      var rand = _.random(index, last);
      // 存储当前值
      var temp = sample[index];
      // 交换
      sample[index] = sample[rand];
      sample[rand] = temp;
    }
    return sample.slice(0, n);
  };

  // Sort the object's values by a criterion produced by an iteratee.
  // 对obj进行排序,如果有iteratee则根据它进行排序
  _.sortBy = function(obj, iteratee, context) {
    var index = 0;
    iteratee = cb(iteratee, context);
    return _.pluck(_.map(obj, function(value, key, list) {
      // 返回一个包含原始数据value、索引index、调用执行函数的结果数值criteria的数组
      return {
        value: value,
        index: index++,
        criteria: iteratee(value, key, list)
      };
    }).sort(function(left, right) {
      var a = left.criteria;
      var b = right.criteria;
      if (a !== b) {
        if (a > b || a === void 0) return 1;
        if (a < b || b === void 0) return -1;
      }
      return left.index - right.index;
    }), 'value');// _.plunk(排好序的数组, 'value')将数组成员的value值抽出成一个数组
  };

  // An internal function used for aggregate "group by" operations.
  var group = function(behavior, partition) {
    return function(obj, iteratee, context) {
      var result = partition ? [[], []] : {};
      iteratee = cb(iteratee, context);
      _.each(obj, function(value, index) {
        var key = iteratee(value, index, obj);
        behavior(result, value, key);
      });
      return result;
    };
  };

  // Groups the object's values by a criterion. Pass either a string attribute
  // to group by, or a function that returns the criterion.
  // 对传入的obj进行分组,可以根据iteratee的返回值进行分组,如果是一个字符串,则是抽取obj里每个对象的该属性进行分组
  _.groupBy = group(function(result, value, key) {
    if (_.has(result, key)) result[key].push(value); else result[key] = [value];
  });

  // Indexes the object's values by a criterion, similar to `groupBy`, but for
  // when you know that your index values will be unique.
  // 和groupBy非常像，但是当你知道你的键是唯一的时候可以使用indexBy
  // 当键不唯一的时候使用这个，那么该键对应的对象将会是最后一个该键的对象
  _.indexBy = group(function(result, value, key) {
    result[key] = value;
  });

  // Counts instances of an object that group by a certain criterion. Pass
  // either a string attribute to count by, or a function that returns the
  // criterion.
  // 计算obj中拥有key属性的成员的个数，然后得到计算结果，结果是key属性对应一个统计数字
  _.countBy = group(function(result, value, key) {
    if (_.has(result, key)) result[key]++; else result[key] = 1;
  });

  var reStrSymbol = /[^\ud800-\udfff]|[\ud800-\udbff][\udc00-\udfff]|[\ud800-\udfff]/g;
  // Safely create a real, live array from anything iterable.
  _.toArray = function(obj) {
    if (!obj) return [];
    if (_.isArray(obj)) return slice.call(obj);
    if (_.isString(obj)) {
      // Keep surrogate pair characters together
      return obj.match(reStrSymbol);//此处应该进行理解
    }
    if (isArrayLike(obj)) return _.map(obj, _.identity);
    return _.values(obj);
  };

  // Return the number of elements in an object.
  _.size = function(obj) {
    if (obj == null) return 0;
    return isArrayLike(obj) ? obj.length : _.keys(obj).length;
  };

  // Split a collection into two arrays: one whose elements all satisfy the given
  // predicate, and one whose elements all do not satisfy the predicate.
  _.partition = group(function(result, value, pass) {
    result[pass ? 0 : 1].push(value);
  }, true);

  // Array Functions
  // ---------------

  // Get the first element of an array. Passing **n** will return the first N
  // values in the array. Aliased as `head` and `take`. The **guard** check
  // allows it to work with `_.map`.
  // 返回数组的第一个数，如果n'不为空，则返回开始的n个元素
  _.first = _.head = _.take = function(array, n, guard) {
    if (array == null) return void 0;
    if (n == null || guard) return array[0];
    return _.initial(array, array.length - n);
  };

  // Returns everything but the last entry of the array. Especially useful on
  // the arguments object. Passing **n** will return all the values in
  // the array, excluding the last N.
  // 返回数组中除了最后一个元素外的其他全部元素。 在arguments对象上特别有用。传递 n参数将从结果中排除从最后一个开始的n个元素
  _.initial = function(array, n, guard) {
    return slice.call(array, 0, Math.max(0, array.length - (n == null || guard ? 1 : n)));
  };

  // Get the last element of an array. Passing **n** will return the last N
  // values in the array.
  // 返回数组最后一个数，如果有n，则是返回数组后n个数
  _.last = function(array, n, guard) {
    if (array == null) return void 0;
    if (n == null || guard) return array[array.length - 1];
    return _.rest(array, Math.max(0, array.length - n));
  };

  // Returns everything but the first entry of the array. Aliased as `tail` and `drop`.
  // Especially useful on the arguments object. Passing an **n** will return
  // the rest N values in the array.
  // 返回数组除了第一个数以外的所有数，如果有n，则返回数组从n开始的所有数
  _.rest = _.tail = _.drop = function(array, n, guard) {
    return slice.call(array, n == null || guard ? 1 : n);
  };

  // Trim out all falsy values from an array.
  // 返回一个去除了所有false值的数组，去除包括false、undefined、null、“”、0、NaN
  _.compact = function(array) {
    return _.filter(array, _.identity);
  };

  // Internal implementation of a recursive `flatten` function.
  var flatten = function(input, shallow, strict, output) {
    // output不为空则使用ouput存储
    output = output || [];
    var idx = output.length;
    for (var i = 0, length = getLength(input); i < length; i++) {
      var value = input[i];
      if (isArrayLike(value) && (_.isArray(value) || _.isArguments(value))) {
        // Flatten current level of array or arguments object
        // 如果shallow为true，则浅展开一层
        if (shallow) {
          var j = 0, len = value.length;
          while (j < len) output[idx++] = value[j++];
        } else {
          // 递归使用，有点类似后序遍历二叉树，这里的数组就是一颗二叉树，
          flatten(value, shallow, strict, output);
          idx = output.length;
        }
      } else if (!strict) {
        // 如果不是数组，则视strict的值决定
        // 目前遇到的一种情况表示strict决定是否返回单层的数组的副本
        output[idx++] = value;
      }
    }
    return output;
  };

  // Flatten out an array, either recursively (by default), or just one level.
  // 将array展开成一个数组，如果shallow为true，则只展开一层
  _.flatten = function(array, shallow) {
    return flatten(array, shallow, false);
  };

  // Return a version of the array that does not contain the specified value(s).
  // 返回array中没有otherArrays中的元素的所有值的集合
  _.without = restArgs(function(array, otherArrays) {
    return _.difference(array, otherArrays);
  });

  // Produce a duplicate-free version of the array. If the array has already
  // been sorted, you have the option of using a faster algorithm.
  // Aliased as `unique`.
  // 去重函数
  _.uniq = _.unique = function(array, isSorted, iteratee, context) {
    if (!_.isBoolean(isSorted)) {// 如果第二个参数不是布尔值
      context = iteratee;
      iteratee = isSorted;// 则视其为迭代用的计算函数
      isSorted = false;// 且视该数组未排序
    }
    if (iteratee != null) iteratee = cb(iteratee, context);
    var result = [];
    var seen = [];
    for (var i = 0, length = getLength(array); i < length; i++) {
      var value = array[i],
          computed = iteratee ? iteratee(value, i, array) : value;
      if (isSorted) {
        if (!i || seen !== computed) result.push(value);
        seen = computed;
      } else if (iteratee) {// 如果有迭代函数
        if (!_.contains(seen, computed)) {// seen是一个标记数组，标记seen里有没有同一个计算结果值computed
          seen.push(computed);
          result.push(value);
        }
      } else if (!_.contains(result, value)) {// 默认下直接查重并插入
        result.push(value);
      }
    }
    return result;
  };

  // Produce an array that contains the union: each distinct element from all of
  // the passed-in arrays.
  // 求出多个数组的并集，返回的元素唯一
  _.union = restArgs(function(arrays) {
    return _.uniq(flatten(arrays, true, true));
  });

  // Produce an array that contains every item shared between all the
  // passed-in arrays.
  // 求交集
  _.intersection = function(array) {
    var result = [];
    var argsLength = arguments.length;
    for (var i = 0, length = getLength(array); i < length; i++) {
      var item = array[i];
      if (_.contains(result, item)) continue;// 如果交集结果中已经判断过了同一个item的值，则直接下一次循环
      var j;
      for (j = 1; j < argsLength; j++) {
        if (!_.contains(arguments[j], item)) break;// 只要有一个数组没有item，则直接退出循环
      }
      if (j === argsLength) result.push(item);// 只有所有数组里拥有同一个item才是交集的元素之一
    }
    return result;
  };

  // Take the difference between one array and a number of other arrays.
  // Only the elements present in just the first array will remain.
  // 如果想去除的参数不是以数组形式传入，那么会发生神奇的bug
  // 因为restArgs会对想去除的参数进行包装，如果传入的是数组，那么包装后rest是[Array]
  // 如果是一个个传，那么包装后是[n1,n2,n3],即Array，注意，不同之处在于上面的形式在Array以外有包了一层Array
  // 然后调用flatten，由于这里的strict传入了true，所以对于单纯的Array（即[n1,n2,n3]）
  // 因为flatten内部要判断Array的value的类型是否类数组，明显地[n1,n2,n3]的value->n1不是类数组
  // 同时又启用了strict，所以直接返回了[]
  // 这里，如果strict不启用，那么[n1,n2,n3]可以正常返回自身，则一切正常
  // 所以这里还是建议使用without
  // difference主要用于获取两个array直接的差集
  // without则用于获取array除了其他参数的集合
  _.difference = restArgs(function(array, rest) {
    // 需要去除的rest浅展开
    rest = flatten(rest, true, true);
    return _.filter(array, function(value){
      return !_.contains(rest, value);
    });
  });

  // Complement of _.zip. Unzip accepts an array of arrays and groups
  // each array's elements on shared indices
  // 将一个包含多个数组的数组进行重组，重组后的结果是result的第n个数组包含了之前多个数组的第n个元素
  _.unzip = function(array) {
    var length = array && _.max(array, getLength).length || 0;
    var result = Array(length);

    for (var index = 0; index < length; index++) {
      result[index] = _.pluck(array, index);
    }
    return result;
  };

  // Zip together multiple lists into a single array -- elements that share
  // an index go together.
  // 与unzip相反
  _.zip = restArgs(_.unzip);

  // Converts lists into objects. Pass either a single array of `[key, value]`
  // pairs, or two parallel arrays of the same length -- one of keys, and one of
  // the corresponding values.
  // 将一个list转换成对象
  // 如果values存在，则是list的第n个元素为键，values的第n个元素为值
  // 如果values不存在，则list的第一个元素是键数组，list的第二个元素是值数组
  _.object = function(list, values) {
    var result = {};
    for (var i = 0, length = getLength(list); i < length; i++) {
      if (values) {
        result[list[i]] = values[i];
      } else {
        result[list[i][0]] = list[i][1];
      }
    }
    return result;
  };

  // Generator function to create the findIndex and findLastIndex functions
  // 供给findIndex和findLastIndex函数调用
  // dir为正数则返回第一个匹配的索引，dir为负数则返回最后一个匹配的索引
  var createPredicateIndexFinder = function(dir) {
    return function(array, predicate, context) {
      predicate = cb(predicate, context);
      var length = getLength(array);
      var index = dir > 0 ? 0 : length - 1;
      for (; index >= 0 && index < length; index += dir) {
        if (predicate(array[index], index, array)) return index;
      }
      return -1;
    };
  };

  // Returns the first index on an array-like that passes a predicate test
  // 和indexOf()方法类似，都是返回寻找值的索引,不过findIndex可以根据function搜索，而indexOf是根据传入的值去搜索
  _.findIndex = createPredicateIndexFinder(1);
  _.findLastIndex = createPredicateIndexFinder(-1);

  // Use a comparator function to figure out the smallest index at which
  // an object should be inserted so as to maintain order. Uses binary search.
  // 这里有个问题，当array找不到obj时，如果obj一直大于array的所有值，最后返回length，反之返回0
  // 当iteratee指定一个字符串时，表示要对比obj里的iteratee变量
  // 当iteratee为null时，只是单纯地比较obj整个对象
  _.sortedIndex = function(array, obj, iteratee, context) {
    iteratee = cb(iteratee, context, 1);
    var value = iteratee(obj);
    var low = 0, high = getLength(array);
    while (low < high) {
      var mid = Math.floor((low + high) / 2);
      if (iteratee(array[mid]) < value) low = mid + 1; else high = mid;
    }
    return low;
  };

  // Generator function to create the indexOf and lastIndexOf functions
  // dir控制正向或方向搜索
  // predicateFind为使用的搜索方法
  // sortedIndex为使用的便捷搜索方法，目前用二分搜索
  var createIndexFinder = function(dir, predicateFind, sortedIndex) {
    return function(array, item, idx) {
      // array为待搜索的数组
      // item为待搜索的值
      // idx为起始值或是否使用二分搜索的标志
      var i = 0, length = getLength(array);
      if (typeof idx == 'number') {
        // 如果idx是一个数字，那么需要获取idx，idx大于0就取idx，如果idx小于0且idx加上length后仍小于0，那么取0
        if (dir > 0) {
          i = idx >= 0 ? idx : Math.max(idx + length, i);
        } else {
          // dir小于0，那么需要获取length，获取小的那个
          length = idx >= 0 ? Math.min(idx + 1, length) : idx + length + 1;
        }
      } else if (sortedIndex && idx && length) {
        // 如果idx是一个boolean值，那么使用更快的二分搜索
        // lastIndexOf不适用二分搜索
        idx = sortedIndex(array, item);
        // 由于二分搜索返回的值不具备判断性，所以返回index后仍然需要判断找到的索引值对应的值是否是想要找的值
        return array[idx] === item ? idx : -1;
      }

      // 以上部分是在sortedIndex不为空的情况下执行，如果为空，则按照以下方法执行

      if (item !== item) {// 若待查找的是NaN
        idx = predicateFind(slice.call(array, i, length), _.isNaN);// 用_.findIndex或_.findLastIndex，根据_.isNaN来查找NaN
        return idx >= 0 ? idx + i : -1;
      }
      for (idx = dir > 0 ? i : length - 1; idx >= 0 && idx < length; idx += dir) {
        // 普通情况则用遍历来查找第一个（或最后一个）匹配的数的索引
        if (array[idx] === item) return idx;
      }
      return -1;
    };
  };

  // Return the position of the first occurrence of an item in an array,
  // or -1 if the item is not included in the array.
  // If the array is large and already in sort order, pass `true`
  // for **isSorted** to use binary search.
  _.indexOf = createIndexFinder(1, _.findIndex, _.sortedIndex);
  // 注意此处不使用二分搜索，即使用户指定true也不启用
  _.lastIndexOf = createIndexFinder(-1, _.findLastIndex);

  // Generate an integer Array containing an arithmetic progression. A port of
  // the native Python `range()` function. See
  // [the Python documentation](http://docs.python.org/library/functions.html#range).
  // 生成一个从start都stop的增量为step的数组
  _.range = function(start, stop, step) {
    if (stop == null) {
      stop = start || 0;
      start = 0;
    }
    step = step || 1;

    var length = Math.max(Math.ceil((stop - start) / step), 0);
    var range = Array(length);

    for (var idx = 0; idx < length; idx++, start += step) {
      range[idx] = start;
    }

    return range;
  };

  // Split an **array** into several arrays containing **count** or less elements
  // of initial array
  // 将一个数组array分割成多个由count个元素组成的数组，再装进数组返回
  _.chunk = function(array, count) {
    if (count == null || count < 1) return [];

    var result = [];
    var i = 0, length = array.length;
    while (i < length) {
      result.push(slice.call(array, i, i += count));
    }
    return result;
  };

  // Function (ahem) Functions
  // ------------------

  // Determines whether to execute a function as a constructor
  // or a normal function with the provided arguments
  var executeBound = function(sourceFunc, boundFunc, context, callingContext, args) {
    // 如果callingContext不是boundFunc的上下文，则使用context调用sourceFunc
    if (!(callingContext instanceof boundFunc)) return sourceFunc.apply(context, args);
    var self = baseCreate(sourceFunc.prototype);
    var result = sourceFunc.apply(self, args);
    if (_.isObject(result)) return result;
    return self;
  };

  // Create a function bound to a given object (assigning `this`, and arguments,
  // optionally). Delegates to **ECMAScript 5**'s native `Function.bind` if
  // available.
  // 将func绑定到context上
  _.bind = restArgs(function(func, context, args) {
    // 这里的args是一开始绑定使用的剩余参数
    if (!_.isFunction(func)) throw new TypeError('Bind must be called on a function');
    var bound = restArgs(function(callArgs) {
      // 这里的callArgs是给后面调用绑定好的函数传参使用的
      return executeBound(func, bound, context, this, args.concat(callArgs));
      // bind调用executeBound一般只进行到第一步，因为func必须是一个函数
    });
    return bound;
  });

  // Partially apply a function by creating a version that has had some of its
  // arguments pre-filled, without changing its dynamic `this` context. _ acts
  // as a placeholder by default, allowing any combination of arguments to be
  // pre-filled. Set `_.partial.placeholder` for a custom placeholder argument.
  // 局部应用一个函数填充在任意个数的 arguments，不改变其动态this值。和bind方法很相近。
  // 你可以传递_ 给arguments列表来指定一个不预先填充，但在调用时提供的参数。
  _.partial = restArgs(function(func, boundArgs) {
    var placeholder = _.partial.placeholder;
    var bound = function() {
      // length获取预填参数的个数
      var position = 0, length = boundArgs.length;
      var args = Array(length);
      for (var i = 0; i < length; i++) {
        // 如果boundArgs[i]是用户指定的待填参数，那么args[i]就会是调用处理好的函数时传入的参数之一
        args[i] = boundArgs[i] === placeholder ? arguments[position++] : boundArgs[i];
      }
      // 如果显式地指定填充和待填的参数都处理好以后
      // 还有未被传入的参数，那么默认这些参数都按顺序从最后一个参数开始传入
      while (position < arguments.length) args.push(arguments[position++]);
      return executeBound(func, bound, this, this, args);
    };
    return bound;
  });

  _.partial.placeholder = _;

  // Bind a number of an object's methods to that object. Remaining arguments
  // are the method names to be bound. Useful for ensuring that all callbacks
  // defined on an object belong to it.
  // 将obj里的多个method绑定到obj上，无论如何调用，method的this始终指向obj
  _.bindAll = restArgs(function(obj, keys) {
    // 将keys展开
    keys = flatten(keys, false, false);
    var index = keys.length;
    if (index < 1) throw new Error('bindAll must be passed function names');
    while (index--) {
      var key = keys[index];
      // obj的key函数经过bind绑定返回一个已经绑定了obj为上下文的函数
      obj[key] = _.bind(obj[key], obj);
    }
  });

  // Memoize an expensive function by storing its results.
  // 将func处理成一个可以缓存计算结果的函数
  // 处理以后的func可以调用func.cache查看每次的计算结果
  // 如func是一个递归函数，那么可以缓存每次递归的结果
  _.memoize = function(func, hasher) {
    var memoize = function(key) {
      var cache = memoize.cache;
      var address = '' + (hasher ? hasher.apply(this, arguments) : key);
      // 如果缓存里没有此记录，那么就记录
      if (!_.has(cache, address)) cache[address] = func.apply(this, arguments);
      return cache[address];
    };
    memoize.cache = {};
    return memoize;
  };

  // Delays a function for the given number of milliseconds, and then calls
  // it with the arguments supplied.
  // 类似setTimeout，等待wait毫秒后调用function。
  // 如果传递可选的参数arguments，当函数function执行时， arguments 会作为参数传入。
  _.delay = restArgs(function(func, wait, args) {
    return setTimeout(function() {
      return func.apply(null, args);
    }, wait);
  });

  // Defers a function, scheduling it to run after the current call stack has
  // cleared.
  // 延迟调用function直到当前调用栈清空为止，类似使用延时为0的setTimeout方法。
  // 对于执行开销大的计算和无阻塞UI线程的HTML渲染时候非常有用。
  // 如果传递arguments参数，当函数function执行时， arguments 会作为参数传入。
  _.defer = _.partial(_.delay, _, 1);

  // Returns a function, that, when invoked, will only be triggered at most once
  // during a given window of time. Normally, the throttled function will run
  // as much as it can, without ever going more than once per `wait` duration;
  // but if you'd like to disable the execution on the leading edge, pass
  // `{leading: false}`. To disable execution on the trailing edge, ditto.
  // 创建并返回一个像节流阀一样的函数，
  // 当重复调用函数的时候，至少每隔 wait毫秒调用一次该函数。
  // 对于想控制一些触发频率较高的事件有帮助。
  // 在wait时间内，只能调用一次func，如果多次调用，那么wait时间内将不会执行
  // 不过将在wait时间后执行最后一次调用
  // 使用了leading，那么执行顺序就是：_____执行_____执行_____执行
  // 不使用leading，那么执行顺序就是：执行_____执行_____执行_____执行
  // 使用了trailing，那么执行顺序就是：执行_____执行_____执行_____
  // 不使用trailing，那么执行顺序就是：执行_____执行_____执行_____执行
  // 使用leading，那么表示不立即调用，且在wait时间段内调用的无数次函数将会在wait后执行最后一次调用
  // 使用trailing，那么表示立即调用，且在wait时间段内调用的无数次函数将不会执行，除非在wait时间后再次调用
  // 两个都不使用，那么表示立即调用，且在wait时间段内调用的无数次函数将会在wait后执行最后一次调用
  _.throttle = function(func, wait, options) {
    var timeout, context, args, result;
    var previous = 0;
    if (!options) options = {};

    var later = function() {
      // 如果启用了leading，那么下一次调用依旧是要在wait时间后执行，而不是立即执行
      previous = options.leading === false ? 0 : _.now();
      timeout = null;
      result = func.apply(context, args);
      if (!timeout) context = args = null;
    };

    var throttled = function() {
      // 获得当前时间戳
      var now = _.now();
      // 第一次？禁用第一次调用？
      // 如果禁用第一次，那么previous=now，那么remaining第一次必定等于wait，所以不执行
      if (!previous && options.leading === false) previous = now;
      // 剩余等待时间
      var remaining = wait - (now - previous);

      // 多次调用而不执行，那么此处的args会一直被更新到最后一个的args
      args = arguments;
      if (remaining <= 0 || remaining > wait) {
        if (timeout) {
          clearTimeout(timeout);
          timeout = null;
        }
        previous = now;
        result = func.apply(context, args);
        if (!timeout) context = args = null;
      } else if (!timeout && options.trailing !== false) {
        timeout = setTimeout(later, remaining);
      }
      return result;
    };

    throttled.clear = function() {
      clearTimeout(timeout);
      previous = 0;
      timeout = context = args = null;
    };

    return throttled;
  };

  // Returns a function, that, as long as it continues to be invoked, will not
  // be triggered. The function will be called after it stops being called for
  // N milliseconds. If `immediate` is passed, trigger the function on the
  // leading edge, instead of the trailing.
  // 返回 function 函数的防反跳版本, 将延迟函数的执行(真正的执行)在函数最后一次调用时刻的 wait 毫秒之后. 
  // 对于必须在一些输入（多是一些用户操作）停止到达之后执行的行为有帮助。 
  // 例如: 渲染一个Markdown格式的评论预览, 当窗口停止改变大小之后重新计算布局, 等等.
  // 传参 immediate 为 true， debounce会在 wait 时间间隔的开始调用这个函数 。
  // 当不立即调用函数时（即immediate为false），若第一次函数调用还没有执行，在调用第一次500ms后调用第二次
  // 预期结果应该是在500ms后函数的执行又被推迟了500ms（假设函数的执行需延迟1000ms）
  // 但是真正的结果是第一次和第二次都执行了
  // 这种情况的概率颇大
  // eg.var result = _.debounce(function(key){
  //      console.log(1+_.now())
  //    },2000,false);
  //    result();
  //    setTimeout(result,500);
  _.debounce = function(func, wait, immediate) {
    var timeout, result;

    var later = function(context, args) {
      timeout = null;
      if (args) result = func.apply(context, args);
    };

    var debounced = restArgs(function(args) {
      // 是否先立即调用
      var callNow = immediate && !timeout;
      // 加入之前有同一个函数调用还未完成，那么清除，时间重新计算
      if (timeout) clearTimeout(timeout);
      if (callNow) {
        // 如果立即调用，那么先设置好定时，再直接调用一次函数
        // 这里的timeout的作用是防止在立即调用该函数后又立即调用了一次
        // 加了timeout可以控制下一次的调用必须是wait间隔以后
        // 否则调用将不会执行
        timeout = setTimeout(later, wait);
        result = func.apply(this, args);
      } else if (!immediate) {
        // 如果不立即调用，那么使用delay延时调用
        // 如果不立即调用，在wait时间内再次调用的话会重新计时
        timeout = _.delay(later, wait, this, args);
      }

      return result;
    });

    debounced.clear = function() {
      clearTimeout(timeout);
      timeout = null;
    };

    return debounced;
  };

  // Returns the first function passed as an argument to the second,
  // allowing you to adjust arguments, run code before and after, and
  // conditionally execute the original function.
  // 注意这里传参和实际调用的顺序相反，即使将func传入了wrapper
  _.wrap = function(func, wrapper) {
    return _.partial(wrapper, func);
  };

  // Returns a negated version of the passed-in predicate.
  // 返回一个判断结果相反的函数predicate
  _.negate = function(predicate) {
    return function() {
      return !predicate.apply(this, arguments);
    };
  };

  // Returns a function that is the composition of a list of functions, each
  // consuming the return value of the function that follows.
  // 返回函数集 functions 组合后的复合函数, 
  // 也就是一个函数执行完之后把返回的结果再作为参数赋给下一个函数来执行. 
  // 以此类推. 在数学里, 把函数 f(), g(), 和 h() 组合起来可以得到复合函数 f(g(h()))。
  _.compose = function() {
    var args = arguments;
    var start = args.length - 1;
    return function() {
      var i = start;
      // 首先执行最后一个函数得到结果
      var result = args[start].apply(this, arguments);
      // 然后从后往前的函数一个个调用前一个的结果
      while (i--) result = args[i].call(this, result);
      return result;
    };
  };

  // Returns a function that will only be executed on and after the Nth call.
  // func只有调用了times次后才能执行
  _.after = function(times, func) {
    return function() {
      if (--times < 1) {
        return func.apply(this, arguments);
      }
    };
  };

  // Returns a function that will only be executed up to (but not including) the Nth call.
  // func的调用次数不超过times次，不包括第times次，第times-1次的结果将被返回
  _.before = function(times, func) {
    var memo;
    return function() {
      if (--times > 0) {
        memo = func.apply(this, arguments);
      }
      if (times <= 1) func = null;
      return memo;
    };
  };

  // Returns a function that will be executed at most one time, no matter how
  // often you call it. Useful for lazy initialization.
  // 只能执行一次的函数
  _.once = _.partial(_.before, 2);

  _.restArgs = restArgs;

  // Object Functions
  // ----------------

  // Keys in IE < 9 that won't be iterated by `for key in ...` and thus missed.
  // IE9-的浏览器有一个bug,当对象属性有与例如toString之类的名称时,将不会出现在for……in枚举中
  // propertyIsEnumerable可以用来检测toString是否可以被枚举，如果不可以则会返回false，hasEmumBug返回true
  var hasEnumBug = !{toString: null}.propertyIsEnumerable('toString');
  // 以下是不可枚举的属性
  var nonEnumerableProps = ['valueOf', 'isPrototypeOf', 'toString',
                      'propertyIsEnumerable', 'hasOwnProperty', 'toLocaleString'];

  // 这个内部函数由别的函数调用，是用于克服IE9-的BUG，将与[[DontEnum]]属性同名的实例属性添加至keys数组中的
  var collectNonEnumProps = function(obj, keys) {
    var nonEnumIdx = nonEnumerableProps.length;
    var constructor = obj.constructor;
    // 获取obj的原型，如果没有就获取Object的原型
    var proto = _.isFunction(constructor) && constructor.prototype || ObjProto;

    // Constructor is a special case.
    var prop = 'constructor';
    // 判断obj中是否有constructor属性，并且keys中还未插入，满足的话就插入keys
    if (_.has(obj, prop) && !_.contains(keys, prop)) keys.push(prop);

    while (nonEnumIdx--) {
      prop = nonEnumerableProps[nonEnumIdx];
      // 如果obj中有prop属性，并且不是原型中的prop属性，并且keys还未插入，都满足就插入keys
      if (prop in obj && obj[prop] !== proto[prop] && !_.contains(keys, prop)) {
        keys.push(prop);
      }
    }
  };

  // Retrieve the names of an object's own properties.
  // Delegates to **ECMAScript 5**'s native `Object.keys`
  _.keys = function(obj) {
    if (!_.isObject(obj)) return [];
    if (nativeKeys) return nativeKeys(obj);
    var keys = [];
    // 这里与allKeys的区别就是这里判断key是否是obj对象的成员，即不查找原型链中的成员
    for (var key in obj) if (_.has(obj, key)) keys.push(key);
    // Ahem, IE < 9.
    if (hasEnumBug) collectNonEnumProps(obj, keys);
    return keys;
  };

  // Retrieve all the property names of an object.
  // 返回对象的所有键，包括原型链里的
  _.allKeys = function(obj) {
    if (!_.isObject(obj)) return [];
    var keys = [];
    for (var key in obj) keys.push(key);
    // Ahem, IE < 9.
    if (hasEnumBug) collectNonEnumProps(obj, keys);
    return keys;
  };

  // Retrieve the values of an object's properties.
  // 返回对象里的所有属性的值，不包括原型链里的
  _.values = function(obj) {
    var keys = _.keys(obj);
    var length = keys.length;
    var values = Array(length);
    for (var i = 0; i < length; i++) {
      values[i] = obj[keys[i]];
    }
    return values;
  };

  // Returns the results of applying the iteratee to each element of the object
  // In contrast to _.map it returns an object
  // 类似于map，但是map是返回计算后的结果集
  // 这里返回的是经过iteratee转换的键值对象
  // 主要用于对象
  _.mapObject = function(obj, iteratee, context) {
    iteratee = cb(iteratee, context);
    var keys = _.keys(obj),
      length = keys.length,
      results = {};
    for (var index = 0; index < length; index++) {
      var currentKey = keys[index];
      results[currentKey] = iteratee(obj[currentKey], currentKey, obj);
    }
    return results;
  };

  // Convert an object into a list of `[key, value]` pairs.
  // 把一个对象转变为一个[key, value]形式的数组。
  _.pairs = function(obj) {
    var keys = _.keys(obj);
    var length = keys.length;
    var pairs = Array(length);
    for (var i = 0; i < length; i++) {
      pairs[i] = [keys[i], obj[keys[i]]];
    }
    return pairs;
  };

  // Invert the keys and values of an object. The values must be serializable.
  // 将obj的键值调换位置，必须保证值是唯一的，否则会发生覆盖
  _.invert = function(obj) {
    var result = {};
    var keys = _.keys(obj);
    for (var i = 0, length = keys.length; i < length; i++) {
      result[obj[keys[i]]] = keys[i];
    }
    return result;
  };

  // Return a sorted list of the function names available on the object.
  // Aliased as `methods`
  // 返回一个对象里的所有函数名，且函数名是排序好的
  _.functions = _.methods = function(obj) {
    var names = [];
    for (var key in obj) {
      if (_.isFunction(obj[key])) names.push(key);
    }
    return names.sort();
  };

  // An internal function for creating assigner functions.
  // 将除obj以外的对象根据keysFunc处理后，将所有对象打包成obj返回
  var createAssigner = function(keysFunc, defaults) {
    return function(obj) {
      var length = arguments.length;
      if (defaults) obj = Object(obj);
      if (length < 2 || obj == null) return obj;
      for (var index = 1; index < length; index++) {
        var source = arguments[index],
            keys = keysFunc(source),
            l = keys.length;
        for (var i = 0; i < l; i++) {
          var key = keys[i];
          // 这里如果defaults没启用，则可以直接插入，但是可能发生覆盖
          // 如果defaults启用了，那么如果obj已存在的键值不会被覆盖
          if (!defaults || obj[key] === void 0) obj[key] = source[key];
        }
      }
      return obj;
    };
  };

  // Extend a given object with all the properties in passed-in object(s).
  // 将多个对象的所有属性（包括原型中）打包成一个新对象返回
  _.extend = createAssigner(_.allKeys);

  // Assigns a given object with all the own properties in the passed-in object(s)
  // (https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)
  // 将传入的需要继承的obj的对象的键值加入到obj里
  // 将多个对象的所有属性（不包括原型中）打包成一个新对象返回
  _.extendOwn = _.assign = createAssigner(_.keys);

  // Returns the first key on an object that passes a predicate test
  // 先将obj的key全部取出成为一个数组，然后一一遍历key对应的值，判断是否相等，最后返回key
  // 找到符合predicate的第一个obj的key
  _.findKey = function(obj, predicate, context) {
    predicate = cb(predicate, context);
    var keys = _.keys(obj), key;
    for (var i = 0, length = keys.length; i < length; i++) {
      key = keys[i];
      if (predicate(obj[key], key, obj)) return key;
    }
  };

  // Internal pick helper function to determine if `obj` has key `key`.
  // 判断key是否在obj中
  var keyInObj = function(value, key, obj) {
    return key in obj;
  };

  // Return a copy of the object only containing the whitelisted properties.
  // 过滤出obj的keys属性的键值
  _.pick = restArgs(function(obj, keys) {
    var result = {}, iteratee = keys[0];
    if (obj == null) return result;
    if (_.isFunction(iteratee)) {
      // 如果是函数，根据函数来判断返回的对象的键值
      if (keys.length > 1) iteratee = optimizeCb(iteratee, keys[1]);
      keys = _.allKeys(obj);
    } else {
      // 若是指定过滤属性值
      iteratee = keyInObj;
      keys = flatten(keys, false, false);// 将keys完全展开成一个数组
      obj = Object(obj);
    }
    for (var i = 0, length = keys.length; i < length; i++) {
      var key = keys[i];
      var value = obj[key];
      if (iteratee(value, key, obj)) result[key] = value;
    }
    return result;
  });

   // Return a copy of the object without the blacklisted properties.
  _.omit = restArgs(function(obj, keys) {
    var iteratee = keys[0], context;
    if (_.isFunction(iteratee)) {
      // 获得相反判断函数
      iteratee = _.negate(iteratee);
      // 获得上下文
      if (keys.length > 1) context = keys[1];
    } else {
      // 将keys全部展开并转化为String类型
      keys = _.map(flatten(keys, false, false), String);
      iteratee = function(value, key) {
        // 如果keys中没有key，返回true
        return !_.contains(keys, key);
      };
    }
    return _.pick(obj, iteratee, context);
  });

  // Fill in a given object with default properties.
  // 对一个对象的空属性，使用另一个对象的属性进行初始化
  _.defaults = createAssigner(_.allKeys, true);

  // Creates an object that inherits from the given prototype object.
  // If additional properties are provided then they will be added to the
  // created object.
  // 创建一个给定prototype的对象，如果props存在，那么同时要继承props自身的属性
  _.create = function(prototype, props) {
    var result = baseCreate(prototype);
    if (props) _.extendOwn(result, props);
    return result;
  };

  // Create a (shallow-cloned) duplicate of an object.
  // 浅拷贝一个对象
  // extend和extendOwn也是浅拷贝
  _.clone = function(obj) {
    if (!_.isObject(obj)) return obj;
    return _.isArray(obj) ? obj.slice() : _.extend({}, obj);
  };

  // Invokes interceptor with the obj, and then returns obj.
  // The primary purpose of this method is to "tap into" a method chain, in
  // order to perform operations on intermediate results within the chain.
  // 作为链式调用中的一环，可以打印链式到当前的结果，而不影响接下去的链式调用
  // 因为它原封不动返回了obj
  _.tap = function(obj, interceptor) {
    interceptor(obj);
    return obj;
  };

  // Returns whether an object has a given set of `key:value` pairs.
  // 判断attrs是否在object中
  _.isMatch = function(object, attrs) {
    var keys = _.keys(attrs), length = keys.length;
    if (object == null) return !length;
    var obj = Object(object);
    for (var i = 0; i < length; i++) {
      var key = keys[i];
      // undefined可以相等，所以要再判断key是否在obj中存在
      if (attrs[key] !== obj[key] || !(key in obj)) return false;
    }
    return true;
  };


  // Internal recursive comparison function for `isEqual`.
  var eq, deepEq;
  eq = function(a, b, aStack, bStack) {
    // Identical objects are equal. `0 === -0`, but they aren't identical.
    // See the [Harmony `egal` proposal](http://wiki.ecmascript.org/doku.php?id=harmony:egal).
    // 如果a===b,那么还要检测0和-0相等的情况
    // 1/0 = Infinity，-1/0 = -Infinity
    if (a === b) return a !== 0 || 1 / a === 1 / b;
    // A strict comparison is necessary because `null == undefined`.
    // 如果a和b有一个是null，那么要比较是否存在undefined和null相比较的情况，两个是不相等的
    if (a == null || b == null) return a === b;
    // `NaN`s are equivalent, but non-reflexive.
    // 当a是NaN，那么判断b是否也是NaN
    if (a !== a) return b !== b;
    // Exhaust primitive checks
    var type = typeof a;
    // 如果a不是函数和对象，b也不是对象，那么就不相等了
    // 以上判断都是基于基本类型的判断
    if (type !== 'function' && type !== 'object' && typeof b != 'object') return false;
    // 如果以上都没有判断出结果，那么明显a和b是引用类型
    return deepEq(a, b, aStack, bStack);
  };

  // Internal recursive comparison function for `isEqual`.
  deepEq = function(a, b, aStack, bStack) {
    // Unwrap any wrapped objects.
    // 如果a和b是underscore的实例，那么需要判断_wrapped值
    if (a instanceof _) a = a._wrapped;
    if (b instanceof _) b = b._wrapped;
    // Compare `[[Class]]` names.
    // 比较a和b的类型
    var className = toString.call(a);
    if (className !== toString.call(b)) return false;
    switch (className) {
      // Strings, numbers, regular expressions, dates, and booleans are compared by value.
      case '[object RegExp]':
      // RegExps are coerced to strings for comparison (Note: '' + /a/i === '/a/i')
      case '[object String]':
        // Primitives and their corresponding object wrappers are equivalent; thus, `"5"` is
        // equivalent to `new String("5")`.
        return '' + a === '' + b;
      case '[object Number]':
        // `NaN`s are equivalent, but non-reflexive.
        // Object(NaN) is equivalent to NaN
        if (+a !== +a) return +b !== +b;
        // An `egal` comparison is performed for other numeric values.
        return +a === 0 ? 1 / +a === 1 / b : +a === +b;
      case '[object Date]':
      case '[object Boolean]':
        // Coerce dates and booleans to numeric primitive values. Dates are compared by their
        // millisecond representations. Note that invalid dates with millisecond representations
        // of `NaN` are not equivalent.
        return +a === +b;
    }

    // 得到a是否是数组
    var areArrays = className === '[object Array]';
    if (!areArrays) {
      // 首先a已经判断不是数组
      // 那么如果a也不是函数，则表示a和b的类型不同
      // 如果a不是函数，是对象
      // 那么b如果不是对象，也就不相等了
      if (typeof a != 'object' || typeof b != 'object') return false;

      // Objects with different constructors are not equivalent, but `Object`s or `Array`s
      // from different frames are.
      var aCtor = a.constructor, bCtor = b.constructor;
      // a和b的构造函数不同
      // a和b的构造函数是函数并且构造函数必须是自己的实例
      // (貌似是指对象的构造函数必须是最原始的，而不是继承来的，
      // 最原始的构造函数是function Function(){},所有function都是他的实例)
      // 如果a和b没有构造函数
      if (aCtor !== bCtor && !(_.isFunction(aCtor) && aCtor instanceof aCtor &&
                               _.isFunction(bCtor) && bCtor instanceof bCtor)
                          && ('constructor' in a && 'constructor' in b)) {
        return false;
      }
    }
    // Assume equality for cyclic structures. The algorithm for detecting cyclic
    // structures is adapted from ES 5.1 section 15.12.3, abstract operation `JO`.

    // Initializing stack of traversed objects.
    // It's done here since we only need them for objects and arrays comparison.
    aStack = aStack || [];
    bStack = bStack || [];
    var length = aStack.length;
    while (length--) {
      // Linear search. Performance is inversely proportional to the number of
      // unique nested structures.
      // 有些奇怪的数组会自己插入自己，此时只要判断a和b相对应的栈是否都各自对应自己
      if (aStack[length] === a) return bStack[length] === b;
    }

    // Add the first object to the stack of traversed objects.
    aStack.push(a);
    bStack.push(b);

    // Recursively compare objects and arrays.
    // 如果是数组的判断
    if (areArrays) {
      // Compare array lengths to determine if a deep comparison is necessary.
      length = a.length;
      // a和b的数组长度不相等，明显它们也不相等
      if (length !== b.length) return false;
      // Deep compare the contents, ignoring non-numeric properties.
      // 深度遍历数组的所有属性进行比较
      while (length--) {
        if (!eq(a[length], b[length], aStack, bStack)) return false;
      }
    } else {
      // Deep compare objects.
      var keys = _.keys(a), key;
      length = keys.length;
      // Ensure that both objects contain the same number of properties before comparing deep equality.
      if (_.keys(b).length !== length) return false;
      while (length--) {
        // Deep compare each member
        key = keys[length];
        if (!(_.has(b, key) && eq(a[key], b[key], aStack, bStack))) return false;
      }
    }
    // Remove the first object from the stack of traversed objects.
    aStack.pop();
    bStack.pop();
    return true;
  };

  // Perform a deep comparison to check if two objects are equal.
  _.isEqual = function(a, b) {
    return eq(a, b);
  };

  // Is a given array, string, or object empty?
  // An "empty" object has no enumerable own-properties.
  _.isEmpty = function(obj) {
    // 如果obj为null，直接判断为空
    if (obj == null) return true;
    // 有length属性的使用length属性进行判断
    if (isArrayLike(obj) && (_.isArray(obj) || _.isString(obj) || _.isArguments(obj))) return obj.length === 0;
    // 如果是对象之类的类型，那么获取对象的所有key的集合，根据key的数量判断
    return _.keys(obj).length === 0;
  };

  // Is a given value a DOM element?
  _.isElement = function(obj) {
    // obj不为空且obj的node类型为1
    return !!(obj && obj.nodeType === 1);
  };

  // Is a given value an array?
  // Delegates to ECMA5's native Array.isArray
  // 如果本地判断数组方法可用就用本地的，否则使用自己创建的
  _.isArray = nativeIsArray || function(obj) {
    return toString.call(obj) === '[object Array]';
  };

  // Is a given variable an object?
  // function类型也算是object
  _.isObject = function(obj) {
    var type = typeof obj;
    return type === 'function' || type === 'object' && !!obj;
  };

  // Add some isType methods: isArguments, isFunction, isString, isNumber, isDate, isRegExp, isError.
  // 判断各种类型的方法
  _.each(['Arguments', 'Function', 'String', 'Number', 'Date', 'RegExp', 'Error'], function(name) {
    _['is' + name] = function(obj) {
      return toString.call(obj) === '[object ' + name + ']';
    };
  });

  // Define a fallback version of the method in browsers (ahem, IE < 9), where
  // there isn't any inspectable "Arguments" type.
  // IE9-对判断arguments的特殊判断方法
  if (!_.isArguments(arguments)) {
    _.isArguments = function(obj) {
      return _.has(obj, 'callee');
    };
  }

  // Optimize `isFunction` if appropriate. Work around some typeof bugs in old v8,
  // IE 11 (#1621), Safari 8 (#1929), and PhantomJS (#2236).
  var nodelist = root.document && root.document.childNodes;
  if (typeof /./ != 'function' && typeof Int8Array != 'object' && typeof nodelist != 'function') {
    _.isFunction = function(obj) {
      return typeof obj == 'function' || false;
    };
  }

  // Is a given object a finite number?
  // obj是否一个有限数字
  _.isFinite = function(obj) {
    return isFinite(obj) && !isNaN(parseFloat(obj));// 原版的isFinite会把null当做Finite
  };

  // Is the given value `NaN`?
  _.isNaN = function(obj) {
    return _.isNumber(obj) && isNaN(obj);
  };

  // Is a given value a boolean?
  _.isBoolean = function(obj) {
    return obj === true || obj === false || toString.call(obj) === '[object Boolean]';
  };

  // Is a given value equal to null?
  _.isNull = function(obj) {
    return obj === null;
  };

  // Is a given variable undefined?
  _.isUndefined = function(obj) {
    return obj === void 0;
  };

  // Shortcut function for checking if an object has a given property directly
  // on itself (in other words, not on a prototype).
  // obj是否有key属性（不包括原型中的）
  _.has = function(obj, key) {
    return obj != null && hasOwnProperty.call(obj, key);
  };

  // Utility Functions
  // -----------------

  // Run Underscore.js in *noConflict* mode, returning the `_` variable to its
  // previous owner. Returns a reference to the Underscore object.
  // 重新设置underscore的引用对象，即可以让_变为你想要的值
  _.noConflict = function() {
    // 将最原始的_变量变回之前状态
    root._ = previousUnderscore;
    return this;
  };

  // Keep the identity function around for default iteratees.
  _.identity = function(value) {
    return value;
  };

  // Predicate-generating functions. Often useful outside of Underscore.
  _.constant = function(value) {
    return function() {
      return value;
    };
  };

  _.noop = function(){};

  _.property = property;

  // Generates a function for a given object that returns a given property.
  // 返回一个获取之前预定好的obj的key的值的函数
  _.propertyOf = function(obj) {
    return obj == null ? function(){} : function(key) {
      return obj[key];
    };
  };

  // Returns a predicate for checking whether an object has a given set of
  // `key:value` pairs.
  _.matcher = _.matches = function(attrs) {
    attrs = _.extendOwn({}, attrs);
    return function(obj) {
      return _.isMatch(obj, attrs);
    };
  };

  // Run a function **n** times.
  // 每一次调用iteratee传递index参数。生成一个返回值的数组。
  _.times = function(n, iteratee, context) {
    var accum = Array(Math.max(0, n));
    iteratee = optimizeCb(iteratee, context, 1);
    for (var i = 0; i < n; i++) accum[i] = iteratee(i);
    return accum;
  };

  // Return a random integer between min and max (inclusive).
  _.random = function(min, max) {
    if (max == null) {
      max = min;
      min = 0;
    }
    return min + Math.floor(Math.random() * (max - min + 1));
  };

  // A (possibly faster) way to get the current timestamp as an integer.
  _.now = Date.now || function() {
    return new Date().getTime();
  };

   // List of HTML entities for escaping.
  var escapeMap = {
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#x27;',
    '`': '&#x60;'
  };
  var unescapeMap = _.invert(escapeMap);

  // Functions for escaping and unescaping strings to/from HTML interpolation.
  var createEscaper = function(map) {
    var escaper = function(match) {
      return map[match];
    };
    // Regexes for identifying a key that needs to be escaped
    var source = '(?:' + _.keys(map).join('|') + ')';
    var testRegexp = RegExp(source);
    var replaceRegexp = RegExp(source, 'g');
    return function(string) {
      string = string == null ? '' : '' + string;
      return testRegexp.test(string) ? string.replace(replaceRegexp, escaper) : string;
    };
  };
  _.escape = createEscaper(escapeMap);
  _.unescape = createEscaper(unescapeMap);

  // If the value of the named `property` is a function then invoke it with the
  // `object` as context; otherwise, return it.
  _.result = function(object, prop, fallback) {
    var value = object == null ? void 0 : object[prop];
    if (value === void 0) {
      value = fallback;
    }
    return _.isFunction(value) ? value.call(object) : value;
  };

  // Generate a unique integer id (unique within the entire client session).
  // Useful for temporary DOM ids.
  var idCounter = 0;
  _.uniqueId = function(prefix) {
    var id = ++idCounter + '';
    return prefix ? prefix + id : id;
  };

  // By default, Underscore uses ERB-style template delimiters, change the
  // following template settings to use alternative delimiters.
  _.templateSettings = {
    evaluate: /<%([\s\S]+?)%>/g,
    interpolate: /<%=([\s\S]+?)%>/g,
    escape: /<%-([\s\S]+?)%>/g
  };

  // When customizing `templateSettings`, if you don't want to define an
  // interpolation, evaluation or escaping regex, we need one that is
  // guaranteed not to match.
  var noMatch = /(.)^/;

  // Certain characters need to be escaped so that they can be put into a
  // string literal.
  var escapes = {
    "'": "'",
    '\\': '\\',
    '\r': 'r',
    '\n': 'n',
    '\u2028': 'u2028',
    '\u2029': 'u2029'
  };

  var escapeRegExp = /\\|'|\r|\n|\u2028|\u2029/g;

  var escapeChar = function(match) {
    return '\\' + escapes[match];
  };

  // JavaScript micro-templating, similar to John Resig's implementation.
  // Underscore templating handles arbitrary delimiters, preserves whitespace,
  // and correctly escapes quotes within interpolated code.
  // NB: `oldSettings` only exists for backwards compatibility.
  _.template = function(text, settings, oldSettings) {
    if (!settings && oldSettings) settings = oldSettings;
    settings = _.defaults({}, settings, _.templateSettings);

    // Combine delimiters into one regular expression via alternation.
    var matcher = RegExp([
      (settings.escape || noMatch).source,
      (settings.interpolate || noMatch).source,
      (settings.evaluate || noMatch).source
    ].join('|') + '|$', 'g');

    // Compile the template source, escaping string literals appropriately.
    var index = 0;
    var source = "__p+='";
    text.replace(matcher, function(match, escape, interpolate, evaluate, offset) {
      source += text.slice(index, offset).replace(escapeRegExp, escapeChar);
      index = offset + match.length;

      if (escape) {
        source += "'+\n((__t=(" + escape + "))==null?'':_.escape(__t))+\n'";
      } else if (interpolate) {
        source += "'+\n((__t=(" + interpolate + "))==null?'':__t)+\n'";
      } else if (evaluate) {
        source += "';\n" + evaluate + "\n__p+='";
      }

      // Adobe VMs need the match returned to produce the correct offset.
      return match;
    });
    source += "';\n";

    // If a variable is not specified, place data values in local scope.
    if (!settings.variable) source = 'with(obj||{}){\n' + source + '}\n';

    source = "var __t,__p='',__j=Array.prototype.join," +
      "print=function(){__p+=__j.call(arguments,'');};\n" +
      source + 'return __p;\n';

    var render;
    try {
      render = new Function(settings.variable || 'obj', '_', source);
    } catch (e) {
      e.source = source;
      throw e;
    }

    var template = function(data) {
      return render.call(this, data, _);
    };

    // Provide the compiled source as a convenience for precompilation.
    var argument = settings.variable || 'obj';
    template.source = 'function(' + argument + '){\n' + source + '}';

    return template;
  };

  // Add a "chain" function. Start chaining a wrapped Underscore object.
  _.chain = function(obj) {
    var instance = _(obj);
    instance._chain = true;
    return instance;
  };

  // OOP
  // ---------------
  // If Underscore is called as a function, it returns a wrapped object that
  // can be used OO-style. This wrapper holds altered versions of all the
  // underscore functions. Wrapped objects may be chained.
  // Helper function to continue chaining intermediate results.
  var chainResult = function(instance, obj) {
    // 如果依旧可以链式调用，那么调用underscore原型中的chain，不是_.chain()，请注意
    // 所以这里的obj是一个运算结果，然后这个结果被新建了一个_实例，且wrapped对应这个结果
    // 然后调用chain又将这个结果传进去，以此来达到链式调用的效果
    console.log(obj)
    return instance._chain ? _(obj).chain() : obj;
  };

  // Add your own custom functions to the Underscore object.
  _.mixin = function(obj) {
    _.each(_.functions(obj), function(name) {
      var func = _[name] = obj[name];
      _.prototype[name] = function() {
        var args = [this._wrapped];
        // args是获取到的最开始的数据
        // arguments是在链式调用中传入的数据
        // 拼接起来，即args始终是第一个
        push.apply(args, arguments);
        return chainResult(this, func.apply(_, args));
      };
    });
  };

  // Add all of the Underscore functions to the wrapper object.
  _.mixin(_);

  // Add all mutator Array functions to the wrapper.
  _.each(['pop', 'push', 'reverse', 'shift', 'sort', 'splice', 'unshift'], function(name) {
    var method = ArrayProto[name];
    _.prototype[name] = function() {
      var obj = this._wrapped;
      method.apply(obj, arguments);
      if ((name === 'shift' || name === 'splice') && obj.length === 0) delete obj[0];
      return chainResult(this, obj);
    };
  });

  // Add all accessor Array functions to the wrapper.
  _.each(['concat', 'join', 'slice'], function(name) {
    var method = ArrayProto[name];
    _.prototype[name] = function() {
      return chainResult(this, method.apply(this._wrapped, arguments));
    };
  });

  // Extracts the result from a wrapped and chained object.
  _.prototype.value = function() {
    return this._wrapped;
  };

  // Provide unwrapping proxy for some methods used in engine operations
  // such as arithmetic and JSON stringification.
  _.prototype.valueOf = _.prototype.toJSON = _.prototype.value;

  _.prototype.toString = function() {
    return '' + this._wrapped;
  };

  // AMD registration happens at the end for compatibility with AMD loaders
  // that may not enforce next-turn semantics on modules. Even though general
  // practice for AMD registration is to be anonymous, underscore registers
  // as a named module because, like jQuery, it is a base library that is
  // popular enough to be bundled in a third party lib, but not be part of
  // an AMD load request. Those cases could generate an error when an
  // anonymous define() is called outside of a loader request.
  if (typeof define == 'function' && define.amd) {
    define('underscore', [], function() {
      return _;
    });
  }
}());
```
