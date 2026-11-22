# Abscission · 离

_A study in leaving. 一场关于离去的研究。_

Abscission is a single-file web piece for autumn. Place a fallen leaf under a webcam microscope and the page turns what the lens sees into drawing and sound: veins become branching lines, cells drift into organic forms, and a slow reading head plays the leaf as music while short lines of poetry follow its colour from green to brown.

It runs entirely in the browser. No server, no build step, no dependencies, and nothing is uploaded.

> _Brown is the colour of having given everything back._

---

## The concept

_Abscission_ is the botanical term for the way a plant lets go of a leaf. The tree seals the connection, the green drains away, and the leaf falls. It is one of the quietest and most deliberate acts in nature, and it is also a decent description of a lot of human life.

The piece is built around that idea:

- **Plate I — Observation** is the leaf as it is right now, annotated like a botanical plate. Its colour is muted by default so the leaf's pigment stays visible without breaking the gallery look; it can be set to true colour or pure black and white.
- **Plate II — Reconstruction** is the leaf as it is remembered. A drawing grows from what the lens found, rests, then comes apart and falls. Then it starts again from whatever is under the lens, so every cycle is a new reading.
- **The sound** is made from the leaf itself. Nothing is pre-recorded.
- **The words** change with the leaf's pigment: chlorophyll, carotene, anthocyanin, tannin. They are about transition, ending and return.

At microscopic scale a leaf and a person are made of the same thing: cells, channels carrying water, a slow cycle of growing, changing and letting go.

---

## Running it

It is one file: `abscission.html`.

1. Open it in **Chrome, Edge or Firefox**.
2. Choose **begin with a camera**, **open a photograph**, or **look at a sample leaf** (a synthetic leaf that slowly turns from green to brown, useful for testing without a camera).
3. If the browser blocks the camera for a local file, serve the folder instead:

   ```bash
   python3 -m http.server
   # then open http://localhost:8000/abscission.html
   ```

If you have a laptop webcam and a USB microscope, the camera is probably the laptop's by default. Click the camera link in the header to cycle through devices.

### Controls

| Key     | Action                                                          |
| ------- | --------------------------------------------------------------- |
| `space` | hold the current frame (useful for adjusting focus)             |
| `c`     | colour of both plates: muted (default) → true → mono            |
| `i`     | invert: black wall, white ink                                   |
| `m`     | sound on / off                                                  |
| `s`     | keep a plate: saves a PNG of both plates with the current verse |

The same actions are in the header. Sound starts on, but browsers only allow audio after a click, so it begins once you choose a source.

### Getting good images

- Backlit leaves on a glass slide show veins most clearly.
- Flatten curled leaves under a slide; the focus depth on a microscope is tiny.
- Even, diffuse light works better than a single hard source.
- Skeletonised leaves (soft tissue rotted away) make beautiful branching drawings.

---

## How it works

Everything is in `abscission.html`, in one `<script>` block, roughly in this order.

### 1. Sources

The page can read from a camera (`getUserMedia`), a still photograph, or the built-in sample leaf (`drawSample`). Whichever it is, each frame is drawn into a small hidden canvas (192 px wide, `GW`) and all analysis is done on that grid, which keeps it fast and also helps smooth sensor noise.

### 2. Analysis (`analyse`)

- **Tone:** grayscale, lightly smoothed over time.
- **Ridges:** the difference between a small and a large box blur (`|blur(1) − blur(4)|`). Thin bright or dark lines, such as veins, stand out. Points above `mean + 1 σ` count as ridges (`rT`). A mean below `0.007` means no leaf is in view.
- **Pigment:** pixels are classified by hue into green, yellow, red/orange and brown. The dominant class (debounced for two seconds) sets the "state" used for poetry and scale.
- **Contours:** marching squares over a smoothed tone map at five levels, drawn as the thin topographic lines on Plate I.
- **Annotations:** up to seven local ridge maxima get a ring, a leader line, a roman numeral and the local vein angle, calculated from the gradient.

### 3. Plate II — cells (`cell`, `drawGen`)

- 96 seed points drift slowly and are re-targeted one at a time, 70% of the time toward strong ridges, so cells get smaller along the veins.
- Each cell is a **Voronoi** region computed by clipping against the other seeds' bisectors (half-plane clipping), then shrunk and rounded with quadratic curves so the walls look organic. A nucleus ring is drawn when the cell is big enough.
- Fill follows the leaf under each seed: in muted or true colour mode each cell samples the live image colour (a small average, smoothed over time) and is muted by the same amount as Plate I; in mono mode, fill darkness follows the tone instead.

### 4. Plate II — branching (`growStep`, `updateTree`)

The branching structure uses **space colonisation** (Runions et al., 2007):

- Attractor points (up to 240) are sampled from the ridge map.
- The tree starts at the bottom centre and, 14 times a second, each node is pulled toward the attractors nearest to it, growing new nodes and removing attractors it reaches.
- Line width follows a pipe model (thickness from the size of the subtree behind it).
- The life cycle has four phases: `wait` → `grow` → `rest` (7 s) → `dissolve` (7 s, the branches fray and fall) → reset from the current frame.

### 5. Sound (Web Audio)

- **Scale and mode:** root is D. The mode comes from the leaf's state: major pentatonic for green, lydian pentatonic for yellow, minor pentatonic for red, hirajoshi for brown, open fifths when no leaf is present.
- **Reading head:** a vertical line sweeps both plates every 18 seconds (`SWEEP_T`). For each column it crosses, the strongest ridge may trigger a soft bell-like note. Vertical position sets pitch, quantised to the scale over three octaves; horizontal position sets stereo pan; ridge strength sets loudness.
- **Pad:** four detuned oscillators through a low-pass filter with a slow LFO. Its chord follows the current mode, and its brightness follows the tone of the image.
- **Events:** branches splitting ring quietly; when the drawing begins to dissolve, a low long chord plays.
- **Space:** a generated convolution reverb and a compressor sit on the master output.

### 6. Poetry (`VERSES`)

Short original lines in English and Chinese, grouped by pigment state, shown one at a time with a slow fade.

### 7. Look

Warm off-white wall, hairline frames, serif type, no icons or gradients. The overlay on Plate I uses a `difference` blend so white lines stay legible on any grey. Text is bilingual: the Chinese is set slightly smaller and lighter under or beside the English.

---

## Tuning

All the important constants are near the top of their sections.

| Constant                | Where        | What it does                                            |
| ----------------------- | ------------ | ------------------------------------------------------- |
| `GW`                    | grid         | analysis resolution (higher is more detail, slower)     |
| `SWEEP_T`               | scan line    | seconds per sweep; larger is slower and calmer          |
| `S.t - lastNoteT < 0.3` | `readColumn` | minimum time between scan notes                         |
| `NC`                    | cells        | number of cells                                         |
| `MAXN`, `STEP`, `KILL2` | tree         | node limit, growth step, kill distance                  |
| `dt * 14`               | `updateTree` | growth steps per second                                 |
| `phaseT > 7`            | `updateTree` | rest and dissolve durations                             |
| `active = m > 0.007`    | `analyse`    | "is there a leaf" threshold; raise it for noisy cameras |
| `MODES`                 | sound        | scales per state; edit to change the music              |
| `VERSES`                | poetry       | add or rewrite lines                                    |

---

## Browser notes

- Tested in Chrome. Edge and Firefox should work. Safari supports most of it, but the grayscale on saved PNGs relies on `CanvasRenderingContext2D.filter`, which Safari may not support.
- Fonts (Cormorant Garamond, Noto Serif SC) load from Google Fonts. If they can't load, the page falls back to system serif fonts.
- The camera needs permission and a secure context (`https`, `localhost`, or a local file in most browsers).

## Credits

- Branching: space colonisation, Runions, Lane, Prusinkiewicz, _Modeling Trees with a Space Colonization Algorithm_ (2007).
- Contours: marching squares. Cells: Voronoi diagram via half-plane clipping.
- Typefaces: Cormorant Garamond, Noto Serif SC (SIL Open Font License).

## License

---

# 中文说明

**Abscission · 离**：一场关于离去的研究。

这是一件为秋天而做的单文件网页作品。把一片落叶放在网络摄像头显微镜下，页面会把镜头里看到的东西变成图像和声音：叶脉长成分枝的线条，细胞漂移成有机的形状，一道缓慢的读取线把叶片弹奏成音乐，诗句则随着叶片的颜色，从绿到褐而变化。

全部在浏览器本地运行：无需服务器、无需构建、无依赖，也不会上传任何内容。

## 概念

_Abscission_（离层 / 落叶）是植物学术语，指植物主动放下一片叶子的过程：树封住连接，绿色退去，叶子落下。这是自然里最安静、也最有意为之的行为之一。

- **图一 · 观察**：此刻的叶片，黑白呈现，像植物图鉴一样做了标注。
- **图二 · 重构**：被记忆的叶片。图形从镜头所见中生长，停留，然后解体、坠落；之后再根据镜头下的新画面重新开始，每一轮都是新的读取。
- **声音**由叶片本身生成，没有任何预先录制的素材。
- **文字**随叶片的色素（叶绿素、胡萝卜素、花青素、单宁）而变化，关于转变、终结与回归。

在微观尺度上，一片叶子和一个人由相同的东西构成：细胞、输送水分的通道，以及缓慢的生长、变化与放下。

## 使用

只有一个文件：`abscission.html`。

1. 用 **Chrome、Edge 或 Firefox** 打开。
2. 选择“以相机开始”“打开一张照片”或“观看示例叶片”（一片会慢慢从绿变褐的合成叶片，没有相机时也能测试）。
3. 如果浏览器不允许本地文件使用相机，请在文件夹中运行 `python3 -m http.server`，然后访问 `http://localhost:8000/abscission.html`。

如果你同时有笔记本摄像头和 USB 显微镜，默认可能是笔记本摄像头，点击页头的相机文字即可切换设备。

**快捷键**：`空格` 定格画面；`c` 切换两幅画面的色彩（淡彩 / 原色 / 黑白）；`i` 反色；`m` 开关声音；`s` 保存当前画面为 PNG。

## 代码结构（简述）

- **采集**：相机、照片或示例叶片，统一绘制到 192 像素宽的隐藏画布上做分析。
- **分析**：灰度、叶脉（大小两个模糊之差）、按色相分类的色素状态、用 marching squares 画出的等高线、以及最多七处带编号和叶脉角度的标注。
- **图二 · 细胞**：96 个种子点做 Voronoi 分割（半平面裁剪），收缩并圆角化，越靠近叶脉细胞越小。
- **图二 · 分枝**：空间殖民算法（space colonization）：从叶脉采样吸引点，树从底部中央生长；经过“生长—停留—解体—重来”的循环。
- **声音**（Web Audio）：以 D 为根音；绿色用大调五声、黄色用利底亚五声、红色用小调五声、褐色用平调子音阶；读取线每 18 秒扫过一次，扫到叶脉就奏出柔和的音，高度决定音高、横向位置决定声像；底部是随叶片色调变化的和弦铺底，并带有混响。
- **诗句**：中英双语，按色素状态分组，缓缓淡入淡出。

可调参数（`GW`、`SWEEP_T`、`NC`、`MAXN`、`MODES`、`VERSES` 等）见上方英文表格。

## 许可
