# 页面内容溢出省略效果

测试环境为 react 18

## 实验过程

### ### 2024/03/19

div 元素单行展示，如下所示。若为 span 元素，需要多设定 display 和元素的 width 。

```jsx
const clsname = useEmotionCss(() => {
  return {
    height: "300px",
    width: "300px",
    backgroundColor: "#ff9c6e",

    // div 元素
    ".view_div": {
      overflow: "hidden",
      whiteSpace: "nowrap",
      textOverflow: "ellipsis",
    },

    // span 元素
    ".view_span": {
      display: "inline-block",
      width: "100%",
      overflow: "hidden",
      whiteSpace: "nowrap",
      textOverflow: "ellipsis",
    },
  };
});

<div className={clsname}>
  <div className="view_div">
    xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  </div>
</div>;
```

div 元素多行展示，超出部分省略展示，代码如下。若为 span 元素也是一样的配置。

```jsx
const clsname = useEmotionCss(() => {
  return {
    height: "300px",
    width: "300px",
    backgroundColor: "#ff9c6e",

    ".view": {
      width: "100%",
      display: "-webkit-box",
      wordBreak: "break-all",
      textOverflow: "ellipsis",
      overflow: "hidden",

      // todo 需测试是否可替代
      "-webkit-line-clamp": "2",
      "-webkit-box-orient": "vertical",
      WebkitLineClamp: "2",
      WebkitBoxOrient: "vertical",
    },
  };
});

<div className={clsname}>
  <div className="view">
    xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  </div>
</div>;
```
