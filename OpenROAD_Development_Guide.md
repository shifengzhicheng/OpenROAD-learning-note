# OpenROAD 项目集成指南

## 一、OpenROAD项目的部署

### 1. 引言

OpenROAD（Foundations and Realization of Open, Accessible Design）项目于2018年启动，旨在解决硬件设计中的高成本、专业知识门槛以及不可预测性问题。由Qualcomm、Arm、多所大学和合作伙伴共同开发，项目由加州大学圣地亚哥分校（UC San Diego）牵头，专注于数字系统芯片（SoC）设计中的RTL到GDSII阶段的全自动开源工具链开发。

该项目的主要目标是提供一个全自动的、无需人工干预的设计流程，支持从RTL（Register Transfer Level）到GDSII的快速设计。OpenROAD的核心特色在于消除设计中的成本、进度风险和不确定性，通过机器学习和智能算法，自动调优设计流程中的各个步骤。

OpenROAD流（flow）包括逻辑综合、底层布局、布局、时钟树综合、布线、寄生提取和时序分析等阶段。它能够使用Tcl脚本命令和Python绑定的API，灵活地控制设计流程，并已成功应用于多个学术和商业场景。

### 2. 报告目标

#### 介绍OpenROAD项目的安装与编译

本报告的第一部分将详细说明如何获取OpenROAD项目源代码、安装必要依赖，以及如何使用CMake工具进行构建与编译。

#### 数字后端的交换文件格式

报告的第二部分将深入探讨数字后端设计中使用的交换文件格式，重点介绍LEF（Library Exchange Format）和DEF（Design Exchange Format）的结构及其在OpenROAD中的作用。

#### 开源数据库OpenDB的常用接口与内部数据结构

第三部分将介绍OpenROAD项目中的核心数据库模块——OpenDB。重点介绍OpenDB提供的常用接口，例如读取和写入LEF/DEF文件的API、如何使用这些API在数据库中构建和操作设计对象（如网络、模块、单元等）。此外，还会介绍OpenDB的内部数据结构，包括dbBlock、dbNet、dbInst等，以及如何使用这些数据结构来表示和管理芯片设计的各个方面。

#### 如何在OpenROAD中集成自定义代码

报告的第四部分将讨论如何在OpenROAD框架中集成自己的EDA工具或算法。具体内容将涵盖如何在OpenROAD项目中引入自定义功能模块，如何扩展OpenDB数据结构，如何通过Tcl或Python API加载自定义功能等。

#### 自定义代码的调用与测试

报告的最后一部分将介绍如何调用和测试集成的自定义代码。内容将包括如何通过OpenROAD的命令行或图形界面运行自定义流程，如何编写和执行回归测试确保代码正确性，以及如何在设计中使用自定义功能并验证其效果。

### 2. 环境准备与安装

#### 安装依赖

``` shell
git clone --recursive https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts
cd OpenROAD-flow-scripts
sudo ./setup.sh
```

#### 本地构建OpenROAD

``` shell
./build_openroad.sh --local
```
:::{Note}
每次构建时，主目录中都会生成一个 `build_openroad.log` 文件。在提交问题时，可以将该文件上传到 OpenROAD-flow-scripts 仓库的[问题表单](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts/issues/new?assignees=&labels=&template=bug_report_with_orfs.yml)的“相关日志输出”部分。
:::

## 验证安装

在设置环境后，二进制文件应该可以通过 `$PATH` 访问。`make` 命令将从 RTL 到 GDSII 生成默认设计 `gcd`，使用 `nangate45` PDK。

``` shell
source ./env.sh
yosys -help
openroad -help
cd flow
make
```

你可以使用以下命令在 OpenROAD GUI 中查看最终布局图像。

``` shell
make gui_final
```

## 数字后端的交换文件格式

在集成电路的物理设计过程中，LEF、DEF 和 LIB 文件各自扮演了重要的角色。它们用于描述芯片的不同方面，包括物理布局、设计连接和电气特性。以下是每个文件格式的介绍及其基本示例。

### 1. LEF (Library Exchange Format) 文件

LEF 文件用于描述标准单元库（cell library）的物理布局抽象信息，定义了电路设计中用于布局和布线的物理参数。

**LEF 文件的关键内容：**
- **图层定义**：设计中不同的金属层和通孔层的属性，如宽度、间距和设计规则。
- **标准单元信息**：单元的物理尺寸（宽度和高度），针脚的位置、方向（输入/输出）和电气属性。
- **阻碍区域**：不能用于布线的区域。
- **制造网格**：定义设计的分辨率和网格大小。
- **天线规则**：防止制造过程中出现天线效应的规则。

**示例：**
```spice
MACRO AND2X1
    CLASS CORE ; 
    ORIGIN (0.0, 0.0) ; 
    SIZE 2.0 BY 1.0 ; 

    PIN A
        DIRECTION INPUT ; 
        USE SIGNAL ; 
        PORT
            LAYER Metal1 ;
                RECT (0.0 0.0) (0.2 0.2) ;
        END
    END A
    
    PIN Y
        DIRECTION OUTPUT ; 
        USE SIGNAL ; 
        PORT
            LAYER Metal1 ;
                RECT (1.8 0.8) (2.0 1.0) ;
        END
    END Y
END AND2X1
```
这个例子定义了一个 AND2X1 标准单元的物理布局，宽度为 2.0，高度为 1.0，针脚 A 是输入，针脚 Y 是输出。

### 2. DEF (Design Exchange Format) 文件

DEF 文件用于描述整个芯片设计的物理布局和连接信息，定义芯片中各组件的位置、布线以及电源和地网络等特殊网络的布局。

**DEF 文件的关键内容：**
- **组件放置**：描述芯片设计中标准单元和宏单元的位置。
- **布线信息**：详细描述电路中各组件之间的布线。
- **净表（netlist）**：组件之间的连接关系。
- **特殊网络**：如电源和地网络的布线。
- **设计约束**：描述设计的布局和布线约束。

**示例：**
```spice
COMPONENTS 2 ;
    - U1 AND2X1 + PLACED (10, 20) N ;
    - U2 INVX1 + PLACED (50, 50) N ;
END COMPONENTS

NETS 1 ;
    - net1 ( U1 Y ) ( U2 A ) ;
END NETS

SPECIALNETS 1 ;
    - VDD + ( U1 VDD ) ( U2 VDD ) ;
END SPECIALNETS
```
这个 DEF 文件描述了一个包含两个组件（U1 和 U2）的设计，它们分别放置在指定的芯片位置，net1 表示它们之间的连接，VDD 表示电源网络。

### 3. LIB (Library) 文件

LIB 文件包含有关标准单元的电气属性、时序和功耗信息，用于时序分析和功耗估算。该文件定义了每个单元的输入输出时序、逻辑功能、功耗、以及工艺条件等。

**LIB 文件的关键内容：**
- **时序信息**：定义了输入输出路径的延迟、建立时间（setup time）和保持时间（hold time）。
- **功耗信息**：定义了单元在不同状态下的功耗，包括漏电功耗（leakage power）和动态功耗。
- **逻辑功能**：定义了每个标准单元执行的逻辑操作。
- **工艺条件**：包括工作电压、温度和工艺角（process corner）。
- **功耗弧（power arc）**：描述不同输入状态下的功耗。

**示例：**
```spice
cell (AND2X1) {
    area : 2.0 ;
    pin(A) {
        direction : input ;
        capacitance : 0.01 ;
    }
    pin(B) {
        direction : input ;
        capacitance : 0.01 ;
    }
    pin(Y) {
        direction : output ;
        function : "A & B" ;
        timing() {
            related_pin : "A" ;
            rise_transition : 0.1 ;
            fall_transition : 0.1 ;
            cell_rise : 0.2 ;
            cell_fall : 0.2 ;
        }
    }
}
```
这个例子定义了一个 AND2X1 单元的电气属性，包括输入输出针脚的方向、时序信息和逻辑功能。

## OpenDB 的常用接口与内部数据结构

OpenDB 是 OpenROAD 项目中的核心数据库模块，用于管理数字集成电路设计中的物理和逻辑信息。OpenDB 的核心功能是通过一系列 API 提供对设计数据的读写、操作和查询，并以高效的方式表示芯片设计的各个部分。下面介绍 OpenDB 的常用接口和内部数据结构。

### 1. 常用接口

OpenDB 提供了一组 API 用于读取、写入、和操作设计数据，特别是通过 LEF 和 DEF 文件对设计进行物理布局的描述。以下是一些常用的接口函数：

#### 读取 LEF 文件：

```cpp
dbDatabase* db = dbDatabase::create();
db->readLef("tech.lef");
```

#### 读取 DEF 文件：

```cpp
db->readDef("design.def");
```

#### 写入数据库文件：

```cpp
db->writeDb("output.db");
```

这些接口允许用户将设计的 LEF 和 DEF 文件加载到 OpenDB 数据库中，并使用数据库操作对象进行处理。此外，还可以将处理后的设计导出为数据库文件，以便后续使用。

### 2. 内部数据结构

OpenDB 的内部数据结构用于表示芯片设计中的组件和连接关系。以下是最常用的几个数据结构：

- **dbBlock**：表示设计中的基本单元块，一个设计通常由一个或多个 dbBlock 组成。dbBlock 记录了与芯片布局相关的所有信息，包括实例、网络、引脚等。

    ```cpp
    dbBlock* block = db->getChip()->getBlock();
    ```

- **dbInst**：表示设计中的实例对象，dbInst 是单元块的具体实例化。例如，一个 AND2X1 单元的实例可以被添加到 dbBlock 中：

    ```cpp
    dbInst* inst = dbInst::create(block, master, "instance_name");
    block->addInst(inst);
    ```

- **dbNet**：表示设计中的网络，dbNet 连接了芯片中的不同组件，用于描述组件之间的信号传递。例如，连接两个实例的网络可以通过以下方式创建：

    ```cpp
    dbNet* net = dbNet::create(block, "net_name");
    ```

- **dbTech 和 dbLib**：用于定义设计中的技术库信息。dbTech 定义了制造工艺的具体信息，例如金属层和设计规则，而 dbLib 则包含了标准单元库的定义。

### 3. 操作流程示例

下面的示例展示了如何使用 OpenDB API 创建一个简单的设计，并通过 LEF 和 DEF 文件进行读写操作：

```cpp
#include "db.h"

int main() {
    // 创建数据库
    dbDatabase* db = dbDatabase::create();

    // 读取 LEF 文件
    db->readLef("tech.lef");

    // 读取 DEF 文件
    db->readDef("design.def");

    // 获取芯片顶层块
    dbBlock* block = db->getChip()->getBlock();

    // 创建一个实例
    dbInst* inst = dbInst::create(block, master, "instance_name");
    block->addInst(inst);

    // 创建网络并连接实例
    dbNet* net = dbNet::create(block, "net_name");
    dbITerm* iterm = inst->getITerm("A");
    net->connect(iterm);

    // 保存设计为数据库文件
    db->writeDb("output.db");

    return 0;
}
```

### 4. 内部操作细节

OpenDB 的数据结构以高效的方式管理芯片布局和连接信息，确保处理大规模设计时的性能。每个设计对象（如 dbBlock, dbInst, dbNet）都有唯一的对象标识符（OID），用于在内存中有效地管理和查找这些对象。OpenDB 使用 C++ 编写，并采用标准库样式的迭代器来操作数据库对象。

## 如何在 OpenROAD 中集成自定义代码

在 OpenROAD 中集成自定义代码，允许用户根据项目需求扩展或修改现有的设计流程。这一部分将展示如何在 OpenROAD 框架中引入自定义的 EDA 工具或算法，包括如何扩展 OpenDB 的数据结构、编写新的功能模块、以及如何通过 Tcl 和 Python API 加载和调用这些自定义功能。

### 1. 扩展 OpenDB 数据结构

OpenDB 是 OpenROAD 的核心数据库模块，管理设计的所有物理和逻辑数据。要在 OpenROAD 中集成自定义代码，首先需要确定是否需要扩展 OpenDB 的数据结构。

**扩展步骤：**

1. **确定要扩展的数据结构**：例如，你可能需要为自定义算法添加新的设计属性，如新的设计规则、连接信息等。

2. **修改相关的数据库类**：

    对于每个数据结构（如 `dbBlock`, `dbInst`, `dbNet`），OpenDB 提供了丰富的接口。扩展这些类可以为自定义数据提供存储空间。在 `src/db` 目录中找到对应的数据结构文件（例如 `dbBlock.h`），添加新的数据成员或方法。

    ```cpp
    class dbBlock {
      ...
      int myCustomData; // 自定义数据
      void setMyCustomData(int data) { myCustomData = data; }
      int getMyCustomData() { return myCustomData; }
      ...
    };
    ```

3. **更新数据库序列化逻辑**：如果需要将自定义数据持久化，可以更新 OpenDB 的序列化机制，将新的数据写入数据库文件中。

    ```cpp
    stream << block->getMyCustomData();
    ```

4. **重新编译 OpenROAD**：修改完成后，使用 CMake 重新编译 OpenROAD，以确保自定义代码正确集成。

    ```bash
    cmake ..
    make
    ```

### 2. 集成自定义功能模块

在集成自定义 EDA 工具或算法时，通常需要在 OpenROAD 中增加新模块。这些模块可以直接在现有设计流程中使用。

**实现步骤：**

1. **编写自定义模块**：首先，在 `src/` 目录下创建新的模块文件。例如，编写一个自定义布局算法 `custom_placer.cpp`。

    ```cpp
    #include "db.h"
    void customPlacer(dbBlock* block) {
      // 实现自定义的布局算法
    }
    ```

2. **在 OpenROAD 中注册新模块**：为了让 OpenROAD 能够调用该模块，需要在 OpenROAD 中注册新的 Tcl 或 Python 命令。

    在 `src/tcl` 目录中，定义新的 Tcl 命令，以便用户通过 Tcl 调用自定义功能。

    ```cpp
    int customPlacerCommand(ClientData, Tcl_Interp* interp, int argc, const char* argv[]) {
      dbBlock* block = getTopBlock(); // 获取当前设计的顶层块
      customPlacer(block); // 调用自定义布局算法
      return TCL_OK;
    }
    ```

    在 `tclAppInit.cpp` 中注册该命令：

    ```cpp
    Tcl_CreateCommand(interp, "custom_placer", customPlacerCommand, (ClientData) NULL, (Tcl_CmdDeleteProc *) NULL);
    ```

3. **编译并测试**：重新编译 OpenROAD，并通过 Tcl 调用自定义的功能模块。

    ```bash
    cmake ..
    make
    ```

4. **在脚本中调用自定义命令**：通过 Tcl 脚本调用自定义的布局算法：

    ```tcl
    custom_placer
    ```

### 3. 通过 Tcl 和 Python API 加载自定义功能

除了直接修改 C++ 源代码，OpenROAD 还支持通过 Tcl 和 Python 接口调用自定义代码。用户可以通过这些脚本语言动态加载功能，从而扩展现有流程。

**使用 Tcl API：**

可以通过定义新的 Tcl 命令将自定义功能注入到 OpenROAD 的设计流程中。例如，定义一个新的命令 `custom_tcl_command`，并将其绑定到自定义的 C++ 函数。

```cpp
int customTclCommand(ClientData, Tcl_Interp* interp, int argc, const char* argv[]) {
  // 实现Tcl命令功能
  return TCL_OK;
}
```

**使用 Python API：**

OpenROAD 还提供了对 Python 的支持，允许用户通过 Python 脚本加载和调用自定义功能。自定义 Python 命令可以直接绑定到 OpenDB 中的对象和数据结构上。

例如，可以通过 `openroad -python` 运行 Python 脚本，直接操作 OpenDB 数据：

```python
import openroad
block = openroad.getTopBlock()
# 调用自定义功能
```

## 调用与测试

在自定义功能模块集成后，进行调用与测试是确保新功能正常工作的关键步骤。OpenROAD 提供了回归测试框架和 GUI 工具，帮助用户调试和验证设计。

**测试步骤：**

1. **编写测试脚本**：为自定义功能编写 Tcl 测试脚本，确保模块能在不同设计条件下正常工作。

    ```tcl
    source custom_script.tcl
    custom_placer
    ```

2. **回归测试**：使用 OpenROAD 内置的回归测试工具，确保集成后的代码没有破坏现有功能。

    ```bash
    ./test/regression
    ```

3. **使用 GUI 调试**：通过 OpenROAD 提供的图形用户界面（GUI），可视化设计流程，观察自定义功能的效果。

    ```bash
    openroad -gui
    ```

## 总结

通过扩展 OpenDB 数据结构、集成自定义功能模块以及使用 Tcl 和 Python API，用户可以灵活地在 OpenROAD 中集成自己的 EDA 工具或算法。通过编写测试脚本和使用 GUI 工具，可以验证和调试自定义代码，确保其在设计流程中的有效性和稳定性。

## 参考文献

[1] T. Ajayi et al., "INVITED: Toward an Open-Source Digital Flow: First Learnings from the OpenROAD Project," 2019 56th ACM/IEEE Design Automation Conference (DAC), Las Vegas, NV, USA, 2019, pp. 1-4.

[2] J. Chen et al., "DATC RDF-2019: Towards a Complete Academic Reference Design Flow," 2019 IEEE/ACM International Conference on Computer-Aided Design (ICCAD), Westminster, CO, USA, 2019, pp. 1-6, doi: 10.1109/ICCAD45719.2019.8942120.

[3] A. B. Kahng, "Looking Into the Mirror of Open Source: Invited Paper," 2019 IEEE/ACM International Conference on Computer-Aided Design (ICCAD), Westminster, CO, USA, 2019, pp. 1-8, doi: 10.1109/ICCAD45719.2019.8942131.

[4] A. Rovinski, T. Ajayi, M. Kim, G. Wang and M. Saligane, "Bridging Academic Open-Source EDA to Real-World Usability," 2020 IEEE/ACM International Conference On Computer Aided Design (ICCAD), San Diego, CA, USA, 2020, pp. 1-7.

[5] A. B. Kahng, "Open-Source EDA: If We Build It, Who Will Come?," 2020 IFIP/IEEE 28th International Conference on Very Large Scale Integration (VLSI-SOC), Salt Lake City, UT, USA, 2020, pp. 1-6, doi: 10.1109/VLSI-SOC46417.2020.9344073.

[6] https://openroad.readthedocs.io/en/latest/main/README.html

[7] https://openroad-flow-scripts.readthedocs.io/en/latest/user/UserGuide.html

[8] [GitHub - The-](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)[OpenROAD](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)[-Project/](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)[OpenROAD](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)[-flow-scripts: ](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)[OpenROAD's](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)[ scripts implementing an RTL-to-GDS Flow. Documentation at ](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)https://openroad-flow-scripts.readthedocs.io/en/latest/