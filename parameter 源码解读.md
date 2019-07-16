# parameter 源码解读

<a name="80lrxi"></a>
## [](#80lrxi)一. parameter 是什么？
[parameter](https://github.com/node-modules/parameter) 是一个 基于 node 的参数校验工具，当我们获取到用户请求的参数后，不可避免的要对参数进行一些校验。这时候我们就可以使用 parameter 进行参数校验。egg 的 [Validate](https://github.com/eggjs/egg-validate) 插件也是基于 parameter 实现的。

<a name="etyqrp"></a>
## [](#etyqrp)二. 源码解读

<a name="bfunyr"></a>
### [](#bfunyr)前提
假设我们定制的规则是
```javascript
// rules
const rules = {
  aaa: 'int',
  bbb: [1,2,3],
  ccc: /\/d+/,
  ddd: 'string?',
  eee: {
    type: 'string',
    required: false,
  }
}
```

假设我们校验的参数对象是
```javascript
{
  aaa: 1,
  bbb: 2,
  ccc: 3,
  ddd: '4',
  eee: '5',
}
```
<a name="gf2uze"></a>
### [](#gf2uze)valiedate
 `valiedate` 是最核心的 api，我们就是通过这个 api 来进行参数校验

```javascript
/**
  * validate
  *
  * @param {Object} rules - 我们制定的校验规则
  * @return {Object} obj - 需要校验的参数对象
  * @api public
  */
  validate(rules, obj) {
    // 如果校验规则不是一个对象，则抛错
    if (typeof rules !== 'object') {
      throw new TypeError('need object type rule');
    }

    // 如果设置了 validateRoot 为 true(即传递进来的参数必须是对象)，
    // 但真正传递进来的参数是其他的数据类型，则抛错
    if (this.validateRoot && (typeof obj !== 'object' || !obj)) {
      return [{
        message: this.t('the validated value should be a object'),
        code: this.t('invalid'),
        field: undefined,
      }];
    }

    var self = this;

    var errors = [];

    // 遍历校验规则
    for (var key in rules) {      
      var rule = formatRule(rules[key]); // 对规则格式化，具体格式化逻辑详见下面 formatRule 的源码解读
      var value = obj[key]; // 要校验的参数的值
      var has = value !== null && value !== undefined; // 检查下我们校验规则里要校验的参数，在参数对象里是否存在

      if (!has) { // 如果不存在
        if (rule.required !== false) { // 但校验规则里又设置该参数为必填项，则抛错
          errors.push({
            message: this.t('required'),
            field: key,
            code: this.t('missing_field')
          });
        }
        continue;
      }

      var checker = TYPE_MAP[rule.type]; // 从类型映射表里找到对应的校验方法
      if (!checker) { // 如果不存在这种数据类型的校验方法，则抛错 
        throw new TypeError('rule type must be one of ' + Object.keys(TYPE_MAP).join(', ') +
          ', but the following type was passed: ' + rule.type);
      }

      
      convert(rule, obj, key, this.convert); // 对参数值进行数据类型转化，具体转化逻辑详见下面 convert 的源码解读
      var msg = checker.call(self, rule, obj[key], obj); // 进行参数校验并返回错误信息
      if (typeof msg === 'string') {
        errors.push({
          message: msg,
          code: this.t('invalid'),
          field: key
        });
      }

      if (Array.isArray(msg)) {
        msg.forEach(function (e) {
          var dot = rule.type === 'object' ? '.' : '';
          e.field = key + dot + e.field;
          errors.push(e);
        });
      }
    }

    if (errors.length) {
      return errors;
    }
  }
};
```

<a name="8vrzyo"></a>
### [](#8vrzyo)formarRule
`formatRule` 是对校验规则进行格式化，因为 parameter 提供了简写的方法， 所以导致传进来的规则数据类型可以有多种情况。

```javascript
/**
 * format a rule
 * - resolve abbr
 * - resolve `?`
 * - resolve default convertType
 *
 * @param {Mixed} rule
 * @return {Object}
 * @api private
 */

function formatRule(rule) {
  rule = rule || {};
  if (typeof rule === 'string') {
    // 我们的参数 aaa, rule 为 'int', 数据类型为 string，
    // 所以格式化为 { type: 'int' }     
    rule = { type: rule };
  } else if (Array.isArray(rule)) {
    // 我们的参数 bbb, rule 为 [1,2,3], 数据类型为数组，
    // 所以格式化为 { type: 'enum', values: [1,2,3] }
    rule = { type: 'enum', values: rule };
  } else if (rule instanceof RegExp) {
    // 我们的参数 ccc, rule 为 /\/d+/, 数据类型为正则表达式，
    // 所以格式化为 { type: 'string', values: [1,2,3] }
    rule = { type: 'string', format: /\/d+/ };
  }

  // 我们的参数 ddd , rule 为 'string?', 最后面带了个问号，代表这个参数是非必填的，
  // 所以格式化为 { type: 'string', required: false },
  // 也就是我们 eee 的数据结构，即规范的结构体
  if (rule.type && rule.type[rule.type.length - 1] === '?') {
    rule.type = rule.type.slice(0, -1);
    rule.required = false;
  }

  // 如果传进来的 rule 已经是规范的结构体， 则直接返回
  // 如果传进来的 rule 不是规范的结构体，额格式化之后再返回
  return rule;
}
```

<a name="p52kab"></a>
### [](#p52kab)convert
```javascript
/**
 * convert param to specific type
 * @param {Object} rule - 参数的校验规则
 * @param {Object} obj - 参数对象
 * @param {String} key - 参数名
 * @param {Boolean} defaultConvert - 全局 convert 参数
 */
function convert(rule, obj, key, defaultConvert) {
  var convertType;
  // 如果对全局参数 convert 设置为 true，则将参数值都转为 CONVERT_MAP 映射表里对应的数据类型
  if (defaultConvert) convertType = CONVERT_MAP[rule.type];
  // 如果有单独设置某个校验字段的 convertType，则将该字段的数据类型转为 convertType 指定的数据类型
  if (rule.convertType) convertType = rule.convertType;
  // 如果全局 convert 和单独字段的 convertType 都没有指定，则直接返回，不做处理
  if (!convertType) return;

  const value = obj[key];
  // convert type only work for primitive data
  // 参数数据类型是对象的话则不做转化
  if (typeof value === 'object') return;

  // convertType support function
  if (typeof convertType === 'function') {
    obj[key] = convertType(value, obj);
    return;
  }

  switch (convertType) {
    case 'int':
      obj[key] = parseInt(value, 10);
      break;
    case 'string':
      obj[key] = String(value);
      break;
    case 'number':
      obj[key] = Number(obj[key]);
      break;
    case 'bool':
    case 'boolean':
      obj[key] = !!value;
      break;
  }
}
```

<a name="8dbseu"></a>
### [](#8dbseu)TYPE_MAP
一个映射表，不同的数据类型，通过不同的校验方法进行参数校验。<br />这些参数校验方法就不一一详解了，就是判断参数符不符合对应的数据类型。

```javascript
/**
 * Simple type map
 * @type {Object}
 */
var TYPE_MAP = Parameter.TYPE_MAP = {
  number: checkNumber,
  int: checkInt,
  integer: checkInt,
  string: checkString,
  id: checkId,
  date: checkDate,
  dateTime: checkDateTime,
  datetime: checkDateTime,
  boolean: checkBoolean,
  bool: checkBoolean,
  array: checkArray,
  object: checkObject,
  enum: checkEnum,
  email: checkEmail,
  password: checkPassword,
  url: checkUrl,
};
```

<a name="6ei0by"></a>
### [](#6ei0by)CONVERT_MAP
数据类型转化映射表。如果将全局参数 convert 设为 true，则会将参数的数据类型转为映射表里对应的数据类型。

```javascript
var CONVERT_MAP = Parameter.CONVERT_MAP = {
  number: 'number',
  int: 'int',
  integer: 'int',
  string: 'string',
  id: 'string',
  date: 'string',
  dateTime: 'string',
  datetime: 'string',
  boolean: 'bool',
  bool: 'bool',
  email: 'string',
  password: 'string',
  url: 'string',
};
```

<a name="95wlta"></a>
## [](#95wlta)三. 不足之处
parmater 基本满足了验参该有的需求，数据类型、是否必穿这些规则都可以很方便的设定，但使用过程中也发现了一些不足之处，也正是因为有些地方使用不方便，从文档看也确实不支持，所以才来看源码，从源码来看是否有优化空间。

<a name="u8vsly"></a>
#### [](#u8vsly)错误信息无法自定义
从源码来看，校验后如果不通过，只会返回两种错误信息。<br />一种是必传参没传的
```javascript
/**
 @example
 // return {
 //  code: 'missing_field',
 //  field: 'aaa',
 //  message: 'required'
 // }
*/
errors.push({
  message: this.t('required'),
  field: key,
  code: this.t('missing_field')
});
```

另一种是数据类型错误的
```javascript
/**
 @example
 // return {
 //  code: 'invalid',
 //  field: 'aaa',
 //  message: 'should be an integer'
 // }
*/
errors.push({
  message: msg,
  code: this.t('invalid'),
  field: key
});
```

这样的错误信息非常有局限性，比如第一种情况，我想在前端展示 「xxx 为必传参」这个错误信息的话，我还得自己在 node 中转一层。对于第二种情况，如果我想展示「xxx 必须为整数」这种错误信息的话，甚至都很难去中转。而这些，我觉得只要对每个参数加多一个 errorMsg 字段让用户自定义错误信息就可以解决了。以第一种情况为例
```javascript
/**
 假设传进来的 errorMsg 为 「aaa 是必传参」
 @example
 // return {
 //  code: 'missing_field',
 //  field: 'aaa',
 //  message: 'aaa 是必传参'
 // }
*/
errors.push({
  message: this.t(rule.errorMsg || 'required'),
  field: key,
  code: this.t('missing_field')
});
```

以上，为我使用 `parameter` 后的一些感想
