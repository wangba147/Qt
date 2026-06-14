# Qt 学习笔记：QGraphicsView / QGraphicsScene / QGraphicsItem

> Graphics View 框架 — Qt 中最高效的 2D 图形交互体系

---

## 一、概念与架构

Qt Graphics View 框架是一个基于 **Model-View** 思想的 2D 图形系统，由三层组成：

```
┌──────────────────────────────────────────┐
│  QGraphicsView  (窗口 — 视口变换/交互)     │
│  ┌────────────────────────────────────┐  │
│  │ QGraphicsScene (世界 — 管理所有Item) │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐       │  │
│  │  │ Item │ │ Item │ │ Item │ ...   │  │
│  │  └──────┘ └──────┘ └──────┘       │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

| 层次 | 类 | 职责 |
|------|-----|------|
| **视图层** | `QGraphicsView` | 提供可滚动的视口，处理缩放/旋转/拖拽等交互 |
| **场景层** | `QGraphicsScene` | 管理所有图形项的容器，负责碰撞检测、事件分发、渲染 |
| **元素层** | `QGraphicsItem` | 单个可绘制的图形对象，拥有独立坐标、变换、事件处理 |

**核心设计思想：**
- Scene 使用 **逻辑坐标**（可任意定义，如毫米、经纬度），View 负责将逻辑坐标映射到屏幕像素
- 每个 Item 有自己的 **局部坐标系**，父子 Item 通过坐标变换级联
- **BSP 树**（Binary Space Partitioning）管理 Item 显示顺序和碰撞检测
- 支持 **百万级 Item** 的高效渲染（比 QWidget 堆叠高效得多）

---

## 二、基础用法

### 2.1 最小示例

```cpp
#include <QApplication>
#include <QGraphicsScene>
#include <QGraphicsView>
#include <QGraphicsRectItem>

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);

    // 1. 创建场景（世界）
    QGraphicsScene scene(-200, -200, 400, 400); // sceneRect: (x,y,w,h)

    // 2. 添加图形项
    auto *rect = scene.addRect(-50, -50, 100, 100,
                                QPen(Qt::red, 2),
                                QBrush(Qt::blue));
    rect->setFlag(QGraphicsItem::ItemIsMovable);      // 可拖拽移动
    rect->setFlag(QGraphicsItem::ItemIsSelectable);   // 可选中

    // 3. 创建视图（窗口）
    QGraphicsView view(&scene);
    view.setRenderHint(QPainter::Antialiasing);
    view.setDragMode(QGraphicsView::RubberBandDrag);  // 橡皮筋框选
    view.setViewportUpdateMode(QGraphicsView::SmartViewportUpdate);
    view.show();

    return a.exec();
}
```

### 2.2 场景坐标系统

```cpp
// 场景矩形范围（逻辑坐标）
scene.setSceneRect(-500, -500, 1000, 1000);

// Item 使用场景坐标定位
item->setPos(100, 200);         // 设置 Item 在场景中的位置
QPointF p = item->scenePos();   // 获取 Item 在场景中的位置

// 坐标转换
item->mapToScene(localPoint);   // 局部 → 场景
item->mapFromScene(scenePoint); // 场景 → 局部
view->mapToScene(viewPoint);    // 视图 → 场景
view->mapFromScene(scenePoint); // 场景 → 视图
```

### 2.3 视图变换

```cpp
// 缩放
view->scale(1.2, 1.2);           // 放大 20%
view->resetTransform();          // 重置所有变换

// 旋转（度）
view->rotate(45);

// 平移（滚动到指定场景坐标）
view->centerOn(scenePoint);

// 适配显示
view->fitInView(scene.sceneRect(), Qt::KeepAspectRatio);

// 矩阵变换
QTransform t;
t.rotate(30).scale(1.5, 0.8);
view->setTransform(t);

// 获取当前矩阵
QTransform current = view->transform();
```

---

## 三、QGraphicsItem 子类大全

### 3.1 内置子类

| 子类 | 用途 | 示例 |
|------|------|------|
| `QGraphicsRectItem` | 矩形 | `addRect(0,0,100,50)` |
| `QGraphicsEllipseItem` | 椭圆/圆 | `addEllipse(0,0,80,80)` |
| `QGraphicsLineItem` | 线段 | `addLine(0,0,100,0)` |
| `QGraphicsPathItem` | 任意路径 | `addPath(painterPath)` |
| `QGraphicsPolygonItem` | 多边形 | `addPolygon(QPolygonF{...})` |
| `QGraphicsPixmapItem` | 图片 | `addPixmap(QPixmap("img.png"))` |
| `QGraphicsTextItem` | 文本 | `addText("Hello Qt")` |
| `QGraphicsSimpleTextItem` | 简单文本（无HTML） | `addSimpleText("Label")` |
| `QGraphicsProxyWidget` | 嵌入 QWidget | `addWidget(button)` |

### 3.2 关键 Flag

```cpp
item->setFlags(
    QGraphicsItem::ItemIsMovable        // 可拖拽移动
    | QGraphicsItem::ItemIsSelectable   // 可选中
    | QGraphicsItem::ItemIsFocusable    // 可获取键盘焦点
    | QGraphicsItem::ItemSendsGeometryChanges  // 位置变化时通知
    | QGraphicsItem::ItemSendsScenePositionChanges
);
```

### 3.3 自定义 Item（继承）

```cpp
class MyNode : public QGraphicsItem
{
public:
    QRectF boundingRect() const override {
        return QRectF(-30, -30, 60, 60); // 包围盒
    }

    void paint(QPainter *painter, const QStyleOptionGraphicsItem *option,
               QWidget *widget) override
    {
        painter->setPen(Qt::NoPen);
        painter->setBrush(isSelected() ? Qt::cyan : Qt::gray);
        painter->drawRoundedRect(boundingRect(), 10, 10);

        painter->setPen(Qt::black);
        painter->drawText(boundingRect(), Qt::AlignCenter, m_label);
    }

    // 自定义形状（用于碰撞检测、点击命中）
    QPainterPath shape() const override {
        QPainterPath path;
        path.addRoundedRect(boundingRect(), 10, 10);
        return path;
    }

private:
    QString m_label = "Node";
};
```

---

## 四、常用 API 速查

### 4.1 QGraphicsScene

| 方法 | 说明 |
|------|------|
| `addItem(QGraphicsItem*)` | 添加自定义 Item |
| `addRect/addEllipse/addLine/...` | 添加内置图形（返回 Item*） |
| `removeItem(QGraphicsItem*)` | 移除 Item（不 delete） |
| `clear()` | 移除并 delete 所有 Item |
| `items()` / `items(rect)` | 获取所有/区域内 Items |
| `items(pos)` / `itemAt(pos, transform)` | 获取某点的 Item(s) |
| `selectedItems()` | 获取所有选中 Item |
| `setFocusItem(item)` | 设置键盘焦点 Item |
| `collidingItems(item)` | 获取与 item 碰撞的 Items |
| `update(rect)` | 触发重绘 |
| `advance()` | 推进所有 Item 动画一帧 |
| `sceneRect()` / `setSceneRect()` | 场景矩形 |
| `views()` | 获取所有关联的 View |

### 4.2 QGraphicsView

| 方法 | 说明 |
|------|------|
| `setScene(scene)` | 设置场景 |
| `setTransform(t)` / `transform()` | 设置/获取变换矩阵 |
| `scale(sx, sy)` / `rotate(angle)` | 缩放/旋转 |
| `centerOn(pos)` | 滚动到指定场景坐标 |
| `fitInView(rect, aspectRatio)` | 适配显示 |
| `mapToScene(vp)` / `mapFromScene(sp)` | 视图⇄场景坐标转换 |
| `setDragMode(mode)` | 拖拽模式：NoDrag/ScrollHandDrag/RubberBandDrag |
| `setRenderHint(hint)` | 渲染提示（抗锯齿、平滑等） |
| `setViewportUpdateMode(mode)` | 视口更新策略 |
| `setCacheMode(mode)` | 缓存模式（提升性能） |
| `resetTransform()` | 重置变换矩阵 |
| `render(painter)` | 渲染到 QPainter（截图/打印） |
| `setOptimizationFlags()` | 性能优化标志 |

### 4.3 QGraphicsItem 关键虚函数

| 虚函数 | 说明 | 是否必须重写 |
|--------|------|:----------:|
| `boundingRect()` | 返回 Item 的包围矩形 | **必须** |
| `paint()` | 绘制 Item 内容 | **必须** |
| `shape()` | 定义精确命中区域 | 可选 |
| `type()` | 返回自定义类型 ID（用于 qgraphicsitem_cast） | 可选 |
| `itemChange(change, value)` | Item 状态变化通知 | 可选 |
| `sceneEvent(event)` | 事件总入口 | 可选 |
| `mousePressEvent/moveEvent/releaseEvent()` | 鼠标事件 | 可选 |
| `keyPressEvent()` | 键盘事件 | 可选 |

---

## 五、进阶技巧

### 5.1 Item 分组与关系

```cpp
// 父子关系（子 Item 随父 Item 移动/变换）
auto *parent = scene.addRect(0, 0, 100, 100);
auto *child = scene.addEllipse(0, 0, 20, 20);
child->setParentItem(parent);  // child 的坐标现在相对于 parent

// 分组（QGraphicsItemGroup）
auto *group = scene.createItemGroup({item1, item2, item3});
group->setFlags(QGraphicsItem::ItemIsMovable);
scene.destroyItemGroup(group);  // 解散分组
```

### 5.2 自定义变换锚点

```cpp
// 默认变换原点是 Item 的 (0,0)
// 修改变换原点：
item->setTransformOriginPoint(50, 50); // 绕 (50,50) 旋转
item->setRotation(45);
```

### 5.3 嵌入 QWidget

```cpp
auto *proxy = scene.addWidget(new QPushButton("Click Me"));
proxy->setPos(100, 100);
proxy->setRotation(15);  // 按钮也可以旋转！
// 注意：嵌入的 Widget 性能开销较大，不宜过多
```

### 5.4 碰撞检测

```cpp
// 检测碰撞
QList<QGraphicsItem*> hits = scene.collidingItems(
    item, Qt::IntersectsItemBoundingRect);

// 自定义碰撞检测
bool MyItem::collidesWithItem(const QGraphicsItem *other,
                               Qt::ItemSelectionMode mode) const {
    // 使用 shape() 进行像素级检测
    return QGraphicsItem::collidesWithItem(other, mode);
}
```

### 5.5 动画

```cpp
#include <QPropertyAnimation>

auto *anim = new QPropertyAnimation(item, "pos");
anim->setDuration(1000);
anim->setStartValue(QPointF(0, 0));
anim->setEndValue(QPointF(300, 200));
anim->setEasingCurve(QEasingCurve::InOutCubic);
anim->start(QAbstractAnimation::DeleteWhenStopped);
```

### 5.6 性能优化

```cpp
// 大量静态 Item 使用缓存
view->setCacheMode(QGraphicsView::CacheBackground);

// 优化标志
view->setOptimizationFlags(
    QGraphicsView::DontSavePainterState     // 不保存 Painter 状态
    | QGraphicsView::DontAdjustForAntialiasing
);

// Item 层面
item->setCacheMode(QGraphicsItem::DeviceCoordinateCache); // 缓存 Item 绘制

// 禁用不需要的 Flag 降低开销
item->setFlag(QGraphicsItem::ItemIsSelectable, false);
item->setFlag(QGraphicsItem::ItemIsMovable, false);

// 大量 Item 时使用 OpenGL 渲染
view->setViewport(new QOpenGLWidget);
```

### 5.7 打印与截图

```cpp
// 渲染场景到图片
QImage image(scene.sceneRect().size().toSize(), QImage::Format_ARGB32);
image.fill(Qt::white);
QPainter painter(&image);
painter.setRenderHint(QPainter::Antialiasing);
scene.render(&painter);
painter.end();
image.save("scene.png");

// 打印
QPrinter printer;
scene.render(&painter, QRectF(), scene.sceneRect());
```

---

## 六、避坑要点

| # | 坑 | 正确做法 |
|---|-----|---------|
| 1 | `scene.clear()` 会 delete 所有 Item，外部持有的指针变野 | 只移除不删除用 `removeItem()` |
| 2 | `boundingRect()` 返回值太小，Item 超出区域不可见 | 包住所有绘制内容 + 笔宽的一半 |
| 3 | 在 `paint()` 外调用 `painter->drawXXX()` | 所有绘制必须在 `paint()` 虚函数内 |
| 4 | 大量 Item 频繁调用 `prepareGeometryChange()` | 位置/大小改变前必须调用 |
| 5 | 嵌入过多 QWidget 作为 Item（每个 Widget 消耗系统资源） | 用自定义 Item 绘制替代简单控件 |
| 6 | 忘记 `setFlag(ItemIsFocusable)` 导致键盘事件不触发 | 需要键盘交互的 Item 必须设置 |
| 7 | 直接修改 Item 的 `pos()` 而非 `setPos()` 触发不了 itemChange | 始终使用 `setPos()` 移动 Item |
| 8 | View 的 `setTransform()` 和多层缩放叠加导致浮点误差 | 使用 `resetTransform()` + 重新变换 |

---

## 七、与 QWidget 绘图的对比

| 特性 | QWidget + paintEvent | Graphics View |
|------|:--------:|:-----------:|
| Item 数量 | 少量（~100） | 百万级 |
| 坐标变换 | 手动 | 内置矩阵变换 |
| 碰撞检测 | 手动实现 | 自动（BSP 树 O(log n)） |
| Item 交互 | 原始事件处理 | Flag 开关 + 虚函数 |
| 动画 | 手动 QTimer | QPropertyAnimation 直接驱动 |
| 缩放/旋转 | 手动 QPainter | View 级一行代码 |
| 嵌入 Widget | 原生 | 通过 QGraphicsProxyWidget |
| 内存/复杂度 | 简单 | 较高 |

---

> **一句话总结：** 当你的 2D 场景有超过几十个需要交互的图形对象，或需要缩放/旋转/拖拽时，Graphics View 框架是唯一正解。
