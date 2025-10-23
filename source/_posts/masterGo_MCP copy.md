---
title: MasterGo DSL Transform 属性转换（Figma 也是如此）
date: 2024-11-03 19:54:06
tags:
  - MasterGo
  - CSS
categories: MasterGo
index_img: /img/mg-logo.png
---

### DSL 中 relativeTransform 对象

```JAVASCRIPT
[
  [0.7071067690849304, 0.7071067690849304, 64.82373046875],
  [-0.7071067690849304, 0.7071067690849304, 46.4097900390625]
]
/**
 * [
 *   [a, c, tx],
 *   [b, d, ty]
 * ]
 */
```

这是一个典型的 2×3 仿射变换矩阵（Affine Transform），与 CSS 中的 transform 几乎一样。

```CSS
  /* transform: matrix(a, b, c, d, tx, ty); */
  transform: matrix(1, 0, 0, 1, 0, 0);
```

<small>MasterGo 和 Figma 的 relativeTransform 与 CSS 中的 `transform: matrix()` 一样，Y 轴的正方向是向下；而 CSS 中的 `transform: rotate(45deg)` 的 Y 轴正方向是向上，两者不同。</small>

| 含义         | CSS `matrix(a,b,c,d,tx,ty)` | Figma `relativeTransform` |
| ------------ | --------------------------- | ------------------------- |
| 第一行第一列 | a                           | matrix[0][0]              |
| 第二行第一列 | b                           | matrix[1][0]              |
| 第一行第二列 | c                           | matrix[0][1]              |
| 第二行第二列 | d                           | matrix[1][1]              |
| 第一行第三列 | tx                          | matrix[0][2]              |
| 第二行第三列 | ty                          | matrix[1][2]              |

更清晰的对照：

```
// CSS → Figma
[
  [a, c, tx],
  [b, d, ty]
]

// Figma → CSS
matrix(a, b, c, d, tx, ty)

```

### DSL 中 relativeTransform、CSS 中 matrix 和单一属性转换

- relativeTransform2SingleAttribute

```
function relativeTransform2SingleAttribute(matrix) {
  const [[a, c, tx], [b, d, ty]] = matrix;

  const translation = { x: tx, y: ty };

  const scaleX = Math.sqrt(a * a + b * b);
  const scaleY = Math.sqrt(c * c + d * d);

  // CSS 设置单一属性时，rotate 的 Y 轴正向相反
  const rotation = -Math.atan2(b / scaleX, a / scaleX) * (180 / Math.PI);

  // 修正浮点误差
  const fix = (n) => Math.abs(n - 1) < 1e-6 ? 1 : n;

  return {
    rotation,
    scale: { x: fix(scaleX), y: fix(scaleY) },
    translation,
  };
}
```

- 在 MasterGo 中，目前存在这样的问题：变换后，DSL 导出的宽高仍是原始元素的宽高，并非外接矩形的宽高；但下载出来的元素却是外接矩形的图片，因此需要手动计算外接矩形宽高。

```
// 传入 MasterGo DSL 的矩阵
// 输出结果仍然要向上取整，因为下载的图片宽高没有小数点，均向上取整。
function getBoundingBoxAfterTransform(matrix, width, height) {
  const [[a, c, tx], [b, d, ty]] = matrix;

  // 原始矩形的四个顶点
  const points = [
    { x: 0, y: 0 },
    { x: width, y: 0 },
    { x: 0, y: height },
    { x: width, y: height },
  ];

  // 经过变换后的点
  const transformed = points.map(p => ({
    x: a * p.x + c * p.y + tx,
    y: b * p.x + d * p.y + ty
  }));

  // 求 min / max
  const xs = transformed.map(p => p.x);
  const ys = transformed.map(p => p.y);

  const minX = Math.min(...xs);
  const maxX = Math.max(...xs);
  const minY = Math.min(...ys);
  const maxY = Math.max(...ys);

  return {
    width: maxX - minX,
    height: maxY - minY,
  };
}
```

- 把 rotation + scale + translation 重新组合成 MasterGo 的 relativeTransform

```
function composeRelativeTransform({ rotation, scale, translation }) {
  const rad = -rotation * Math.PI / 180; // 取反，因为 CSS 单独设置 rotate 时的 Y 轴方向与此相反。

  const a = Math.cos(rad) * scale.x;
  const b = Math.sin(rad) * scale.x;
  const c = -Math.sin(rad) * scale.y;
  const d = Math.cos(rad) * scale.y;

  const tx = translation.x;
  const ty = translation.y;

  return [
    [a, c, tx],
    [b, d, ty],
  ];
}

```
