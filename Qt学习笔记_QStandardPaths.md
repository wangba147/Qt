# QStandardPaths — 跨平台标准路径定位类（Qt Core）

> 每日 Qt 学习 | 2026-05-27

---

## 一、概念与背景

### QStandardPaths 是什么

`QStandardPaths`（Qt Core）提供一组静态方法，用于在 Windows、macOS、Linux 上获取操作系统定义的标准目录路径。它屏蔽了三大平台文件系统规范的差异，让开发者用同一行代码拿到"正确"的路径。

- **模块**：Qt Core
- **兼容**：Qt 5 / 6
- **头文件**：`#include <QStandardPaths>`
- **特质**：纯静态类，无需实例化，全部是 `static` 方法

### 为什么不用硬编码路径

| 做法 | 问题 |
|------|------|
| `"C:\\Users\\xxx\\AppData\\Local"` | 只在当前用户/当前机器有效 |
| `QDir::homePath() + "/.myapp"` | 在 Windows 上不符合 AppData 规范，杀毒软件可能拦截 |
| `"/tmp/myapp"` | Linux 上 ok，Windows 上不存在 `/tmp` |
| **QStandardPaths** | **自动适配平台规范，通过应用商店审核** |

### 背后的平台规范

| 平台 | 规范 | 关键目录 |
|------|------|----------|
| **Windows** | Known Folders (SHGetFolderPath) | `%LOCALAPPDATA%`, `%APPDATA%`, `%USERPROFILE%` |
| **macOS** | Apple File System Programming Guide | `~/Library/Application Support`, `~/Library/Caches` |
| **Linux** | XDG Base Directory Specification | `~/.local/share`, `~/.config`, `~/.cache` |

---

## 二、StandardLocation 枚举速查

### 应用专属路径（含 AppName）

| 枚举值 | Windows | Linux | macOS | 用途 |
|--------|---------|-------|-------|------|
| `AppDataLocation` | `%LOCALAPPDATA%\AppName` | `~/.local/share/AppName` | `~/Library/Application Support/AppName` | **应用持久化数据**（数据库、状态文件） |
| `AppConfigLocation` | `%LOCALAPPDATA%\AppName\conf`（Qt 5）/ `%APPDATA%\AppName`（Qt 6） | `~/.config/AppName` | `~/Library/Preferences/AppName` | 应用配置文件 |
| `AppLocalDataLocation` | 同 AppDataLocation | 同 AppDataLocation | 同 AppDataLocation | Qt 5 遗留，与 AppDataLocation 完全相同 |

> **Qt 6 变化**：`AppConfigLocation` 在 Qt 6 中改为返回 `%APPDATA%`（漫游），Qt 5 中返回 `%LOCALAPPDATA%`（本地）。

### 通用路径（不含 AppName）

| 枚举值 | Windows | Linux | macOS | 用途 |
|--------|---------|-------|-------|------|
| `GenericDataLocation` | `%LOCALAPPDATA%` | `~/.local/share` | `~/Library/Application Support` | 通用数据（多应用共享） |
| `GenericConfigLocation` | `%APPDATA%` | `~/.config` | `~/Library/Preferences` | 通用配置 |
| `GenericCacheLocation` | `%LOCALAPPDATA%\cache`（Qt 5）/ `%LOCALAPPDATA%`（Qt 6） | `~/.cache` | `~/Library/Caches` | 通用缓存 |
| `TempLocation` | `%LOCALAPPDATA%\Temp` | `/tmp` 或 `$TMPDIR` | `/tmp` 或 `$TMPDIR` | 临时文件 |
| `DownloadLocation` | `~/Downloads` | `~/Downloads` 或 `XDG_DOWNLOAD_DIR` | `~/Downloads` | 用户下载目录 |
| `DocumentsLocation` | `~/Documents` | `~/Documents` 或 `XDG_DOCUMENTS_DIR` | `~/Documents` | 文档目录 |
| `PicturesLocation` | `~/Pictures` | `~/Pictures` 或 `XDG_PICTURES_DIR` | `~/Pictures` | 图片目录 |
| `MusicLocation` | `~/Music` | `~/Music` 或 `XDG_MUSIC_DIR` | `~/Music` | 音乐目录 |
| `MoviesLocation` | `~/Videos` | `~/Videos` 或 `XDG_VIDEOS_DIR` | `~/Movies` | 视频目录 |
| `DesktopLocation` | `~/Desktop` | `~/Desktop` 或 `XDG_DESKTOP_DIR` | `~/Desktop` | 桌面 |
| `HomeLocation` | `C:\Users\xxx` | `/home/xxx` | `/Users/xxx` | 用户主目录 |

### 运行时 / 系统路径

| 枚举值 | Windows | Linux | macOS | 用途 |
|--------|---------|-------|-------|------|
| `RuntimeLocation` | 不支持（返回空） | `$XDG_RUNTIME_DIR`（通常 `/run/user/1000`） | 不支持（返回空） | 运行时套接字、锁文件 |
| `ApplicationsLocation` | `C:\Users\xxx\AppData\Roaming\Microsoft\Windows\Start Menu\Programs` | `/usr/share/applications` 或 `~/.local/share/applications` | `/Applications` | 已安装应用目录 |
| `ExecutablesLocation` | 应用安装目录 | `~/bin` 或 `.local/bin` | 应用安装目录 | 可执行文件位置 |

---

## 三、核心静态方法

### 路径获取（最常用）

```cpp
#include <QStandardPaths>
#include <QCoreApplication>

// 基础用法 — 获取单个路径
QString dataPath = QStandardPaths::writableLocation(QStandardPaths::AppDataLocation);
// → "C:/Users/xxx/AppData/Local/MyApp"

// 所有搜索路径（按优先级排列）
QStringList allPaths = QStandardPaths::standardLocations(QStandardPaths::AppDataLocation);
// → ["C:/Users/xxx/AppData/Local/MyApp",
//     "C:/ProgramData/MyApp",       // 系统级回退
//     "C:/Users/xxx/AppData/Local/MyApp"]  // 可能重复
```

### writableLocation vs standardLocations

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `writableLocation(type)` | `QString` | 返回**第一个可写**的路径，适合创建文件 |
| `standardLocations(type)` | `QStringList` | 返回**所有搜索路径**，适合读取已存在的文件 |

> **关键区别**：`writableLocation` 保证路径可写，但可能返回空字符串。`standardLocations` 可能包含只读路径。

### 路径查找与定位

```cpp
// 在标准路径中查找一个已存在的可执行文件/文件
QString exe = QStandardPaths::findExecutable("git");
// → "C:/Program Files/Git/bin/git.exe"

QString exe2 = QStandardPaths::findExecutable("python3", {"/usr/local/bin"});
// → 第二个参数指定额外搜索路径

// 查找应用在系统中的桌面入口
QString desktopFile = QStandardPaths::locate(QStandardPaths::ApplicationsLocation,
                                               "myapp.desktop");
// → "/usr/share/applications/myapp.desktop"
```

### 辅助方法

```cpp
// 检查路径是否存在 + 可选自动创建
bool exists = QStandardPaths::locate(QStandardPaths::AppDataLocation,
                                      "config.ini") != QString();

// Qt 6.6+：确保目录存在
#include <QStandardPaths>
// 确保可写目录存在，如果不存在则创建
QString cacheDir = QStandardPaths::writableLocation(QStandardPaths::GenericCacheLocation);
QDir().mkpath(cacheDir);  // 通用做法，QStandardPaths 本身不自动创建目录
```

---

## 四、自定义应用名与组织名

`writableLocation` 中的 `AppName` 来自 `QCoreApplication` 的元数据设置：

```cpp
int main(int argc, char *argv[]) {
    QCoreApplication::setApplicationName("MyApp");
    QCoreApplication::setOrganizationName("MyCompany");
    QCoreApplication::setOrganizationDomain("mycompany.com");
    // Qt 6 还可设置：
    // QCoreApplication::setApplicationVersion("1.0.0");

    QCoreApplication app(argc, argv);

    // AppDataLocation 实际路径取决于平台规则：
    // Windows: %LOCALAPPDATA%\MyCompany\MyApp  (组织名在前)
    // Linux:   ~/.local/share/MyApp             (只用应用名)
    // macOS:   ~/Library/Application Support/MyApp
}
```

### 路径构造规则

| 平台 | AppDataLocation 构造规则 |
|------|--------------------------|
| **Windows** | `%LOCALAPPDATA%\<OrganizationName>\<ApplicationName>` |
| **macOS** | `~/Library/Application Support/<ApplicationName>` |
| **Linux** | `~/.local/share/<ApplicationName>`（遵循 XDG，不用组织名） |

### 覆盖默认路径（测试 / 便携模式）

```cpp
// 设置自定义路径前缀（影响所有 App* 类型）
QStandardPaths::setTestModeEnabled(true);
// Windows: %LOCALAPPDATA%\Local\MyCompany\MyApp
// 用于单元测试隔离

// Qt 5 无官方便携模式，但可以手动处理：
#ifdef PORTABLE_MODE
    QString dataPath = QCoreApplication::applicationDirPath() + "/data";
#else
    QString dataPath = QStandardPaths::writableLocation(QStandardPaths::AppDataLocation);
#endif
```

---

## 五、实战场景

### 场景 1：应用配置文件管理

```cpp
class AppConfig {
public:
    static QString configPath() {
        QString dir = QStandardPaths::writableLocation(QStandardPaths::AppConfigLocation);
        QDir().mkpath(dir);  // 确保目录存在
        return dir + "/settings.json";
    }

    static void save(const QJsonObject &config) {
        QFile file(configPath());
        if (file.open(QIODevice::WriteOnly)) {
            file.write(QJsonDocument(config).toJson(QJsonDocument::Indented));
        }
    }

    static QJsonObject load() {
        QFile file(configPath());
        if (file.exists() && file.open(QIODevice::ReadOnly)) {
            return QJsonDocument::fromJson(file.readAll()).object();
        }
        return {};
    }
};
```

### 场景 2：日志文件存放

```cpp
// 日志放缓存目录（可被系统清理，不影响用户数据）
QString logDir = QStandardPaths::writableLocation(QStandardPaths::GenericCacheLocation)
                 + "/" + QCoreApplication::applicationName() + "/logs";
QDir().mkpath(logDir);

QString logFile = logDir + "/app_" +
    QDateTime::currentDateTime().toString("yyyyMMdd") + ".log";
```

### 场景 3：临时文件处理

```cpp
// 生成唯一临时文件路径
QString tempFile = QStandardPaths::writableLocation(QStandardPaths::TempLocation)
                   + "/" + QCoreApplication::applicationName()
                   + "_XXXXXX.tmp";
QTemporaryFile tmp(tempFile);
tmp.setAutoRemove(true);  // 析构时自动删除

if (tmp.open()) {
    tmp.write(someData);
    tmp.close();
    // 将路径传给外部工具处理...
    processFile(tmp.fileName());
}
```

### 场景 4：OSGQt_Buddy — 地球数据缓存

```cpp
// 地形瓦片缓存放 GenericCacheLocation（不影响应用数据，可安全清理）
QString tileCacheDir = QStandardPaths::writableLocation(QStandardPaths::GenericCacheLocation)
                       + "/OSGQt_Buddy/tiles";
QDir().mkpath(tileCacheDir);

// 应用配置（相机位置、图层设置）放 AppConfigLocation
QString appConfigDir = QStandardPaths::writableLocation(QStandardPaths::AppConfigLocation);
QDir().mkpath(appConfigDir);
// → %LOCALAPPDATA%/MyCompany/OSGQt_Buddy/
```

---

## 六、API 速查

### QStandardPaths 静态方法一览

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `writableLocation(type)` | `QString` | 获取指定类型的可写路径 |
| `standardLocations(type)` | `QStringList` | 获取所有搜索路径（按优先级） |
| `findExecutable(name, paths)` | `QString` | 在 PATH 和指定路径中查找可执行文件 |
| `locate(type, fileName, options)` | `QString` | 在标准路径中定位文件 |
| `locateAll(type, fileName, options)` | `QStringList` | 定位所有匹配文件 |
| `displayName(type)` | `QString` | 获取目录的显示名称（如"Downloads"的中文翻译） |
| `setTestModeEnabled(enabled)` | `void` | 启用测试模式（使用独立的测试路径） |
| `isTestModeEnabled()` | `bool` | 是否处于测试模式 |

### LocateOption 枚举

| 值 | 说明 |
|----|------|
| `LocateFile` | 只查找文件（默认） |
| `LocateDirectory` | 只查找目录 |

---

## 七、避坑要点

### 坑 1：writableLocation 可能返回空字符串

当 `QCoreApplication` 未设置 applicationName 时，某些平台返回空路径。**始终在 main() 开头设置应用名**。

```cpp
// 危险：未设置应用名
auto path = QStandardPaths::writableLocation(QStandardPaths::AppDataLocation);
// 可能返回 "" 或一个不完整的路径

// 安全：先设置应用名
QCoreApplication::setApplicationName("MyApp");
auto path = QStandardPaths::writableLocation(QStandardPaths::AppDataLocation);
```

### 坑 2：QStandardPaths 不自动创建目录

`writableLocation` 只返回路径字符串，**不会创建目录**。首次使用前务必 `QDir().mkpath()`。

### 坑 3：Qt 5 与 Qt 6 的 AppConfigLocation 不同

- Qt 5：`AppConfigLocation` → `%LOCALAPPDATA%\AppName`（本地，不漫游）
- Qt 6：`AppConfigLocation` → `%APPDATA%\AppName`（漫游，域登录同步）

如果需要跨版本一致行为，显式使用 `GenericConfigLocation` 或手动拼路径。

### 坑 4：GenericCacheLocation 的 Qt 版本差异

- Qt 5：`%LOCALAPPDATA%\AppName\cache`（带应用名后缀）
- Qt 6：`%LOCALAPPDATA%`（直接返回本地数据目录，不再加 cache 后缀）

### 坑 5：Windows 权限问题

某些系统目录（如 `ProgramData`）可能不可写。始终用 `writableLocation` 而不是 `standardLocations` 的第一个元素来写入文件。如果 `writableLocation` 返回的是受限目录，考虑回退到 `TempLocation`。

### 坑 6：路径中的空格和特殊字符

Windows 用户名可以包含空格和非 ASCII 字符。QStandardPaths 返回的路径已正确处理这些情况，但手动拼接时需要用 `QDir::separator()` 或 `QDir` 操作，避免硬编码 `/`。

---

## 八、最佳实践清单

1. **始终在 main() 中设置应用名和组织名**，否则 `App*` 类型路径不可靠
2. **写入前 `QDir().mkpath()`**，QStandardPaths 不保证目录存在
3. **持久化数据**用 `AppDataLocation`，**配置**用 `AppConfigLocation`，**缓存**用 `GenericCacheLocation`
4. **临时文件**用 `QTemporaryFile` + `TempLocation`，设置 `setAutoRemove(true)`
5. **不要**把缓存文件放 `AppDataLocation`，否则占用用户空间且不会被系统清理
6. **单元测试**用 `setTestModeEnabled(true)` 隔离文件系统
7. **跨 Qt 版本项目**注意 `AppConfigLocation` 在 Qt 5/6 上的路径差异
8. **可执行文件查找**优先用 `findExecutable()`，不要自己拼 PATH 搜索逻辑

---

*累计已学 17 个 Qt 类：QTimer, QThread, QSignalMapper, QObject, QSettings, QFileSystemWatcher, QProperty/QBindable, QVariant, QSortFilterProxyModel, QEventLoop, QRegularExpression, QJsonDocument, QTranslator, QElapsedTimer, QProcess, QFuture/QtConcurrent, QStandardPaths*
