># Windows 下编译安装 open62541（含 OpenSSL 加密 + 动态库）

本文档记录在 Windows 上从源码编译 open62541 的完整步骤，最终产出：**启用 OpenSSL 加密**、**动态库（.dll + .lib）**、**安装到自定义目录**，可作为依赖库供其他项目引用。

---

## 一、准备工作

### 1.1 所需工具

| 工具 | 用途 | 说明 |
|------|------|------|
| Git | 克隆源码 | 安装时确保加入 PATH |
| CMake | 配置与构建 | 安装时勾选 "Add to PATH" |
| OpenSSL (Win64) | 加密后端 | 从 Shining Light Productions 下载完整安装包（非仅 DLL 运行时） |

> 编译由 CMake 根据本机环境自动选择生成器，无需手动指定编译器。

### 1.2 目录规划（本文约定）

| 用途 | 路径 |
|------|------|
| OpenSSL 安装目录 | `D:\opcuaSDK\OpenSSL-Win64` |
| open62541 源码目录 | `D:\workspace\github\open62541` |
| open62541 最终安装目录 | `D:\opcuaSDK\open62541` |

> 建议所有路径**不含空格**，避免 CMake 解析问题。

---

## 二、安装 OpenSSL

1. 下载并安装 Win64 OpenSSL（完整开发包）。
2. 安装到 `D:\opcuaSDK\OpenSSL-Win64`。
3. 确认目录结构：

```text
D:\opcuaSDK\OpenSSL-Win64\
├── include\
│   └── openssl\        ← 头文件
└── lib\
    ├── libcrypto.lib   ← 若不存在，见下方说明
    └── libssl.lib
```
### 关键步骤：修正 .lib 文件位置

Shining Light 安装包通常把 VS 用的库文件放在 `lib\VC\x64\MD\`（或 `MDd`）子目录下，而 CMake 的 `FindOpenSSL` 只在 `lib\` 根目录查找，会报错：

```text
Could NOT find OpenSSL ... (missing: OPENSSL_CRYPTO_LIBRARY OPENSSL_INCLUDE_DIR)    
```
>解决办法：进入 lib\VC\x64\MD\（Release）或 MDd（Debug），把 libcrypto.lib 和 libssl.lib 复制到 lib\ 根目录下。

## 三、获取 open62541 源码

```bash
git clone https://github.com/open62541/open62541.git
cd open62541
git submodule update --init --recursive
```

>git submodule update 不可省略，否则构建可能失败。

## 四、配置、构建与安装

### 4.1 创建构建目录并配置

```bash
mkdir build
cd build
```

执行 CMake 配置命令（**在 build 目录内**）：

```bash
cmake -DUA_ENABLE_ENCRYPTION=OPENSSL -DOPENSSL_ROOT_DIR="D:/opcuaSDK/OpenSSL-Win64" -DBUILD_SHARED_LIBS=ON ..
```

**各参数含义：**

| 参数 | 作用 |
|------|------|
| `-DUA_ENABLE_ENCRYPTION=OPENSSL` | 启用加密功能，后端使用 OpenSSL。**不加此项则库中不含 `UA_ServerConfig_setDefaultWithSecurityPolicies` 等加密相关函数**，链接时会报 LNK2019 |
| `-DOPENSSL_ROOT_DIR="D:/opcuaSDK/OpenSSL-Win64"` | 告诉 CMake OpenSSL 的根目录（包含 include 和 lib 的那一层）。路径使用**正斜杠 `/`** |
| `-DBUILD_SHARED_LIBS=ON` | 生成动态库（`.dll` + 导入库 `.lib`）。若省略则生成静态库 |
| `..` | 指向上一级的源码目录（含 CMakeLists.txt） |

> 注意：`-DUA_ENABLE_ENCRYPTION=ON` 是旧写法，新版本推荐直接指定后端 `OPENSSL` 或 `MBEDTLS`。

### 4.2 构建 Release 版本

```bash
cmake --build . --config Release
```

### 4.3 安装到指定目录

```bash
cmake --install . --config Release --prefix "D:\opcuaSDK\open62541"
```

**注意**：`--prefix` 使用**反斜杠 `\`**（Windows 原生路径），与前面 `OPENSSL_ROOT_DIR` 的正斜杠写法不同，这是因为 `--install` 的 prefix 由 CMake 直接按系统路径处理。

---

## 五、安装结果

执行完成后，`D:\opcuaSDK\open62541` 目录结构大致如下：

```text
D:\opcuaSDK\open62541\
├── include\                    ← 所有对外公开头文件
│   └── open62541\
│       ├── server.h
│       ├── client.h
│       ├── ua_types.h
│       └── ...
├── bin\                        ← 动态库（.dll）
│   └── open62541.dll
├── lib\                        ← 导入库（.lib，用于链接）
│   └── open62541.lib
└── lib\cmake\open62541\        ← CMake 配置文件
    └── open62541Config.cmake
```

> 这是**非合并模式**（未启用 `UA_ENABLE_AMALGAMATION`）的产物，是标准的"库 + 头文件"形态，适合作为依赖被其他项目引用。

---

## 六、在其他项目中引用

### 6.1 使用 CMake（推荐）

```cmake
list(APPEND CMAKE_PREFIX_PATH "D:/opcuaSDK/open62541")
find_package(open62541 REQUIRED)
target_link_libraries(你的项目名 PRIVATE open62541::open62541)
```

由于开启了 OpenSSL 加密，你的项目**可能还需要链接 OpenSSL 的库**（`libssl.lib`、`libcrypto.lib`），具体取决于 open62541 的链接方式（静态/动态）。如果出现未解析符号，把 OpenSSL 的 `lib` 目录加入"附加库目录"，并在链接器输入中加入 `libssl.lib` 和 `libcrypto.lib`。

### 6.2 使用 Visual Studio 手动配置

1. **附加包含目录**：`D:\opcuaSDK\open62541\include`
2. **附加库目录**：`D:\opcuaSDK\open62541\lib`
3. **链接器输入**：`open62541.lib`（以及可能的 `libssl.lib`、`libcrypto.lib`）
4. **运行时**：确保 `open62541.dll`（及 OpenSSL 的 DLL）位于可执行文件同目录或系统 PATH 中。

---

## 七、常见问题排查

### 问题 1：`Could NOT find OpenSSL ... (missing: OPENSSL_CRYPTO_LIBRARY OPENSSL_INCLUDE_DIR)`

- 确认 `OPENSSL_ROOT_DIR` 路径正确、无空格。
- 确认 `lib\` 根目录下**直接**存在 `libcrypto.lib` 和 `libssl.lib`（见第二节的关键步骤）。
- 路径使用正斜杠 `/` 或双反斜杠 `\\`。

### 问题 2：链接时报 `LNK2019 无法解析的外部符号 UA_ServerConfig_setDefaultWithSecurityPolicies`

说明当前链接的库**没有启用加密**。需按本文档重新用 `-DUA_ENABLE_ENCRYPTION=OPENSSL` 编译，并确保项目链接的是新产物。

### 问题 3：`static int LONG_MAX = ...` 报 C2059 语法错误

`LONG_MAX` 是 `<limits.h>` 中的标准宏，不能作为变量名。改用自定义常量名，或直接使用标准宏 `ULONG_MAX`。

### 问题 4：`winnt.h` 报 E1097 未知特性 `no_init_all`

这是 VS IntelliSense 与旧版 Windows SDK（如 10.0.19041.0）的兼容问题，**不要修改系统头文件**。在项目属性 → C/C++ → 预处理器 → 预处理器定义中新增：

```text
no_init_all
```

即可屏蔽该误报。

---

## 八、命令速查（完整流程）

```bash
# 1. 获取源码
git clone https://github.com/open62541/open62541.git
cd open62541
git submodule update --init --recursive

# 2. 配置（启用 OpenSSL 加密 + 动态库）
mkdir build
cd build
cmake -DUA_ENABLE_ENCRYPTION=OPENSSL -DOPENSSL_ROOT_DIR="D:/opcuaSDK/OpenSSL-Win64" -DBUILD_SHARED_LIBS=ON ..

# 3. 构建
cmake --build . --config Release

# 4. 安装到指定目录
cmake --install . --config Release --prefix "D:\opcuaSDK\open62541"
```


