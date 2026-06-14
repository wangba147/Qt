# QFuture 与 QtConcurrent — 现代 Qt 异步编程模式

> 每日 Qt 学习 | 2026-05-26

---

## 一、概念与背景

### QFuture / QtConcurrent 是什么

QtConcurrent（Qt Core）是 Qt 基于 **QThreadPool** 构建的高层异步 API，提供函数式编程风格的并行计算能力。QFuture 是异步计算结果的"占位符"（类似 std::future）。

- **模块**：Qt Core（`#include <QtConcurrent>` + `#include <QFuture>`）
- **兼容**：Qt 5 / 6（Qt 6 新增 QPromise + .then() 链式调用）
- **定位**：替代手动 QThread 管理，比 std::async 更强大的 Qt 生态集成

### 核心角色定位

| 组件 | 职责 |
|------|------|
| `QThreadPool` | 全局线程池（默认最大线程数 = CPU 核心数） |
| `QtConcurrent::run/map/filter/...` | 任务提交 API |
| `QFuture<T>` | 异步计算结果容器 |
| `QFutureWatcher<T>` | 将 QFuture 状态转换为 Qt 信号 |
| `QPromise<T>`（Qt 6） | 生产者端控制结果、进度、取消 |

### 三件套对比

| 维度 | QThread | QtConcurrent | std::async |
|------|---------|-------------|------------|
| 线程管理 | 手动创建/销毁 | 自动线程池复用 | 系统决定 |
| 返回值 | 无，需信号/共享变量 | QFuture<T> 直接返回 | std::future<T> |
| 信号驱动 | emit signal | QFutureWatcher 信号 | 需轮询或 .get() 阻塞 |
| 进度报告 | 手动实现 | QPromise::setProgressValue | 不支持 |
| 任务取消 | 手动标志位 + quit() | cancel() + QPromise 协作 | 不支持 |
| 并行集合操作 | 手动拆分 | map/filter/reduce | 需手写 |
| 链式调用 | 不支持 | Qt 6: .then()/.onFailed() | 不支持 |

---

## 二、基础用法

### 1. QtConcurrent::run() — 最简异步

```cpp
#include <QtConcurrent>
#include <QFuture>

QFuture<int> future = QtConcurrent::run([]() -> int {
    int sum = 0;
    for (int i = 0; i < 1000000; ++i) sum += i;
    return sum;
});
// 主线程继续执行
```

### 2. QtConcurrent::map() — 并行映射

```cpp
QList<QImage> images = loadImages();
QFuture<void> future = QtConcurrent::map(images, &QImage::invertPixels);
future.waitForFinished();
```

### 3. QtConcurrent::filtered() — 并行过滤

```cpp
bool isLargeFile(const QString &path) {
    return QFileInfo(path).size() > 1024 * 1024;
}

QStringList files = getAllFiles();
QFuture<QString> future = QtConcurrent::filtered(files, isLargeFile);
QStringList largeFiles = future.results();  // 获取全部结果
```

### 4. QtConcurrent::filteredReduced() — MapReduce

```cpp
void accumulate(qint64 &total, const QString &path) {
    total += QFileInfo(path).size();
}

QFuture<qint64> future = QtConcurrent::filteredReduced<qint64>(
    files, isLargeFile, accumulate,
    QtConcurrent::SequentialReduce);  // 或 UnorderedReduce | OrderedReduce
qint64 totalSize = future.result();
```

> **Reduce 选项**：SequentialReduce（顺序聚合，默认）、UnorderedReduce（乱序但更快）、OrderedReduce（保序聚合）

---

## 三、QFuture 核心 API

### 取值方法

| 方法 | 说明 |
|------|------|
| `result()` | 阻塞等待第一个结果（GUI 线程慎用） |
| `results()` | 获取全部结果列表 |
| `resultAt(int index)` | 按索引取第 n 个结果 |
| `takeResult()` | 取出并移除第一个结果 |

### 状态查询

| 方法 | 说明 |
|------|------|
| `isStarted()` | 是否已开始执行 |
| `isRunning()` | 是否运行中 |
| `isFinished()` | 是否已完成 |
| `isCanceled()` | 是否已被取消 |
| `isPaused()` | 是否暂停中 |
| `progressValue()` | 当前进度值 (0 ~ progressMaximum) |
| `progressText()` | 进度文本描述 |

### 控制方法

| 方法 | 说明 |
|------|------|
| `cancel()` | 请求取消（需 QPromise 协作） |
| `pause()` | 请求暂停 |
| `resume()` | 请求恢复 |
| `togglePaused()` | 切换暂停状态 |
| `waitForFinished()` | 阻塞等待完成（可选超时参数） |

### 链式调用（Qt 6）

```cpp
QtConcurrent::run(loadData)
    .then([](Data d) { return process(d); })
    .then([](Result r) { return save(r); })
    .onFailed([](const QException &e) {
        qWarning() << "Failed:" << e.what();
    })
    .onCanceled([] { cleanup(); });
```

---

## 四、QFutureWatcher — 信号驱动模式

QFutureWatcher 将 QFuture 的状态变化转换为 Qt 信号，是连接异步线程与主线程 UI 更新的桥梁。

### 经典模式

```cpp
void DataLoader::loadAsync() {
    auto *watcher = new QFutureWatcher<QByteArray>(this);

    connect(watcher, &QFutureWatcher<QByteArray>::finished,
            this, [watcher]() {
        QByteArray data = watcher->result();
        updateUI(data);
        watcher->deleteLater();
    });

    connect(watcher, &QFutureWatcher<QByteArray>::progressValueChanged,
            this, [this](int v) {
        progressBar->setValue(v);
    });

    watcher->setFuture(QtConcurrent::run([this] { return downloadFile(); }));
}
```

### 可用信号一览

| 信号 | 触发时机 |
|------|----------|
| `started()` | 任务开始执行 |
| `finished()` | 任务完成 |
| `canceled()` | 任务被取消 |
| `paused()` / `resumed()` | 暂停/恢复 |
| `resultReadyAt(int)` | map/filter 中某个元素完成 |
| `resultsReadyAt(int, int)` | 一批元素完成 |
| `progressValueChanged(int)` | 进度值变化 |
| `progressRangeChanged(int, int)` | 进度范围变化 |

---

## 五、QPromise — 自定义 Future 数据源（Qt 6）

QFuture 默认只能"读取"。当你需要从生产者端控制结果、进度和取消时，使用 QPromise。

```cpp
QFuture<QString> processLargeFile(const QString &path) {
    QPromise<QString> promise;
    QFuture<QString> future = promise.future();

    QtConcurrent::run([promise = std::move(promise), path]() mutable {
        QFile file(path);
        file.open(QIODevice::ReadOnly);
        qint64 total = file.size();

        while (!file.atEnd()) {
            if (promise.isCanceled()) {
                file.close();
                return;
            }

            QString line = file.readLine();
            processLine(line);

            promise.setProgressValue(file.pos() * 100 / total);
            promise.suspendIfRequested();
        }

        promise.finish();
    });

    return future;
}
```

### QPromise 核心方法

| 方法 | 说明 |
|------|------|
| `addResult(T)` | 添加单个结果 |
| `addResults(container)` | 批量添加结果 |
| `finish()` | 标记完成（必须最后调用） |
| `setProgressValue(int)` | 设置当前进度 |
| `setProgressRange(min, max)` | 设置进度范围 |
| `setProgressValueAndText(int, QString)` | 同时设置进度和文本 |
| `isCanceled()` | 检查是否被请求取消 |
| `suspendIfRequested()` | 如果被请求暂停则挂起 |
| `future()` | 返回对应的 QFuture |

---

## 六、实战与避坑

### 实战 1: 异步地形数据加载（OSGQt_Buddy）

```cpp
void TerrainLoader::loadAsync(const QString &path) {
    auto *watcher = new QFutureWatcher<HeightField>(this);

    connect(watcher, &QFutureWatcher<HeightField>::finished,
            this, [this, watcher]() {
        m_heightField = watcher->result();
        updateTerrain();
        watcher->deleteLater();
    });

    connect(watcher, &QFutureWatcher<HeightField>::progressValueChanged,
            this, [this](int v) {
        statusBar()->showMessage(QString("Loading... %1%").arg(v));
    });

    watcher->setFuture(QtConcurrent::run([path]() {
        return HeightField::fromGeoTiff(path);
    }));
}
```

### 实战 2: 批量图片处理

```cpp
QFuture<QImage> future = QtConcurrent::map(images, [](const QImage &img) {
    return img.scaled(512, 512, Qt::KeepAspectRatio, Qt::SmoothTransformation);
});
future.waitForFinished();
QList<QImage> thumbnails = future.results();
```

### 避坑要点

1. **result() 阻塞 UI**：永远不要在 GUI 线程中直接调用 `future.result()`，使用 `QFutureWatcher::finished` 信号获取结果。

2. **Watcher 生命周期**：`QFutureWatcher` 通常 new 后 setFuture，在 finished 信号中 `deleteLater`。栈上的 watcher 可能在信号到达前析构。

3. **cancel() 是请求不是强制**：`future.cancel()` 只设置标志位，必须在线程内通过 `QPromise::isCanceled()` 检查并自行返回。

4. **线程安全**：QtConcurrent 的任务在多线程中并行执行，不要操作 GUI 对象或非线程安全的共享数据。

5. **QThreadPool 资源**：默认最大线程数是 `QThread::idealThreadCount()`。所有 QtConcurrent 任务共享同一个全局线程池，大量阻塞型任务可能耗尽线程。

### 最佳实践

- GUI 线程用 `QFutureWatcher` 信号模式，非 GUI 线程可用 `waitForFinished()`
- Qt 6 项目优先 `.then()` 链式调用，代码更清晰
- 可取消的长任务用 QPromise + 周期性 `isCanceled()` 检查
- 大数量 map/filter 任务设置 `QThreadPool::setMaxThreadCount()` 防止 CPU 过载

---

*累计已学 16 个 Qt 类：QTimer, QThread, QSignalMapper, QObject, QSettings, QFileSystemWatcher, QProperty/QBindable, QVariant, QSortFilterProxyModel, QEventLoop, QRegularExpression, QJsonDocument, QTranslator, QElapsedTimer, QProcess, QFuture/QtConcurrent*
