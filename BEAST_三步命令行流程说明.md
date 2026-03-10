# BEAST 三步命令行流程说明（PowerShell）

由于BEAST系列软件的exe文件BUG较多，运行不稳定，且不可修改运行内存，当exe文件运行出错时建议使用命令行方法。
这份文档给第一次用 BEAST 命令行的人准备。
三个步骤分别为：
1. 先跑 MCMC（BEAST）
2. 再整理树文件（LogCombiner）
3. 最后生成注释树（TreeAnnotator）

## 1. 这 3 步分别在做什么

1. `beast.bat`：按 `XML` 模型做采样，产出 `.log` 和 `.trees`
2. `logcombiner.bat`：对 `.trees` 去掉前期不稳定样本（burn-in）并降采样
3. `TreeAnnotator`：把后验树样本汇总成一棵代表树（常见是 MCC 树）

## 2. 前置条件（先准备好再跑）

1. 安装好 BEAST，并确认目录存在，例如：`D:\Software\Tools\BEAST`
2. 能找到以下程序：
   - `D:\Software\Tools\BEAST\bat\beast.bat`
   - `D:\Software\Tools\BEAST\bat\logcombiner.bat`
   - `D:\Software\Tools\BEAST\jre\bin\java`
3. 准备好输入文件：
   - `InputModel.xml`：BEAST 模型配置
   - `InputTrees.trees`：MCMC 产生的树日志
   - `CombinedTrees.trees`：用于注释汇总的树（可用步骤二输出）
4. 如需 GPU 加速，准备好 BEAGLE（下面有详细安装步骤）
5. 如果你的模型使用 SA（Sampled Ancestors）等扩展模型，先在 Package Manager 安装对应包

## 3. 占位符怎么替换

文档里会出现占位符，请按你的实际情况替换：

- `BEAST_HOME`：BEAST 根目录，例如 `D:\Software\Tools\BEAST`
- `InputModel.xml`：例如 `24GROUP3.xml`
- `InputTrees.trees`：例如 `24GROUP3-year.trees`
- `ResampledTrees.trees`：例如 `24GROUP3-year_resample10000_b20.trees`
- `CombinedTrees.trees`：例如 `V3yueguizuCP127-yueguizuCP131mafftCOM.trees`
- `AnnotatedTree.tree`：例如 `V3yueguizuCP127-yueguizuCP131mafftCOMmcmc.tree`

## 4. BEAGLE 安装与验证

> 如果你只想先跑通流程，可以跳过 BEAGLE，直接用 CPU 命令。
- 建议使用，通过GPU加速可大幅缩短运算时间
### 4.1 先判断你是否需要 BEAGLE

- 数据量较大、链很长、机器有 NVIDIA GPU：建议配置 BEAGLE + GPU
- 只是先测试流程是否可跑：可先用 CPU，不影响正确性

### 4.2 首次配置时安装

1. 安装/更新显卡驱动（NVIDIA 官方驱动）
2. 安装 CUDA Runtime（版本与 BEAGLE 二进制兼容）
（上面两部一般不用额外操作，计算机默认已经具备，仅在出错时考虑）
3. 安装 BEAGLE 库（Windows 对应版本，Github：https://github.com/beagle-dev/beagle-lib/releases）
4. 重启电脑（建议，非必需）

### 4.3 若BEAST识别不到BEAGLE

1. 安装程序自动把 BEAGLE 动态库放入系统可搜索路径（`PATH`）
2. 手动把 BEAGLE 动态库所在目录加入系统 `PATH`
3. 将相关动态库放在 BEAST 可加载到的位置（不推荐新手手工拷贝，容易混版本）

### 4.4 验证 BEAGLE 是否被识别
- 法1：打开BEAST.EXE，查看“RUN”按钮左边“BEAGLE INFO”按钮是否存在，点击后是否能检测到GPU型号等信息。

- 法2：在 PowerShell 执行：

```powershell
Set-Location "D:\Software\Tools\BEAST"
.\bat\beast.bat -beagle_info
```

如果成功，通常会看到可用资源（CPU/GPU）信息。

### 4.5 常见 BEAGLE 报错与处理

1. 报 `Cannot find BEAGLE`：
   - 说明 BEAST 没找到 BEAGLE 动态库
   - 检查 BEAGLE 是否安装成功、`PATH` 是否生效、重开终端
2. 报 `No compatible GPU`：
   - 可能是驱动/CUDA/BEAGLE 版本不匹配
   - 先改 CPU 方式跑通，再回头调 GPU
3. GPU 模式卡住或报数值错误：
   - 尝试保留 `-beagle_scaling dynamic`
   - 尝试减少 `-threads` 或改用 CPU 测试模型

## 5. SA（Sampled Ancestors）包安装说明

如果 XML 用到了 SA 相关模型，但包未安装，会在启动时报包/类找不到。

建议操作：
1. 打开 `BEAUti` 或 `Package Manager`
2. 搜索并安装 SA 相关包
3. 重新打开 BEAST
4. 先用 `-validate` 检查 XML：

```powershell
& "D:\Software\Tools\BEAST\bat\beast.bat" -validate "D:\Software\Tools\BEAST\InputModel.xml"
```

#这里注意，目前2.7.8版本的BEAST与最新的SA 版本2.1.1不兼容，所以不要更新BEAST版本，使用2.7.7即可。

## 6. PowerShell 里怎么执行

1. 打开 PowerShell
2. 切换目录到 BEAST 根目录：

```powershell
Set-Location "D:\Software\Tools\BEAST"
```

3. 看当前目录是否正确：

```powershell
Get-Location
```

4. 执行命令时注意：
   - 含空格路径必须加双引号
   - 执行绝对路径程序通常写 `& "完整路径" 参数...`
   - 执行当前目录 bat 可写 `.\bat\xxx.bat`

## 7. 步骤一：运行 BEAST（MCMC 采样）

意义：输入 `InputModel.xml`，输出链结果（通常包含 `.trees` 、`.status` 和 `.log`）。

### 7.1 GPU 版（BEAGLE），推荐

```powershell
& "BEAST_HOME\bat\beast.bat" -beagle -beagle_GPU -beagle_order 1 -threads 12 -instances 1 -beagle_scaling dynamic "BEAST_HOME\InputModel.xml"
```

### 7.2 CPU 版（不使用 GPU）

```powershell
& "BEAST_HOME\bat\beast.bat" -threads 12 -instances 1 -beagle_scaling dynamic "BEAST_HOME\InputModel.xml"
```

### 7.3 参数解释（常用）

- `-threads 12`：使用 12 个线程（使用当前电脑的CPU的最大线程数。）
- `-instances 1`：BEAGLE 实例数（也可多链并行，根据需求选择，默认单条链选择1）
- `-beagle_scaling dynamic`：动态缩放，减少下溢/上溢风险
- `-beagle`：启用 BEAGLE
- `-beagle_GPU`：优先使用 GPU
- `-beagle_order 1`：资源顺序（多设备时可调整）

### 7.4 如何判断步骤一成功

- 终端无致命报错并持续输出采样进度
- 目录中出现或更新了对应输出文件（如 `.log`、`.trees`、`.state`）

### 7.5 常见错误快速排查

1. XML 报错：先执行 `-validate`
2. GPU 报错：先改 CPU 命令确认模型正常
3. 闪退、内存不够：减少线程，或给 Java 更大内存（在启动脚本层面调整）

## 8. 步骤二：LogCombiner（tree file + burnin + resample）

你的目标设置是：
- 选择 `tree file`
- `resample states at lower frequency = 10000`
- `burnin = 20`

命令：

```powershell
.\bat\logcombiner.bat -log .\InputTrees.trees -o .\ResampledTrees.trees -b 20 -resample 10000
```

关键说明：
- 命令行没有单独 `tree file` 开关
- 当 `-log` 输入是 `.trees` 文件时，就等价于 GUI 选择 `tree file`

参数解释：
- `-log`：输入日志文件，不要改（ `.log` 或 `.trees`均可传）
- `-o`：输出文件名
- `-b 20`：去掉前 20%（burn-in）
- `-resample 10000`：每 10000 个状态保留一个样本

## 9. 步骤三：TreeAnnotator 生成注释树

意义：把后验树样本压缩成一棵代表树，便于可视化和报告。

命令：

```powershell
& "BEAST_HOME\jre\bin\java" -Xms4g -Xmx16g -Xss4m -cp "BEAST_HOME\lib\launcher.jar" beast.pkgmgmt.launcher.TreeAnnotatorLauncher -burnin 0 -height median "BEAST_HOME\CombinedTrees.trees" "BEAST_HOME\AnnotatedTree.tree"
```

参数解释：
- `-Xms4g`：初始内存 4G
- `-Xmx16g`：最大内存 16G（根据电脑最大内存选择，若电脑最大为32G，选择16G，若最大为16G，选择8G；内存选的太小，生成树时内存不足会失败）
- `-Xss4m`：线程栈大小
- `-burnin 0`：这里不再去 burn-in（因为步骤二已处理）
- `-height median`：节点高度取后验中位数

## 10. 一份可直接套用的完整命令模板

```powershell
# 0) 进入 BEAST 目录
Set-Location "BEAST_HOME"

# 1) 可选：验证 XML 是否能解析
& "BEAST_HOME\bat\beast.bat" -validate "BEAST_HOME\InputModel.xml"

# 2) 运行 MCMC
#CPU 版：
& "BEAST_HOME\bat\beast.bat" -threads 12 -instances 1 -beagle_scaling dynamic "BEAST_HOME\InputModel.xml"
#GPU 加速版：
& "BEAST_HOME\bat\beast.bat" -beagle -beagle_GPU -beagle_order 1 -threads 12 -instances 1 -beagle_scaling dynamic "BEAST_HOME\InputModel.xml"

# 3) 处理树文件（tree file + burnin 20 + resample 10000）
.\bat\logcombiner.bat -log .\InputTrees.trees -o .\ResampledTrees.trees -b 20 -resample 10000

# 4) 生成注释树
& "BEAST_HOME\jre\bin\java" -Xms4g -Xmx16g -Xss4m -cp "BEAST_HOME\lib\launcher.jar" beast.pkgmgmt.launcher.TreeAnnotatorLauncher -burnin 0 -height median "BEAST_HOME\CombinedTrees.trees" "BEAST_HOME\AnnotatedTree.tree"
```

## 11. 建议的执行顺序（第一次跑建议）

1. 先跑 `-validate`，确保 XML 没问题
2. 先 CPU 跑通一次
3. 再尝试启用 GPU（BEAGLE）提速，若GPU不支持或实在配置不成功再退回CPU运行
4. 跑完后再做 LogCombiner 和 TreeAnnotator
