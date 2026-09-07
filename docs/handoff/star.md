# 星光藍寶蛋面交接（star）

- **定位**：星光藍寶（P1，id:'star'，標本卡第二位）＝asterism 星光現象；反射光路六芒星＋手電筒移光＋剛玉色系滑桿＋星心點狀化。
- **目前版本**：P1 v5（含 P1 v4–v5 星石批改），已推 GitHub Pages（commit 4180c4e 起，最新 e485cc5）。
- **下一步**：無待辦；待用戶截圖標註批改。

> 蛋面共用機制（DOME、前光、limb brightening、手電筒 UI 共用）見 `dome-shared.md`；總表在專案根 `CLAUDE.md`。科研依據＝`星光藍寶石_科研報告.md`（gitignore）。

## 程式地圖（star 專屬 row）

| 想改什麼 | 找這個 |
|---|---|
| 星光藍寶（P1，id:'star'） | 本體＝sliceLinear `#ifdef SLICE_STAR`（乳藍剛玉：uMilk 乳霧＋uZoneAmt 六方環帶＋吸紅透藍 sigma）；星光＝DOME 尾段 `#ifdef SLICE_STAR`：三組針方向**局部座標常數**（自旋星臂自動跟轉）、`u=nl−H·dot(nl,H)`（反射點 u=0＝光動星動）、帶k=exp(−dot(u,fk)²·1600)⊥fk 過星心、三帶交會＝六芒星＋taper＋星心乳暈；強度＝uStarInt×前光（**反射光路**，與達碧茲透射相反）。**色系滑桿 uStarTint**（0藍→0.34紫→0.67紅寶→1黑星；黑星端星線自動轉古銅金＝赤鐵礦絲）＋**星線銳利度 uStarSharp**（縮放帶指數，1=用戶認可預設）。**手電筒模式**【人類決 C 案】：`state.flash`＋`_ptrX/_ptrY`（canvas pointermove 永遠追蹤）→ 主迴圈 uKeyDir=游標方向（優先序：手電筒＞跟隨＞固定）；桌面 hover 移光拖曳轉石、**觸控單指拖=移光不轉石【AI 代決可改】**；hash `flash=1`、localStorage `lumenlens.flash`、`#flashRow` 星石/亞歷顯示（v5 起亞歷＝暖束混色，見亞歷 row）。科研依據＝`星光藍寶石_科研報告.md`（gitignore） |
| 蛋面光點/limb brightening（星石批改） | 星心＝真鏡面 `pow(dot(nl,H),320)`＋2 副點（**u 空間偏移 0.2/0.28、微偏離帶軸**——正壓臂上會被臂亮度吞掉、H 偏移太小會縮進主點，都踩過）；星帶 **max 不 sum**（交會不自加亮）；**受光半球 gate `hemiS`**（u 空間高斯不分 θ/180°−θ，反位半球會鬼影出對角線微光，踩過）。limb brightening（`uRimGlow` 通用項）見 `dome-shared.md` |

## 狀態史（v1→v5）

- **P1 星光藍寶 v1 已實作**（2026-07-17，標本卡第二位；先科研：`星光藍寶石_科研報告.md`）：四行為驗證✓（星隨光走/自旋星臂跟轉/傾斜星平移/uStarInt 0 全滅）。星線預設粗細已獲用戶認可。**P1 v2**（同日）＝手電筒模式（C 案：toggle 並存，限星石）＋剛玉色系滑桿（藍/紫/星光紅寶/黑星，黑星金線）＋星線銳利度滑桿。**已推 GitHub Pages**。**P1 v3 星石批改**＝星心點狀化（鏡面反射點＋2 副點，非柔光帽）＋輪廓月牙光暈（uRimGlow 通用項）——四行為截圖驗證✓。
- **P1 v4–v5＋亞歷 v2–v3**（用戶末輪批改全修完）：月牙真兇＝v3 天使光環弧帶（星石/亞歷撤除）、limb brightening 接手邊緣光；星心真鏡面點＋副點（u 空間 0.2/0.28 微偏帶軸）＋星帶 max＋受光半球 gate（殺對角線鬼影）；亞歷 milk&honey 統一 ua 座標系（keyL.x 定側，禁每像素 H.x）＋無中心光點。星石 v3–v5＋亞歷 v1–v3 已推 GitHub（commit 4180c4e）。
- 星石手電筒迴歸 ✓（亞歷 v5 混色手電筒導入時驗證，見 `alex.md`）。
