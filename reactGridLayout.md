# react-grid-layout 布局系统

## 实验课题

### 高度自适应

> 2024/03/20

在 react-grid-layout 的布局是采用栅栏布局，*列*指定等分列数，*行*指定 px 像素。对于其中的 div 只需要指定 x、y、w、h 即可，x 和 y 表示左上角点的坐标，w 和 h 表示在布局下占多少“单位列”和“单位行”。

因为*列*是根据页面总宽度等分而成的，所以其本身具有随着页面大小的变化自动伸缩的能力。*行*是直接指定像素高，所以其不会随着页面总高度的变化而变化。

如果希望行也能根据可视区域高度而自适应，那么有两种解决方案：第一种是实时调整单位行高；第二种是实时调整占据行数。具体如何选择主要需要看是否需要同时存在*定高*和*自适应高*两种。如果是，则只能选择方案二。反之两个方案即可。

按照方案一，行与列的管理方式都是一致的，都是指定需要划分多少个栅格，代码如下。这样设计的特点有：1、当屏幕的高度较大时，图也会较大；2、数据存储上无新增；

```jsx
// 方案一 实时调整单位行高
function calcRowHeight(totalHeight: number, splitNum: number) {
  const minRowHeight = 10;

  if (totalHeight == 0) {
    return minRowHeight;
  }

  const latestRowHeight = totalHeight / splitNum;

  if (latestRowHeight < minRowHeight) {
    return minRowHeight;
  }

  return latestRowHeight;
}

// React.FC Inner
const divRef = React.useRef<HTMLDivElement>(null);
const [vheight, setVheight] = useState<number>(0);

useEffect(() => {
  if (!divRef.current) return;

  const resizeObserver = new ResizeObserver(() => {
    if (!divRef.current) return;

    const lh = divRef.current.offsetHeight;
    setVheight(lh);
  });

  resizeObserver.observe(divRef.current);

  return () => {
    resizeObserver.disconnect();
  };
}, []);

const rowHeight = calcRowHeight(vheight, 40);

return (
  <div ref={divRef}>
    <ResponsiveGridLayout rowHeight={rowHeight}>
      {/* content */}
    </ResponsiveGridLayout>
  </div>
)
```

按照方案二，假设单位行高为 20px ，初始布局是在可视区域高度为 919px 下设定的值。如果是自定义高度，则会等按照目前的高度等比例的进行伸缩。这样设计的特点是：1、卡片可以单独配置是否需要自适应高度；2、全局需要保存多一个字段设计布局时的可视区域高度。

```jsx
// 方案二 实时调整占据行数
interface CardProps {
  i: string;
  x: number;
  y: number;
  w: number;
  h: number;
  fixHeight: boolean;
}

interface WorkspaceProps {
  viewHeight: number;
  cards: CardProps[];
}

function transformGL(wsprops: WorkspaceProps, actualHeight: number) {
  const { viewHeight, cards } = wsprops;

  const layout = cards.map((c) => {
    const { i, x, y, w, h } = c;

    let rh = h;
    if (!c.fixHeight) {
      rh = (actualHeight * h) / viewHeight;
    }

    return { i, x, y, w, h: rh } as ReactGridLayout.Layout;
  });

  return layout;
}

// React.FC Inner
const wsprops: WorkspaceProps = {
  viewHeight: 919,
  cards: [
    { i: "1", x: 0, y: 0, w: 60, h: 18, fixHeight: true },
    { i: "2", x: 60, y: 0, w: 60, h: 18, fixHeight: false },
    { i: "3", x: 0, y: 18, w: 60, h: 18, fixHeight: false },
    { i: "4", x: 60, y: 18, w: 60, h: 18, fixHeight: true },
  ],
};

const divRef = React.useRef<HTMLDivElement>(null);
const [layout, setLayout] = useState<ReactGridLayout.Layout[]>([]);
const [vheight, setVheight] = useState<number>(0);

useEffect(() => {
  if (!divRef.current) return;

  const resizeObserver = new ResizeObserver(() => {
    // 在这里处理屏幕大小变化的逻辑
    if (!divRef.current) return;

    const lh = divRef.current.offsetHeight;
    console.log(lh);
    setVheight(lh);
  });

  resizeObserver.observe(divRef.current);

  return () => {
    resizeObserver.disconnect();
  };
}, []);

useEffect(() => {
  const relayout = transformGL(wsprops, vheight);
  setLayout(relayout);
}, [vheight]);

return (
  <div ref={divRef}>
    <ResponsiveGridLayout rowHeight={20} layouts={{ lg: layout }}>
      <!-- content -->
    </ResponsiveGridLayout>
  </div>
)
```

因为有设定最小高度，当浏览器可视窗口高度持续缩小时，必然会出现垂直滚动条。此时，因为垂直滚动条的出现占据了一部分的宽度，导致也会同步出现水平滚动条，水平滚动条实际使用上不好用，这就需要将实际展示数据的宽度减少掉垂直滚动条的宽度（默认 17px ）。代码如下。

```jsx
// React.FC Inner
const divRef = React.useRef<HTMLDivElement>(null);
const glRef = React.useRef<HTMLDivElement>(null);
const [hasScroll, setHasScroll] = useState<boolean>(false);

useEffect(() => {
  if (!divRef.current) return;

  const resizeObserver = new ResizeObserver(() => {
    if (!divRef.current || !glRef.current) return;

    const ele = divRef.current;
    const glele = glRef.current;

    const lhs = glele.offsetHeight > ele.offsetHeight;
    setHasScroll(lhs);
  });

  resizeObserver.observe(divRef.current);

  return () => {
    resizeObserver.disconnect();
  };
}, []);

const clsname = useEmotionCss(() => {
  return {
    height: "100vh",
    width: hasScroll ? "calc(100vw - 17px)" : "100vw",
  };
});

return (
  <div className={clsname} ref={divRef}>
    <div ref={glRef}>
      <ResponsiveGridLayout />
    </div>
  </div>
)
```

### 全屏展示

> 2024/03/20

按 F11 全屏展示可以将 windows 底部导航栏和浏览器顶部导航栏部分也变成可视区域。Chrome浏览器中默认点击 F11 进入/退出全屏的时间没办法监听到。需要在代码中添加监听 F11 按键的 keydown 事件。这样才能被 fullscreenchange 事件监听到，代码如下。

```jsx
// React.FC Inner
const [fullscreen, setFullscreen] = useState<boolean>(false);

useEffect(() => {
  if (!divRef.current) return;

  const handleFullscreenChange = () => {
    const isFs = document.fullscreenElement !== null;
    setFullscreen(isFs);
  };

  const handleKeydown = (event: any) => {
    if (!divRef.current) return;

    // F11键的keyCode为122
    if (event.keyCode === 122) {
      event.preventDefault();
      divRef.current.requestFullscreen();
    }
  };

  addEventListener("fullscreenchange", handleFullscreenChange);
  addEventListener("keydown", handleKeydown);

  return () => {
    removeEventListener("fullscreenchange", handleFullscreenChange);
    removeEventListener("keydown", handleKeydown);
  };
}, []);

return (
  <div ref={divRef}>
    {/* content */}
  </div>
)
```

todo 全屏状态下如何布局问题
todo 如果在“变形”页面上进入编辑模式，需要考虑如何展示
todo 小窗口变换处理

---


在 Chrome 浏览器中在*非全屏状态*下可视区域高度**始终**小于在*全屏状态*下的可视区域高度。浏览器最大能拉伸的高度也不会超过全屏状态下的可视区域高度。在 Edge 浏览器中也是如此。
