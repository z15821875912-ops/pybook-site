# PYbook 迭代日志（CHANGELOG）

每个版本"加了什么"，一眼看完。新版本在最上面。

## 1.0.4 —— 3D 可编程管线 + Godot 级编译器（2026-09-25）

这一版分两条线：**3D_highway 从"固定管线小引擎"变成"真 OpenGL 4.6 渲染管线"**，
**编译器从"比 Python 慢 80 倍"追成"同题同量级"**。

### 3D 可编程管线（GLSL）—— 本版主菜
- 新增 `hw.preset(槽,名字)`：6 个内置着色器（中英别名）
  `lambert`/朗伯、`toon`/卡通（`uSteps` 可调色阶）、`fresnel`/边缘光（`uRim`/`uRimPow`）、
  `checker`/棋盘（程序化纹理）、`normal`/法线可视化、`unlit`/自发光
- 新增 `hw.shader(槽,顶点源码,片段源码)`：装自定义 GLSL，返回 1/0 不中断
- 新增 `hw.shaderuse(槽)`：**进命令流**，所以能在一帧里中途切换
  （世界用着色器画 → 切回固定管线画 gizmo 辅助线 → 再切回来）
- 新增 `hw.uni(名, a[,b[,c[,d]]])`：设 uniform（float/vec2/vec3/vec4）
- 引擎自动喂三个约定 uniform：`uLight`（跟着 `hw.light`）、`uTex`（纹理单元 0）、`uTexMix`
- **架构关键**：着色器用 **GLSL 120**，兼容 profile 下能直接读
  `gl_Vertex`/`gl_Normal`/`gl_Color`/`gl_MultiTexCoord0`/`gl_ModelViewProjectionMatrix`，
  这些量由现成的 `glBegin` 几何自动喂进来 —— **1400 行几何生成代码一行都没改成 VBO**
- 把"固定管线是天花板"这个判断彻底推翻：`_gl_probe.c` 实测驱动是
  **OpenGL 4.6 兼容 profile**，`glCreateShader`/VBO/VAO/FBO/compute shader 全部可达，
  缺的只是"那几百行还没写"

### 3D_highway 其余补全
- **球 / 圆柱 / 圆锥**：`hw.sphere(x,y,z,半径,r,g,b)`、`hw.cyl(x,y,z,下半径,高,上半径[,边数],r,g,b)`
  —— 一个函数吃下圆柱/圆锥/倒锥/六棱柱；16 纬×24 经 `GL_TRIANGLE_STRIP`，sin/cos 细分表按段数缓存
- **线段**：`hw.line(x1,y1,z1,x2,y2,z2,r,g,b)`（gizmo / 坐标轴 / 包围盒 / 弹道）
- **真方向光**：`hw.light(方向x,方向y,方向z,强度)` 按面法线做 Lambert，
  替换原来写死的 100%/62%/78% 假明暗；方向内部归一化，球/柱共用
- **视口可调**：`hw.fov(角度)`（钳 5~170）、`hw.near(距离)`（原来 70° / 1.0 是硬编码常量）
- **贴图**：`hw.tex(槽,路径)` / `hw.texuse(槽)` / `hw.uv(重复)`
  —— stb_image 解码（png/jpg/bmp/tga），中文路径走 `_wfopen`，
  透明 PNG 用 alpha 裁剪（不做排序，绕开半透明排序难题），`GL_REPEAT` 支持 UV 平铺
- **父子层级**：`hw.push(x,y,z,绕x,绕y,绕z)` / `hw.pop()` —— 最多 20 层嵌套，
  "转动桌子桌上的东西跟着转"从做不到变成一行；忘配对 pop 帧末自动清栈
- **架构改造：op 命令流** —— 图元不再"按类型分批画"而是"按脚本提交顺序画"，
  push/pop/shaderuse 只有顺序执行才有语义；盒/球/柱全部补 `glNormal3f`
- 配套：3D 场景摆放器原型（放置/拾取/三轴旋转/复制/吸附/FOV 与近裁剪面调节/线框图元切换）
- 修复：`glUniform1f` 设 `sampler2D` 会吃 `GL_INVALID_OPERATION`（探针实测抓出）

### Godot 级编译器优化（速度对决翻身仗）
- 第一阶段：`rm.around` 热点**标量化** —— int/float 纯算术变量整圈走 C 原生标量
  （`pk_mod`/`pk_fdiv` 保语义），候选分析 + 值域逃逸装盒 + 自动回退
- 第二阶段：`RtDict` **哈希索引** —— 桶表 + 懒建 + 删除脏标记，六处线性扫描换 `dict_hfind`，
  小字典自动回退线性
- 修：标量化拦截下沉到 stmt 全深度（嵌套 if/循环内的候选赋值原先进不去，导致计数码归零）
- **结果**：速度对决从"比 Python 3.11 慢约 80 倍（73.6s vs 0.89s）"追到 **1.68s vs 0.62s**；
  神经网络示例 100% 准确率与损失逐位恢复

### 新标准库
- `js.` —— JSON 解析 / 序列化（纯 C 递归下降，`%.17g` 保真，中文键值/嵌套/往返一致性全过）
- `csv.` —— RFC4180（引号、转义、引号内换行与逗号，往返验收过）
- `bs.` —— **位运算库**（15 个）：`and/or/xor/not`、`shl/shr`、`get/set`、`popcount`、
  `band/mask`、`hex/bin/frombin/fromhex`
- `mm.` MyMove —— **列表即像素画**：`mm.draw_away` 落成真 PNG（手写 store 模式编码器，零依赖）、
  `mm.贴` 窗口直画（SDL 纹理 alpha 混合）、`mm.色` 自定义 token、`mm.动` 逐帧动画；
  双模式自动识别（多字符 token / 单字符连写）；三张 PNG 像素级比对全对
- **音乐库扩展——列表做音乐**：`音.音(名, 频率, 波形)` 注册自定义音符（正弦/三角/噪声）、
  `音.奏(表, "名.wav", 速度)` 把乐谱列表渲染成 WAV 落盘、
  `音.谱(表, 速度)` 即时播放 —— 和 `mm.draw_away` 对称的"表即作品"路线
- `list.flat` / `list.zip` / `list.count` 三定式
- 空 / `none` 空值字面量（`js.dump` 输出 `null`）

### VM 后端（pyvm）
- **字典字面量**：新增 `DICT_NEW` / `DICT_SET` 指令，VM 模式也能用字典
- **标准库调用**：新增 `CALL_LIB` 指令（库名进常量池 + 实参按书写序压栈，按 arity 弹参），
  VM 模式不再只能跑"纯语言子集"

### 语言与工具链
- **`查(条件, "名字")` 测试语句** + `--test`：断言计数、失败逐条报、
  失败数作进程退出码（AI 的"改-验"循环可以直接用退出码判断）
- **`引用("模块")` 模块合类**：把模块的 `lb.` 类、函数、顶层变量全部并入主文件
  （深度优先、防循环引用、重名带行号报错）
- **`web.块` 组件复用**：文件顶层 `web.块 名字(参数){ web.* 语句 }`，
  GOING 里当函数调；可嵌套
- `web.页("名"){...}` 多页输出（页间相对路径跳转）+ `web.draw.始(宽,高)` 画布尺寸可调
- 表单短板三连修：`web.置(id,值)` 程序回写输入框、`web.textbox` 第三参绑按钮（回车=点一下）、
  `web.存(键,缺省)` 读 localStorage
- 词法器支持 `/* */` 块注释（修：`AI走迷宫` 被 C 风格注释弄坏）
- **多核加速**：`pybook_mt` 线程池（64 块切分、主线程参与、`PYBOOK_THREADS` 可调），
  `tb.*` 与 `np.*` 大矩阵自动并行、小任务零开销串行 —— 实测整体 **2.64×**
- **积木编辑器**：单文件零构建离线可用，画笔海龟 + 控制流积木，点击拼装，
  实时生成 PYbook 代码，一键下载 `.pybk` 接现有编译管线

### 对决与基准（本版新增的"实力证据"）
- 速度对决：PyBench 标量循环 PYbook vs Python 3.11（翻身仗）
- 对决赛道三（桌面窗口）：弹球实验室 PYbook 26 行 vs Python tkinter 39 行
- 对决赛道四（动画）：MyMove 火箭动画 41 行 / 1297 字符 vs Python tkinter 48 行 / 1815 字符
- web 对决：弹球 PYbook 29 行 vs React18 76 行；待办清单行数胜、交互体验认输（已记录边界）
- 四语言对比：C++ / Java 同题实测（弹球 16 vs 26/28，数字统计 11 vs 20/17，词频 21 vs 12/12）

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
