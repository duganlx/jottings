# s2Table 使用总结

官方文档: https://s2.antv.vision/zh

目前开发中，大多数表格的展示都慢慢转用 s2 表格，而不是用 Ant Design 的表格组件，原因是 s2 在数据量大的场景下用户体验更好。s2 表格的使用可分为两类，透视表 和 明细表。以下是开发过程中，碰到的一些经典的使用场景。环境版本是 `"@antv/s2": "^1.47.1"`, `"@antv/s2-react": "^1.40.0"`

### 明细表, 标题行背景色

资产概要中的 S2 表格因为有非常多的列，s2 原本的表头背景色都是一样的，容易看花。所以用户是希望按照 收益、资产、负债等抽象概念将一些列放在一块，并且标记不一样的颜色以示区分。

代码实现如下所示（只保留最关键的代码）

```tsx
class CustomColCell extends ColCell {
  getBackgroundColor(): { backgroundColor: string; backgroundColorOpacity: number } {
    const { meta } = this;
    const oriParam = super.getBackgroundColor();
    if (
      meta.field.search(/.*totalAssetPnl.*/) === 0 ||
      meta.field.search(/.*alpha.*/) === 0 ||
      meta.field.search('xnqz_ticket') === 0 ||
      meta.field.search(/isJoinCalc.*/) === 0
    ) {
      return { backgroundColor: '#ffe7ba', backgroundColorOpacity: 0.8 };
    }

    if (meta.field.search(/.*Liability.*|.*Debt.*/) === 0) {
      return { backgroundColor: '#ffccc7', backgroundColorOpacity: 0.8 };
    }

    if (meta.field.search(/.*Deposit.*|.*Withdraw.*/) === 0) {
      return { backgroundColor: '#d9d9d9', backgroundColorOpacity: 0.8 };
    }

    return oriParam;
  }
}

const s2Options = {
  colCell: (node: Node, s2: SpreadSheet, headConfig: unknown) => {
    return new CustomColCell(node, s2, headConfig);
  },
}

<SheetComponent
  sheetType="table"
  options={s2Options as any}
/>
```

### 明细表, 数据行背景色

在结算的资产概要中，有个需求是希望如果某个交易日的市值变化超过 10%则该交易日的当日盈亏就不计算入累计盈亏中。在实现在决定在 s2 明细表中添加一列 "参与计算" 来表达该天是否应该参与计算，并且还希望将不参与计算的行用灰色进行标记。

代码实现如下所示（只保留最关键的代码），有几点需要注意下：

1. 如果在 `CustomDataCell` 类使用外部的变量，当前发现只能通过本地缓存`localStorage` 来完成，测试使用构造器传参方式发现不可行。
2. 如果需要取得当前所在行，需要通过`meta.spreadsheet.dataSet.getCellData({query: {rowIndex: meta.rowIndex}})`获取, 不能直接用 `meta.rowIndex`。

```tsx
class CustomDataCell extends DataCell {
  getBackgroundColor():
    | {
        backgroundColor: string;
        backgroundColorOpacity: number;
        intelligentReverseTextColor?: undefined;
      }
    | {
        backgroundColor: string;
        backgroundColorOpacity: number;
        intelligentReverseTextColor: boolean;
      } {
    const oriparam = super.getBackgroundColor();
    const { meta } = this;

    const balanace = meta.spreadsheet.dataSet.getCellData({
      query: { rowIndex: meta.rowIndex },
    }) as any as UniverseData;

    const showCalcPercentage = localStorage.getItem('assetoutlineview#showCalcPercentage');
    const alphaPercentageFM = localStorage.getItem('assetoutlineview#alphaPercentageFM');

    if (alphaPercentageFM === 'rcccsz' && showCalcPercentage === 'pnlPercentage') {
      if (!balanace.isJoinCalc_sz) {
        // 将背景色设置为灰色
        return {
          ...oriparam,
          backgroundColor: '#d9d9d9',
          backgroundColorOpacity: 0.4,
        };
      }
    }

    return oriparam;
  }
}

const s2Options = {
  dataCell: (viewMeta: ViewMeta) => {
    return new CustomDataCell(viewMeta, viewMeta?.spreadsheet);
  }
}

<SheetComponent
  sheetType="table"
  options={s2Options as any}
/>
```

