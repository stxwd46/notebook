# jq 和 zepto 中 .attr() 和 .data() 的区别

$.attr() 和 $.data() 本质上属于 DOM 属性和 Jquery 对象属性的区别。

- $.attr()每次都从 `DOM元素` 中取属性的值，即和视图中标签内的属性值保持一致。

  - $.attr('data-foo') 会从标签内获得 data-foo 属性值；
  - $.attr('data-foo', 'world') 会将字符串 'world' 塞到标签的 'data-foo' 属性中；
- $.data()是从 `Jquery对象` 中取值，由于对象属性值保存在内存中，因此可能和视图里的属性值不一致的情况。

  - $.data('foo') 会从 `Jquery对象` 内获得foo的属性值，不是从标签内获得 data-foo 属性值；
  - $.data('foo', 'world') 会将字符串 'world' 塞到 `Jquery对象` 的'foo'属性中，而不是塞到视图标签的data-foo属性中。

[]()<a name="5db9fd7c"></a>
#### 小结

所以 $.attr() 和 $.data() 应避免混合用，也就是应该尽量避免如下两种情况的出现：

通过 $.attr() 来进行 set 属性，然后通过 $.data() 进行 get 属性值；<br />
通过 $.data() 来进行 set 属性，然后通过 $.attr() 进行 get 属性值。<br />
同时从性能的角度来说，建议使用 $.data() 来进行 set 和 get 操作，因为它仅仅修改的 Jquey 对象的属性值，不会引起额外的 DOM 操作。
