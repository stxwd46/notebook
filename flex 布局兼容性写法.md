# flex 布局兼容性写法


```css
.parent {
    display: -webkit-box;
    display: box;
    display: -webkit-flex;
    display: flex;
    .child {
        -webkit-box-flex: 1;
        flex: 1;
    }
}
```

