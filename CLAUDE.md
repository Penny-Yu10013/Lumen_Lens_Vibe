# Lumen Lens 寶石透鏡 — 開發交接文件 (CLAUDE.md)

> 給未來 session：這是**目前實作狀態**的交接。原始提案在 `Lumen_Lens_Vibe提案表.md`（不要改，當願景參考）。
> 前作（架構複用來源）：`C:\Users\yu2_7\Downloads\isekai_gemcraft_claude\`（v1=切割幾何；本作 v2=內部結構與光學現象）。

---

## 0. 一句話

黑底寶石光學現象觀察台：切片放光桌上透光看，拖曳旋轉/傾斜，彩虹瑪瑙傾到對的角度才出彩虹。單一 `index.html`，Three.js r128 CDN，全程序化零外部資產。

## 1. 怎麼跑 / 怎麼驗

- **跑**：雙擊 `index.html`（需連網載 Three.js CDN）。
- **Claude Code 自動驗證**：`.claude/launch.json` 有 `lumenlens`（python http.server, port 8737）→ preview 工具開 `http://localhost:8737`。
  - **Browser pane 若 hidden（截圖逾時、rAF 停擺）**：改用 headless Chrome 截圖（批次腳本模板在 session scratchpad `shoot.ps1`，要點：每張獨立 `--user-data-dir`、45s 逾時 kill、**`--virtual-time-budget=1500` 別調高**——SwiftShader 下 rAF 會連跑 budget÷16ms 幀，iris shader 最慢，5000 會爆 45s；`#maxframes=N` 停 rAF 的方案不可行，headless compositor 會把停掉的 canvas 變黑）。逾時是隨機的，重試即可。
  - **數值驗證**：`#debug` 下有 `window.__ll`（renderer/scene/state/sliceMats + `probe([[x,y],..])` 同步 render+readPixels），Browser pane 隱藏也能用，比截圖快，適合「哪個 mesh 沒畫/顏色對不對」。
- **狀態直達 URL**（hash 參數，自動驗證核心）：
  `#debug&slice=iris&tilt=25&spin=40&backlight=1.2&zoom=1.1`
  - `debug`＝左上 fps overlay＋每 5 秒 console `[fps]`；`shot`＝隱藏全部 UI（宣傳截圖/錄屏）；`auto`＝傾角自動正弦掃＋緩慢自旋（錄屏用）
- 手機版：preview mobile preset（375×812）模擬。

## 2. 架構決策（別推翻）

- **拖曳轉切片，不是 OrbitControls**【人類決】：相機固定微俯視（`CAM_BASE=(0,-0.75,3.25)`），水平拖=自旋、垂直拖=傾斜（clamp ±40°）、pinch/滾輪=縮放（0.6–1.8）。目標值+lerp 慣性（鬆手彩虹滑一小段，是 Iris 體感的一部分）。
- **全部光都是 shader uniform，場景零 THREE.Light**：背光=`uBacklight` 同時餵光桌/切片/光暈三材質。
- **切片外形不建 mesh**：單一 2.2×2.2 quad，fragment 內 `shapeSD()` SDF 裁形＋磨邊亮線。draw call 恆定 3（光桌/切片/光暈）。
- **一份 GLSL 模板 + defines 編三份材質**（`SLICE_TRAPICHE`/`SLICE_LIDDI`/`SLICE_IRIS`），啟動 `precompile()` 全編譯，切換零卡頓。不用 uniform 動態分支。
- **視角相關性**：JS 每幀把切片 quaternion 逆轉成 `uInvRot`（mat3），fragment 把 viewDir 轉進切片局部空間。物理項用真 viewDir，藝術調校用純量 `uTilt`。
- **嵌入式視口坑（踩過）**：Browser pane / in-app WebView 載入瞬間 `innerWidth=0` 且之後不發 resize → `animate()` 每幀比對視口尺寸，變了呼叫 `onResize()`（`onResize` 對 0 尺寸直接 return）。
- **渲染順序三連坑（都踩過，2026-07）**：
  1. `precompile()` 換材質編譯後**必須還原**當前選片材質，否則永遠畫最後一份（症狀：選什麼都是 iris）。
  2. 光暈材質**刻意 `transparent:false`**：transparent 會進透明 pass 畫在所有不透明物之後，把切片洗白；不透明 pass 的 additive 才吃 renderOrder（table 0 → glow 1 → slice 2，painter's order）。
  3. 光桌/發光類 shader 的尾巴要壓到線性 <0.01：tonemap+gamma 會把 0.03 放大成 52/255 的整螢幕灰霧。發光 quad 的 dither 用乘法（`a*=1+n`）不用加法，否則 quad 邊界在黑底上浮出灰框。

## 3. Iris 繞射彩虹（本作技術核心）

光柵方程近似：`λ = d_eff(r)·|sv|`，`sv = dot(帶法向, vLocal.xy)`。
- `d_eff = uD0·(1 + uDGrad·(r-0.5) + 0.15·fbm)` **沿半徑漸變** → 同傾角不同半徑不同 λ = 彩虹梯度帶，傾角變 → 整帶沿半徑平移 =「掃過」。
- 相機基礎視角 `REST_ANGLE≈13°`，tilt=0 時 λ 落在紫外 → 無彩虹；傾 10–35° 掃過可見光。
- λ→RGB：三峰 Gaussian `spectral()`（出可見光自動熄滅）＋ m=2 級 `spectral(λ/2)×0.3`。
- **防油膜感四道收斂**：只在細帶區（gate）／色相沿帶法向有序漸變不隨機染斑／去飽和 mix 0.30＋uIrisGain 壓暗／單視角只出窄段光譜。
- 調參對照 `參考圖\` 錨點照（規格見該資料夾 README）。

## 4. 程式地圖（index.html 內用 [SEC:xxx] 註解區段劃界，Grep 定位）

| 想改什麼 | 找這個 |
|---|---|
| hash 參數/驗證旗標 | `[SEC:HASH]` |
| 中英雙語 | `[SEC:I18N]`（`UI_LANG`/`tx()`/`I18N_STATIC`/`applyStaticLang`；**加新 UI 文字一律走這套**） |
| 標本資料/滑桿定義（加新切片改這） | `[SEC:SLICES]` `SLICES` 陣列 |
| 切片/光桌/光暈 shader | `[SEC:GLSL]`（`FRAG`/`TABLE_FRAG`/`GLOW_FRAG`；切片外形=各 define 的 `shapeSD`） |
| 場景/材質建立 | `[SEC:SCENE]`（`sliceMats`/`precompile`） |
| 互動手感 | `[SEC:INPUT]`（pointer events，pinch）＋`[SEC:STATE]`（clamp 範圍） |
| 面板/標本卡/切換 | `[SEC:UI]`（全部由 SLICES 驅動：`setSlice`/`buildSliders`/`renderSpecimen`） |
| 玻璃降級 | `[SEC:GLASS]`（supports 偵測→fps 哨兵自動→手動 toggle，localStorage `lumenlens.glass`） |
| fps 哨兵/debug HUD | `[SEC:FPS]`（<45fps 連續 3 秒：先關玻璃、再降 pixelRatio 1.5） |
| 光暈染 UI 的顏色 | `[SEC:GLOWCOLOR]`（Iris 傾斜時 JS 端 `spectralJS` 跟 shader 同步光譜色） |
| P1/P2 擴充 | `[SEC:MODES]`（P1 星石=註冊新模式；P2=獨立 `octahedron.html`） |
| 蛋面（M9a） | `[SEC:GLSL]` 的 `#ifdef DOME` 區塊（`domeSample`/橢球法線常數 a=b=1.06,c=0.60 與 `domeGeo` 綁定）＋`[SEC:SCENE]` `domeGeo`/`domeMats`/`sliceGroup`＋`[SEC:UI]` `applyDome`/`domeSupported`/色散滑桿（buildSliders 尾段）；SLICES 加 `dome:true` 即支援新切片 |
| 蛋面色散 | `uDispStr`（藝術誇張量，物理 Δn 在蛋面尺度 <1px 不可見）＋`uDisp` 開關（哨兵第一段降級；hash `disp=0`） |
| 主迴圈/姿態/uniform 更新 | `[SEC:LOOP]` |
| CSS：玻璃/LOGO/手機版 | `[SEC:CSS-GLASS]`/`[SEC:CSS-LOGO]`（LOGO SLOT 註解=可替換插槽）/`[SEC:CSS-MOBILE]` |

## 5. 狀態（隨開發更新）

- M0–M5 完成並截圖驗證（2026-07-16）：場景/互動/三切片 shader/Iris 繞射彩虹（tilt −25° 光譜弧帶成立，符合「掃過」機制；相機基礎俯角 ≈12.6°，正傾 ~+13° 附近是彩虹死區、負傾出彩快——物理性質，不是 bug）
- M6 UI/i18n 實作完成（`#lang=en` 驗證 hash 可用）
- M7 直向自動退鏡頭（`_fitK`）已加；fps 哨兵/no-glass 降級已實作（`#debug` 下不自動降級）
- M8：README/LICENSE/.gitignore 已建；OG 文案是【人類決】佔位；**尚未建 git repo/未推 Pages**（用戶 GitHub Desktop 操作）
- **錨點照第一輪調參已做**（2026-07-16，用戶已丟 21 張參考圖進 `參考圖\` 含【參照】筆記）：trapiche=細臂/小六角核/濃郁祖母綠/秘密花園霧域/扇區明暗差；liddi=六方輪廓/粉色 Mercedes 星線/橄欖外圈/黑殼/新色序 palette；iris=乳白冷色體/琥珀 crust/玉髓核/帶振幅由內向外。**第二輪等用戶看成品後截圖標註再收斂**
- 星石×4、GIA×4 參考圖已在 `參考圖\`（P1/P2 開工直接用）
- **M9a 蛋面完成第一輪**（2026-07-16）：達碧茲半球折射（函數式真折射：橢球 refract→ray-plane 求交→重算圖案）＋三波長色散滑桿＋Fresnel/果凍 rim＋哨兵降級鏈（色散→玻璃→pixelRatio）。決策與技術脈絡見 `3D折射方向_討論稿.md`。**未上線**（用戶看過成品再推）。M9b 瑪瑙厚牆/M9c 後處理/M9d 萬花筒未做（萬花筒緩議）
- **蛋面類 shader 的截圖驗證**：headless Chrome 會逾時（SwiftShader 扛不住），改走「隱藏分頁 pixelRatio 0.5 同步 render → canvas.toDataURL → fetch POST 到本機一次性 HttpListener（scratchpad `recv.ps1`，port 8766）→ Read jpg」
- 【懸置】Logo 正式視覺（CSS placeholder 在跑）；OG 文案（README/index.html 有插槽註解）

## 6. 用戶背景備忘

- 程式小白，但寶石礦物學/金工/Rhino 底子，3D 幾何講得通；寶石學術語當同行講。
- Vibe 工作法：Code 全包並自動驗證；給選項 A/B/C 不要開放式問題；繁中、結論先講。
- 推送用 GitHub Desktop GUI（不走 git CLI）；部署 GitHub Pages 純靜態。
- 視覺類決策（鏡頭/構圖）永遠先問；用戶的參考圖是正式答案。
