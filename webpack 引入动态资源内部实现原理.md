# webpack 引入动态资源内部实现原理


<a name="1e125dba"></a>
## 一. 背景
我们在开发过程中，有一些业务场景需要我们动态去引入资源。比如我们现在想要遍历一个数组引入数组里所有的 js 文件，我们可以这样写

```javascript
const indexArr = [1, 2, 3, 4];
indexArr.map(index => {
  // 动态引入 test1、test2、test3、test4 四个 js 文件
  return require(`test${index}.js`);
});
```

那么这种写法， webpack 是如何去解析编译的呢？

<a name="65a29ebe"></a>
## 二. 流程
webpack 的整体流程可以用一张图来概括<br />![webpack流程图.jpg](https://intranetproxy.alipay.com/skylark/lark/0/2019/jpeg/20513/1548236137854-b95948bb-665e-4e35-b01e-ac690b12fbaa.jpeg#align=left&display=inline&height=714&name=webpack%E6%B5%81%E7%A8%8B%E5%9B%BE.jpg&originHeight=4244&originWidth=4436&size=1145836&width=746)

<a name="d1631432"></a>
## 三. 具体实现
从流程图可以看出，编译 `require` 等依赖发生在 `_addModuleChain()` 阶段。所以我们直接看这一块的源码。<br />备注: 这里只展示相关的源码，无关部分不展示

<a name="_addModuleChain"></a>
#### _addModuleChain
`_addModuleChain` 方法会 build 入口模块文件，build 完之后执行 `processModuleDependencies`  开始处理模块依赖
```javascript
// Compilation.js

/**
	*
	* @param {string} context context string path
	* @param {Dependency} dependency dependency used to create Module chain
  * @param {OnModuleCallback} onModule function invoked on modules creation
	* @param {ModuleChainCallback} callback callback for when module chain is complete
	* @returns {void} will throw if dependency instance is not a valid Dependency
	*/
  _addModuleChain(context, dependency, onModule, callback) {
    // ...
    
    const afterBuild = () => {
      if (addModuleResult.dependencies) {
        // 处理模块中的依赖
        this.processModuleDependencies(module, err => {
          if (err) return callback(err);
          callback(null, module);
        });
      } else {
        return callback(null, module);
      }
    };
    
    // 对入口 moudle 创建实例
    moduleFactory.create({
      ...props,
    },
   	(err, module) => {
     	// build 入口模块文件
   	 	this.buildModule(module, false, null, null, err => {
     		afterBuild();
     	}
   	}
  }
                                         
```

<a name="processModuleDependencies"></a>
#### processModuleDependencies
`processModuleDependencies` 主要是处理模块的依赖(具体做了什么处理跟这次的主题不相干，所以不在这里提及)。并执行 `addModuleDependencies` 加入到依赖数组
```javascript
// Compilation.js

/**
	* @param {Module} module to be processed for deps
	* @param {ModuleCallback} callback callback to be triggered
	* @returns {void}
	*/
  processModuleDependencies(module, callback) {
    // ...
      
    // 将模块依赖的 dependencies 加入依赖数组
    this.addModuleDependencies(
      module,
      sortedDependencies,
      this.bail,
      null,
      true,
      callback
    );
  }
```

<a name="addModuleDependencies"></a>
#### addModuleDependencies
`addModuleDependencies` 对依赖的 module 调用 `factory.create` 创建实例，创建过程需要留意，详见 `ContextModuleFactory.js` 。创建完成后会执行 `buildModule` 对依赖的 module 进行 build。
```javascript
// Compilation.js

/**
	* @param {Module} module module to add deps to
  * @param {SortedDependency[]} dependencies set of sorted dependencies to iterate through
  * @param {(boolean|null)=} bail whether to bail or not
	* @param {TODO} cacheGroup optional cacheGroup
	* @param {boolean} recursive whether it is recursive traversal
	* @param {function} callback callback for when dependencies are finished being added
	* @returns {void}
	*/
  addModuleDependencies(
    module,
    dependencies,
    bail,
    cacheGroup,
    recursive,
    callback
  ) {
    // 对依赖的 module 创建实例，创建过程需要留意，详见 ContextModuleFactory.js
    factory.create({
      ...props
    },
    (err, dependentModule) => {
      // 对依赖的 module 进行 build
      this.buildModule(
        dependentModule,
        isOptional(),
        module,
        dependencies,
        err => {}
      );
    }
  }
```

<a name="factory.create"></a>
#### factory.create
这里创建了一个 ContextModule 实例，同时我们可以留意到，它给这个实例注册了一个 `resolveDependencies` 方法，这个方法具体做了什么我们后面会说，这里先提及一下。

```javascript
// ContextModuleFactory.js

 create(data, callback) {
   // ...
   this.hooks.afterResolve.callAsync(
     Object.assign(
       {
         ...props,
         resolveDependencies: this.resolveDependencies.bind(this) // 赋予了 ContextModuleFactory 声明的 resolveDependencies 方法
       },
       ...otherProps,
     ),
     (err, result) => {
       // 可以看到，create 最终返回的是一个 ContextModule 实例
       return callback(
         null,
         new ContextModule(result.resolveDependencies, result)
       );
     }
   );
 }
```

<a name="buildModule"></a>
#### buildModule
实例创建完毕后我们会开始对依赖的 module 进行 build。因为我们通过 `factory.create` 创建出来的实例继承 ContextModule，所以这里的 module.build 实际是执行 ContextModule.js 里的 build 方法。详见 ContextModule.js
```javascript
// Compilation.js

/**
	* Builds the module object
	*
	* @param {Module} module module to be built
	* @param {boolean} optional optional flag
	* @param {Module=} origin origin module this module build was requested from
	* @param {Dependency[]=} dependencies optional dependencies from the module to be built
	* @param {TODO} thisCallback the callback
	* @returns {TODO} returns the callback function with results
	*/
  buildModule(module, optional, origin, dependencies, thisCallback) {
    // 因为创建出来的实例继承 ContextModule，所以这里的 module.build 实际是执行 ContextModule.js 里
    // 的 build 方法。详见 ContextModule.js
  	module.build(
      this.options,
    	this,
     	this.resolverFactory.get("normal", module.resolveOptions),
      this.inputFileSystem,
      error => {
      }
    );
  }
```

<a name="module.build"></a>
#### module.build
我们可以看到，build 里执行了 `resolveDependencies` ，也就是我们前面提及的创建实例的时候注册的方法。所以我们现在可以重新回到 ContextModuleFactory.js 看下 resolveDependencies 的具体逻辑
```javascript
// ContextModule.js
class ContextModule extends Module {
  constructor(resolveDependencies, options) {
		this.resolveDependencies = resolveDependencies;
  }
  build(options, compilation, resolver, fs, callback) {
    // ...
    
    // 可以看到，build 里执行了 resolveDependencies，而 resolveDependencies 这个方法，是在创建实例
    // 的时候声明的。所以我们可以从 ContextModuleFactory.js 里看下 resolveDependencies 的具体逻辑
    this.resolveDependencies(fs, this.options, (err, dependencies) => {
      // ...
    }
  }
}
```

<a name="resolveDependencies"></a>
#### resolveDependencies
`resolveDependencies` 其实就只执行了一个函数 `addDirectory` , 而真正处理动态引入文件查询的逻辑就在这里！其实它就是通过一个正则表达式，从入口文件所在的路径开始，递归查询是否有符合正则匹配的文件。有的话则对该文件进行 build。<br />这个正则表达式是怎么来的呢，其实就是根据你动态引入文件时的写法生成的一个正则。比如我们上面的例子，它就会生成一个 `/^\.\/test.*\.js$/`  的正则表达式。
```javascript
// ContextModuleFactory.js

resolveDependencies(fs, options, callback) {
    const cmf = this;
    let resource = options.resource; // 入口文件所在路径
    let resourceQuery = options.resourceQuery;
    let recursive = options.recursive;
    let regExp = options.regExp; // 文件名正则表达式，拿我们上面的例子来说，regRxp 为 /^\.\/test.*\.js$/
    let include = options.include;
    let exclude = options.exclude;
    if (!regExp || !resource) return callback(null, []);

    // 真正处理动态引入文件查询的逻辑就在这里！
    const addDirectory = (directory, callback) => {
 
      // 从入口文件所在目录开始查询
      fs.readdir(directory, (err, files) => {
        if (err) return callback(err);
        files = cmf.hooks.contextModuleFiles.call(files);
        if (!files || files.length === 0) return callback(null, []);
        asyncLib.map(
          files.filter(p => p.indexOf(".") !== 0),
          (segment, callback) => {
            // segment 为当前目录下的文件名，subResource 为文件的具体路径
            const subResource = path.join(directory, segment);

            if (!exclude || !subResource.match(exclude)) {
              fs.stat(subResource, (err, stat) => {
                if (stat.isDirectory()) { // 如果是目录，则递归查询
                  if (!recursive) return callback();
                  addDirectory.call(this, subResource, callback);
                } else if ( // 否则，开始查找匹配的文件
                  stat.isFile() &&
                  (!include || subResource.match(include))
                ) {
                  const obj = {
                    context: resource,
                    request:
                      "." +
                      subResource.substr(resource.length).replace(/\\/g, "/")
                  };

                  this.hooks.alternatives.callAsync(
                    [obj],
                    (err, alternatives) => {
                      if (err) return callback(err);
                      alternatives = alternatives
                        .filter(obj => regExp.test(obj.request)) // 匹配出满足正则的文件名
                        .map(obj => {
                          const dep = new ContextElementDependency(
                            obj.request + resourceQuery,
                            obj.request
                          );
                          dep.optional = true;
                          return dep;
                        });
                      callback(null, alternatives);
                    }
                  );
                } else {
                  callback();
                }
              });
            } else {
              callback();
            }
          },
          (err, result) => {
          	// ...
          }
        );
      });
    };

    addDirectory(resource, callback);
  }
```
<a name="d41d8cd9"></a>
## 
<a name="1ef28f2b"></a>
## 四. 弊端
<a name="46364856"></a>
#### 1. 有可能引入不必要的文件
因为动态引入的文件，是根据正则匹配查找的，只要满足正则匹配的就进行 build。所以比如上面的例子，我把 `indexArr` 改为只有 1 和 2，最终构建出来的文件，也还是包含了 test1、test2、test3、test4 四个文件

```javascript
// 假设我文件目录下存在 test1、test2、test3、test4 四个 js 文件
const indexArr = [1, 2];
indexArr.map(index => {
  return require(`test${index}.js`);
});

// 上面的写法初衷只是想引入 test1、test2 两个文件，但由于目录下存在四个文件，且都满足正则匹配，所以
// 最终构建出来的 js 还是包含了 test1、test2、test3、test4 四个文件
```

<a name="e712711c"></a>
## 五. 参考文献

1. [https://juejin.im/entry/5b0e3eba5188251534379615](https://juejin.im/entry/5b0e3eba5188251534379615)
1. [http://taobaofed.org/blog/2016/09/09/webpack-flow/](http://taobaofed.org/blog/2016/09/09/webpack-flow/)

