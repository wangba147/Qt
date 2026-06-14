# Qt 学习笔记 — QPainter

> QPainter 是 Qt 的 2D 矢量绘制引擎，承担所有 QPaintDevice 上的图形绘制工作。

---

## 1. 绘制架构

```
┌─────────────┐
│  QPainter   │ ← 画笔/画刷/字体/变换 → 命令下发
├─────────────┤
│ QPaintEngine│ ← 平台抽象（Raster / OpenGL / PDF / Print）
├─────────────┤
│QPaintDevice │ ← 物理输出（QWidget / QPixmap / QImage / QPrinter）
└─────────────┘
```

## 2. QPaintDevice 子类

| 设备类 | 用途 | 典型场景 |
|--------|------|----------|
| QWidget | 窗口/控件表面 | paintEvent() 自定义控件 |
| QPixmap | 屏幕位图缓存 | 双缓冲、图标缓存、截图 |
| QImage | 像素级图像操作 | 图像处理、像素读写 |
| QPicture | 矢量指令录制 | 序列化保存与回放 |
| QPrinter | 打印输出 | 报表打印、导出 PDF |
| QOpenGLPaintDevice | OpenGL 加速 | 3D/2D 混合渲染 |

## 3. 两种绘制入口

**paintEvent（事件驱动，最常用）**：
```cpp
void MyWidget::paintEvent(QPaintEvent *) {
    QPainter painter(this);  // 析构时自动 end()
    painter.setRenderHint(QPainter::Antialiasing);
    // ... 绘制代码
}
```

**begin/end（主动绘制到非 Widget 设备）**：
```cpp
QImage img(400, 300, QImage::Format_ARGB32);
QPainter painter;
painter.begin(&img);
painter.drawText(...);
painter.end();
```

## 4. 画笔与画刷

- **QPen**：控制轮廓 — 颜色、宽度、样式（实线/虚线/点线）、端点/连接样式
- **QBrush**：控制填充 — 纯色、线性/径向/锥形渐变、纹理图案

配置**持续生效**，直到被再次修改。

## 5. 基础绘制 API

| 方法 | 说明 |
|------|------|
| drawPoint / drawPoints | 点 |
| drawLine / drawLines | 线段 |
| drawRect / drawRoundedRect | 矩形 / 圆角矩形 |
| drawEllipse | 椭圆（矩形内切） |
| drawArc / drawChord / drawPie | 弧 / 弓形 / 扇形（角度 = 1/16 度） |
| drawPolygon / drawPolyline | 多边形 / 折线 |
| drawPath | QPainterPath 复合路径 |
| drawText | 文本 |
| drawPixmap / drawImage | 图像 |
| eraseRect | 清除区域 |

## 6. QPainterPath

支持组合运算：`united()` / `intersected()` / `subtracted()`

```cpp
QPainterPath path;
path.moveTo(20, 30);
path.lineTo(120, 30);
path.cubicTo(120, 100, 60, 20, 20, 100);
path.closeSubpath();
painter.drawPath(path);
```

## 7. 坐标变换

四种基本变换：`translate()` / `rotate()` / `scale()` / `shear()`

**变换顺序 = 反向阅读**（右乘累积）

```cpp
painter.save();
painter.translate(100, 100);   // 原点移到 (100,100)
painter.rotate(45);            // 绕新原点旋转 45°
painter.drawText(0, 0, "Rotated");
painter.restore();             // 恢复
```

- `setWindow()` / `setViewport()` — 世界坐标 → 设备坐标映射
- `resetTransform()` — 重置为单位矩阵

**黄金规则**：始终成对使用 `save()`/`restore()` 包裹局部变换。

## 8. 渲染 Hint

```cpp
painter.setRenderHint(QPainter::Antialiasing);           // 线条/字体抗锯齿
painter.setRenderHint(QPainter::TextAntialiasing);       // 仅文本抗锯齿
painter.setRenderHint(QPainter::SmoothPixmapTransform);  // 图片缩放平滑
painter.setRenderHint(QPainter::HighQualityAntialiasing);// Qt 6 高质量
```

## 9. 合成模式（CompositionMode）

常用：`SourceOver`（默认）/ `Source` / `Clear` / `Xor`（橡皮擦）/ `Multiply` / `Screen`

## 10. 裁剪

```cpp
painter.setClipRect(QRect(50, 50, 200, 200));
painter.setClipPath(clipPath);
painter.setClipping(false);  // 禁用裁剪
```

## 11. 双缓冲

先用 QPixmap 离屏绘制，再整体贴到控件上，消除闪烁。Qt 5+ 默认已启用双缓冲。

## 12. 避坑要点

1. **不要在 paintEvent 外创建 QPainter(this)**
2. **save()/restore() 必须成对**
3. **QPen width=0 是"发丝线"**（始终 1px，不受变换影响）
4. **QPixmap 用于屏幕**（平台优化、更快），**QImage 用于图像处理**（平台无关像素数组）
5. 大量小字体可关闭 TextAntialiasing 提升性能
6. 优先使用 `update()`（异步合并重绘），而非 `repaint()`（同步强制）
