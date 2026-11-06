# dart-tui-AGENTS.md

> 用纯 Dart（不依赖 Flutter）写跨平台终端 TUI 的实战手册。
> 内容全部来自本仓库 `mihomo-tui` 的实现（原 Python CLI 重写为 Dart TUI），记录了**容易踩坑的点**、**关键实现方式**和**可直接搬运的代码骨架**。
>
> 用法：把这份文档放在新项目根目录，改名为 `AGENTS.md`，按第 8 节的顺序搭骨架；遇到具体问题时查第 7 节的速查表。

---

## 0. 适用场景

适合：

- 把已有的 CLI / 脚本升级成可交互的终端面板
- 需要鼠标、表单、列表、多面板、实时刷新的 TUI
- 必须编译成单文件、同时支持 Linux 与 Windows（含 arm）的工具

不适合：一次性输出、纯管道命令、只需要 ANSI 上色的简单输出。

---

## 1. 依赖与项目骨架

### 依赖（保持极少）

```yaml
dependencies:
  ffi: ^2.1.3      # 仅 Windows 平台后端与进程管理需要
  yaml: ^3.1.2     # 解析业务配置文件
  archive: ^4.6.0  # 导出 zip / 解压内核压缩包
dev_dependencies:
  test: ^1.25.15
  lints: ^5.1.1
```

不要找现成的 Dart TUI 框架：生态里没有能同时满足「鼠标 + 跨平台 + CJK 宽度」的库，自研终端层反而更快。

### 分层

```
bin/<app>.dart            入口：参数解析 + 无 TTY 子命令 + 启动 TUI
lib/src/term/             终端层（与业务无关，可整目录复制到新项目）
  keys.dart               事件模型：KeyEvent / MouseEvent / ResizeEvent / ColorReportEvent
  width.dart              runeWidth / textWidth / truncate / wrapText
  style.dart              颜色与 SGR 生成（truecolor + 256 回退）
  screen.dart             Cell / Screen（帧缓冲）+ ScreenRenderer（diff + 输出）
  input_decoder.dart      ANSI/CSI/SS3/OSC/SGR 鼠标 解析
  terminal.dart           平台后端（POSIX / Windows）+ 备用屏幕 + 原始模式
  win32.dart              Windows FFI（控制台 + 进程）
lib/src/ui/               UI 层
  theme.dart              调色板与预设强调色
  painter.dart            Rect / Painter / 命中区域 Hit
  list_state.dart         列表光标 + 视口（滚动）状态
  overlay.dart            弹窗：确认 / 表单 / 信息 / 帮助 / 选择 + TextInput
  app.dart                应用壳：布局、按键路由、渲染循环、后台任务
  app_context.dart        视图能用的接口（避免循环依赖）
  views/                  各页面：view.dart 定义接口，其余实现
lib/src/service/          业务层：配置、进程、HTTP、下载、导出、系统集成
lib/src/util/             无状态小工具：format / shell / clipboard / net / log_entry
```

---

## 2. 终端层（最容易被低估的部分）

### 2.1 渲染模型：cell 缓冲 + 帧间 diff

每帧把整个屏幕画进一个 `Screen`（`List<Cell>`，每个 cell = rune + Style），再和上一帧比较，只把变化的连续区间写出去。

```dart
class Cell {
  int rune = 0x20;
  Style style = Style.plain;
  bool continuation = false; // 宽字符的右半格
}
```

渲染器用两条 `Int32List` 缓存上一帧的 `rune` 和 `styleId`：

- **Style 要 intern 成 int**：`Map<Style,int>`，比较 int 而不是对象，diff 才够快（10k cell × 60fps）
- 宽字符占两格：左格存 rune，右格 `continuation=true, rune=0`；输出时跳过 continuation 格
- diff 区间两端要"扩展"到宽字符边界：起点落在 continuation 上就往左退一格，终点右边是 continuation 就吞进来
- 整行没变化就完全跳过；跨行保留 `lastStyleId`，减少重复 SGR

### 2.2 宽度与字符集（决定边框是否对齐）

- 自建 `runeWidth` 表：CJK/全角 = 2，组合符/零宽 = 0，其余 = 1；**不要依赖 `characters` 包或系统 wcwidth**
- 所有布局（截断、填充、对齐、光标定位）都按**显示列**算，不按码点
- **界面里只用窄字符**：`-`、`|`、`~` 代替 `—`（U+2014）、`·`（U+00B7）、`…`（U+2026）；`上下/左右` 代替 `↑↓/←→`；掩码用 `*` 代替 `•`。这些"歧义宽度（East Asian Ambiguous）"符号在中文终端/字体里常被渲染成 2 列，一出现就会顶飞整行边框
- 框线字符（`╭╮╰╯─│┃┊`）保留：终端普遍按 1 列渲染

### 2.3 渲染器的绝对定位（emoji 救星）

终端对 emoji / 国旗 / ZWJ 组合的实际宽度与 wcwidth 不一致（`🇭🇰` 可能是 2 列而不是 4 列，`👨‍👩‍👧` 可能是 2 列而不是 6 列）。整行连续输出时，一个字符窄了，后面全部左移，**行尾边框写不到、旧内容残留**。

解法：把非 ASCII 字形按"风险段"处理。

```dart
// 安全字形：ASCII、框线、CJK/全角区段 → 连续输出（快）
bool _safeWidth(int rune) { /* ASCII / 0x2500-0x257F / CJK 区段 */ }

// 行内输出
var expectedCol = -1;
for (var col = first; col <= last; col++) {
  final cell = cells[rowStart + col];
  if (cell.continuation) continue;
  final rune = cell.rune;
  if (rune < 0x20) continue;

  if (_safeWidth(rune)) {
    if (col != expectedCol) moveTo(row, col);
    out.writeCharCode(rune);
    expectedCol = col + runeWidth(rune);
    continue;
  }

  // 收集连续的"风险"段（同一样式），先清空占位再整段画，段后强制重定位
  final start = col;
  final buffer = StringBuffer();
  var end = col;
  while (col <= last) { ...; buffer.writeCharCode(cell.rune); end = col + runeWidth(cell.rune); col++; }
  col--;
  moveTo(row, start);
  out.write(' ' * (end - start));  // 清掉可能残留的旧字符
  moveTo(row, start);
  out.write(buffer.toString());    // 整段一起写，旗帜/ZWJ 仍能正确组合
  expectedCol = -1;
}
```

要点：风险段**整段一次写出**（否则国旗会被拆成两个字母），但段的前后都用绝对定位兜底。

### 2.4 输入解析

`stdin.echoMode = false; stdin.lineMode = false;`（Dart 内置，POSIX 有效），然后自己解析字节流：

- 单字节：`\r`/`\n` = Enter、`\t` = Tab、`0x7f`/`0x08` = Backspace、`<0x20` = Ctrl+字母
- `ESC [`（CSI）：方向键 `A/B/C/D`、Home/End `H/F`、`1;5C` 这类带修饰键的、`3~/5~/6~` 等
- `ESC O`（SS3）：F1–F4、方向键
- `ESC <printable>` = Alt+char
- `ESC ]`（OSC）：终端颜色回复（见 2.10），用 BEL 或 `ESC \` 结束
- 鼠标（SGR 1006）：`ESC [ < b ; x ; y M|m`，`b&64` 是滚轮，`b&32` 是移动，`b&3` 是键位，`b&4/8/16` 是修饰键
- UTF-8 多字节要跨 read 拼接；**单独一个 ESC 要等 30–40ms 超时**再当 Escape 事件

### 2.5 stdin 只能订阅一次（外部编辑器必读）

Dart 的 `stdin` 是单订阅流：

- `subscription.cancel()` 之后再 `listen` 会抛 `Stream has already been listened to`
- 更坑的是：**cancel 之后 fd 0 对子进程不可用**，`$EDITOR` 启动会报"标准输入不是一个终端"

正确做法：打开编辑器时 `pause()`，回来 `resume()`，绝不 cancel。

```dart
void suspend() {                 // 交给子进程
  _subscription?.pause();
  _resizeSub?.pause();
  stdin.echoMode = _originalEcho;
  stdin.lineMode = _originalLine;
  stdout.write('\x1b[0m\x1b[?1003l\x1b[?1006l\x1b[?1000l\x1b[?25h\x1b[?1049l');
}
void resume() {                  // 子进程结束
  stdin.echoMode = false;
  stdin.lineMode = false;
  stdout.write('\x1b[?1049h\x1b[?25l\x1b[?1000h\x1b[?1002h\x1b[?1003h\x1b[?1006h\x1b[2J\x1b[H');
  _subscription?.resume();
  _resizeSub?.resume();
  renderer.invalidate();         // 强制全量重画
}
```

外部编辑器：`Process.start(editor, [file], mode: ProcessStartMode.inheritStdio)`，等 `exitCode`。

### 2.6 Ctrl+C 与 ISIG（否则表单里按 Ctrl+C 直接退出）

Dart 的 `lineMode=false` 只关掉 ICANON/ECHO，**ISIG 还开着**，所以 Ctrl+C 不进输入流，而是变成 SIGINT 把程序杀掉——你的输入框永远收不到 Ctrl+C。

解法：进入 TUI 时用 `stty` 关掉 ISIG，退出/挂起时恢复。

```dart
Future<void> _setSignalKeys(bool enabled) async {
  final flag = enabled ? 'isig' : '-isig';
  // 关键：用 sh 从 /dev/tty 拿 stdin，且丢弃 stderr
  await Process.run('sh', ['-c', 'stty $flag < /dev/tty 2>/dev/null']);
}
```

两个细节：

- **不要用 `inheritStdio` 让 stty 继承 fd 0**：退出流程里 stdin 订阅已 cancel，子进程会拿到坏句柄并打印 `stty: 标准输入: 错误的文件描述符`
- 恢复 ISIG 要放在 cancel stdin 订阅**之前**；拿不到 `/dev/tty` 就静默跳过（退化成 Ctrl+C 直接退出）

### 2.7 备用屏幕、鼠标、还原

```dart
// 进入
'\x1b[?1049h\x1b[?25l\x1b[?1000h\x1b[?1002h\x1b[?1003h\x1b[?1006h\x1b[2J\x1b[H'
// 退出（务必在 finally / 信号处理里执行）
'\x1b[0m\x1b[?1003l\x1b[?1002l\x1b[?1006l\x1b[?1000l\x1b[?25h\x1b[?1049l'
```

- `1000` 点击、`1002` 拖拽、`1003` 纯移动（hover 需要）、`1006` SGR 坐标
- 用 `stdout.add` 写 UTF-8 字节；终端已关闭时写入要 try/catch
- 监听 `SIGINT` / `SIGTERM` / `SIGHUP`，收到就"优雅退出"（还原终端再退出），否则 tmux/终端关掉后会留孤儿进程

### 2.8 尺寸变化

- POSIX：`SIGWINCH` **延迟 40ms 再读尺寸**（很多终端在信号瞬间还报旧值）
- 兜底：主循环每秒比对一次 `stdout.terminalColumns/terminalLines` 与帧缓冲尺寸，不一致就 resize（否则长行会折行、看起来像边框被破坏）
- Windows：`GetConsoleScreenBufferInfo` 轮询

### 2.9 剪贴板（无插件）

按平台尝试命令，全部加超时与异常兜底：

| 平台 | 写 | 读 |
| --- | --- | --- |
| Linux | `wl-copy` → `xclip -selection clipboard` → `xsel --clipboard --input` | `wl-paste --no-newline` → `xclip -o -selection clipboard` → `xsel -o --clipboard` |
| macOS | `pbcopy` | `pbpaste` |
| Windows | `clip` | `powershell -NoProfile -Command Get-Clipboard -Raw` |

**异常绝不能逃逸**：`unawaited(Clipboard.write(...))` 抛出的未捕获异常会直接杀掉进程。每个分支都 `try/catch`，`Process.run/exitCode` 都加 `timeout`。

### 2.10 深/浅色探测

1. 发 `\x1b]11;?\x1b\\`，解析回复 `\x1b]11;rgb:RRRR/GGGG/BBBB`（也支持 `#RRGGBB`），250ms 超时
2. 兜底读环境变量 `COLORFGBG`（如 `15;0` → 背景是 0 号色），用 xterm 16 色调色板映射
3. Windows 用 `GetConsoleScreenBufferInfo` 的 `wAttributes` 高 4 位取背景色索引
4. 都拿不到 → 默认深色；同时在设置里提供「明暗模式：跟随终端 / 深色 / 浅色」

配色方案建议：**不涂背景色（透明）**，只画文字、边框、强调色，选中态用 `>` + 加粗（或反色），这样深浅色终端都不用改配色；强调色给 6 个预设（蓝/紫/绿/橙/青/玫红），从强调色派生一个 `accentSoft`（向黑/白偏移 22%）给"取值类文字"用。

### 2.11 Windows 后端（FFI 要点）

只用一个文件、懒加载 `kernel32.dll`：

- 输出：`SetConsoleMode(ENABLE_VIRTUAL_TERMINAL_PROCESSING)` + `SetConsoleOutputCP(65001)`，之后就能直接写 ANSI
- 输入：**不要用 `stdin.listen`**（Windows 控制台不给你按键）；用 `ReadConsoleInputW` + 16ms 定时器轮询
- `INPUT_RECORD` = 20 字节：`EventType` 在 0；`KEY_EVENT` 偏移 `4(bKeyDown) / 8(repeat) / 10(vk) / 12(scan) / 14(unicode) / 16(controlState)`；`MOUSE_EVENT` 偏移 `4(x int16) / 6(y) / 8(buttons) / 12(control) / 16(flags, 4=wheel)`
- 结构体用 `Struct` + `@IntPtr()` 声明指针字段（不要手算偏移，32/64 位布局不同）
- 后台进程：`CreateProcessW` + `STARTUPINFO` 重定向 stdout/stderr 到日志文件（见 4.1），`CREATE_NO_WINDOW`；杀进程用 `OpenProcess`+`TerminateProcess`
- 变量：`STARTF_USESTDHANDLES`、`SECURITY_ATTRIBUTES{inherit=true}`、`OPEN_ALWAYS`、`FILE_APPEND_DATA`

---

## 3. UI 层

### 3.1 Painter：画什么 + 哪里有可点区域

```dart
class Painter {
  final Screen screen;
  final Theme theme;
  final List<Hit> hits = [];          // 每帧重建
  void text(int x, int y, String s, Style style, {int? maxWidth});
  void fill(Rect r, Style style);
  void drawBox(Rect r, {bool focused, String? title});
  void hit(Rect r, {String id, void Function()? onPress, void Function(int)? onWheel,
                    void Function()? onHover});
}
```

- 命中区域在**绘制时**注册，事件按 `hits.reversed` 命中（后画的在上层）
- 弹窗绘制在最后 → 自然获得最高优先级
- hover：`onHover` 只在鼠标进入时触发一次（用 `hoverId` 比较），列表里让它**移动光标**

### 3.2 布局与边框规则（避免 `││` 双边框）

- 顶栏 1 行、底栏 1 行，中间是内容区；侧栏固定宽度，内容区吃剩余
- **每个视图自己拥有最外层边框**（App 不再画一层内容框），标题写在边框上，聚焦时边框换色
- 需要分区的页面（比如"列表 + 详情"）用两块**独立卡片**：列表占满剩余高度，详情卡片固定在底部、高度 = 内容行数 + 2（**不要留空行**）
- 边框**最后画**：即使文字因为宽度问题溢出半格，也会被边框覆盖回来
- 滚动条画在**边框列本身**（只换颜色/字形），不要占用内容列，否则紧贴边框形成 `││`
- 表格列要留间隙：`name | nodes | ports | updated | url` 之间各留 1 列

### 3.3 焦点与按键路由

```
覆盖层（弹窗栈最上层） > 全局键（q、?、h、1-5）> 侧栏/内容区 > 当前视图 > 回退（Esc/Tab 回侧栏）
```

- 侧栏和内容区两个焦点域，`Tab` 在侧栏与内容区之间切（内容区视图可以自己消费 Tab，比如双栏切换）
- 视图 `onKey` 返回 bool，未消费才回退
- 大写快捷键要用**区分大小写**的匹配（`isCharExact('R')`），否则 `isChar('R')` 会把 `r` 也吃掉

### 3.4 列表滚动状态（避免"底部空行"和"滚轮不动"）

```dart
class ListState {
  int selected = 0, scroll = 0, total = 0, visible = 1;
  int get maxScroll => total > visible ? total - visible : 0;

  void bind({required int total, required int visible}) { /* clamp selected/scroll */ }
  void wheel(int delta) { scroll = (scroll + delta).clamp(0, maxScroll);
                          if (selected < scroll) selected = scroll;
                          if (selected >= scroll + visible) selected = scroll + visible - 1; }
  void moveBy(int delta) { selected = (selected + delta) % total; ensureVisible(); }
  void ensureVisible() { /* 光标出了视口就把视口拉过来，并夹紧 */ }
}
```

三条铁律：

1. 渲染循环用 `visible = min(容量, total - scroll)`，**不要**写死容量（否则滚到底会出现空行）
2. 容量要按真实行高算（行高 2 行 + 1 行间隔的列表，1 行 height 装不下 2 行）
3. **滚轮事件要能落在任意位置**：给行命中区多覆盖下面那行间隔，并在没有命中任何区域时回落到 `view.onWheel()`

### 3.5 高亮风格（用户偏好）

- 光标行：`>` 标记 + 加粗 + **不画背景色块**（背景色块会盖住边框、也在透明终端里很丑）
- 悬停：鼠标移动即移动光标（列表内），侧栏悬停只高亮不切页
- 语义色单独保留：绿=启用/成功、红=错误/未运行、黄=警告；其余"取值类文字"用 `accentSoft`

### 3.6 表单与文本输入

`TextInput` 需要支持：光标（rune 索引）、选区（anchor）、`insert/backspace/deleteForward/deleteWordBefore/deleteToEnd`、`selectAll/deleteSelection`、`moveLeft/Right/Home/End({extend})`、`setCursorFromColumn`（鼠标点击定位）。

快捷键映射（在表单层，不要在输入框里判断 Ctrl）：

| 键 | 行为 |
| --- | --- |
| `Ctrl+A` | 全选 |
| `Ctrl+C` | 复制选区（无选区时复制整行） |
| `Ctrl+X` | 剪切选区 |
| `Ctrl+V` | 粘贴（换行/制表符转空格） |
| `Ctrl+U/K/W/E` | 清空 / 删到行尾 / 删前一个词 / 光标到行尾 |
| `Shift+←/→/Home/End` | 扩展选区 |
| `Tab` / `Enter` / `Esc` | 切字段 / 提交 / 取消 |

渲染细节：

- 选区用 `reverse: true` 覆盖（不引入背景色，深浅色终端都可见）
- 光标用"反色字符块"画，不要用硬件光标（定位麻烦）
- **横向滚动跟随光标**：`offset = clamp(cursorColumns - visibleWidth + 1, 0, maxOffset)`，否则长 URL 永远只能看到尾部
- 密码字段用 `*` 掩码；CJK 光标列用显示宽度算

### 3.7 弹窗

统一基类 `Overlay { draw(painter); handleKey; handleMouse; dismiss(); }`，App 维护一个栈：只让最上层处理按键；`handleMouse` 返回 true 表示已消费（点外部关闭）。

现有类型：`ConfirmOverlay`（确认/危险操作）、`FormOverlay`（多字段 + 复选框 + 校验）、`InfoOverlay`（长文本滚动）、`HelpOverlay`（快捷键表）、`MirrorOverlay`（带状态的选择列表）。

### 3.8 主题

```dart
Theme.build({required bool light, String accent = 'blue'})  // 明暗 + 强调色
```

- 调色板只保留：`text / muted / border / borderFocus / accent / green / red / yellow / purple / cyan / chipText`
- 背景、panel、selection 全部 `null`（透明），深浅色只影响文字与边框
- `accentSoft` = 强调色向黑/白偏移 22%，给路径、端口、订阅名这类"取值"用

---

## 4. 业务层常见件

### 4.1 后台进程 + 日志（最容易踩的坑）

**不要用管道托管长期进程**：父进程退出后管道关闭，Go 程序（mihomo/clash）写 stdout 会 SIGPIPE **直接死掉**。

- POSIX：用 `sh` 启动并重定向到日志文件，自己拿走 pid

```dart
const script = r'''
log="$1"; shift
mkdir -p "$(dirname "$log")" || exit 3
nohup "$@" >> "$log" 2>&1 < /dev/null &
echo $!
''';
final p = await Process.start('/bin/sh', ['-c', script, 'sh', logPath, exe, ...args]);
final pid = int.parse((await p.stdout.transform(utf8.decoder).join()).trim());
```

- Windows：`CreateProcessW` + `STARTUPINFO.hStdOutput = 日志文件句柄`
- pid 用文件记录（`run/<x>.pid`），"是否在跑"优先用 API 探活，其次 pid 存活检查，最后扫进程名
- **区分"自己启动的"和"外部启动的"进程**，动外部进程前必须二次确认（可能是别的工具在托管）
- 启动后等 500–800ms 再检查存活；死了就把日志尾部几行弹出来
- 日志文件超过 5MB 轮转为 `.1`

### 4.2 HTTP API 客户端

- `HttpClient` 共享实例，`connectionTimeout` 2–3s，每请求 `timeout`
- 统一的 `ApiException(message)`：连接失败 / 超时 / 非 2xx（带 body 片段）/ 解析失败
- 认证头 `Authorization: Bearer <secret>`；URL 里拼名字要 `Uri.encodeComponent`
- 探测不要每秒都打重接口：代理/连接这类列表按 1–3s 节流，版本号缓存

### 4.3 下载与校验

- 订阅下载：`HttpClient` + 代理（见 `util/net.dart`，尊重 `https_proxy`/`all_proxy`，**回环地址永不走代理**）
- 先写 `.part`，校验通过再原子 rename；失败清理临时文件
- 内容用 `package:yaml` 校验结构，不合法就报错并保留旧文件
- UA 用 `clash.meta/<app>`：很多机场靠 UA 返回 Clash 配置

### 4.4 系统集成

| 功能 | Linux | Windows |
| --- | --- | --- |
| 系统代理 | `gsettings set org.gnome.system.proxy ...`（先读 `mode` 判断状态） | 写 `HKCU\...\Internet Settings` + `InternetSetOptionW` 刷新 |
| 开机自启 | 写 `~/.config/systemd/user/<x>.service` + `systemctl --user enable --now` | 写 `HKCU\...\CurrentVersion\Run` |
| 打开 URL | `xdg-open` | `cmd /c start "" <url>` |
| 剪贴板 | 见 2.9 | 见 2.9 |

关闭/卸载也要实现，且**关闭前确认**（用户可能不是用它设的）。

### 4.5 在线更新核心 + 镜像

- 版本：`GET https://api.github.com/repos/<org>/<repo>/releases/latest`；**被限流（60/h/IP）时回退**到 `releases/latest` 的 302 `Location` 解析 tag；支持 `GITHUB_TOKEN` 提额
- 资产名按约定拼：`<bin>-<os>-<arch>-<tag>.<ext>`，跳过 `-v1/-v2/-v3`、`-go1xx` 变体，回退到 `-compatible`
- 解包：`.gz` 用 `GZipCodec()`，`.zip` 用 `archive`；chmod 755
- 校验：装完跑 `<bin> -v` 解析版本号，失败自动用 `.bak` 还原
- 镜像：前缀拼在原 URL 前（`https://ghproxy.net/<url>`），列表可配置；**连通性检测**用 `Range: bytes=0-65535` 请求首块测 TTFB（10s 超时），结果显示在 UI（`可用 365ms` / `不可用（超时）`），不要做测速排名

---

## 5. 测试与调试

### 5.1 单元测试（纯逻辑，不碰终端）

必测：`runeWidth/textWidth/truncate/wrapText`、按键解码（含 SGR 鼠标、OSC 颜色、分片到达）、`Screen.putText` 宽字符、渲染器 diff（连续两帧输出为空）、配置解析、仓库 CRUD（用临时目录 + 本地 `HttpServer` 当订阅源）、导出 zip 能回读、排序、日志行解析。

`dart test` 要能离线跑：涉及网络的测试自己起 `HttpServer`（绑 `127.0.0.1:0`）。

### 5.2 tmux 端到端（推荐）

```sh
tmux new-session -d -s t -x 118 -y 34 ./build/app
tmux capture-pane -t t -p            # 看画面
tmux send-keys -t t 'j'              # 按键
tmux send-keys -t t -H 1b 5b 3c 30 3b 37 31 3b 37 4d   # 注入 SGR 鼠标点击
tmux capture-pane -t t -e -p         # 带 SGR 的原始输出，用来验证颜色/反色
tmux resize-window -t t -x 100 -y 24 # 验证 resize
```

- 用 `capture-pane -e` 验证颜色（`48;2;r;g;b`）、反色（`\x1b[7m`）
- 检查边框：写个脚本按 wcwidth 求每行第 N 列字符，确认是 `│`；非 0 即边框被破坏
- 测完记得 `tmux kill-server`、停掉测试进程

### 5.3 伪造后端

- 假内核：Python/Shell 脚本模拟 `-v`、写日志、起一个假 REST API（`/version`、`/configs`、`/proxies`、`/connections`），端口用 19xxx 避开真实服务
- 假订阅服务：`python3 -m http.server` 或几行 `http.server`，返回带节点/分组的 YAML
- 用 `--data /tmp/test-data --home /tmp/test-home --bin /tmp/fake/core` 隔离，不碰真实配置
- 测试脏数据：超长 CJK 名称、emoji/国旗/ZWJ 名称、嵌套分组、0 节点配置

### 5.4 常见现象 → 原因

| 现象 | 原因 / 检查点 |
| --- | --- |
| 边框错位/缺一格 | 行内有歧义宽度字符；或 emoji 宽度不一致（见 2.3）；或写到了最后一列 |
| 滚动后残留旧字 | 宽字符 continuation 被跳过但占位没清；或整行连续输出时发生宽度漂移 |
| 第 116 列不是 `│` | 文本 maxWidth 没给边框留位；或边框不是最后画的 |
| `││` 双边框 | 内容框与视图框嵌套；应让视图自己拥有外框 |
| 底部出现空行 | 列表可见容量写死；见 3.4 |
| 表单里 Ctrl+C 退出程序 | ISIG 没关（2.6） |
| 编辑器启动报"标准输入不是终端" | stdin 订阅被 cancel（2.5） |
| 父进程退出后内核也死了 | 用管道托管了子进程（4.1） |
| 进程留下孤儿 | 没监听 SIGHUP/没还原终端 |
| `stty: 标准输入: 错误的文件描述符` | stty 继承了我们已 cancel 的 fd 0（2.6） |

---

## 6. CI/CD

```yaml
test:   dart format --output=none --set-exit-if-changed .
        dart analyze --fatal-infos
        dart test
        dart compile exe bin/app.dart -o dist/app && ./dist/app --selftest
build:  matrix: ubuntu-latest(x64) / ubuntu-24.04-arm(arm64) /
                windows-latest(x64) / windows-11-arm(arm64)
        dart compile exe ... -o dist/app[.exe]
release: 仅 tag v* 触发；下载全部 artifact，生成 SHA256SUMS，softprops/action-gh-release
```

要点：

- `dart compile exe` **不会创建输出目录**，先 `mkdir -p dist`；覆盖正在运行的二进制会报 `Text file busy`
- 32 位 ARM 没有 hosted runner：`docker run --platform linux/arm/v7` + qemu，在容器里下载 `dartsdk-linux-arm-release.zip` 再编译（该 job 标 `continue-on-error`）
- release job 用 `if: always() && startsWith(github.ref, 'refs/tags/v')`，避免某个平台失败把整个发布吞掉
- `.gitignore`：`build/`、`dist/`、`.dart_tool/`、本地下载的内核目录

---

## 7. 踩坑速查表（浓缩版）

1. **Ctrl+C**：关 ISIG，`sh -c 'stty -isig < /dev/tty 2>/dev/null'`；恢复放在 cancel 之前
2. **外部编辑器**：`pause()/resume()`，绝不 cancel stdin
3. **子进程**：重定向到文件，不要管道；否则 Go 子进程 EPIPE 直接死
4. **宽字符**：自建宽度表；ASCII/CJK 连续写，emoji 段单独定位 + 先清占位
5. **歧义宽度字符**：界面里一律不用
6. **边框**：视图自己拥有最外层框；边框最后画；文本右侧留一列
7. **列表滚动**：`ListState`，容量按真实行高算，滚轮事件回落到视图
8. **高亮**：`>` + 加粗，不画背景
9. **剪贴板/async**：所有 `unawaited` 的地方都要 try/catch，否则未捕获异常杀进程
10. **resize**：SIGWINCH 延迟 40ms + 每秒比对尺寸
11. **大写快捷键**：`isCharExact`，别让 `isChar('R')` 吃掉 `r`
12. **颜色探测**：OSC 11 → COLORFGBG → 控制台属性 → 默认深色

---

## 8. 复用到新项目的顺序（checklist）

1. 复制 `lib/src/term/`（keys/width/style/screen/input_decoder/terminal/win32）——**这层与业务无关，能直接用**
2. 复制 `lib/src/ui/painter.dart`、`list_state.dart`、`overlay.dart`，删掉业务相关的弹窗
3. 写 `theme.dart`（透明背景 + 一个强调色 + accentSoft）
4. 写 `app.dart` 骨架：顶栏/侧栏/内容/底栏 + 事件路由 + 每帧 diff 渲染 + 1s ticker
5. 定义 `AppContext`（视图需要的服务接口），避免视图直接依赖 App
6. 业务层：paths/store/models → 进程或 API → 各视图
7. 无 TTY 子命令（`--status`、`--selftest` 等）先写好，CI 用它们做冒烟
8. 单测补齐（宽度/解码/渲染/diff/解析/CRUD）
9. tmux 手动过一遍：鼠标、滚轮、resize、退出还原
10. workflow 矩阵 + `v*` 发布，先发一个 v0.x 验证资产

---

## 9. 本仓库文件地图（对照用）

| 关注点 | 文件 |
| --- | --- |
| 帧缓冲与 diff 渲染 | `lib/src/term/screen.dart` |
| 宽度/截断/换行 | `lib/src/term/width.dart` |
| ANSI/OSC/鼠标解码 | `lib/src/term/input_decoder.dart` |
| POSIX/Windows 后端、ISIG、尺寸 | `lib/src/term/terminal.dart` |
| Windows FFI | `lib/src/term/win32.dart` |
| 主题/强调色/透明 | `lib/src/ui/theme.dart` |
| 命中区域/绘制 | `lib/src/ui/painter.dart` |
| 列表滚动 | `lib/src/ui/list_state.dart` |
| 表单/选区/剪贴板快捷键 | `lib/src/ui/overlay.dart` + `lib/src/util/clipboard.dart` |
| 布局、路由、渲染循环 | `lib/src/ui/app.dart` |
| 进程托管/日志/pid | `lib/src/service/launcher.dart`、`mihomo_core.dart` |
| REST 客户端 | `lib/src/service/mihomo_api.dart` |
| 下载与代理 | `lib/src/service/downloader.dart`、`lib/src/util/net.dart` |
| 导出 zip | `lib/src/service/exporter.dart` |
| 系统代理 / 自启动 | `lib/src/service/system_proxy.dart`、`autostart.dart` |
| 在线更新内核 + 镜像 | `lib/src/service/core_updater.dart` |
| CI 矩阵与发布 | `.github/workflows/build.yml`、`tool/build-linux-arm.sh` |