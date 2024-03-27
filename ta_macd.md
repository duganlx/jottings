# MACD

平滑异同平滑平均线（Moving Average Convergence Divergence，MACD）Geral Appel 于 1979 年提出的，它是一项利用短期（常用为 12 日）移动平均线与长期（常用为 26 日）移动平均线之间的聚合与分离状况，对买进、卖出时机做出研判的技术指标。在现有的技术分析软件中，MACD 常用参数是快速平滑移动平均线为 12，慢速平滑移动平均线参数为 26. 此外，MACD 还有一个辅助指标柱状图（Bar）。在大多数期货技术分析软件中，柱状线是有颜色的，在低于 0 轴以下是绿色，高于 0 轴以上是红色，前者代表趋势较弱，后者代表趋势较强。

假设收盘价 `close = {p1, p2, ..., pn}`, 则 macd 计算如下

$$
ema_i(12) = \frac{2}{13} \times p_i + \frac{11}{13} \times ema_{i-1}(12), ema_1=p_1
$$

$$
ema_i(26) = \frac{2}{27} \times p_i + \frac{25}{27} \times ema_{i-1}(26), ema_1=p_1
$$

$$
dif_i = ema_i(12) - ema_i(26)
$$

$$
dea_i = \frac{2}{10} \times dif_i + \frac{8}{10} \times dea_{i-1}, dea_1=dif_1
$$

$$
macd_i = (dif_i - dea_i) \times 2
$$

代码实现

```py
# MACD 代码实现
import pandas as pd
import numpy as np

def calc_macd(close) -> pd.DataFrame:
    """
    计算平滑异同平均线 MACD
    """
    ema12, ema26 = [], []
    for ele in close.array:
        if len(ema12) == 0:
            ema12.append(ele)
            ema26.append(ele)
            continue

        pre_ema12, pre_ema26 = ema12[-1], ema26[-1]
        cur_ema12 = 2/13 * ele + 11/13 * pre_ema12
        cur_ema26 = 2/27 * ele + 25/27 * pre_ema26

        ema12.append(cur_ema12)
        ema26.append(cur_ema26)

    ema12 = pd.Series(ema12, name='EMA12')
    ema26 = pd.Series(ema26, name='EMA26')
    dif = pd.Series(ema12 - ema26, name='DIF')

    dea = []
    for ele in dif.array:
        if len(dea) == 0:
            dea.append(ele)
            continue

        pre_dif = dea[-1]
        cur_dea = 2/10 * ele + 8/10 * pre_dif
        dea.append(cur_dea)

    dea = pd.Series(dea, name='DEA')
    macd = pd.Series((dif - dea) * 2, name='MACD')

    df = pd.concat([dif, dea, macd], axis=1)
    return df

def rolling_df(df, window, func, name):
    """
    对Dataframe按一行为单位进行滚动, 并且生成结果Series
    """
    res = []
    for i in range(len(df)):
        window_df = df.iloc[(i-window+1):i+1, :]

        if window_df.empty:
            res.append(np.nan)
            continue

        this_res = func(window_df)
        res.append(this_res)

    res = pd.Series(res, name=name)
    return res
```

#### 研究

通过 macd 的计算公式，我们可以到 白线 dif 是用 12 日的指数移动平均线减去 26 日的指数移动平均线。黄线 dea 是在 dif 的基础上求了个 9 日的指数移动平均线。利用移动平均线其实就是平滑曲线，所以 dif 会比 dea 反应更加剧烈一点，我习惯用快慢线来描述：dif 快线、dea 慢线。柱状图 macd 则是用快线减去慢线所得到的差值（扩大两倍）。

这么看来，其实 macd 有点类似套娃的过程了。指数移动平均线 EMA 比普通的移动平均线会更加侧重当日的股价变化，dif 用 12 日的 EMA（快线）减去 26 日 EMA（慢线），其实就有三种情况，分别是大于 0、等于 0、小于 0 三种情况。当股价处于上升状态时，dif 大于 0；当股价下跌时，dif 小于 0。而 dif 等于 0 其实就是移动平均线的金叉或死叉信号，当 dif 从负转零，即为金叉，反之为死叉。当 dif 的绝对值变大时，代表长短期均线的开口越大，代表短期股价的上涨或下跌呈现加速状态，反之则呈现减弱态势。

dea 是在 dif 上计算了一次均值。dif 上穿 dea 时形成金叉，根据金叉位置与零轴的关系可以分为零上金叉和零下金叉。零上金叉意味着 dif 在变大，股价的长短期均线距离在变大，股价上涨势头越来越猛；而零下金叉即为 dif 的绝对值在变小，股价长短期均线距离在变小，股价下跌势头变弱。dif 下穿 dea 时形成死叉。零上死叉，意味着 dif 正在变小，股价长短期均线的距离正在变小，股价上涨势头变弱；零下死叉，意味着 dif 绝对值在变大，长短期均线的距离变大，股价下跌势头越来越强。

柱子 macd 就是为了更加明显的看 dif 和 dea 之间的差值。

**参考**

1. [【微信公众号】为什么骗子推荐的股票总是涨？](https://mp.weixin.qq.com/s?__biz=MzAxMjM4MTEwNg==&mid=2651704635&idx=1&sn=49048b7810552fd91f08d654bf75fe7f&chksm=804bd3e6b73c5af0f38b23376b7579b1bbf75eb4cc918227dd880978fe9d9c7e14835c0940d0&scene=21&poc_token=HK_aOWWjQWolLIigvhayAevZmBoQ7VnR98YUgnpY)
2. [【知乎】macd 指标的内在逻辑是什么？](https://www.zhihu.com/question/29954111)
3. [【富途牛牛】什么是指数平滑异同移动平均线（MACD）？](https://www.futunn.com/learn/detail-what-is-the-exponential-smoothing-moving-average-macd-63265-0)
4. [【雪球·天街看锦】指标之王——MACD 的所有用法，终于有一文完完整整明明白白讲清楚了！](https://xueqiu.com/6611881023/124133829)
