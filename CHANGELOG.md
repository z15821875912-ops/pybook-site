# PYbook 迭代日志（CHANGELOG）

每个版本"加了什么"，一眼看完。新版本在最上面。

## 1.0.3 —— 新版 Web + torchpybk 神经网络（2026-09-19）

### torchpybk（tb.*）分类能力补全
- 损失：`tb.softmax`（数值稳定版）、`tb.crossent`（交叉熵）、`tb.crossent_grad`、`tb.argmax`
- 优化器：`tb.sgd(W,梯度,lr)` 就地更新；`tb.adam(W,梯度,M,V,lr,b1,b2,步数)` 带偏差校正
- 数据原语：`tb.shuffle(列表,种子)`（同种子同序列）、`tb.onehot(n,类别数)`、`tb.batch(列表,批号,大小)`（批号语义：批 i 取 [i×大小, i×大小+大小)，末尾不足截断、越界给空表）
- 自此 MNIST 式分类任务全链路可跑：示例 `神经网络示例.pybk` 两层网络 + Adam 训练到 **100% 准确率、损失 0.0007**

### Web（web.*）交互全套
- 页面控件：`web.textbox(占位, id)`、`web.button(文字, id)`
- 读取表达式（网页/实时块专用）：`web.text("id")`、`web.down("id")`、`web.click()`、`web.mx()`、`web.my()`
- 页面能力：`web.audio("文件")` / `web.audio.stop()`、`web.title`、`web.align`、`web.gap`、`web.style("css")`、`web.storage("键","值")`、`web.open("网址")`、`web.waitsec(秒)`
- 画布新动词：`web.draw.图翻`（翻转贴图，6 参）

### 修复（1.0.2 里一用就崩/必错的三处）
- `web.audio.stop()`：代码生成引用未定义变量，编译即崩 → 修复
- `web.waitsec`：等待计数未初始化，首次使用即崩 → 修复
- `mp.mat_scale`：前端注册 3 参、实现 4 参，按任何写法都调不对 → 参数数改为 4

### 文档（AI/新人友好）
- PYBOOK_AI.md 新增五章：`tb.*` 神经网络、`mp.*` math+、`np.*` numpybk、`web.*` 网页输出、`hw.*` 3D（全部示例经 --check 验证可编译）
- 新增本文件（CHANGELOG.md），以后每次发版同步更新

## 1.0.2 —— 编辑器/IDE exe + AI 编程规范 + 语言升级（2026-09-19）

- 场景编辑器 `editor.exe`：可视化搭场景 + 一键真机播放
- `PYbookIDE.exe`：写代码 + 编译 + 运行一体化（C 原生启动器）
- AI 编程四件套：
  - `--check` 只验语法不弹窗（AI 改-验循环）；`--check-json` 结构化结果（含行号）
  - 运行时错误带行号：`运行时错误(第 N 行): 信息`
  - 报错带规则与改法（拼错的标准库前缀会提示可用前缀）
  - `PYBOOK_AI.md` + `llms.txt`：给 AI 的完整编程规范
- 语言升级：字典字面量/推导式、try/catch/onerror、`.get/.at` 安全访问、`list.sorted/rsorted/byfield/unique/merge`、`dirkeys()`、`frame/sec`、多定义/交换/解包、切片/负索引/步长、三元、复合赋值、字符串插值 `$(...)`、成员判断 `in/not in`、字典双变量遍历、默认参数 + 单行函数
- 网页画布与动画：`web.draw.始/色/圆/方/线/清/字/图/图缩/终`、预烤帧动画 `web.帧`/`web.播放`、浏览器实时块 `web.实时`（参数每帧重算）
- 全部新游戏与四语言基准（games/、基准/）

## 1.0.1 —— 网页模式起步 + 物理实验室（2026-09-13）

- 官网新增《物理引擎实验室》：PYbook 网页模式实时物理演示（滑条/按钮/输入框 + 鼠标交互）
- web 段落输出：`web.big/mid/show/img/line/link`
- `ns.show(...)/magic(...)`：输出段落 + HTML 修饰

## 1.0.0 —— 首发（2026-09-13）

- 中文优先语法：`GOING:{...}STOP` 帧循环、`vg.` 声明、`lb.` 类、`rm.around(n) -> i`、`func`/`ret`
- 编译到 C（gcc）：原生 SDL2 窗口，2D 绘图 + 声音 + 输入
- 标准库：`mt.` 数学、`te.` 字符串、`list.`/`dict.`、`rd.` 随机、`file.` 文件、`net.` 网络、`tm.` 时间、`音.`/`mu.` 声音
- 3D 模块 `ON 3D_highway`（`hw.*`：相机/盒子/鼠标锁定/帧循环）
- math+（`mp.*`：向量/矩阵/统计/随机）、numpybk（`np.*`：一维数组逐元素运算）
- torchpybk（`tb.*`）：张量 = 二维列表，全连接/激活/损失 + 手写反向传播算子
- 内置游戏：占点突击、飞船大战、AI 走迷宫、伪 3D 走廊、鼠标勇士、霓虹突袭、地牢屠龙记 等
