# dsh-room · 用 AI 做完这 242 件素材（非美术版操作指南）

> 面向：不会画画、但会用工具、有耐心的人。
> 配套：[`dsh-room-美术素材清单.md`](dsh-room-美术素材清单.md)（要做什么）· [`dsh-room-设计方案.md`](dsh-room-设计方案.md)（为什么这么做）
>
> **先说结论**：AI 能把这件事从"不可能"变成"10 天做出能跑的原型、20 天做出接近清单的质量"。
> 但它**不是一键生成**——真正的活是"定风格 → 分层 → 对齐 → 统一色"这四步，AI 只在第一步和生成素材上帮最大忙。

---

## 0. 一张表看懂：哪些能靠 AI，哪些不能

| 素材类型 | 件数 | AI 直接可用 | AI + 手工修 | 建议别用 AI |
|---|---|---|---|---|
| 家具（立牌） | 33 | ✅ 高 | 修描边/对齐 | — |
| 道具（小件） | 22 | ✅ 很高 | 修手持点 | — |
| tile（地板/墙） | 10 | ⚠️ 中（接缝易翻车） | 必须检查无缝 | — |
| 特效（蒸汽/火花…） | 31 | ⚠️ 中 | 抽帧/调透明度 | — |
| 角色**单帧立绘** | — | ✅ 高 | 分层切割 | — |
| 角色**三方向一致** | 47 | ❌ 低 | 全靠人工逐件比对 | 能用 AI 一次出三视图，但比例一定不一致 |
| 角色**表情差分** | 6 | ⚠️ 中 | 只改眼嘴（局部重绘） | 一次出 6 个基本必废 |
| 角色**逐帧动画** | 50 | ⚠️ 中 | 视频抽帧 + 抠像 | 直接让 AI 出"序列帧"必帧间跳 |
| UI 件 | 15 | ⚠️ 低 | 建议用开源组件改 | ❌ AI 出不了九宫格 |
| 图标 | 31 | ❌ 低 | 用 Lucide/Tabler 重描 | ❌ 24px 网格 AI 画不准 |

**一句话**：**AI 负责"生成画面"，你负责"把画面对齐"**。后者才是这套素材真正的工作量。

---

## 1. 先做一个决定：要不要把规格降下来

清单里的 **47 层角色** 是"专业美术 + 骨骼动画"的规格。你没画过画，切 47 层会崩。有两条路：

| | 方案 A：按原清单 | 方案 B：AI 友好简化（推荐小白） |
|---|---|---|
| 角色层 | 47 | **22–25**（见下表） |
| 序列帧 | 50 | **18–20**（只保留 4 个关键演出） |
| 家具状态 | 每件 2 态 | **每件 1 态 + 运行时染色/叠加**（省 17 件） |
| 图集 | 4 张 196 帧 | 4 张 ~120 帧 |
| 质量上限 | 高（接近商业素材） | 中（"精致原型"） |
| 小白+AI 耗时 | 15–20 天，且要会分层 | **8–10 天** |
| 适合 | 准备正式发布 | 先跑通、先给别人看 |

**方案 B 的角色层（22 件）**：头（down/up/side = 3）· 后发（3）· 表情 6（只做 down）· 躯干整块（3）· 裙摆（3）·
手臂整条 ×2（down/side = 4）——加起来 22。省掉的"上臂/前臂分开"用**整条手臂旋转**代替，
视觉上只损失"手腕翻转"，但工作量掉一半。

> **建议**：先按 B 做一版，截图给别人看。如果反应是"挺好"，就停在 B；如果要求更高，再按 A 补。
> 这不是妥协——B 已经能覆盖清单里所有**交互**（抓/递/按/放/跑），只影响精细度。

**如果你选 B，告诉我，我把清单改成 B 版（删掉对应行、调小计）。**

---

## 2. 工具选型（免费优先）

| 环节 | 免费 | 付费/在线 | 备注 |
|---|---|---|---|
| 出图（主力） | **ComfyUI + SDXL**（显卡 ≥8G） | **即梦 / 豆包 / 通义万相**（国内，出图稳）· Midjourney $10/月 | 小白建议先用在线平台，别一上来装 ComfyUI |
| 角色一致性 | SD + IP-Adapter / LoRA | **Midjourney `--cref`**（最省事）· 即梦"角色参考" | 这是最关键的一项 |
| 抠图/去背景 | **rembg** · `backgroundremover` | remove.bg · Photopea 一键 | rembg 本地免费、批量 |
| 分层切割 | **Photopea**（浏览器，免费）· Krita | Photoshop（套索+蒙版更顺） | **这一步全是手工**，别指望 AI |
| 局部重绘 | ComfyUI 的 inpaint | 各平台的"局部重绘/消除" | 修手、改表情、去多余物件 |
| 视频转帧 | **ffmpeg**（命令行一条） | 剪映（导出图片序列） | 大演出用 |
| 拼图集 | TexturePacker（付费） | `free-tex-packer`（网页版免费） | 之后用我的 `assets:check` 验 |
| 校色/统一色板 | **Python + Pillow**（我给你脚本） | — | 12 色量化，一次搞定全批 |
| 图标 | **Lucide / Tabler**（开源 MIT） | — | 直接改 stroke 宽度就行 |

**最低配起步**：即梦/豆包（出图）+ Photopea（切图）+ rembg（抠图）+ ffmpeg（抽帧）+ free-tex-packer（打包）。
全程零成本，只花时间。

---

## 3. 12 步流水线（每步都有验收）

### Step 1 · 定风格锚（半天）

只做一件事：出一张**你自己满意的"风格样张"**——一个书架 + 一盆植物 + 一小块地板，摆在一起。

- 目的不是交付，是拿到"以后所有图都往它靠"的那张基准。
- 验收：**色板是否只有那 12 色**、描边是不是 2px、有没有渐变/阴影。
- 拿到后：把它当 `--sref`（风格参考）或 img2img 的底图，后面每一批都带上它。

### Step 2 · 角色立绘（1 天）

先只出**正面一张**，别管三视图。

- 提示词见 §4 的 T2，生成 20–40 张挑 1 张。
- 挑的标准：**头身比 3.5、脚在画面底部、手是分开的（不是抱胸/插兜）**。
- 手要分开——这是后面挂道具的前提。

### Step 3 · 三方向（0.5 天，AI 只能帮一半）

用 `--cref`（角色参考）或"角色一致性"功能，分别出正面 / 背面 / 侧面**三张独立的图**（不要一次出三视图）。
然后：**在 Photopea 里把三张的脚底对齐到同一条线、把身高缩放到同一高度**。
> 三视图一次生成的通病是"三个身高不一样"，修比重画更费时。

### Step 4 · 分层切割（1–2 天，最难的手工活）

在 Photopea/Photoshop 里把正面图切成 6–8 块（方案 B）：头、后发、躯干（含裙）、左臂、右臂、腿、配饰、表情层。

**三条经验**：

1. **每块向外多留 8px 重叠**。旋转手臂时不会露缝。
2. 先复制整图，再对每块"隐藏其余部分 → 导出 PNG"，避免切漏。
3. 命名严格照清单：`char/torso_down`、`char/upperarm_down_L`……

验收：把所有块按原位置叠回去，和原图**像素级一致**（说明没切错）。

### Step 5 · 表情差分（0.5 天）

在 Photopea 里**只改眼睛和嘴**（新建一层画），或用平台的"局部重绘"框住五官改。
不要重生成整张脸——发型会变。

### Step 6 · rig 描述（0.5 天，不用画）

填一张表：每块的 pivot（旋转中心）和 6 个 socket 坐标（清单 §2.2 已给参考值）。
**这一步是纯数据，我可以直接给你 `rig.json` 样板，你只需要在 Photopea 里量一下改数字。**

### Step 7 · 家具（2 天，AI 性价比最高）

一件一件出，用统一前缀提示词（T4）。每件出 3–5 张挑 1 张。
两态（方案 B 可省）：用 img2img 强度 0.2–0.3，只加"暖光/火苗"，保证轮廓不动。

### Step 8 · 道具（1 天）

22 件小件，尺寸都小，AI 命中率很高。
验收：**手持点**——道具重心要能落在手心里（清单给了 pivot），否则像浮在手上。

### Step 9 · tile（0.5 天，最容易翻车）

- 提示词里必须写 `seamless tileable`，出图后**横向复制 3 次看接缝**。
- 有缝就修：Photopea 里用"位移 + 修补工具"把接缝抹平。
- 地板只要 3 种变体，别贪多。

### Step 10 · 特效（1 天）

蒸汽/火花/彩带这类，**用视频抽帧比自己画帧快**：
绿幕或纯黑底出 3 秒视频 → ffmpeg 抽帧 → rembg 抠 → 取 4–6 帧。
> 加色类特效（蒸汽/火花）用纯**黑底**更好抠：之后在运行时用 `screen` 混合，黑就是透明。

### Step 11 · UI 与图标（1 天）

- **UI 别用 AI**：气泡/按钮/输入框用 Photopea 画圆角矩形 + 2px 描边，10 分钟一个，比 AI 稳。
- **图标别用 AI**：直接下 Lucide/Tabler，改 `stroke-width=2`、`stroke=#26313F`，导出 24×24。
  缺的 5–6 个（工位/状态）照着风格补画——都是几何图形，比 AI 出的准。

### Step 12 · 统一色 + 打包 + 校验（1 天）

1. **一键把全批量化到 12 色**（我给你 Python 脚本，见 §7）。
2. 用 free-tex-packer 打 4 张图集，导出 `atlas.json`。
3. 跑 `assets:check`（我写）：尺寸 / 命名 / 锚点 / 未引用帧 / 调色板 / 图集字段，全绿才算完。

---

## 4. 提示词模板（可直接抄）

> 原则：**前缀固定、只换 `<物件>`**。前缀固定是"风格不飘"的唯一可靠办法。

### 通用负向词（每次都带）

```
3D render, realistic, photorealistic, perspective, isometric, side view, drop shadow, ground shadow,
gradient, glow, noise, texture grain, text, watermark, multiple objects, busy background, thin outline,
oversaturated
```

### T1 · 风格锚定样张

```
2.5D game asset, orthographic front elevation, hand-painted flat color, 2px dark outline (#26313F),
limited 12-color palette (cream #F7E9D7, wood #E0B98A, brick #D9544B, teal #8FD3C1, warm lamp #FFD79A,
navy #2F3A48, apron white #F7F3EC), no gradient, no noise, no glow, no cast shadow,
single object centered, thin visible top face, clean solid background, --ar 1:1
```

### T2 · 角色正面立绘

```
chibi character sheet, 3.5 heads tall, blue twin-tail hair (#7FC8F8), navy maid dress (#2F3A48) with
white apron (#F7F3EC), flat hand-painted coloring, 2px dark outline, neutral expression, arms slightly
separated from body, standing straight, feet at bottom edge, plain white background, no shadow,
no perspective, --ar 3:4
```

### T3 · 角色背面 / 侧面

```
same character as reference image, back view / side view, identical proportions and outfit,
same 2px outline, plain white background, no shadow, --ar 3:4 --cref <正面图URL> --cw 100
```

### T4 · 家具（一件一行提示词）

```
<物件中文名 + 关键细节> , 2.5D game furniture, orthographic front elevation, flat hand-painted,
2px dark outline, wood/metal palette (#E0B98A #C08A5A #9AA6AE), single object centered,
thin top face, no shadow, plain background, --ar 2:1
```
两态（激活）：`same object, identical silhouette, add warm rim light on the top face only, img2img strength 0.25`

### T5 · 可平铺地板

```
seamless tileable wooden floor texture, 64x64, 3 horizontal planks, slightly darker seams,
flat hand-painted, no shadow, no perspective, orthographic top-down
```
> 出图后：`PIL` 做 3×3 平铺拼图，肉眼找缝。

### T6 · 道具（小件）

```
<道具名>, small game prop, flat hand-painted, 2px outline, single object centered,
plain white background, no shadow, --ar 1:1
```

### T7 · 演出视频（用于抽帧）

```
green screen background, locked camera, no camera movement, chibi maid character doing <动作>,
full body, flat animation style, 10 seconds, steady timing
```
> 抽帧：`ffmpeg -i in.mp4 -vf fps=12 out/%03d.png`，取 6–8 帧。

### T8 · 图标（只在补齐时用）

```
24x24 line icon, 2px round stroke, single color #26313F, no fill, orthographic, centered,
representing "<概念>"
```
> 图标更推荐：下 Lucide 的同义图标改描边。

---

## 5. 一致性四件套（决定成败）

| 手段 | 怎么做 | 解决什么 |
|---|---|---|
| **风格锚图** | Step 1 那张，每批都作为 `--sref` / img2img 底图 | 整批画风不飘 |
| **角色参考** | 正面上半身当 `--cref`（Midjourney）/ IP-Adapter（SD） | 脸、发型、服装一致 |
| **固定 seed + 固定前缀** | 同一批用同一个 seed，只换 `<物件>` | 笔触、光照方向一致 |
| **后处理调色板** | 最后统一量化到 12 色（§7 脚本） | 救回所有"偏色"的图 |

> 前三个是"事前"，最后一个是"事后"。**只做前三个不做第四个，整批会花**。

---

## 6. 三条动画路线（别硬做逐帧）

| 路线 | 做法 | 质量 | 时间 | 适合 |
|---|---|---|---|---|
| **① rig 程序化** | 只切层，不做帧；运行时旋转/平移部件（呼吸、走路、打字、递纸、按铃） | 中上 | 0.5 天 | **12 个循环 clip 全靠它**，不用画 |
| **② 关键姿势插值** | 一个动作出 3–5 张关键姿势（AI 出，人工对齐），运行时补间 | 中 | 2 天 | 结印、庆祝、抱头 |
| **③ 视频抽帧** | 绿幕视频 → 抽帧 → 抠像 | 高 | 1 天/动作 | 只有 2–3 个"大招"值得 |

**建议配比**：① 全部循环动作 + ② 结印/庆祝/抱头 + ③ 一个"分身术"。
这样 50 帧的手绘量压到 **20 帧左右**（方案 B）。

---

## 7. 一键统一色板（Python，可直接跑）

```python
# 用法：python palette.py assets/src/out/*.png   → 就地量化到 12 色
import sys, colorsys
from PIL import Image

PALETTE = ["#F7E9D7","#E0B98A","#C08A5A","#D9544B","#8FD3C1","#FFD79A",
           "#7FC8F8","#5FB4E8","#2F3A48","#F7F3EC","#FFE3D0","#26313F",
           "#7ED99A","#9AA6AE","#F6A9A9"]     # 12 主色 + 3 辅助

def nearest(p, palette):
    return min(palette, key=lambda q: sum((a-b)**2 for a, b in zip(p, q)))

pal = [tuple(int(h[i:i+2], 16) for i in (1,3,5)) for h in PALETTE]
for path in sys.argv[1:]:
    img = Image.open(path).convert("RGBA")
    px = img.load()
    for y in range(img.height):
        for x in range(img.width):
            r,g,b,a = px[x,y]
            if a < 8:                      # 全透明不动
                continue
            nr,ng,nb = nearest((r,g,b), pal)
            px[x,y] = (nr,ng,nb,a)
    img.save(path)
    print("quantized", path)
```
> 它会牺牲一点细腻度，换来"整批看起来是一套"——对非美术来说，这个交换非常划算。

---

## 8. 10 天排期（方案 B，每天 4–6 小时）

| 天 | 做什么 | 交付物 |
|---|---|---|
| 1 | 风格锚定 + 角色正面 | style.png · char_front.png |
| 2 | 角色背/侧 + 对齐 | char_back.png · char_side.png（等高） |
| 3 | 分层切割（最难，留足时间） | 22 个 PNG |
| 4 | 表情 6 个 + rig.json | 表情层 + 数据 |
| 5 | 家具第一批（书架/工作台/显示器/灶台/柜台/白板/传送带） | 7 件 |
| 6 | 家具第二批 + 道具 10 件 | 9 + 10 |
| 7 | 道具剩余 + tile | 12 + 6 |
| 8 | 特效（抽帧）+ 光照 | 约 20 |
| 9 | UI + 图标 | 15 + 31 |
| 10 | 统一色 + 打包 + `assets:check` | 4 张图集 |

> 每天收工前做一件事：**把今天的图和 style.png 并排看一眼**。偏了就当天重出，别攒到最后。

---

## 9. 最容易翻车的 7 个点

| # | 坑 | 症状 | 防法 |
|---|---|---|---|
| 1 | 脚底线不一致 | 角色走路上下跳 | 每个方向件都压在 `y=448`（@2x） |
| 2 | 三方向比例不一 | 转身时"变高/变胖" | Photopea 里先统一身高再切 |
| 3 | 透明边白圈 | 边缘一圈脏白 | 导出前扩 1px，颜色取相邻色 |
| 4 | 调色板漂移 | 一批图里"奶油色有 5 种" | §7 脚本兜底 |
| 5 | tile 有缝 | 地板出现网格线 | 3×3 平铺自检 |
| 6 | 道具浮在手上 | 拿纸像贴纸 | 按清单的手持 pivot 对位 |
| 7 | 素材越做越多 | 图集爆到 8MB | `assets:check` 拦"未引用帧" |

---

## 10. 版权与合规（别忽略）

- **Midjourney**：付费订阅可商用；免费额度**不可**商用。
- **SD / ComfyUI 本地生成**：模型许可证各异（SDXL 允许商用；部分二次模型有非商用限制，看模型卡）。
- **国内平台（即梦/豆包/通义）**：各自条款不同，商用前读一遍"生成内容权属"。
- **开源图标（Lucide/Tabler）**：MIT，可商用，保留 LICENSE 即可。
- **参考 dsh-pet 的画风是可以的**（风格不受版权保护），但**不能直接把它的 png/webm 拼进来**（其素材明确禁商用）。

---

## 11. 我能直接帮你做的（说一声就行）

1. **`rig.json` 样板**：把 22 个部件、6 个 socket、26 个 clip 的 JSON 结构先写好，你只改坐标数字。
2. **`assets:check` 校验脚本**：尺寸 / 命名 / 锚点 / 未引用帧 / 调色板 / 图集字段，一条命令出报告。
3. **批量对齐脚本**：给定"脚底 y"和"中线 x"，自动把一批 PNG 裁边 + 补透明 + 对齐。
4. **提示词包**：把 §4 的 8 个模板做成可复制文件（含 33 件家具的逐条提示词）。
5. **清单切成 B 版**：如果你选简化方案，我把美术素材清单改成 22 层 / 20 帧版本。

---

## 附：一句话回答"我该怎么开始"

> **今天只做一件事**：打开即梦或豆包，用 §4 的 T1 出一张"书架 + 盆栽 + 地板"的风格样张。
> 看着它能不能接受——能，就按 §8 的排期往下走；不能，就改提示词再出，直到能接受。
> **不要先装 ComfyUI，不要先切图，不要先写代码。** 风格没定，后面全是白工。