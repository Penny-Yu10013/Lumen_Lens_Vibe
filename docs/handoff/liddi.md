# 李迪克碧璽切片交接（liddi）

- **定位**：Liddicoatite 碧璽（id:'liddi'，`SLICE_LIDDI`）＝色帶現象；目前只有 P0 薄切片版（單 quad SDF＋色序 palette）。
- **目前版本**：P0 薄切片，已隨 P0 推 GitHub Pages；無蛋面／厚牆版本。
- **下一步**：厚牆切片，工法同瑪瑙 M9b，排在瑪瑙之後；開工前先讀 `完稿標準_光學標本.md`（gitignore 僅本機）＝後續光學標本案例的施工與驗收規範。

> 總表在專案根 `CLAUDE.md`；若走蛋面路線則另讀 `dome-shared.md`（SLICES 加 `dome:true` 即接上）。厚牆工法先看 `iris.md`（M9b 做完後此處補連結）。

## 錨點參考（`參考圖\`，規格見該資料夾 README）

- liddi=六方輪廓/粉色 Mercedes 星線/橄欖外圈/黑殼/新色序 palette

## 程式地圖

- 本體＝`[SEC:GLSL]` `FRAG` 的 `SLICE_LIDDI` define（`shapeSD` 六方外形＋色帶 palette）；滑桿定義在 `[SEC:SLICES]`。目前沒有專屬 DOME 區塊。

## 狀態史

- P0 三切片之一（M0–M5 完成並截圖驗證，2026-07-16）；錨點照第一輪調參已做（見上方錨點參考）；第二輪等用戶看成品後截圖標註再收斂。
- 厚牆版未開工。
