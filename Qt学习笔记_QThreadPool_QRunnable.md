# QThreadPool 与 QRunnable — Qt 线程池与任务调度

> 每日 Qt 学习 | 2026-06-06

---

## 一、概念与背景

### QThreadPool 是什么

QThreadPool（Qt Core）是 Qt 提供的**线程池管理器**，负责复用已有线程、控制并发度、避免频繁创建/销毁线程的开销。QRunnable 是提交到线程池的任务接口。

- **模块**：Qt Core（`#include <QThreadPool>` + `#include <QRunnable>`）
- **兼容**：Qt 5 / 6（核心 API 稳定）
- **定位**：管理 QThread 生命周期，比手动 `new QThread` 更高效

### 核心角色

| 组件 | 职责 |
|------|------|
| `QThreadPool::globalInstance()` | 全局单例线程池，QtConcurrent 底层依赖 |
| `QRunnable` | 任务接口，继承后重写 `run()` |
| `QThreadPool` | 管理线程复用、优先级队列、并发上限 |
| `QThread` | 底层工作线程（线程池内部持有，通常不直接接触） |

### 与 QThread 手动管理的对比

| 维度 | 手动 QThread | QThreadPool |
|------|-------------|-------------|
| 线程创建开销 | 每次 new/delete | 复用，几乎为零 |
| 并发控制 | 手动管理上限 | `maxThreadCount()` 自动控制 |
| 优先级 | 需自行实现队列 | 内置优先级调度 |
| 任务积压 | 需自行排队 | 内置 FIFO + Priority 队列 |
| 适用场景 | 长时间运行的后台任务 | 大量短小的并行任务 |

---

## 二、基础用法

### 方式一：继承 QRunnable

```cpp
class MyTask : public QRunnable {
public:
    MyTask(int id) : m_id(id) {}
    
    void run() override {
        qDebug() << "Task" << m_id << "running on" << QThread::currentThread();
        // 执行耗时操作...
        QThread::msleep(100);
        qDebug() << "Task" << m_id << "done";
    }

private:
    int m_id;
};

// 使用
auto *task = new MyTask(42);
task->setAutoDelete(true);  // 执行完毕后自动 delete
QThreadPool::globalInstance()->start(task);
```

### 方式二：Lambda + QRunnable（便捷写法）

```cpp
// Qt 6.3+ / Qt 5.15（部分支持）
auto *task = QRunnable::create([=] {
    qDebug() << "Lambda task running on" << QThread::currentThread();
    // 耗时操作...
});
task->setAutoDelete(true);
QThreadPool::globalInstance()->start(task);
```

### 方式三：与 QtConcurrent 的关系

```cpp
// QtConcurrent::run 内部就是使用 QThreadPool::globalInstance()
QFuture<int> future = QtConcurrent::run([] {
    return heavyComputation();
});

// 等价于手动提交 QRunnable + QPromise（Qt 6）
```

---

## 三、常用 API 速查

### QThreadPool 核心方法

| 方法 | 说明 |
|------|------|
| `static globalInstance()` | 获取全局单例线程池 |
| `start(QRunnable*, int priority=0)` | 提交任务，可指定优先级 |
| `tryStart(QRunnable*)` | 如果有空闲线程则立即执行（同一线程），否则返回 false |
| `waitForDone(int msecs=-1)` | 阻塞等待所有任务完成 |
| `clear()` | 清除队列中尚未开始的任务 |
| `setMaxThreadCount(int n)` | 设置最大线程数（默认 QThread::idealThreadCount()） |
| `maxThreadCount()` | 获取最大线程数 |
| `activeThreadCount()` | 当前正在工作的线程数 |
| `reserveThread()` | 预留一个线程（增加活跃线程数） |
| `releaseThread()` | 释放预留的线程 |
| `contains(const QThread*)` | 检查线程是否属于此线程池 |
| `tryTake(QRunnable*)` | 从队列中取出指定任务（未运行），成功返回 true |

### QRunnable 核心方法

| 方法 | 说明 |
|------|------|
| `virtual void run() = 0` | 纯虚函数，必须重写 |
| `bool autoDelete()` | 是否自动删除（默认 true） |
| `void setAutoDelete(bool)` | 设置自动删除标志 |

---

## 四、进阶技巧

### 4.1 任务优先级

```cpp
QThreadPool::globalInstance()->start(task1, 0);   // 默认优先级
QThreadPool::globalInstance()->start(task2, 5);   // 高优先级（先执行）
QThreadPool::globalInstance()->start(task3, -5);  // 低优先级（后执行）
```

### 4.2 自定义线程池（隔离资源）

```cpp
QThreadPool pool;
pool.setMaxThreadCount(2);       // 限制并发数
pool.setExpiryTimeout(30000);    // 空闲线程 30s 后回收

for (int i = 0; i < 100; ++i) {
    pool.start(new MyTask(i));
}
pool.waitForDone();              // 等待全部完成
```

### 4.3 reserveThread / releaseThread

适合需要保证特定任务有线程可用的场景：

```cpp
QThreadPool::globalInstance()->reserveThread();
// 现在保证还有至少 1 个空闲线程（实际 maxThreadCount 被临时 +1）
QThreadPool::globalInstance()->start(urgentTask);
QThreadPool::globalInstance()->releaseThread();
```

### 4.4 tryStart() — 当前线程立即执行

```cpp
auto *task = new QuickTask();
if (!QThreadPool::globalInstance()->tryStart(task)) {
    // 没有空闲线程，加入队列等待
    QThreadPool::globalInstance()->start(task);
}
```

### 4.5 批量任务 + 进度跟踪

```cpp
QAtomicInt counter(0);
int total = 100;

for (int i = 0; i < total; ++i) {
    auto *task = QRunnable::create([&, i] {
        processItem(i);
        counter.fetchAndAddAcquire(1);  // 线程安全的计数
    });
    task->setAutoDelete(true);
    QThreadPool::globalInstance()->start(task);
}

// 轮询进度（实际项目中可用 QTimer + 信号）
while (counter.loadRelaxed() < total) {
    qDebug() << "Progress:" << counter << "/" << total;
    QThread::msleep(50);
}
```

---

## 五、避坑要点

| 坑 | 说明 | 解决方案 |
|----|------|----------|
| **autoDelete 默认 true** | 忘记关闭 autoDelete 导致 task 被双重删除 | `task->setAutoDelete(false)` 后手动管理生命周期 |
| **无法取消正在运行的任务** | `clear()` 只影响队列，不管运行中的 | 用 `QAtomicInt` 标志位让任务自行退出 |
| **工作线程不能操作 GUI** | `run()` 中操作 QWidget 会崩溃 | 通过信号/槽传递结果到主线程 |
| **共享数据必须加锁** | 多线程访问同一变量 | `QMutex` / `QReadWriteLock` / `QAtomicInt` |
| **过多小任务反而不如单线程** | 任务切换和队列锁开销 > 任务本身 | 合理拆分粒度，合并小任务 |
| **局部 QThreadPool 生命周期** | 池销毁时可能还有任务在跑 | 先调用 `waitForDone()` 再析构 |
| **waitForDone() 不能从任务内调用** | 会死锁，任务在等自己完成 | 只在提交线程（通常是主线程）调用 |

---

## 六、完整实战：并行图片缩略图生成器

```cpp
#include <QThreadPool>
#include <QRunnable>
#include <QImage>
#include <QMutex>
#include <QDir>

class ThumbnailTask : public QRunnable {
public:
    ThumbnailTask(const QString &src, const QString &dst, QSize size)
        : m_src(src), m_dst(dst), m_size(size) {}

    void run() override {
        QImage img(m_src);
        if (img.isNull()) {
            emitError("Failed to load: " + m_src);
            return;
        }
        QImage thumb = img.scaled(m_size, Qt::KeepAspectRatio, Qt::SmoothTransformation);
        thumb.save(m_dst);
    }

signals:  // QRunnable 不能直接使用信号，需用 QObject 包装
    // 这里简化为回调函数方式，实际项目建议用 QObject + moveToThread

private:
    QString m_src, m_dst;
    QSize m_size;
};

void generateThumbnails(const QStringList &images, const QString &outDir, QSize size) {
    QThreadPool pool;
    pool.setMaxThreadCount(QThread::idealThreadCount());  // CPU 核心数

    for (const QString &img : images) {
        QFileInfo fi(img);
        QString out = outDir + "/thumb_" + fi.fileName();
        pool.start(new ThumbnailTask(img, out, size));
    }

    pool.waitForDone();  // 等待所有缩略图生成完毕
    qDebug() << "All thumbnails generated.";
}
```

---

## 七、小结

- **QThreadPool** 是 QtConcurrent 和所有异步 API 的底层引擎
- **QRunnable** 是你需要手动控制的轻量任务接口
- 全局线程池适合大多数场景，局部线程池适合资源隔离
- **autoDelete = true** 是默认行为，手动管理要记住关掉
- 永远不要在工作线程中操作 UI，用信号/槽解耦
- 配合 `QAtomicInt` 实现优雅的取消和进度跟踪

---

*上一篇：QMimeDatabase & QMimeType | 下一篇：待定*
