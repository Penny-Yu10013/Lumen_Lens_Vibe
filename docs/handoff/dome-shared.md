# 蛋面通用交接（dome-shared）

- **定位**：蛋面（M9a）所有標本共用的機制——DOME 折射區塊、色散、前光、limb brightening、球面形體陰影、邊緣濕亮線、法線雙軌制、蛋面截圖驗證方法、星石/亞歷共用手電筒 UI。
- **目前版本**：v5.1 收尾兩修已 commit+push；隨各標本推進（達碧茲 v6.1／星石 v5／亞歷 v5.2，全部已推 GitHub Pages，commit e485cc5）。
- **下一步**：無獨立待辦。新蛋面標本（李迪蛋面等）在 SLICES 加 `dome:true` 即接上本檔機制；開工前先讀 `完稿標準_光學標本.md`（gitignore 僅本機）。

> 標本專屬內容在 `trapiche.md` / `star.md` / `alex.md`；總表在專案根 `CLAUDE.md`。

## 程式地圖（蛋面通用 row，index.html 內用 [SEC:xxx] 註解區段劃界，Grep 定位）

| 想改什麼 | 找這個 |
|---|---|
| 蛋面（M9a） | `[SEC:GLSL]` 的 `#ifdef DOME` 區塊（`domeSample`；橢球 a=b=1.06 固定、c=`uDomeH` 可調）＋`[SEC:SCENE]` `domeGeo`/`applyDomeH`（高度滑桿=CPU 頂點重縮放+uniform，非每幀）/`domeMats`/`sliceGroup`＋`[SEC:UI]` `applyDome`/`domeSupported`/色散+高度滑桿（buildSliders 尾段）；SLICES 加 `dome:true` 即支援新切片 |
| 蛋面色散 | `uDispStr`（藝術誇張量，物理 Δn 在蛋面尺度 <1px 不可見）＋`uDisp` 開關（哨兵第一段降級；hash `disp=0`） |
| 法線雙軌制（蛋面） | **法線雙軌制（架構決定，別改回去）**：`nl0`=乾淨橢球法線餵折射/domeSample/wallOD/Fresnel/形體陰影，`nl`=溝擾動後**只餵反光層**（spec/halo/前光/貓眼/AO）——溝若扭曲折射，臂會被抹成閃電折線或花瓣暗斑（兩輪用戶回報的總根源） |
| 球面形體陰影（蛋面） | DOME 主函數 `form`（mix 0.26–1 × smoothstep(ndi 0.10–0.44)）只乘透射體＝外環暗帶（球體素描）；Fresnel 反光/rim glow/前光弧不吃 form（【人類決】最外環反光保留）。後接**邊緣濕亮線**：`pow(1-ndi,14)`×背光＝拋光腰稜細高光弧，不吃 grooveAO（拋光面連續，亮線越過臂端） |
| limb brightening（蛋面） | **limb brightening**＝`pow(1-ndi,6)`×光源方位遮罩×`uRimGlow`（fresnel 邊緣連續漸亮；**月牙/弧帶類貼片全禁**——「天使光環弧帶」在星石/亞歷已 `halo=0.0` 撤除，僅達碧茲保留） |
| 前光（天使光環，蛋面限定） | DOME 區塊 `front` 項：**輪廓座標系弧帶**（`vWorldPos.xy` 半徑 0.50–0.84＋`uKeyDir` xy 投影方位 gate），**不是法線空間 lobe——扁蛋面上必糊成整片光帽（踩過兩輪）**。JS 每幀餵 `uKeyDir`：跟隨式＝相機方向 y+1.25、固定式＝(-0.35,0.45,0.82)。UI＝前光滑桿＋跟隨 checkbox（`[SEC:UI]`，蛋面模式才顯示）；hash `front=`/`keyfix=1`；localStorage `lumenlens.keyfollow` |
| 手電筒 UI（星石/亞歷共用） | 手電筒 UI 現為星石/亞歷共用（星石=移反射光源、亞歷=暖束）。達碧茲專屬 dome 滑桿已 gate（buildSliders `if(s.id==='trapiche')`）。實作細節：星石端見 `star.md`「星光藍寶」row 的手電筒模式、亞歷端見 `alex.md` 的混色手電筒 |

## 蛋面截圖驗證

- **蛋面類 shader 的截圖驗證**：headless Chrome 會逾時（SwiftShader 扛不住），改走「隱藏分頁 pixelRatio 0.5 同步 render → canvas.toDataURL → fetch POST 到本機一次性 HttpListener（scratchpad `recv.ps1`，port 8766）→ Read jpg」
- 一般驗證流程（launch.json / `#debug` `window.__ll` probe / hash 參數）見總表 `CLAUDE.md` 第 1 節。

## 狀態史（蛋面通用）

- **v5.1 收尾兩修**（用戶標註）＝①扁蛋面白殼根除：dome mesh xy 半徑 1.06＞外形 ~0.95，低 uDomeH 折射近直穿、外圈透視「外形外＝白光桌」fallback＝寬灰白殼（每個蛋面都有）——`domeSample` 出界採樣沿 shapeSD 拉回石緣（材質延伸；預設高度 transect 迴歸零差異）；②EN chips 破版：300px 卡 5 顆英文長名 flex 均分 46px 必爆——`#chips.en` 3 欄網格＋10px 字級＋允許折行（規則放 CSS-MOBILE media query 後面蓋掉手機 nowrap），buildChips 依 UI_LANG 掛 class。已 commit+push。
- 真機基準：桌機 53fps@pxr1.25（哨兵有降）、iPhone 60fps@pxr2 蛋面全開
- M9a 蛋面第一版（半球折射／色散／哨兵降級鏈／高度滑桿）是在達碧茲身上做出來的，該段狀態史保留在 `trapiche.md`。
