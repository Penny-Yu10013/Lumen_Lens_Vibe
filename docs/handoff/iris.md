# 彩虹瑪瑙切片交接（iris）

- **定位**：Iris agate 彩虹瑪瑙（id:'iris'，`SLICE_IRIS`）＝繞射現象，本作技術核心；目前為 P0 薄切片版（單 quad SDF＋光柵方程近似）。
- **目前版本**：P0 薄切片，已隨 P0 推 GitHub Pages；M9b 厚牆未做。
- **下一步**：M9b 厚牆切片，光學標本待做順位第一；開工前先讀 `完稿標準_光學標本.md`（gitignore 僅本機）＝後續光學標本案例的施工與驗收規範。

> 總表在專案根 `CLAUDE.md`；M9c 後處理／M9d 萬花筒（緩議）的排程在總表第 5 節。

## 錨點參考（`參考圖\`，規格見該資料夾 README）

- iris=乳白冷色體/琥珀 crust/玉髓核/帶振幅由內向外

## Iris 繞射彩虹（本作技術核心，原 CLAUDE.md 第 3 節）

光柵方程近似：`λ = d_eff(r)·|sv|`，`sv = dot(帶法向, vLocal.xy)`。
- `d_eff = uD0·(1 + uDGrad·(r-0.5) + 0.15·fbm)` **沿半徑漸變** → 同傾角不同半徑不同 λ = 彩虹梯度帶，傾角變 → 整帶沿半徑平移 =「掃過」。
- 相機基礎視角 `REST_ANGLE≈13°`，tilt=0 時 λ 落在紫外 → 無彩虹；傾 10–35° 掃過可見光。
- λ→RGB：三峰 Gaussian `spectral()`（出可見光自動熄滅）＋ m=2 級 `spectral(λ/2)×0.3`。
- **防油膜感四道收斂**：只在細帶區（gate）／色相沿帶法向有序漸變不隨機染斑／去飽和 mix 0.30＋uIrisGain 壓暗／單視角只出窄段光譜。
- 調參對照 `參考圖\` 錨點照（規格見該資料夾 README）。

## 程式地圖

- 本體＝`[SEC:GLSL]` `FRAG` 的 `SLICE_IRIS` define；光暈染 UI 顏色同步在 `[SEC:GLOWCOLOR]`（Iris 傾斜時 JS 端 `spectralJS` 跟 shader 同步光譜色，總表程式地圖有列）；滑桿定義在 `[SEC:SLICES]`。目前沒有專屬 DOME 區塊。
- 驗證要點：headless Chrome 截圖時 iris shader 最慢（見總表第 1 節 `--virtual-time-budget` 註記）。

## 狀態史

- P0 三切片之一（M0–M5 完成並截圖驗證，2026-07-16）：tilt −25° 光譜弧帶成立，符合「掃過」機制；相機基礎俯角 ≈12.6°，正傾 ~+13° 附近是彩虹死區、負傾出彩快——物理性質，不是 bug。
- 錨點照第一輪調參已做（見上方錨點參考）；第二輪等用戶看成品後截圖標註再收斂。
- M9b 厚牆未開工。
