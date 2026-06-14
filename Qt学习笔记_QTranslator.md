# QTranslator — Qt 国际化翻译核心类（Qt Core）

> 每日 Qt 学习 | 2026-05-23

---

## 一、概念与背景

### QTranslator 是什么

`QTranslator`（Qt Core）是 Qt 国际化（i18n）体系的核心运行时类，负责加载编译后的 `.qm` 翻译文件，在程序运行时将 `tr()` 中的源文本替换为目标语言。

- **模块**：Qt Core
- **兼容**：Qt 5 / 6
- **头文件**：`#include <QTranslator>`

### 为什么需要国际化

- **市场覆盖**：同一份代码发布多语言版本，降低维护成本
- **翻译解耦**：翻译工作可交给非程序员完成（Qt Linguist）
- **运行时切换**：无需重启即可动态切换语言
- **Qt 生态标准**：与 QLocale、QCollator、QTextBoundary 等无缝配合

### 核心角色定位

| 组件 | 职责 |
|------|------|
| `QObject::tr()` | 源码中标记可翻译字符串 |
| `lupdate` | 提取 .cpp/.ui/.qml 中所有 tr() 生成 .ts |
| `Qt Linguist` | GUI 工具，编辑 .ts 翻译条目 |
| `lrelease` | 将 .ts 编译为 .qm（二进制，运行时用） |
| **QTranslator** | **运行时加载 .qm，翻译 tr() 字符串** |
| `QCoreApplication` | 管理翻译器栈，协调查找 |

> **注意**：QTranslator 本身不做翻译文本的编辑，那是 Qt Linguist 的工作。QTranslator 只负责在运行时加载 .qm 文件并提供翻译查找。

### i18n 完整工作流

```
C++ 源码 tr("Hello")  →  lupdate 提取字符串  →  Qt Linguist 编辑 .ts
                                                        ↓
         用户界面显示翻译  ←  QTranslator 加载 .qm  ←  lrelease 编译 .ts→.qm
```

### 翻译查找链（LIFO 后进先出）

```
QCoreApplication::translate()
        ↓
最后安装的 QTranslator     ← 优先级最高
        ↓ (未匹配)
倒数第二个 QTranslator     ← 次优先级
        ↓ (未匹配)
最早安装的 QTranslator     ← 优先级最低
        ↓ (全部未匹配)
返回原文（源字符串）
```

---

## 二、基础用法

### 最小化 i18n 三步

#### Step 1: 在源码中用 tr() 标记字符串

```cpp
// 必须继承 QObject（直接或间接），才能使用 tr()
class MainWindow : public QMainWindow {
    Q_OBJECT
public:
    MainWindow(QWidget *parent = nullptr) {
        setWindowTitle(tr("My Application"));
        QPushButton *btn = new QPushButton(tr("Click Me"), this);
    }
};
```

#### Step 2: 用 lupdate 提取 + lrelease 编译

```bash
# .pro 文件中添加（或 CMake 用 qt_add_lrelease）
TRANSLATIONS += myapp_zh_CN.ts

# 命令行
lupdate myapp.pro           # → 生成 myapp_zh_CN.ts
# 用 Qt Linguist 打开 .ts 翻译
lrelease myapp.pro          # → 编译为 myapp_zh_CN.qm
```

#### Step 3: main.cpp 中加载翻译器

```cpp
#include <QTranslator>
#include <QLocale>
#include <QCoreApplication>

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);

    QTranslator translator;
    // 方式 A：按系统语言自动加载
    if (translator.load(QLocale(), "myapp", ":"))
        app.installTranslator(&translator);

    MainWindow w;
    w.show();
    return app.exec();
}
```

> **load() 的搜索路径模式**：`load(locale, filename, prefix, suffix)`
> 实际查找：`prefix + filename + "_" + locale.name() + suffix`
> 例如 `load(QLocale("zh_CN"), "myapp", ":/i18n/", ".qm")` → 查找 `:/i18n/myapp_zh_CN.qm`

---

## 三、翻译流程实战

### 运行时动态切换语言

```cpp
void MainWindow::switchLanguage(const QString &lang) {
    // 移除旧翻译器
    qApp->removeTranslator(m_translator);

    // 加载新翻译
    if (m_translator->load(":/i18n/myapp_" + lang + ".qm")) {
        qApp->installTranslator(m_translator);
    }

    // 关键：tr() 在构造时求值，需要重新设置
    // 方式1：retranslateUi（Qt Designer 生成）
    ui->retranslateUi(this);

    // 方式2：手动逐个更新
    ui->titleLabel->setText(tr("Welcome"));
    ui->okButton->setText(tr("OK"));
    ui->menuFile->setTitle(tr("&File"));

    // 方式3：使用 changeEvent + QEvent::LanguageChange
}
```

### QEvent::LanguageChange 事件模式（推荐）

```cpp
// installTranslator 后，QCoreApplication 会向所有 Widget
// 发送 QEvent::LanguageChange 事件，可以统一处理：
bool MainWindow::event(QEvent *event) {
    if (event->type() == QEvent::LanguageChange) {
        ui->retranslateUi(this);   // 重新翻译 UI
        updateCustomTexts();       // 重新翻译动态文本
    }
    return QMainWindow::event(event);
}
```

### CMake 项目配置（Qt 6）

```cmake
find_package(Qt6 REQUIRED COMPONENTS LinguistTools)
qt_add_translations(MyApp
    SOURCES ${PROJECT_SOURCES}
    QM_FILES_OUTPUT_DIR ${CMAKE_BINARY_DIR}/translations
)
```

### .pro 项目配置（Qt 5）

```
TRANSLATIONS += \
    translations/myapp_zh_CN.ts \
    translations/myapp_ja_JP.ts
```

---

## 四、常用 API 速查

### QTranslator 核心方法

| 方法 | 说明 |
|------|------|
| `load(const QString &filename)` | 从文件路径加载 .qm 文件 |
| `load(const QLocale &, const QString &, const QString &prefix = {}, const QString &suffix = {})` | 按 locale 命名规则自动查找加载 |
| `load(const uchar *data, int len, const QString &directory)` | 从内存数据加载（嵌入资源） |
| `isEmpty()` | 翻译器是否为空（未加载或加载失败） |
| `language()` | 返回翻译器中的目标语言名 |

### QCoreApplication 翻译管理方法

| 方法 | 说明 |
|------|------|
| `installTranslator(QTranslator *)` | 安装翻译器（加入查找栈） |
| `removeTranslator(QTranslator *)` | 移除翻译器并释放 |
| `translate(const char *context, const char *sourceText, ...)` | 手动翻译（等价于 tr()） |

### tr() 系列函数

| 函数 | 说明 |
|------|------|
| `tr("text")` | 标准翻译，context 为类名 |
| `tr("text", nullptr, n)` | 复数形式（%n 占位符） |
| `QT_TR_NOOP("text")` | 标记翻译但不立即翻译（非 QObject 上下文） |
| `qtTrId("id_string")` | 用唯一 ID 翻译（不依赖源文本） |

### QTranslator 虚函数（子类化重写）

| 虚函数 | 说明 |
|--------|------|
| `translate(const char *context, const char *sourceText, const char *disambiguation, int n) const` | 核心查找函数，自定义翻译器重写此方法 |
| `isEmpty() const` | 翻译器是否为空 |
| `language() const` | 返回目标语言字符串 |

---

## 五、高级技巧

### 技巧1: 多翻译器叠加（fallback 机制）

```cpp
// 应用通用翻译 + 库/插件专属翻译
QTranslator *appTranslator = new QTranslator;
QTranslator *libTranslator = new QTranslator;

appTranslator->load("myapp_zh_CN.qm", ":/i18n");
libTranslator->load("qtbase_zh_CN.qm", ":/i18n");

// 先装通用，后装专用 → 专用优先
qApp->installTranslator(appTranslator);  // 低优先级
qApp->installTranslator(libTranslator);  // 高优先级
```

### 技巧2: 翻译 Qt 自带 UI 字符串

```cpp
// QFileDialog、QMessageBox 等对话框的按钮文本
// 需要额外加载 Qt 自身翻译文件
QTranslator qtTranslator;
#if QT_VERSION >= QT_VERSION_CHECK(6, 0, 0)
qtTranslator.load("qtbase_" + QLocale::system().name(),
                  QLibraryInfo::path(QLibraryInfo::TranslationsPath));
#else
qtTranslator.load("qt_" + QLocale::system().name(),
                  QLibraryInfo::location(QLibraryInfo::TranslationsPath));
#endif
qApp->installTranslator(&qtTranslator);
```

### 技巧3: 复数形式处理

```cpp
// 源码中：
int count = getUnreadCount();
label->setText(tr("You have %n message(s).", nullptr, count));

// .ts 翻译文件中可分别为单数/复数提供不同翻译：
// <numerusform>您有 %n 条消息。</numerusform>  (单数)
// <numerusform>您有 %n 条消息。</numerusform>  (复数)
```

### 技巧4: 翻译含上下文消歧

```cpp
// 同一词在不同上下文含义不同
tr("File", "File menu");       // → 菜单中的"文件"
tr("File", "File on disk");   // → 磁盘上的"文件"
// Qt Linguist 中会分别显示为两行，避免混淆
```

### 技巧5: 非 QObject 类中使用翻译

```cpp
// 方法1：QT_TR_NOOP 宏 + 后续 translate
static const char *msg = QT_TR_NOOP("Connection failed");
// 使用时：
qApp->translate("MyClass", msg);

// 方法2：直接调用 QCoreApplication::translate
QString text = QCoreApplication::translate("Context", "Hello");
```

---

## 六、避坑要点

### 坑1: tr() 在构造函数外的时机问题
`tr("OK")` 等价于 `this->QObject::tr("OK")`，在构造函数中翻译器尚未安装时可能返回原文。确保在 `installTranslator` **之后**再构建 UI，或使用 `retranslateUi` / `QEvent::LanguageChange` 刷新。

### 坑2: 不能拼接 tr()
`tr("File") + tr("Manager")` 翻译器无法识别为一个整体，不同语言词序也不同。正确做法：`tr("%1 Manager").arg(tr("File"))` 或 `tr("File Manager")`。

### 坑3: 字符串变量无法被翻译
`QString s = "Hello"; tr(s);` — tr() 接收变量时，lupdate 无法提取该字符串。**tr() 的参数必须是字符串字面量**。

### 坑4: load() 失败不报错
`load()` 加载失败返回 `false`，但不会抛异常。程序会静默使用原文。务必检查返回值或用 `isEmpty()` 确认。

### 坑5: QTranslator 生命周期
`QTranslator *t = new QTranslator;` 安装后如果 `t` 被销毁（栈上变量出作用域），`removeTranslator` 会访问悬空指针。确保翻译器对象生命周期覆盖整个应用，或在 `removeTranslator` 之后再销毁。

### 坑6: 动态文本需要手动刷新
`tr()` 只在调用时求值一次。切换语言后，已显示的文本不会自动变化。必须通过 `LanguageChange` 事件或手动重新 `setText(tr(...))`。

---

## 七、最佳实践清单

1. 所有面向用户的字符串都用 `tr()`（包括按钮文本、菜单、标题、tooltip、状态栏消息）
2. 日志、调试信息、配置键名 **不要** 用 `tr()`
3. 用 `QStringLiteral()` 或 `u"..."` 包裹源文本，避免运行时 UTF-16 转换
4. .qm 文件放入 Qt 资源系统 (`.qrc`)，避免部署时路径问题
5. CMake 项目使用 `qt_add_translations()`，让构建系统自动处理 lupdate/lrelease
6. 翻译文件命名规范：`appname_locale.qm`（如 `myapp_zh_CN.qm`）

---

*累计已学 13 个 Qt 类：QTimer, QThread, QSignalMapper, QObject, QSettings, QFileSystemWatcher, QProperty/QBindable, QVariant, QSortFilterProxyModel, QEventLoop, QRegularExpression, QJsonDocument, QTranslator*
