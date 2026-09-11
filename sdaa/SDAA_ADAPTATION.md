# p2rank SDAA 平台适配说明

## 1. 结论

p2rank 是预测蛋白配体结合位点的 **CPU Java** 工具（Groovy + WEKA/FasterForest 随机森林），
不依赖 CUDA / GPU，不存在算子级、设备级适配内容。

本适配内容是「**在 LoongArch64 平台上可构建、可运行**」：

- **代码零改动**：`adapt/sdaa` 与上游 `ce05641e` 无 diff，`patches/` 为空
- 平台差异全在环境层面（JDK、构建编码、gradle 发行版获取）
- 预测结果与官方 release 2.5.1 在同一台机器上**逐字段一致**

## 2. 环境

| 项 | 值 |
|---|---|
| 平台 | LoongArch64，宿主 node499（10.71.15.115），容器 `sdaa`（Loongnix 23.1） |
| 代码版本 | 2.6-alpha.7，`ce05641e` |
| JDK | `java-17-openjdk-17.0.5.0.8-4.lns23.loongarch64`（`yum install -y java-17-openjdk java-17-openjdk-devel`） |
| 构建 | Gradle 9.7.1（wrapper） |
| 机器 | 1012 GB 内存 / 128 核 |

## 3. 安装与平台相关处理（不涉及代码）

### 3.1 JDK 17 安装

`build.gradle:32` 要求 Java 17，容器原生只有 OpenJDK 11（`java-11-openjdk-11.0.17.0.8-4.lns23.loongarch64`）。
JDK 17 由 Loongnix 官方源提供，容器内以 root 安装：

```bash
yum install -y java-17-openjdk java-17-openjdk-devel
```

安装结果（`yum` 装的是 `java-17-openjdk-17.0.5.0.8-4.lns23.loongarch64`）：

```bash
ls -d /usr/lib/jvm/java-17-openjdk-*
# /usr/lib/jvm/java-17-openjdk /usr/lib/jvm/java-17-openjdk-17.0.5.0.8-4.lns23.loongarch64
/usr/lib/jvm/java-17-openjdk-17.0.5.0.8-4.lns23.loongarch64/bin/javac -version
# javac 17.0.5
```

`java -version` 仍是 11（alternatives 未被切换，不影响容器内其他工具），构建与运行统一用
`JAVA_HOME` 显式指定 17。

### 3.2 Gradle 发行版获取

wrapper 需下载 `gradle-9.7.1-bin.zip`，默认地址 `services.gradle.org` 在本网络不通
（HTTP 307 后重定向目标超时，`*.zip.part` 长期为 0 字节）。两种方式任选：

**方式一：替换下载地址**（本次采用；构建完成后已还原该文件，以保证仓库零 diff）

```bash
sed -i 's|services.gradle.org/distributions/|mirrors.cloud.tencent.com/gradle/|' \
  gradle/wrapper/gradle-wrapper.properties
```

**方式二：预置发行版包**（离线环境）

```bash
# 先执行一次 ./gradlew，让它创建缓存目录（下载会卡住，Ctrl-C 退出）
D=$(ls -d $GRADLE_USER_HOME/wrapper/dists/gradle-9.7.1-bin/*/ | head -1)
curl -L -o $D/gradle-9.7.1-bin.zip https://mirrors.cloud.tencent.com/gradle/gradle-9.7.1-bin.zip
unzip -t $D/gradle-9.7.1-bin.zip >/dev/null && touch $D/gradle-9.7.1-bin.zip.ok
```

wrapper 检测到 zip 与 `.ok` 标记后即跳过下载（目录名由 wrapper 按 `distributionUrl` 计算，
换 URL 会变，所以用上面 `ls` 取实际目录）。

### 3.3 构建编码

容器 `LANG` 为空，javac 默认 US-ASCII，源码注释里的 `—` 会报
`unmappable character (0xE2) for encoding US-ASCII` 并中止。构建时给 `LANG=C.UTF-8` 即可，
源码无需改动。

### 3.4 p2rank 发行版直接安装（免构建）

不构建源码时可直接用官方二进制发行版（同样要求 JDK 17~23，见发行版 README）：

```bash
curl -LO https://github.com/rdk/p2rank/releases/download/2.5.1/p2rank_2.5.1.tar.gz
tar -xzf p2rank_2.5.1.tar.gz
cd p2rank_2.5.1
./prank predict -f test_data/1fbl.pdb -o /data/application/xuqiang/p2rank_release_artifacts
```

实测（`p2rank_2.5.1.tar.gz` 262 MB，JDK 17.0.5）：`Finished successfully in 6.814 seconds`
（内部计算 5.095 s），4 个口袋，`predictions.csv` 与源码构建版（2.6-alpha.7）逐字段一致。

下载页 `https://github.com/rdk/p2rank/releases`，当前最新为 2.6-alpha（`p2rank_2.6-alpha.tar.gz`）。

## 4. 构建与运行

```bash
# 进容器（-u/-e HOME 是为了以 xuqiang 身份写盘，产物属主正确）
docker exec -it -u 1007:1007 -e HOME=/data/application/xuqiang \
  -e LANG=C.UTF-8 -e LC_ALL=C.UTF-8 sdaa bash
```

```bash
# 容器内：指定 JDK 17
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-17.0.5.0.8-4.lns23.loongarch64
export PATH=$JAVA_HOME/bin:$PATH
```

```bash
# 构建
cd /data/application/xuqiang/p2rank && ./make.sh
```

```bash
# 预测（进安装目录，用法即 README 的 `prank predict ...`）
cd /data/application/xuqiang/p2rank/distro
./prank predict -f test_data/1fbl.pdb -o /data/application/xuqiang/p2rank_test_artifacts -threads 8
```

`-threads` 需显式指定：默认取 `核数 + 1`（本机 128 核 → 129），单结构下反而更慢（见 5.4）。
批量与其他子命令同样在命令行末尾追加该参数。

用 `distro/prank`（生产启动器）而非仓库根的 `prank.sh`（开发启动器，需往仓库根拷 `misc/local-env.sh`）。
**不要把它软链到 `/usr/local/bin`**：脚本用 `dirname "${BASH_SOURCE[0]}"` 定位自身目录，软链会让
`INSTALL_DIR` 指向软链所在位置，实测报 `Could not find or load main class cz.siret.prank.program.Main`。

## 5. 结果

### 5.1 构建产物

```
build/bin/p2rank.jar    3,053,844 B
build/classes           1139 个 .class
distro/bin/p2rank.jar   同步产出
```

### 5.2 预测结果

预测 1fbl.pdb（`-threads 8`）：

```
[INFO] Model - Loading model from directory (v3 format): distro/models/default
[INFO] FeatureSetup - effectively enabled features: [chem, volsite, protrusion, bfactor, atom_table]
[INFO] Dataset - processing dataset [1fbl.pdb] using 8 threads
[INFO] SingleLinkageClustering - LIGANDABLE POINTS: 47 / CLUSTERS: 7 / FILTERED CLUSTERS: 4
[INFO] PocketPredictor - pocket 1 - surf_atoms: 40   points:  20   score:    9.8
[INFO] PocketPredictor - pocket 2 - surf_atoms: 18   points:  10   score:    3.0
[INFO] PocketPredictor - pocket 3 - surf_atoms: 15   points:   7   score:    2.9
[INFO] PocketPredictor - pocket 4 - surf_atoms: 17   points:   5   score:    1.9
[INFO] Writable - predicting pockets finished in 0 hours 0 minutes 6.451 seconds
[INFO] Console - Finished successfully in 0 hours 0 minutes 8.341 seconds.
```

输出目录：`1fbl.pdb_predictions.csv` / `1fbl.pdb_residues.csv` / `params.txt` / `run.log` /
`visualizations/`。`predictions.csv` 与默认线程数下（129）的产物 `diff` 结果一致。

与官方 release 2.5.1（同一台 loongarch64 机器）对照：

| 项 | 2.6-alpha.7（源码构建） | 2.5.1（release） |
|---|---|---|
| pocket 数 | 4 | 4 |
| score | 9.77 / 3.04 / 2.94 / 1.87 | 9.77 / 3.04 / 2.94 / 1.87 |
| pocket1 中心 | 70.5274, 83.4375, -11.5099 | 70.5274, 83.4375, -11.5099 |
| residues.csv 行数 | 368 | 368 |

`predictions.csv` 的 rank / score / probability / sas_points / surf_atoms / center / residue_ids /
surf_atom_ids 全部字段逐行一致。

native 库：`FasterForest-2.13.0.jar` 不带 loongarch64 原生库（只有 x86_64 / windows），
运行日志中无 `UnsatisfiedLinkError` 或 native 警告，预测正常完成。

### 5.3 其他子命令

各命令均在 `distro` 目录下执行，末尾带 `-threads 8`。

| 命令 | 输入 | 结果 |
|---|---|---|
| `predict <dataset.ds>`（批量） | `test_data/basic.ds`（2W83 + 1fbl） | 两结构的 predictions / residues 均产出，总 8.994 s（内部 7.131 s） |
| `rescore <dataset.ds> -c rescore_2024` | `test_data/fpocket.ds`（4 个蛋白） | 输出 `*_rescored.csv`（含 old_rank / change 重排序），总 13.330 s（内部 10.951 s） |
| `traineval -t <ds> -e <ds> -loop 1`（训练路径） | `test_data/test.ds`（5 个蛋白） | 训练 + 评估完成，输出 DCC / DSO / DPA 等指标，avg training time 1.097 s，总 12.109 s |

dataset 走位置参数传入，不要用 `-f`：`-f test_data/test.ds` 会把 .ds 当单个结构文件加载，
日志报 `Structure with no protein chain atoms!` 并产出空结果。

训练命令用 `./prank.sh`（需临时拷 `misc/local-env.sh`，跑完删除），因为它设 `-Xmx31G`，
而 `distro/prank` 固定 `-Xmx2048m`。traineval 输出落在 `distro/test_output/`（已被 `.gitignore` 忽略）。

### 5.4 性能

测试结构 `distro/test_data/1fbl.pdb`：**367 残基**（2 链）/ 蛋白原子 2902 / 结构原子 3294 /
SAS 表面点 5168 / 暴露原子 1825，运行参数 `-threads 8`（日志 `processing dataset [1fbl.pdb]
using 8 threads`）。

吞吐按内部计算耗时 **6.451 s** 计（日志 `Finished successfully in 8.341 seconds`，其中含
JVM 启动、类加载、模型反序列化）：

| 口径 | 数值 |
|---|---|
| **残基吞吐** | **56.9 residues/s** |
| **原子吞吐** | **450 atoms/s** |
| 表面点吞吐 | 801 surface points/s |
| 结构吞吐 | 0.155 structures/s；558 structures/h |

同一台机器上官方 release 2.5.1 的对照（同为 1fbl，`predictions.csv` 与源码构建版逐行一致）：
内部计算 **5.095 s**，与源码构建版 `-threads 8` 的 6.451 s 同一量级。

线程数的影响：与代码、模型、特征无关（两者的模型 `model.zst` 完全一致，特征集同为
`[chem, volsite, protrusion, bfactor, atom_table]`，表面点同为 5168）。单结构任务并行收益
有限，线程数远超工作量时调度开销占主导。实测（同一结构，内部计算耗时）：

| 配置 | 线程数 | 内部计算 |
|---|---|---|
| 2.6-alpha.7 默认 | 129（核数 + 1） | 10.904 s |
| 2.6-alpha.7 `-threads 1` | 1 | 5.899 s |
| 2.6-alpha.7 `-threads 8` | 8 | 5.303 s |
| 2.6-alpha.7 `-threads 16` | 16 | 5.334 s |
| 2.5.1 release（默认） | 129 | 4.601 ~ 5.474 s |

批量（多结构）：`predict test_data/test.ds`（5 个蛋白），四种线程配置串行执行：

| 配置 | 线程数 | 内部计算 | 总耗时 |
|---|---|---|---|
| 2.6-alpha.7 `-threads 1` | 1 | 8.985 s | 10.841 s |
| 2.6-alpha.7 `-threads 8` | 8 | 5.850 s | 7.821 s |
| 2.6-alpha.7 `-threads 16` | 16 | 5.315 s | 7.475 s |
| 2.6-alpha.7 默认 | 129 | 5.821 s | 7.930 s |
| 2.5.1 release `-threads 16` | 16 | 4.539 s | 6.177 s |
| 2.5.1 release（默认） | 129 | 4.612 s | 6.542 s |

## 6. 已知限制

1. 首次构建需联网获取 Gradle 发行版与 Maven 依赖（离线步骤见 3.2）。
2. 未执行 `crossval`（交叉验证）与 `ploop` / `hopt` 等超参实验命令。
3. 未在 PDB 量级的大数据集上验证内存与耗时（本次最大为 5 蛋白的 `test.ds`）。
