# Godot Pck 工具

一个用于解包和打包 Godot .pck 文件的独立可执行文件。

## 命令行用法

查看工具帮助信息：

```sh
./godotpcktool -h
# 或者
./godotpcktool --help
```

### 查看 pck 文件内容

列出 pck 文件内的文件：

```sh
./godotpcktool Thrive.pck
```

完整形式：

```sh
./godotpcktool --pack Thrive.pck --action list
```

### 解压 pck 文件

解压 pck 文件的内容：

```sh
./godotpcktool Thrive.pck -a e -o extracted
```

完整形式：

```sh
./godotpcktool --pack Thrive.pck --action extract --output extracted
```

### 添加文件到 pck

将文件添加到现有的 pck 或创建新的 pck。创建新 pck 时，可以使用 `set-godot-version` 参数指定 Godot 版本。

```sh
./godotpcktool Thrive.pck -a a extracted --remove-prefix extracted
```

完整形式：

```sh
./godotpcktool --pack Thrive.pck --action add --remove-prefix extracted --file extracted
```

文件会使用命令行指定的路径添加，但会移除指定的前缀。例如，如果有一个名为 `extracted/example.png` 的文件，使用上述命令后，该文件在 pck 中的路径将为 `res://example.png`。

建议在添加文件后使用列表命令查看 pck 内的结果数据，以验证操作是否正确。当新文件与 pck 内的路径完全匹配时，该文件将被替换。

如需更控制 pck 内的路径，请参阅下文关于 JSON 命令的说明。

## 过滤器

过滤器可用于仅处理 pck 文件或文件系统中的部分文件。

### 最小文件大小

指定最小大小，低于该大小的文件将被排除：

```sh
./godotpcktool --min-size-filter 1000
```

这将排除大小为 999 字节及以下的文件。

### 最大文件大小

指定最大大小，超过该大小的文件将被排除：

```sh
./godotpcktool --max-size-filter 1000
```

注意：如果同时使用最大和最小大小进行操作，应该从大小值中减一，否则会重复操作相同的文件。

如果想要处理特定大小的文件，可以指定相同的大小两次：

```sh
./godotpcktool --min-size-filter 1 --max-size-filter 1
```

### 按名称包含

可以使用正则表达式列表来选择要处理的文件，只处理匹配的文件。例如，列出名称中包含 "po" 的所有文件：

```sh
./godotpcktool --include-regex-filter po
```

或者只匹配文件扩展名（注意不同的 shell 需要不同的转义）：

```sh
./godotpcktool -i '\.po'
```

多个正则表达式可以用逗号分隔，或多次指定选项：

```sh
./godotpcktool -i '\.po,\.txt'
./godotpcktool -i '\.po' -i '\.txt'
```

如果未指定包含过滤器，所有文件都将通过。不指定包含过滤器意味着"处理所有文件"。

注意：过滤区分大小写。

### 按名称排除

文件也可以根据匹配的正则表达式被排除：

```sh
./godotpcktool --exclude-regex-filter txt
```

如果同时指定了包含和排除过滤器，则先应用包含过滤器，然后使用排除过滤器过滤通过第一个过滤器的文件。例如，查找包含 "po" 但不包含 "zh" 的文件：

```sh
./godotpcktool -i '\.po' -e 'zh'
```

### 覆盖过滤器

如果需要更复杂的过滤，可以使用 `--include-override-filter` 指定正则表达式，使匹配的文件即使被其他过滤器排除也会被包含。例如，设置文件大小限制后覆盖特定类型：

```sh
./godotpcktool --min-size-filter 1000 --include-override-filter '\.txt'
```

### JSON 批量操作

为了更好地控制 pck 文件内的路径，提供了 JSON 操作 API。

首先需要创建一个 JSON 文件（下面的例子中使用 `commands.json`，但可以使用任何名称），格式如下（可以根据需要指定任意数量的文件）：

```json
[
    {
        "file": "/path/to/file",
        "target": "overridden/path/file"
    },
    {
        "file": "LICENSE",
        "target": "example/path/LICENSE"
    }
]
```

然后运行以下命令（JSON 命令可以在添加操作时指定）：

```sh
./godotpcktool Thrive.pck -a a --command-file commands.json
```

这将读取 `/path/to/file`（可以是绝对或相对路径），并将其保存到 pck 中的 `res://overridden/path/file`，以及将 `LICENSE` 文件保存为 `res://example/path/LICENSE`。这样就可以使用绝对路径并指定文件在 pck 中的最终路径，以获得最大控制权。

注意：JSON 的 `target` 属性需要填写 pck 内的完整路径（不含 `res://` 前缀）。此模式不支持仅指定文件夹，因此多个文件使用 `pck/folder` 作为 `target` 会相互覆盖，而不是放置在 `pck/folder` 中。因此，指定 JSON 命令时始终应使用完整路径，如 `pck/folder/README.txt`，而不是 `pck/folder/`。

## 高级选项

### 指定引擎版本

创建 .pck 文件时可以指定 .pck 文件声明的 Godot 引擎版本：

```sh
./godotpcktool NewPack.pck -a a some_file.txt --set-godot-version 3.5.0
```

请注意，这种方法**不会**覆盖现有 .pck 文件中的引擎版本号。这目前仅适用于新的 .pck 文件。

### 脚本化使用

可以在不创建临时文件的情况下使用 JSON 批量 API。方法是指定 `-` 作为要添加的文件，然后向工具的 stdin 写入 JSON 并关闭它：

```sh
./godotpcktool NewPack.pck -a a -
```

启动上述命令后，工具将读取 stdin 的所有行直到关闭，然后解析输入的 JSON。由于需要关闭 stdin 才能继续，因此不适用于交互式使用，仅适用于脚本。请参阅上文关于 JSON 命令文件的说明以了解可用的命令行版本。

接受的 JSON 格式与 JSON 命令文件格式相同，请参阅上文。

## 常规信息

完整形式中可以包含多个文件：

```sh
./godotpcktool ... --file firstfile,secondfile
```

如果文件包含空格，请务必使用引号，否则文件将被解释为其他选项。

简短形式中文件可以列在其他命令后面。如果文件以 `-` 开头，可以在参数和文件列表之间添加 `--` 来防止被解释为参数。

## 编译

本项目使用 CMake，第三方库已包含在源代码中。

### Linux / macOS

```sh
mkdir build
cd build
cmake ..
make
```

编译后的二进制文件位于 `build/src/godotpcktool`。

### Windows (MinGW)

```sh
mkdir build
cd build
cmake .. -G "MinGW Makefiles"
mingw32-make
```

### 在 Linux 上交叉编译 Windows 版本

```sh
mkdir build
cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=../cmake/i686-w64-mingw32.cmake -DCMAKE_BUILD_TYPE=Release
make
```

或者使用提供的 Makefile：

```sh
make all-install
```

### Podman 编译

为了与旧版 Linux 发行版获得最大兼容性，可以使用最旧的受支持 Ubuntu LTS 进行编译：

```sh
make compile-podman
```

由于 C++17 和 cmake 的要求，本地编译需要 Ubuntu 22.04 或更高版本。
