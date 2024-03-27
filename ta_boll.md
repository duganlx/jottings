# BOLL

布林线指标 Bollinger Bands, 是研判股价运动趋势的一种中长期技术分析工具. 一共有三根线组成, 中轨线, 上轨线, 下轨线.

假设收盘价$CLOSE=\{p_1,p_2,...,p_n\}$, 布林线观察窗口为 20 日, 且上下轨取 2 倍标准差

$$
mid_i = \frac{p_{i-19} + p_{i-18} + ... + p_{i}}{20}
$$

$$
std_i = \sqrt{\frac{\sum_{k=i-19}^{i}(p_k - mid_i)^2}{20}}
$$

$$
upper_i = mid_i + 2 \times std_i
$$

$$
lower_i = mid_i - 2 \times std_i
$$

代码实现如下

```py
def boll(close, wins=20, k=2) -> pd.DataFrame:
    mid = close.rolling(window=wins).mean()
    std = close.rolling(window=wins).std()
    upper = mid + k * std
    lower = mid - k * std

    mid = pd.Series(mid, name='mid')
    upper = pd.Series(upper, name='upper')
    lower = pd.Series(lower, name='lower')
    df = pd.concat([mid, upper, lower], axis=1)

    return df
```

## 参考

1. [【知乎】怎么看懂「布林线（Boll）指标」，如何分析与运用？](https://www.zhihu.com/question/384284854)
2. [【知乎】图文详解：BOLL 线的真正用法，胜过读万本股票书](https://zhuanlan.zhihu.com/p/47995278)
