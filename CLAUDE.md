# Lumen Lens 寶石透鏡 — 開發交接文件 (CLAUDE.md)

> 給未來 session：這是**目前實作狀態**的交接。原始提案在 `Lumen_Lens_Vibe提案表.md`（不要改，當願景參考）。
> 前作（架構複用來源）：`C:\Users\yu2_7\Downloads\isekai_gemcraft_claude\`（v1=切割幾何；本作 v2=內部結構與光學現象）。

---

## 0. 一句話

黑底寶石光學現象觀察台：切片放光桌上透光看，拖曳旋轉/傾斜，彩虹瑪瑙傾到對的角度才出彩虹。單一 `index.html`，Three.js r128 CDN，全程序化零外部資產。

### 0.1 為什麼是 shader 模擬，不是「騙眼睛」（2026-07-17，【人類決】）

內部結構沒有對應的立體 mesh，是像素級算出來的——這是效能取捨，不是要唬弄誰。寶石不像機械零件能 3D 印出來練手感，真要學等比例的內部光學現象，得買研磨機或真的達碧茲標本、架顯微鏡，鑑定師課程又貴又要熬年資（30–40 歲才有機會拿到證照，還要英文好）。這筆錢跟時間拿來在 20 幾歲用 vibe coding 探索興趣，門檻低很多。**程序化 shader 逼近物理公式，是在「印不出寶石」這個現實限制下最接近的替代方案，不是取巧**——這決定了整個專案的技術路線，別因為外部質疑就懷疑這個方向。

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
| 蛋面（M9a） | `[SEC:GLSL]` 的 `#ifdef DOME` 區塊（`domeSample`；橢球 a=b=1.06 固定、c=`uDomeH` 可調）＋`[SEC:SCENE]` `domeGeo`/`applyDomeH`（高度滑桿=CPU 頂點重縮放+uniform，非每幀）/`domeMats`/`sliceGroup`＋`[SEC:UI]` `applyDome`/`domeSupported`/色散+高度滑桿（buildSliders 尾段）；SLICES 加 `dome:true` 即支援新切片 |
| 蛋面色散 | `uDispStr`（藝術誇張量，物理 Δn 在蛋面尺度 <1px 不可見）＋`uDisp` 開關（哨兵第一段降級；hash `disp=0`） |
| 透射貓眼（trapiche 蛋面限定） | DOME 尾段 `#ifdef SLICE_TRAPICHE`：**透射現象不是反光**（科研報告 §4-bis 大衛之星）——徑向位錯束散射穿石背光，眼線＝零跡線 `aE=dot(rdn.xy,fT)+dot(LiE,fT)`（折射視線＋光桌點光扇；**平行光假設會讓零跡線塌在圓心**，踩過）。銳線＋柔暈雙 Gaussian×增益 4.0；亮度載體 `trans.g`＝臂切開六眼/背光驅動/前光無關全免費。`uCatEye` 滑桿（hash `cat=`）；**LiE 係數 0.19＝眼線半徑，probe 校準過別憑感覺調**。反射貓眼已拆除勿復活。臂消光＝`grooveAO` 0.82（碳質縫隙=光陷阱，亮壁玻璃管感的根源）。同段後面＝裂隙碎光（前光×uCrackAmt/uCrackSpread，hash `crack=`） |
| 核心尺寸/型態（trapiche 蛋面） | `uCoreSize` 滑桿（0.06–0.28）＝等效選切位（核是錐形），wallOD 的 hex 常數全掛鉤；頂點六壁邊界＝表層 hex 輪廓線（@ uCoreSize*0.95）；臂緣銳利度綁 uEnrich（edgeK）；B 型外緣碳質暗邊（sliceLinear rimC × uEnrich）。科研依據＝`達碧茲晶體_科研報告.md`（gitignore） |
| 六臂 V 溝（trapiche 蛋面限定） | DOME 區塊開頭 `#ifdef SLICE_TRAPICHE` V 溝層：法線沿垂直臂軸切向倒向兩壁（帶號弧距 `sG` 定側、溝寬跟 `uArmWidth`）＋`grooveAO` 壓暗反射項。**法線雙軌制（架構決定，別改回去）**：`nl0`=乾淨橢球法線餵折射/domeSample/wallOD/Fresnel/形體陰影，`nl`=溝擾動後**只餵反光層**（spec/halo/前光/貓眼/AO）——溝若扭曲折射，臂會被抹成閃電折線或花瓣暗斑（兩輪用戶回報的總根源） |
| 球面形體陰影（蛋面） | DOME 主函數 `form`（mix 0.26–1 × smoothstep(ndi 0.10–0.44)）只乘透射體＝外環暗帶（球體素描）；Fresnel 反光/rim glow/前光弧不吃 form（【人類決】最外環反光保留）。後接**邊緣濕亮線**：`pow(1-ndi,14)`×背光＝拋光腰稜細高光弧，不吃 grooveAO（拋光面連續，亮線越過臂端） |
| 實體六壁＋核柱（trapiche 蛋面限定） | `wallOD`（domeSample 後面）：蛋面下臂/富集/核**不畫底面**（sliceLinear 內 `#ifndef DOME` 守衛），改對折射線解析求交積分——3 條垂直牆板＋海星蓋（近入射點採樣的頂面表層，俯視=六爪星、斜視=頂點暗蓋；**不能做成全深牆板加寬**＝側視大寬翼）＋六角蜂巢核柱（hexRim 壁濃內部淡＋`depthW` 垂直漸層頂暗底透＋吸收色透綠不透紅藍＝深處墨綠）。**結構性圖案必須體積化**：底面採樣會被 V 溝折射踢成閃電形轉折（踩過）；質地性圖案（花園霧/生長紋）留底面沒事。牆積分只用主波長算一次；臂不能吃單點 z 權重（垂直視線 tj 退化會單邊變暗，踩過） |
| 星光藍寶（P1，id:'star'） | 本體＝sliceLinear `#ifdef SLICE_STAR`（乳藍剛玉：uMilk 乳霧＋uZoneAmt 六方環帶＋吸紅透藍 sigma）；星光＝DOME 尾段 `#ifdef SLICE_STAR`：三組針方向**局部座標常數**（自旋星臂自動跟轉）、`u=nl−H·dot(nl,H)`（反射點 u=0＝光動星動）、帶k=exp(−dot(u,fk)²·1600)⊥fk 過星心、三帶交會＝六芒星＋taper＋星心乳暈；強度＝uStarInt×前光（**反射光路**，與達碧茲透射相反）。**色系滑桿 uStarTint**（0藍→0.34紫→0.67紅寶→1黑星；黑星端星線自動轉古銅金＝赤鐵礦絲）＋**星線銳利度 uStarSharp**（縮放帶指數，1=用戶認可預設）。**手電筒模式**【人類決 C 案，限星石】：`state.flash`＋`_ptrX/_ptrY`（canvas pointermove 永遠追蹤）→ 主迴圈 uKeyDir=游標方向（優先序：手電筒＞跟隨＞固定）；桌面 hover 移光拖曳轉石、**觸控單指拖=移光不轉石【AI 代決可改】**；hash `flash=1`、localStorage `lumenlens.flash`、`#flashRow` 只在星石顯示。科研依據＝`星光藍寶石_科研報告.md`（gitignore）。達碧茲專屬 dome 滑桿已 gate（buildSliders `if(s.id==='trapiche')`） |
| 亞歷山大貓眼（id:'alex'） | 本體＝sliceLinear `#ifdef SLICE_ALEX`：**Cr³⁺ 雙透射窗必須分 4 頻帶採樣**（650/583/515/455 過 `spectral()` 合成——RGB 三通道塞不下「紅綠都透、黃吸收」）；吸收固定、光源權重 `uWarm` 0/1 切換（日光/燭光**兩鍵瞬切不做滑桿**【人類決】，`#lightSrcRow` 限 alex 顯示，hash `warm=1`）；光源權重差距要敢拉、燭光藍端補超物理量才會紫紅不橘棕（踩過兩輪）。貓眼＝DOME `#ifdef SLICE_ALEX`，v4 批改後＝**|ua| 走廊剖面**（跟眼線同座標：核心線→乳光走廊 corrG=exp(−uaP²·7)→深陰影翼；`prof=mix(0.18,1.25,corrG)`——**翼底 0.18 是 tonemap 反推值**，col/(1+col)+gamma 吃對比極兇（0.58 只剩螢幕比 0.73），要壓到螢幕比 0.55–0.6 得給 0.18，probe 校準過別用線性直覺調）＋**milk & honey＝剖面整體往光側平移 0.09＋光側翼 ±0.18 偏亮**（`sideW`＝clamp(keyL.x·1.7)，keyL.x **每幀常數**——每像素 H.x 會生直線弦切雙色皮球，踩過；正面光 keyL.x≈0 自動對稱）＋**眼暈變色**（暈色＝`haloC`＝trans 色度 `bodyT` 混 25% 白＝體色乳光，日光綠乳/燭光粉乳自動跟 uWarm；核心線 mix(white,haloC,0.18) near-white）＋走廊染色乳光加法項（corrG·bodyT·0.10）＋雙層帶（K45000/K600 × `uAlexSharp²` 眼線銳利度滑桿）＋縱向低頻調變＋放射狀邊緣壓暗＋受光半球 gate＋`eyeK=min(uAlexEye,1)` gate 整個剖面（eye=0 全滅）。**頭燈根除（v4）：alex 的 spec/front 整項＝0**——0.22 倍壓不掉，pow240 鏡面點（spec＋front 各一顆）在眼線上段聚成獨立白球；鏡面點是星石規格，亞歷的光只有一條線。絲光條紋振幅 0.08 別拉高（會變木紋珠）。科研依據＝`亞歷山大貓眼_科研報告.md`（gitignore） |
| 蛋面光點/limb brightening（星石批改） | 星心＝真鏡面 `pow(dot(nl,H),320)`＋2 副點（**u 空間偏移 0.2/0.28、微偏離帶軸**——正壓臂上會被臂亮度吞掉、H 偏移太小會縮進主點，都踩過）；星帶 **max 不 sum**（交會不自加亮）；**受光半球 gate `hemiS`**（u 空間高斯不分 θ/180°−θ，反位半球會鬼影出對角線微光，踩過）。**limb brightening**＝`pow(1-ndi,6)`×光源方位遮罩×`uRimGlow`（fresnel 邊緣連續漸亮；**月牙/弧帶類貼片全禁**——「天使光環弧帶」在星石/亞歷已 `halo=0.0` 撤除，僅達碧茲保留） |
| 前光（天使光環，蛋面限定） | DOME 區塊 `front` 項：**輪廓座標系弧帶**（`vWorldPos.xy` 半徑 0.50–0.84＋`uKeyDir` xy 投影方位 gate），**不是法線空間 lobe——扁蛋面上必糊成整片光帽（踩過兩輪）**。JS 每幀餵 `uKeyDir`：跟隨式＝相機方向 y+1.25、固定式＝(-0.35,0.45,0.82)。UI＝前光滑桿＋跟隨 checkbox（`[SEC:UI]`，蛋面模式才顯示）；hash `front=`/`keyfix=1`；localStorage `lumenlens.keyfollow` |
| 主迴圈/姿態/uniform 更新 | `[SEC:LOOP]` |
| CSS：玻璃/LOGO/手機版 | `[SEC:CSS-GLASS]`/`[SEC:CSS-LOGO]`（LOGO SLOT 註解=可替換插槽）/`[SEC:CSS-MOBILE]` |

## 5. 狀態（隨開發更新）

- M0–M5 完成並截圖驗證（2026-07-16）：場景/互動/三切片 shader/Iris 繞射彩虹（tilt −25° 光譜弧帶成立，符合「掃過」機制；相機基礎俯角 ≈12.6°，正傾 ~+13° 附近是彩虹死區、負傾出彩快——物理性質，不是 bug）
- M6 UI/i18n 實作完成（`#lang=en` 驗證 hash 可用）
- M7 直向自動退鏡頭（`_fitK`）已加；fps 哨兵/no-glass 降級已實作（`#debug` 下不自動降級）
- M8：README/LICENSE/.gitignore 已建；OG 文案是【人類決】佔位；**尚未建 git repo/未推 Pages**（用戶 GitHub Desktop 操作）
- **錨點照第一輪調參已做**（2026-07-16，用戶已丟 21 張參考圖進 `參考圖\` 含【參照】筆記）：trapiche=細臂/小六角核/濃郁祖母綠/秘密花園霧域/扇區明暗差；liddi=六方輪廓/粉色 Mercedes 星線/橄欖外圈/黑殼/新色序 palette；iris=乳白冷色體/琥珀 crust/玉髓核/帶振幅由內向外。**第二輪等用戶看成品後截圖標註再收斂**
- 星石×4、GIA×4 參考圖已在 `參考圖\`（P1/P2 開工直接用）
- **M9a 蛋面完成兩輪**（2026-07-16）：v1=達碧茲半球折射（函數式真折射：橢球 refract→ray-plane 求交→重算圖案）＋三波長色散滑桿＋Fresnel/果凍 rim＋哨兵降級鏈（色散→玻璃→pixelRatio）。v2（v1 驗收未過補做）=**六扇區獨立貓眼絲光**（反光層，隨拖曳掃動，六道分開=硬驗收條件）＋**蛋面高度滑桿**（uDomeH 0.35–1.05 預設 0.72）＋色散上限 0.12。決策與技術脈絡見 `3D折射方向_討論稿.md`（已 gitignore，僅本機）。**v3**（2026-07-17，v2「大致完美，上修一點」的兩個修改點）＝六臂 V 溝立體化（法線擾動＋grooveAO）＋前光「天使光環」（輪廓座標系弧帶；【人類決】跟隨視角先行、固定燈位保留為面板選項）。v3 俯視角已獲用戶認可；**v3.1**＝臂/核體積化（`wallOD` 實體六壁＋頂點核柱，修大傾角閃電形轉折，用戶確認有進步）；**v3.2**＝中心結構上修（海星蓋＋六角蜂巢核＋垂直明暗頂暗底透墨綠，對照用戶背光標本照）；**v3.3**＝低參數水墨修正（臂邊緣改比例式，禁用固定 +0.03 常數）＋核柱增實（depthW 0.72–1.90）＋臂末端破圖修正（蛋面關磨邊亮線＋V 溝邊緣提早淡出）；**v3.4**＝法線雙軌制（V 溝退出折射路徑，根治花瓣/閃電類 artifact）＋海星縮小柔化＋球面形體陰影（外環暗帶、最外環反光保留）；**v3.5**＝貓眼改前光驅動＋帶形放寬＋裂隙碎光；**v3.6**（先科研再修改）＝`達碧茲晶體_科研報告.md`（gitignore，結論→shader 映射表）＋貓眼弧帶修正（lobe 22、獨立 uCatEye 滑桿與前光脫鉤）＋頂點六壁 hex 輪廓＋裂隙雙滑桿＋核尺寸滑桿（B↔C 偏 B）＋臂緣銳利綁富集＋外緣碳質暗邊。**v3.7**（用戶選 A 路線「遵循物理與礦物學」）＝透射貓眼（拆掉反射制；眼線=折射視線×光桌點光扇的散射零跡線，載體 trans.g）＋臂消光化（grooveAO 0.82）＋桌面面板限高可捲（滑桿多到頂不到背光的 bug）。v3.7 用戶驗收通過（達碧茲本體滿意）；**v3.8**＝核柱實體化（depthW 1.25–2.25＋內部加權、hex 壁對比壓低——垂直俯視 tc 夾底部拿到最低吸收＝幽靈空心感的根源）＋邊緣濕亮線＋形體陰影界線增強（用戶標本照紅/紫箭頭）。**v3.8 已推 GitHub Pages**。**P1 星光藍寶 v1 已實作**（2026-07-17，標本卡第二位；先科研：`星光藍寶石_科研報告.md`）：四行為驗證✓（星隨光走/自旋星臂跟轉/傾斜星平移/uStarInt 0 全滅）。星線預設粗細已獲用戶認可。**P1 v2**（同日）＝手電筒模式（C 案：toggle 並存，限星石）＋剛玉色系滑桿（藍/紫/星光紅寶/黑星，黑星金線）＋星線銳利度滑桿。**已推 GitHub Pages**。**P1 v3 星石批改**＝星心點狀化（鏡面反射點＋2 副點，非柔光帽）＋輪廓月牙光暈（uRimGlow 通用項）——四行為截圖驗證✓。**亞歷山大貓眼 v1 已實作**（標本卡第三位；先科研：`亞歷山大貓眼_科研報告.md`）：4 頻帶雙透射窗變色（日光/燭光兩鍵瞬切【人類決】）＋單帶貓眼＋點狀星心；行為全過、**體色飽和度偏粉彩待用戶批改**（解藥＝絲光濃度↑前光↓）。**P1 v4–v5＋亞歷 v2–v3**（用戶末輪批改全修完）：月牙真兇＝v3 天使光環弧帶（星石/亞歷撤除）、limb brightening 接手邊緣光；星心真鏡面點＋副點（u 空間 0.2/0.28 微偏帶軸）＋星帶 max＋受光半球 gate（殺對角線鬼影）；亞歷 milk&honey 統一 ua 座標系（keyL.x 定側，禁每像素 H.x）＋無中心光點。星石 v3–v5＋亞歷 v1–v3 已推 GitHub（commit 4180c4e）。**亞歷 v4**（2026-07-19，用戶三項批改）＝頭燈根除（spec/front 砍零）＋眼暈變色（暈＝體色乳光隨 uWarm 換）＋亮度剖面重做（|ua| 走廊剖面＋光側平移）＋眼線銳利度滑桿 uAlexSharp；probe 驗證全過（線上無亮團／日光綠乳燭光粉乳／翼走廊螢幕比 0.59／側光剖面平移／eye=0 全滅）——**v4 未 commit、待用戶驗收**。陣容定位：達碧茲（生長結構）/星光（asterism）/李迪（色帶）/瑪瑙（繞射）/亞歷（chatoyancy+變色）＝五現象零重複。開發順序：達碧茲→星石→亞歷→李迪→瑪瑙（次序前三完成，剩李迪/瑪瑙切片組）。**完稿標準已立**：`完稿標準_光學標本.md`（gitignore 僅本機）＝後續光學標本案例（李迪蛋面/M9b 瑪瑙/P1 星光）的施工與驗收規範，開工先讀。M9b 瑪瑙厚牆/M9c 後處理/M9d 萬花筒未做（萬花筒緩議）。真機基準：桌機 53fps@pxr1.25（哨兵有降）、iPhone 60fps@pxr2 蛋面全開
- 【人類決】Logo：用戶滿意現有 placeholder 方向，**將自繪正式視覺**，現在不動
- **蛋面類 shader 的截圖驗證**：headless Chrome 會逾時（SwiftShader 扛不住），改走「隱藏分頁 pixelRatio 0.5 同步 render → canvas.toDataURL → fetch POST 到本機一次性 HttpListener（scratchpad `recv.ps1`，port 8766）→ Read jpg」
- 【懸置】Logo 正式視覺（CSS placeholder 在跑）；OG 文案（README/index.html 有插槽註解）

## 6. 用戶背景備忘

- 程式小白，但寶石礦物學/金工/Rhino 底子，3D 幾何講得通；寶石學術語當同行講。
- Vibe 工作法：Code 全包並自動驗證；給選項 A/B/C 不要開放式問題；繁中、結論先講。
- 推送用 GitHub Desktop GUI（不走 git CLI）；部署 GitHub Pages 純靜態。
- 視覺類決策（鏡頭/構圖）永遠先問；用戶的參考圖是正式答案。
