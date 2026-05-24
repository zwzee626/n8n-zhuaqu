# 肺炎关键词批签发（2026）每周五抓取方案（n8n）

> 目标：每周五上午 09:00 自动抓取 `bio.nifdc.org.cn/pqf/search.do?formAction=pqfGs` 下与“肺炎”相关的记录，进入子页面提取批签发时间（子网站标题），仅保留 2026 年数据，并写入/更新表格。

## 表头

```text
序号
官网序号
产品名称
批号
有效期至
上市许可持有人
证书编号
签发结论
批签发机构
批签发时间（子网站标题）
```

## n8n 工作流节点设计

1. **Schedule Trigger**
   - Cron：`0 9 * * 5`
   - 时区建议：`Asia/Shanghai`
2. **HTTP Request - 搜索页**
   - URL: `https://bio.nifdc.org.cn/pqf/search.do?formAction=pqfGs`
   - Method: `POST`
   - Body/Form 参数包含关键词：`肺炎`
3. **HTML Extract - 列表解析**
   - 解析列表字段：官网序号、产品名称、批号、有效期至、上市许可持有人、证书编号、签发结论、批签发机构、详情链接
4. **Split In Batches**
   - 遍历每条列表数据
5. **HTTP Request - 子页面详情**
   - 请求详情链接
6. **Code - 提取子页面标题并过滤 2026**
   - 从子页面标题中提取批签发时间
   - 仅保留 `2026` 年
7. **Google Sheets / 表格节点（Upsert）**
   - 按唯一键（官网序号 + 批号）更新，不存在则新增

## Code 节点示例（过滤 2026 + 组装输出）

```javascript
const item = $input.item.json;

const title = item.detailTitle || '';
const yearMatch = title.match(/(20\d{2})/);
const year = yearMatch ? Number(yearMatch[1]) : null;

if (year !== 2026) {
  return [];
}

return [{
  json: {
    序号: item.index,
    官网序号: item.officialIndex,
    产品名称: item.productName,
    批号: item.batchNumber,
    有效期至: item.validUntil,
    上市许可持有人: item.mah,
    证书编号: item.certificateNumber,
    签发结论: item.conclusion,
    批签发机构: item.issuer,
    '批签发时间（子网站标题）': title,
  },
}];
```

## 去重与更新策略

- 建议将 `官网序号 + 批号` 拼接作为唯一键，例如：`{{官网序号}}-{{批号}}`。
- 写入前先查表：
  - 命中则更新整行（确保签发结论等变化会同步）
  - 未命中则新增。

## 定时说明

- 每周五早上 9 点（中国时区）执行：`0 9 * * 5`
- 若 n8n 实例不在中国时区，请在 **Schedule Trigger** 中显式设置 `Asia/Shanghai`。

## 合规提示

- 请先确认目标网站 robots、服务条款及访问频率限制。
- 若页面结构变更，需要同步更新 HTML 解析选择器。
