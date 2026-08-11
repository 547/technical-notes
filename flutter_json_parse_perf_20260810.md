# Flutter JSON→Model 解析性能方案（格力品类规则数据）

## 背景 / 真实经验
- 用户真实经验：之前几万条时**卡在 json 转 model 阶段**（主线程同步解析 + jsonDecode 大字符串），不是渲染。
- 不能假设"纯解析不会慢"。用户踩过坑 = 真信号。

## 数据结构量级（最坏上限）
- category 最外层：≤20
- 每 category 的 areaRows：≤20（累计 400）
- 每 areaRow 的 cells：≤200（累计最坏 80,000 cell）
- 每 cell 的 models：平均 ~3
- 真实数据：每 areaRow 仅 2 cell → 实际约 800 cell，全量解析 <10ms

## 核心结论（修正版）
1. UI 层每次 ≤200 cell 转 ViewModel：永远主线程，<5ms，无瓶颈。
2. 数据层一次性全量解析（最坏 80k cell）：**隔离到 compute / Isolate，冷启动只跑一次**。
   - 原因：用户有慢的历史 + 最坏 80k，隔离后无论 800 还是 80k 都不挡 UI 帧。
3. 解析只跑一次，结果缓存（全局/仓储/单例），绝不放进 build()。

## 落地代码
```dart
// parse_categories.dart —— 顶层函数（compute 必须是 top-level）
import 'dart:convert';
import 'category.dart';

List<Category> parseCategories(String jsonString) {
  final list = jsonDecode(jsonString) as List<dynamic>;
  return list
      .map((e) => Category.fromJson(e as Map<String, dynamic>))
      .toList();
}

// 调用处（冷启动 / 仓储层只跑一次）
final cats = await compute(parseCategories, jsonString);
```

UI 层保持内联懒转换：
```dart
List<CellViewModel> cellsForArea(AreaRow row) =>
    row.cells.map((c) => c.toViewModel()).toList(); // ≤200，主线程无感
```

## 压低"解析本身"成本
1. 只读展示 → 删掉 `explicitToJson: true`（只影响 toJson 生成，对 fromJson 无用）。
2. `modelNames` 是冗余字段，UI 直接用，不要 `models.map((m)=>m.name)` 重算。
3. 只 parse 一次，build() 只读现成 Model 树。
4. model→viewmodel 一次到位，别二次 parse。

## compute 回传坑
- 回传 80k 对象树要跨 isolate 再序列化一次，有成本。
- 但最贵的是 decode+alloc，隔离后不挡 UI，回传通常可接受。
- 若回传也重 → 升级常驻 worker isolate（持数据、只回传当前屏子集）。80k 量级通常 compute 够。

## 测量判读（Stopwatch）
```dart
final sw = Stopwatch()..start();
final cats = parseCategories(jsonString);
sw.stop();
print('全量解析: ${sw.elapsedMilliseconds}ms, cell 总数: ${cats.totalCells}');
```
- <50ms：主线程 parse 一次即可，不用 compute
- 50~300ms：上 compute，UI 丝滑
- 回传也重：上常驻 worker isolate

## 排除项（已规避）
- 一次性渲染几万 widget（没用 ListView.builder）→ 懒加载已排除
- build 里反复 jsonDecode+fromJson → parse 一次已排除
