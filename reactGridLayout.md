# react-grid-layout 布局系统

react-grid-layout 可以很方便的实现拖拽式布局页面展示。很贴合 DIY 工作台的场景。

使用的版本为 react-grid-layout@1.4.4, @types/react-grid-layout@1.3.5

## 实验课题

### 高度自适应

> 修改日志:   
> - 2024/03/20 高度自适应方案调研      
>

在 react-grid-layout 的布局是采用栅栏布局，*列*指定等分列数，*行*指定 px 像素。对于其中的 div 只需要指定 x、y、w、h 即可，x 和 y 表示左上角点的坐标，w 和 h 表示在布局下占多少“单位列”和“单位行”（这四个值都不一定要整数）。

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

因为有设定最小高度，当浏览器可视窗口高度持续缩小时，必然会出现垂直滚动条。此时，因为垂直滚动条的出现占据了一部分的宽度，导致也会同步出现水平滚动条，水平滚动条可用性不高，这就需要将实际展示数据的宽度减少掉垂直滚动条的宽度（默认 17px ）来去除。代码如下。需要注意是，该滚动条是浏览器的滚动条是浏览器的滚动条。

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

> 2024/03/21

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

在进入全屏状态之前，页面存在三种展示情况。1、页面内容未占据全部可视区域；2、页面内容刚好占据全部可视区域；3、页面内容超出了可视区域。进一步分析为两种情况，即是否出现垂直滚动条。在没有滚动条的页面进入全屏状态时，自动将内容铺满整个可视区域；而如果有滚动条的页面进入全屏状态，则页面内容不进行变化。

铺满全屏的方式有两种：1、按照尺寸等比例的进行拉伸进行铺满；2、按照固定的列数将每个卡片块按照*相同的尺寸*进行平铺（至少有一个列能保证高度占满整块屏）。采用方案一进行铺满全屏需要解决的问题是*如何确定每一个卡片需要放大的比例？*；采用方案二会出现有空白的情况出现（卡片块数量非列数的整数倍时）。个人感觉方案二不会满足业务需求，所以只考虑方案一情况。方案一中卡片的放大比例按照场景可以分为三种情况：1、卡片块数量为列数的整数倍；2、卡片块数量非列数的整数倍时；3、纯搞怪型，无法判断列数。情况3主要出现在测试中，所以如果是该情况，则不进行任何处理直接展示。情况 1 和 2 则按照公式计算扩大。假设 rowHeight 固定不变的情况下，代码如下。每列需要扩大的间距可能会不一致，所以需要按照列去逐个单独计算。不建议使用组件 ResponsiveGridLayout 的 margin 设置边距，会导致计算不准放大的程度，替代方案是卡片块 padding。另外，全屏的伪类是 `:fullscreen::backdrop`。

```jsx
const rowHeight = 28;

function analyzeRowByCol(layout: ReactGridLayout.Layout[]) {
  // n: 总行数 h: 总行高 px
  const colMap: Record<string, { n: number; h: number }> = {};
  const colset = new Set<string>();
  layout.forEach((item) => {
    const { x, w, h } = item;
    const key = `${x}:${x + w}`;
    colset.add(key);
    if (colMap[key] === undefined) {
      colMap[key] = { n: 0, h: 0 };
    }

    colMap[key].n += 1;
    colMap[key].h += h * rowHeight;
  });

  let valid = true;
  const cols = Array.from(colset);

  for (let i = 0; i < cols.length; i++) {
    for (let j = i + 1; j < cols.length; j++) {
      const [bi, ei] = cols[i].split(":");
      const [bj, ej] = cols[j].split(":");

      if (!(ei <= bj || ej <= bi)) {
        valid = false;
      }
    }
  }

  return { valid, colMap };
}

function calcRowExpByCol(vh: number, cMap: Record<string, { n: number; h: number }>) {
  const rMap: Record<string, number> = {};

  let tooMuch = false;
  Object.keys(cMap).forEach((col) => {
    const { n, h } = cMap[col];
    if (vh - h < 0) {
      tooMuch = true;
    }

    const remaingRowNum = (vh - h) / rowHeight;
    // console.log(remaingRowNum, col, "calc row expand by col");
    const rowExp = remaingRowNum / n;
    rMap[col] = rowExp;
  });

  if (tooMuch) { 
    // 如果出现滚动条，则不进行变化
    Object.keys(cMap).forEach((col) => {
      rMap[col] = 0;
    });
  }

  return rMap;
}

function generateFsLayout(ori: ReactGridLayout.Layout[], rMap: Record<string, number>) {
  return ori.map((item) => {
    const { i, x, y, w, h } = item;
    const key = `${x}:${x + w}`;
    const exp = rMap[key];
    const lh = exp + h;

    return { i, x, y, w, h: lh };
  });
}

function renderGLayout(ori: ReactGridLayout.Layout[], fs: boolean, vh: number) {
  const report = analyzeRowByCol(ori);
  // console.log(report.valid, report.colMap, "analyze row number by col");
  if (fs) {
    if (report.valid) {
      const rMap = calcRowExpByCol(vh, report.colMap);
      // console.log(rMap, "calc row expand by col");
      const layout = generateFsLayout(ori, rMap);
      return layout;
    }
  }

  return ori;
}

// React.FC inner
const [vheight, setVheight] = useState<number>(0);
const [fullscreen, setFullscreen] = useState<boolean>(false);
const [layout, setLayout] = useState<ReactGridLayout.Layout[]>(ORI_LAYOUT);

useEffect(() => {
  const nlo = renderGLayout(ORI_LAYOUT, fullscreen, vheight);
  setLayout(nlo);
}, [fullscreen, vheight]);

const clsname = useEmotionCss(() => {
  return {
    height: "100vh",
    width: hasScroll ? "calc(100vw - 17px - 1px)" : "100vw",
    backgroundColor: "#fff7e6",
    overflow: fullscreen ? "auto" : "",

    ".glview": {
      width: "calc(100% - 1px)",
    },
    ".card-wrap": {
      padding: "1px",
    },
    ".card": {
      width: "100%",
      height: "100%",
      backgroundColor: "#ffd8bf",
      padding: "3px",
    },
  };
});

return (
  <div className={clsname} ref={divRef}>
    <div className="glview" ref={glRef}>
      <ResponsiveGridLayout
        margin={[0, 0]}
        rowHeight={rowHeight}
        layouts={{ lg: layout }}
      >
        {layout.map((item) => {
          return (
            <div className="card-wrap" key={item.i}>
              <div className="card">
                {/* content */}
              </div>
            </div>
          );
        })}
      </ResponsiveGridLayout>
    </div>
  </div>
)
```

在全屏状态下要出现滚动条，需要在调用 requestFullscreen 的那个元素块上管理滚动条，但是在非全屏状态下，如果让该元素块上管理滚动条，则会出现两个滚动条（元素块滚动条 + 浏览器滚动条），且元素块滚动条是失效的。所以，只有全屏状态下，才交由元素块维护滚动问题，代码如下。

```jsx
// 全屏调用 divRef 的 requestFullscreen(), fullscreen 是否是全屏状态
const [fullscreen, setFullscreen] = useState<boolean>(false);

const clsname = useEmotionCss(() => {
  return {
    height: "100vh",
    // hasScroll 解决非全屏状态下出现水平滚动条问题，全屏状态失效
    width: hasScroll ? "calc(100vw - 17px)" : "100vw",
    backgroundColor: "#fff7e6",
    overflow: fullscreen ? "auto" : "", // 全屏状态下滚动条方案
  };
});

return (
  <div className={clsname} ref={divRef}>
    {/* content */}
  </div>
)
```


todo 如果在“变形”页面上进入编辑模式，需要考虑如何展示
todo 小窗口变换处理

---


在 Chrome 浏览器中在*非全屏状态*下可视区域高度**始终**小于在*全屏状态*下的可视区域高度。浏览器最大能拉伸的高度也不会超过全屏状态下的可视区域高度。在 Edge 浏览器中也是如此。
