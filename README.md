# Lumen Lens 寶石透鏡

> A black-table observatory for the *inside* of gemstones.
> 黑底光桌上的寶石光學現象觀察台——刻面退居配角，內部世界是主角。

<!-- 【人類決】正式宣傳文案/截圖/Pages 網址上線前由用戶定稿，以下為佔位結構 -->

## 現在可以看什麼（P0 光桌切片模式）

| 標本 | 現象 |
|---|---|
| 達碧茲祖母綠 Trapiche Emerald | 六個生長扇區邊界的碳質包裹體——透光現六輻黑臂 |
| 李迪克碧璽 Liddicoatite | 垂直 c 軸切片的三重對稱同心色帶靶心 |
| 彩虹瑪瑙 Iris Agate | 紋帶細到接近光波長＝天然繞射光柵，**傾到對的角度彩虹才現身** |

## 怎麼玩

拖曳＝旋轉標本｜上下拖曳＝傾斜｜滾輪/雙指＝縮放｜背光滑桿＝光桌亮度。
彩虹瑪瑙請開背光、慢慢傾斜——彩虹是「掃」出來的。

## 技術

單一 HTML 檔＋Three.js＋自訂 ShaderMaterial。所有內部結構皆為程序化圖案數學（fragment shader），零外部資產、零 build。

- 繞射彩虹：光柵方程近似 `λ = d(r)·sinθ`，帶間距沿半徑漸變 → 彩虹隨傾角沿半徑掃動
- 透射：Beer–Lambert，傾斜時光程變長、顏色自然變濃

## 相關

- 姊妹作 v1（動手切）：[isekai_gemcraft](https://penny-yu10013.github.io/isekai_gemcraft/) — 寶石切割模擬器
- 本作 v2（看內在）：寶石內部結構觀察台

## License

GPL-3.0
