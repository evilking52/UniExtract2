# UniExtract2 aardio 重构设计规格

- 日期：2026-07-23
- 仓库：`evilking52/UniExtract2`
- 目标分支：`refactor/aardio`
- 状态：已批准

## 1. 目标

在 `refactor/aardio` 分支中，用 aardio 完全替换 AutoIt 主程序，保留现有第三方提取器、定义资料、图标及必要资源，不调用、不发布旧版 `UniExtract.exe`。

首个可运行版本覆盖：ZIP、7Z、RAR、TAR、GZIP、BZIP2、XZ、CAB、MSI、MSU、Inno Setup、NSIS、ISO、WIM、DMG、通用 EXE 扫描；支持 GUI、命令行、单任务 FIFO 队列、密码列表、日志、进度、取消和批处理。

## 2. 硬约束

- Windows 10/11。
- aardio 主程序发布为 x86。
- 外部提取器沿用其原有架构。
- 完全禁止旧 AutoIt 可执行程序参与运行和发布。
- 使用重新设计的 aardio 原生界面，不逐像素复制旧界面。
- 程序始终便携，所有可变数据写入程序目录。
- 程序目录不可写时，启动检测失败并直接退出；不提权、不切换 AppData、不进入只读模式。
- 首版只支持简体中文、繁体中文、英文。
- 规则文件使用 UTF-8 无 BOM 的标准 INI 子集。
- 同一时间只执行一个任务。

## 3. 仓库策略

在 `refactor/aardio` 分支直接重建仓库根目录。旧 AutoIt 源码仅保留在 Git 历史和 `master` 分支，不复制到新发布目录。

## 4. 项目结构

```text
UniExtract2/
├─ main.aardio
├─ UniExtract2.aproj
├─ app/
│  ├─ bootstrap.aardio
│  ├─ paths.aardio
│  ├─ environment.aardio
│  └─ constants.aardio
├─ core/
│  ├─ task.aardio
│  ├─ taskQueue.aardio
│  ├─ ruleLoader.aardio
│  ├─ detector.aardio
│  ├─ dispatcher.aardio
│  ├─ executor.aardio
│  ├─ processRunner.aardio
│  ├─ progressParser.aardio
│  ├─ passwordProvider.aardio
│  ├─ cleanup.aardio
│  └─ result.aardio
├─ handlers/
│  ├─ generic/archive.aardio
│  ├─ archive/multipart.aardio
│  ├─ archive/encrypted.aardio
│  ├─ installer/inno.aardio
│  ├─ installer/nsis.aardio
│  ├─ installer/msi.aardio
│  ├─ image/iso.aardio
│  ├─ image/wim.aardio
│  ├─ image/dmg.aardio
│  └─ executable/peScan.aardio
├─ rules/
│  ├─ extensions.ini
│  ├─ signatures.ini
│  ├─ extractors.ini
│  ├─ priorities.ini
│  └─ aliases.ini
├─ ui/
│  ├─ mainForm.aardio
│  ├─ settingsForm.aardio
│  ├─ scanResultForm.aardio
│  ├─ passwordForm.aardio
│  └─ aboutForm.aardio
├─ services/
│  ├─ config.aardio
│  ├─ language.aardio
│  ├─ logger.aardio
│  ├─ history.aardio
│  └─ shellIntegration.aardio
├─ config/settings.ini
├─ lang/zh-CN.ini
├─ lang/zh-TW.ini
├─ lang/en-US.ini
├─ passwords/passwords.txt
├─ logs/
├─ temp/
├─ bin/
├─ def/
├─ support/
├─ tests/
└─ docs/
```

以上是职责边界，不强制制造只有少量转发代码的小文件。实现时允许合并强相关模块，但禁止反向依赖和职责混杂。

## 5. 模块边界

### 5.1 `main.aardio`

只负责导入启动模块、初始化应用、创建主窗口、启动消息循环、退出前请求队列安全停止。禁止放入格式识别、进程命令、配置解析和提取逻辑。

### 5.2 `app/`

负责计算程序根目录、创建与验证便携目录、检查写权限、检查规则和关键提取器、初始化配置/语言/日志，并将 aardio 路径转换为外部程序可识别的绝对 Windows 路径。

### 5.3 `core/`

与 UI 无关的任务引擎：任务模型、FIFO 队列、规则加载、检测、处理器选择、任务生命周期、外部进程、进度解析、密码重试、清理和统一结果。

### 5.4 `handlers/`

简单格式走 `generic.archive`。只有 Inno、NSIS、MSI/MSU、EXE 多阶段识别、分卷、密码轮询等存在条件分支的流程使用专用 handler。

### 5.5 `rules/`

只保存声明式数据，不包含条件、循环、正则替换脚本、aardio 代码或任意命令。复杂逻辑必须转为 handler。

### 5.6 `ui/`

只负责用户输入和展示。UI 向任务队列发送命令，通过窗口代理接收工作线程状态；工作线程不直接访问原生控件。

## 6. 依赖方向

```text
UI
 ↓
TaskQueue
 ↓
Executor
 ↓
Detector → Dispatcher → Handler
                  ↓
             ProcessRunner
                  ↓
        外部提取器与文件系统
```

横向服务：Config、Language、Logger、Paths。

禁止：`core/` 引用 `ui/`、handler 弹窗、INI 执行任意命令、UI 拼接工具命令、多个模块修改共享全局状态。

## 7. 任务状态

```text
queued
scanning
ready
extracting
password_required
cleaning
succeeded
failed
canceled
skipped
```

状态转换只由 `executor` 控制。UI 不得自行标记成功或失败。

## 8. INI 规则规范

所有规则文件：UTF-8 无 BOM；`[section]`；`key=value`；仅 `;` 注释；禁止重复节、重复键、嵌套节和可执行表达式。列表使用编号键。

### 8.1 `extensions.ini`

扩展名映射候选格式。按最长复合扩展名优先，例如 `tar.gz` 优先于 `gz`。

```ini
[extension.exe]
format.1=installer.inno
format.2=installer.nsis
format.3=archive.7z.sfx
format.4=executable.pe
```

### 8.2 `signatures.ini`

描述固定偏移二进制签名和有限文本特征。首版不支持通配十六进制、无界扫描或 INI 内复杂正则。

```ini
[signature.7z]
format=archive.7z
offset=0
hex=377ABCAF271C
priority=100
```

### 8.3 `extractors.ini`

描述外部工具路径、工作目录、参数模板、退出码、超时和能力。

允许模板变量：`{input}`、`{output}`、`{temp}`、`{password}`、`{programDir}`、`{fileName}`、`{fileBase}`、`{fileExt}`。

默认直接启动可执行文件，不经 `cmd.exe`。INI 禁止环境变量展开、管道、重定向和多命令拼接。

### 8.4 `priorities.ini`

最终候选分数：检测器权重 + 格式权重 + 规则优先级。相同分数时：专用 handler 优先、签名优先、具体格式优先；仍无法区分时进入方法选择。

### 8.5 `aliases.ini`

保存内部别名和日志回退显示名。正常 UI 文本仍来自语言包。

## 9. 启动校验

主窗口显示前完成：

1. 解析根目录。
2. 实际创建并删除测试文件，验证 `config/`、`logs/`、`temp/`、`passwords/` 可写。
3. 加载配置与语言。
4. 加载并校验全部规则。
5. 校验 handler、格式、提取器和模板引用。
6. 校验首版必需提取器。

错误必须给出文件、节、键和原因。核心规则或工具缺失时禁止启动。

## 10. 任务数据流

```text
用户添加文件
 → 规范化绝对路径
 → 验证输入与输出
 → 创建 queued 任务
 → FIFO 取任务
 → 创建独立临时目录
 → 扩展名初筛
 → 二进制签名检测
 → 文本/PE/安装器专用检测
 → 候选合并去重排序
 → 选择 handler 与 extractor
 → 执行提取
 → 解析输出、进度、退出码
 → 验证产物
 → 清理临时目录
 → 返回结构化结果
```

不能只凭退出码判定成功。必须验证新增产物、handler 明确产物或有效扫描结果。

## 11. 密码流程

首次无密码执行；明确识别为密码错误后，按 `passwords.txt` 顺序重试；全部失败后，非静默模式允许用户输入，静默模式返回密码失败。

密码不得写入普通日志，命令行日志必须遮蔽密码参数。每次重试前清理该次失败产物。

## 12. 输出冲突

默认输出到原文件同目录的文件名子目录。策略：`ask`、`merge`、`replace`、`rename`。GUI 默认 `ask`；批处理或静默模式未指定时使用 `rename`。禁止默认删除已有目录。

## 13. 失败与取消

失败：不修改输入文件；按设置保留部分正式产物；清理临时目录；记录工具、退出码和验证失败原因。

取消：设置取消标志、停止后续步骤、终止当前进程树、等待退出、清理临时目录、标记 `canceled`。默认继续下一项。

程序异常退出后，启动时检测残留任务目录，只支持清理、查看和保留，不实现伪任务恢复。

## 14. 主界面

布局由顶部菜单/工具栏、任务列表、右侧任务详情、总体进度和底部日志组成。

工具栏：添加文件、添加文件夹、移除、开始、暂停、停止、上移、下移、清空、设置。

任务列表列：序号、状态、文件名、大小、格式、方法/工具、进度、输出路径。允许调整列宽，不支持自由增删列。

暂停定义为“当前任务完成后暂停队列”，不挂起外部进程线程。

停止当前任务与停止全部分开。清空不得删除正在执行任务。

## 15. 设置

设置页：常规、输出、扫描、密码、队列、系统集成。

当前用户右键菜单可直接配置；所有用户右键菜单只有用户明确操作时才申请管理员权限。

配置保存使用临时文件和原子替换，写入失败保留旧配置并明确提示。

## 16. 语言

首版语言：`zh-CN.ini`、`zh-TW.ini`、`en-US.ini`，全部 UTF-8 无 BOM。缺失键回退英文，不加载旧 AutoIt 语言文件。

## 17. 线程模型

UI 线程只处理界面。解压工作在独立线程运行时中执行，依赖在工作线程显式导入。线程间仅传递普通 table 和基本值，不传控件、COM、进程对象或复杂闭包。

事件：`task_state`、`task_progress`、`task_log`、`task_result`、`queue_state`、`password_request`、`method_request`。

取消使用可共享取消状态，并在检测步骤、启动进程、输出读取、密码重试和清理前检查。强制结束 worker 仅作最后兜底。

## 18. 进程执行

`processRunner` 负责验证可执行文件范围、绝对路径、工作目录、启动进程、捕获 stdout/stderr、退出码、超时、取消、终止进程树和结构化结果。

默认不使用 `cmd.exe /c`。可执行文件与参数边界分别处理，输入路径不能改变命令结构。

## 19. CLI

```text
UniExtract.exe [options] <input...>
```

支持：`/extract`、`/scan`、`/output`、`/sub`、`/here`、`/password-file`、`/conflict`、`/silent`、`/log`、`/format`、`/method`、`/no-open`、`/?`。

退出码：

- 0 全部成功
- 1 至少一项失败
- 2 参数错误
- 3 输入不存在
- 4 规则或配置失败
- 5 必需提取器缺失
- 6 任务取消
- 7 静默模式需要密码
- 8 程序目录不可写
- 9 内部错误

GUI、CLI 和静默模式共用同一 TaskFactory、TaskQueue 和 Executor，只替换交互适配层。

## 20. 测试与验收

测试包含规则加载、候选排序、命令构造、输出冲突、进度解析、各格式集成、取消和 CLI。

每种首版格式至少覆盖正常、损坏、错误扩展名、Unicode 路径、空格路径、长文件名、只读输入；压缩格式额外覆盖密码、分卷、空档案和已存在输出目录。

发布必须从干净目录运行，不依赖 IDE 或开发机额外库；正确携带 `bin/`、`rules/`、`lang/`；不包含 AutoIt 主程序、AutoIt 依赖、测试文件和开发文档。

## 21. 风险与止损

停止增加功能并优先修复的条件：

1. 进程执行器不能稳定捕获退出码、输出或取消。
2. Unicode/空格路径破坏参数。
3. handler 出现大量跨格式例外，表明抽象错误。
4. INI 出现控制流需求。
5. 发布目录与 IDE 行为不一致。
6. 取消后残留进程或锁定文件。
7. 规则候选排序不可重复。

## 22. 实施原则

- 先建立可验证的进程执行器、路径层、规则加载器和队列状态机，再迁移格式。
- 每个格式处理器必须有固定样本与验收结果。
- 不为未来功能提前增加并发、插件脚本语言或复杂扩展框架。
- 每次提交只完成一个明确可验证的垂直切片。
