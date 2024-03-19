# JS 的 setInterval 实验测试

setInterval 是 JavaScript 中的定时器，它主要接收两个参数，第一个参数是一个函数 func，第二个参数是一个毫秒级的间隔时间 ms。该定时器就会每隔 ms 执行一次 func。

通过实验测试，发现 js 是单线程的，存在多个 setInterval 时，定时器执行频率会不符合预期。

## 实验过程

在 index.tsx 中调用了三次 CardView 组件，而在 CardView 组件挂载时设置一个定时器，代码如下。根据 console 中的打印，每隔 1s 就会有三次 001 打印出来。

```jsx
// components/card.tsx
let tid: number = 0;

const CardView: React.FC = () => {
  useEffect(() => {
    tid = setInterval(() => {
      console.log("001");
    }, 1000) as unknown as number;

    return () => {
      if (tid) {
        clearInterval(tid);
        tid = 0;
      }
    };
  }, []);

  return <>card</>;
};


// index.tsx
const TrialView: React.FC = () => {
  return (
    <>
      <CardView />
      <CardView />
      <CardView />
    </>
  );
};
```

在 index.tsx 中调用三次 CardView 组件，而在 CardView 组件睡眠 3s 后才进行渲染，代码如下。根据 console 中的打印，每个 CardView 都需要 3s 进行渲染，单个 CardView 不会进行先渲染，而是等待三个组件都渲染完成之后，页面才显示（9s）。

```jsx
// components/card.tsx
function sleep(sec: number) {
  const end = dayjs().add(sec, "milliseconds");

  while (true) {
    if (dayjs().unix() > end.unix()) return;
  }
}

const CardView: React.FC = () => {
  console.log("--1");
  sleep(3000);
  console.log("--2");

  return <>card</>;
};

// index.tsx
<>
  <CardView />
  <CardView />
  <CardView />
</>;
```

CardView 组件接收一个参数 ms，用于设置睡眠时间，而在 index.tsx 中调用三次该组件，并且对 ms 参数分别设置为 1s, 2s, 3s，代码如下。页面在 6s 时后才渲染完成，每个 CardView 组件也在设定的时间睡眠后完成组件的渲染。

```jsx
// components/card.tsx
const CardView: React.FC<{ ms: number }> = (props) => {
  const { ms } = props;

  console.log("--1");
  sleep(ms);
  console.log("--2");

  return <>card</>;
};

// index.tsx
<>
  <CardView ms={1000} />
  <CardView ms={2000} />
  <CardView ms={3000} />
</>;
```

在 CardView 组件设置一个定时器，并且定时器的时间固定为 1s，在定时器执行的函数中会有一个 sleep 函数，sleep 的时间在 props.ms 进行指定。在 index.tsx 中调用三次该组件，并且 ms 参数分别设置为 1s, 2s, 3s，代码如下。根据 console 的打印结果为，先完整执行完 CardView(cid=a) 的 setInterval 函数后 ，再完整执行完 CardView(cid=b) 的 setInterval 后，再完整执行完 CardView(cid=c) 的 setInterval 函数。随后重复这一过程。而三个 CardView 中的 sleep 是正常的，都是 sleep 了指定的时间后才继续下一步，详细过程如下。

```jsx
// components/card.tsx
const { ms, cid } = props;

useEffect(() => {
  tid = setInterval(() => {
    console.log("--1", cid);
    sleep(ms);
    console.log("--2", cid);
  }, 1000) as unknown as number;

  return () => {
    if (tid) {
      clearInterval(tid);
      tid = 0;
    }
  };
}, []);

// index.tsx
<>
  <CardView ms={1000} cid="a" />
  <CardView ms={2000} cid="b" />
  <CardView ms={4000} cid="c" />
</>
```

```text
# 执行过程
--1 a
--2 a （1s 后，打印时间 base + 2s）
--1 b
--2 b （2s 后，打印时间 base + 4s）
--1 c
--2 c （3s 后，打印时间 base + 6s）
... （循环上述过程）
```

在 CardView 组件中设定一个定时器，时间间隔由 props.ms 指定，而在定时执行的函数中会 sleep 1s。在 index.tsx 中调用三次该组件，并且 ms 参数分别设定为 1s, 2s, 4s ，代码如下。根据 console 的打印，在初始的循环中还会有设定的 1s, 2s, 4s 的打印规律，到后面就变成每间隔 1s 打印一条 `--2 cid` ，其中的 cid 的规律就是 a、b、c 如此，但是仍然是完成执行完一个组件的定时器函数后才会执行下一个函数，执行过程如下。

```jsx
// components/card.tsx
const { ms, cid } = props;

useEffect(() => {
  tid = setInterval(() => {
    console.log("--1", cid);
    sleep(1000);
    console.log("--2", cid);
  }, ms) as unknown as number;

  return () => {
    if (tid) {
      clearInterval(tid);
      tid = 0;
    }
  };
}, []);

// index.tsx
<>
  <CardView ms={1000} cid="a" />
  <CardView ms={2000} cid="b" />
  <CardView ms={4000} cid="c" />
</>
```

```text
--1 a
--2 a
--1 a
--2 a
--1 b
--2 b
--1 c
--2 c
--1 a
--2 a
--1 b
--2 b
--1 a
--2 a
--1 c
--2 c
... （后续规律即 a、b、c 依次 --1, --2 的循环打印）
```

在 CardView 组件中编写了一个异步的 sleep 函数，并且直接在组件内使用，而在 index.tsx 中调用三次该组件，代码如下。在 console 中打印的结果是按照 a、b、c 的顺序依次完整打印 --1, --3, --2 ，而页面是等所有的打印都完成之后才进行渲染。

如果将 console.log 那段代码放入 useEffect 函数中，则表现为先依次打印 a、b、c 的 --1 和 --3 。接着依次打印 a、b、c 的 --2 日志。页面仍然是等所有的打印都完成之后才进行渲染。

```jsx
// components/card.tsx
async function sleep(sec: number) {
  const end = dayjs().add(sec, "milliseconds");

  while (true) {
    if (dayjs().unix() > end.unix()) return;
  }
}

const { ms, cid } = props;
// console.log 代码段
// == begin ==
console.log("--1", cid);
sleep(ms).then(() => {
  console.log("--2", cid);
});
console.log("--3", cid);
// == end ==

// index.tsx
<>
  <CardView ms={1000} cid="a" />
  <CardView ms={2000} cid="b" />
  <CardView ms={4000} cid="c" />
</>;
```

---

在开发卡片集市中的自定义指数卡片中，因为该卡片需要的分钟级行情数据来自于 kLineDs ，前端 js 会利用 setInterval 在每分钟的开始时刻调用 kLineDs 提供的 rpc 接口去获取数据。将获取回来的数据进行处理后在卡片中进行展示。而此时有个有趣的现象，在存在多个自定义指数卡片时，只会有一个 setInterval 在获取数据，简化代码如下。

经过 debug 分析，由于 tid 是全局变量，并且在 useEffect 有判断只有当 tid 为 0 时才会设置定时器。所以只有第一个 TestCard 组件才设置了，后续的由于 tid 都不为 0，都直接跳过设置定时器。所以将 tid 的声明放到 useEffect 中即可解决该问题。

```jsx
// components/card.tsx
let tid: number = 0;

const TestCard: React.FC<{ conf: string; cid: string }> = (props) => {
  const { conf, cid } = props;

  useEffect(() => {
    if (conf === undefined) return;

    // 解决只有第一个组件才设置了定时器BUG
    // let tid: number = 0;

    if (!tid) {
      tid = setInterval(() => {
        const curTick = +dayjs().format("ss");
        console.log(curTick, "tick");
      }, 1000) as unknown as number;
    }

    return () => {
      if (tid) {
        clearInterval(tid);
        tid = 0;
      }
    };
  }, [conf]);

  return <>card</>;
};

const CardFactory: React.FC<{ ct: string; t: string; c: string }> = (p) => {
  const { ct, t, c } = p;

  switch (ct) {
    case "test":
      return <TestCard cid={t} conf={c} />;
    default:
      throw new Error(`未知卡片类型: ${ct}`);
  }
};

// index.tsx
const TrialView: React.FC = () => {
  const cards = [
    { title: "a1", conf: "b" },
    { title: "a2", conf: "b" },
    { title: "a3", conf: "b" },
  ];

  return (
    <>
      {cards.map((c) => {
        return <CardFactory ct="test" t={c.title} c={c.conf} />;
      })}
    </>
  );
};
```

由于 js 是单线程的，所以每个自定义指数中的 setInterval 其实类似队列（Queue）一样，理论上的触发一到就往队列放入一个定时器函数。队列里的定时器函数一个一个的进行执行。

如果在卡片里边进行耗时的操作，会导致 setInterval 执行频率与预期的不一致，可能会导致 BUG 的出现。比如在 TestCard 组件的 setInterval 是 1s 执行一次，但在 setInterval 定时函数需要执行 2s ，代码如下。根据 console 的结果，setInterval 实际执行频率为 ≥ 3s （=3s 是在只有一个 TestCard 下；>3s 是在多个 TestCard）。

这样的话，在 setInterval 定时函数中执行情况也会与预期的不一致。比如，在定时函数中，设定在每分钟的开始（:00s ~ :01s）时去发送 grpc 请求去取分钟级 k 线数据，setInterval 的定时频率为 500ms ，可能会错过分钟开始而错误的未请求数据（同步、异步都存在该情况）。<u>如果定时器执行频率远大于定时器函数的耗时，则不会发生此问题。</u>

如果按照该例子的话，能够命中取数逻辑的时间段只有 1s ，setInterval 的频率是 500ms ，只要取数的时间稍微长一点，或者卡片数量多一点，就必然会出现丢失请求的情况发生。

```jsx
// components/card.tsx - TestCard
async function sleep(sec: number) {
  const end = dayjs().add(sec, "milliseconds");

  while (true) {
    if (dayjs().unix() > end.unix()) return;
  }
}

useEffect(() => {
  if (conf === undefined) return;

  let tid: number = 0;

  if (!tid) {
    tid = setInterval(() => {
      const curTick = +dayjs().format("ss");
      console.log(curTick, cid, "tick");
      sleep(2000);
    }, 1000) as unknown as number;
  }

  return () => {
    if (tid) {
      clearInterval(tid);
      tid = 0;
    }
  };
}, [conf]);
```
