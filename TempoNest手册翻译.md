# TempoNest手册翻译

## INTRODUCTION

此文章的安装部分，仅可作为参考，想要真正安装，请看 **安装实操过程**，此外数学公式的符号可能有错误，请参考原手册

TempoNest 提供了一种同时分析线性或非线性确定性脉冲星定时模型和附加随机参数的方法。它使用贝叶斯推断工具 MultiNest 来高效地探索这个联合参数空间，同时使用 TEMPO2 作为在该空间中每个点上评估定时模型的已建立手段。前者可以在 <http://ccpforge.cse.rl.ac.uk/gf/project/multinest/> 下载，后者可以在 <http://sourceforge.net/projects/tempo2/> 上获得。本文档旨在作为 TempoNest 的安装和操作指南，不包括例如 Tempo2 拟合的定时模型的具体细节

第二节提供了更多关于安装过程的详细信息，第四节描述了当前可用的不同模型，第五和第六节描述了 TempoNest中所有用户设置参数。在第七节中，我们描述了 TempoNest 中存在的模拟设施(simulation facilities)，在第八节中，我们描述了贝叶斯分析产生的输出文件的内容。最后，在第九节中，我们使用 Examples 子文件夹中包含的数据给出一组完整的示例。

## INSTALLATION

编译已使用 gcc 4.6.1 进行测试。因此，其他编译器或编译器的组合目前不受支持。 TempoNest 依赖于 Tempo2 和 MultiNest，但除此之外，还需要访问 gsl 函数以及安装 LAPACK 线性代数库，如果要使用 GPU 运行 TempoNest，则需要安装 CULA 线性代数库。

### Install MultiNest

下载MultiNest的最新版本(3.x) (注意：为了使用 MultiNest ，您需要访问可以从(<http://www.netlib.org/lapack/)下载的> lapack 线性代数库)，并打开根目录中的 Makefile 文件。您需要更改编译器，使得：

```bash
FC = gfortran
CC = gcc
CXX = g++
FFLAGS += -O3 -ffree-line-length-none
CFLAGS += -O3
LAPACKLIB = -llapack
```

然后输入`make`命令，允许源代码进行编译。接下来，您需要创建一个共享版本的`libnest3.a`库，可以使用以下命令完成：

```bash
make libnest3.so
```

TempoNest 的自动配置脚本将通过环境变量`MULTINEST_DIR`查找此文件。

### Install Tempo2

使用以下命令通过 cvs 下载可用的最新版本 Tempo2：

```bash
cvs -z3 -d :pserver:anonymous@tempo2.cvs.sourceforge.net:/cvsroot/tempo2 co -P tempo2
```

进入根目录，并将`T2runtime`文件夹复制到您选择的位置。您需要设置环境变量`TEMPO2`为该目录的位置。然后输入

```bash
./bootstrap
./configure --prefix=install_directory
```

并允许脚本运行，这将为您设置 Makefile，无需您自己进行任何操作。一旦`configure`脚本运行，请输入：

```bash
make
make install
make plugins
make plugins-install
```

这将编译并安装代码到由前缀命令指定的目录。您需要链接到 TempoNest 的库和包含文件将安装到安装文件夹中的 `lib` 和`include`子目录中，分别应存储为环境变量`TEMPO2_LIB`和`TEMPO2_INC`。

### Install TempoNest

TempoNest 支持 CPU 和 GPU，但是在这个阶段，`configure`脚本仅支持 CPU 安装，GPU 安装需要更多的手动干预。

#### 不使用 GPU 的  TempoNest

进入包含`TempoNest`的根目录，然后输入：

```bash
./autogen.sh
./configure
```

这将生成安装 TempoNest 所需的`Makefile`。您应该检查此输出以确保正确找到 multinest 和 tempo2 库。然后只需输入：

```bash
make && make install
```

然后 TempoNest 插件将被编译并复制到您的 Tempo2 插件目录中。

#### TempoNest with GPUs

在 TempoNest 源代码文件夹的`autoconf`子目录中，有一个名为`cula.m4`的文件，您需要首先打开此文件并从第66和70行中删除“Fake”一词，这将使configure脚本正确识别CULA库并在生成插件时使用正确的编译选项。 然后，在运行configure脚本之前，您需要使用 nvcc 编译器编译`TempoNestGPUFuncs.cu`文件，生成可共享对象`libTNGPU.so`，然后可以链接到 TempoNest 安装。可以使用以下命令完成此操作：

```bash
nvcc -arch sm_20 -Xcompiler -O3 --shared -I${CULA_INC_PATH} -o libTNGPU.so TempoNestGPUFuncs.cu -L${CULA_LIB_PATH_64} -lcula_lapack -lcuda --compiler-options '-fPIC'
```

将文件`libTNGPU.so`移动到`TEMPO2_LIB`目录中，然后像正常安装一样输入：

```bash
./autogen.sh
./configure
make && make install
```

这将检测到 CULA 库和 GPU 函数，并允许您在 TempoNest 中使用 GPU 进行所有线性代数运算。

## INTERFACE AND OPERATION

TempoNest 旨在为任何具有（或没有）使用 Tempo2 的经验的人提供简单易用的设计。因此，一旦设置了任何相关参数，并给出了 par 文件`X.par`和 tim 文件`X.tim`，就可以在其根目录中使用以下命令从命令行运行 TempoNest：

```bash
./tempo2 -gr temponest -f X.par X.tim
```

在`.par`文件中设置的用于拟合的参数将包括在 TempoNest 中的定时模型拟合中，因此，任何习惯于使用 Tempo2 的人都可以立即使用 TempoNest 分析他们的数据。在接下来的章节中，我们将详细描述用于脉冲星定时数据分析的不同设置，以及如何在 TempoNest 本身中模拟数据。

## INCLUDED MODELS

TempoNest 允许同时对确定性的定时模型和超出数据中 TOA 误差条所描述的白噪声之外的额外随机信号进行贝叶斯分析。对于贝叶斯分析和采样器 MultiNest 的一般概述， TempoNest中 所有可用模型选择的详细描述，以及其应用于模拟数据的示例，请参考 TempoNest 论文（正在准备中）。下面，我们将总结所包含模型的关键方面，但不涉及数学细节。

### The Timing Model

TempoNest 使用 Tempo2 来评估脉冲星参数文件中的确定性脉冲星定时模型。这里需要考虑的一个问题是Tempo2 必须使用`long double`精度来存储定时模型参数，然而 MultiNest 操作的是双精度。因此，MultiNest 将每个参数`m`的定时模型拟合参数化为以`m0`为参考值的偏差的倍数，倍数由一个缩放参数`mϵ`决定。因此，在每次似然函数评估中，MultiNest 将为每个定时模型参数`m`选择一个值`mp`，然后通过以下公式将其转换为`mt`:

```bash
mt = m0 ± (mp × mϵ)
```

然后可以将其传递给 Tempo2，以便评估定时模型的定时模型参数集`mt`。

### Additional White Noise

在处理脉冲星定时数据时，白噪声的特性可以分为两个组成部分：

1. 对于给定的脉冲星，每个 TOA 都有一个相关的误差条，其大小将在一组观测中变化。因此，我们可以引入一个额外的自由参数，即 EFAC 值，来考虑这种辐射计噪声的可能误差。EFAC 参数因此作为给定脉冲星的所有 TOA 误差条的乘数，在使用特定系统进行观测时起作用。TempoNest 允许为给定脉冲星的所有 TOA 估计单个 EFAC 参数，或者在每个 TOA 都已被标记为观测系统的情况下，可以包括单独的 EFAC。
2. 独立于误差条大小的第二个白噪声组成部分也用于表示一些额外的时间无关噪声。我们将该参数称为 EQUAD。原则上，此参数表示脉冲星的某些物理特性，例如脉冲发射时间的固有抖动，因此应该与观测系统无关。因此，可以为所有 TOA 包括单个 EQUAD 参数。

因此，我们可以重新定义每个 TOA i相关的误差 σ i为 ˆσi ，即:

```bash
σ2i = (αiσi)2 + β2
```

其中，α 和 β 分别代表应用于 TOA i的 EFAC 和 EQUAD 参数。

### Red Noise and DM Models

TempoNest 目前支持两种方法来建模红噪声或 DM。第一种是在 vHL2013 和 Lee et al 2013（在准备中）中描述的时间域方法，其中使用幂律来模拟红噪声或 DM，第二种是在 Lentati L. et al. 2013（L2013）中描述的模型无关的频域方法，其中模型中包含的每个频率的功率都单独参数化。TempoNest 还允许对使用模型无关方法的相同频率进行幂律拟合，以便可以同时使用建模和模型无关方法。

## TEMPONEST PARAMETERS

在 TempoNest 源文件的`Examples/Example1`文件夹中，有一个名为`defaultparameters.conf`的文件，其中包含运行 TempoNest 所需的所有用户定义参数。在本节中，我们将提供所有这些参数及其在程序中的功能的完整列表。这里列出的值将是默认设置的值。

### General Parameters

`root = results/Example1- TempoNest`
生成的输出文件的根名称，详细信息将在第九部分中描述。在 Worked 示例中，这将导致在`Example/results`文件夹中生成一组文件，每个文件都以“Example1-”为前缀，后跟脉冲星的名称和文件扩展名。

`numTempo2its = 1;`
在开始分析之前，Tempo2 执行的迭代次数。 TempoNest 将此拟合的预拟合值作为以下分析的先验分布中心，将1 sigma误差重新分配为拟合的降低卡方根的“sigma”值（请参阅下面的 FitSig）。因此，如果您希望使用`.par`文件中的值作为先验分布的中心，则应将其设置为1。但是，其值将取决于 Tempo2 是否能够收敛于您希望使用 TempoNest 分析的问题。如果无法收敛，则应将其设置为1或0，并手动设置先验分布。

`doLinearFit = 0;`
“doLinearFit” 参数可以切换在时序模型的非线性分析(`doLinearFit=0`)和使用在时序模型参数空间某一点处计算的线性逼近(`doLinearFit=1`)之间。线性化发生的点遵循在 TempoNest 中设置的选项的等级顺序。如果手动设置了任何时间模型参数的先验中心值，则这些值将用于线性化。如果已对联合参数空间执行了最大似然分析，则将使用此过程返回的参数值。否则，如果没有采取这些选项中的任何一项，则将使用在初始 Tempo2 评估期间进行的参数估计值。

`doMax=0;`
这个参数控制是否对所要研究的参数空间执行最大似然分析(`doMax=1`)或不执行(`doMax=0`)。此过程返回的计时模型参数值优先于最初 Tempo2 分析返回的值，但被手动设置的任何先验信息将覆盖它们。

### Marginalisation

TempoNest 可以通过解析地进行时间模型参数的子集边缘化，以降低问题的维数。例如，在拟合跳跃时，描述跳跃的基函数是线性的，因此边缘化振幅可以大大简化分析。TempoNest支持几种选项来进行这个过程：

```bash
doTimeMargin=0
doTimeMargin=1
doTimeMargin=2
```

这些设置允许您在不进行边缘化、边缘化二次自旋减速以及边缘化所有时间模型模型（除了跳跃）之间进行切换（分别为0、1、2）。

```bash
doJumpMargin=0
doJumpMargin=1 
```

这些设置允许您在参数文件中不对任何跳跃进行边缘化或对所有跳跃进行边缘化（分别为0和1）。 请注意，如果执行任何分析边缘化，则默认情况下也将对偏移量进行边缘化处理。 还可以对自定义参数集进行边缘化。这是通过在TempoPriors数组中设置与特定参数相对应的标志来完成的，例如，如果要拟合的参数为（RA，DEC，F0，F1，DM，PMRA，PMDEC，PX），并且您只想分析地对DM进行边缘化，则可以在defaultparameters.conf文件中包含以下内容： `TempoPriors[5][2] = 1;`这里，5表示 Tempo2 将读取的是第五个参数，2是保存此信息的元素。

### Model Selection

1. EFAC
`incEFAC = 0;`
在分析中包括额外的EFAC白噪声参数的标志。支持三个可能的值:

```bash
0: Include No Additional EFAC parameters
1: Include one EFAC parameter for all error bars
2: Include one EFAC parameter for each -sys flag in the tim file for all error bars associated with that flag
```

2.EQUAD
`incEQUAD = 0;`
这是一个用于在分析中包括额外的 EQUAD 白噪声参数的标志。支持两个可能的值：

```bash
0: Include No Additional EQUAD parameters
1: Include one EQUAD parameter for all TOA error bars
```

3.Red Noise
`incRED = 0;`
该标志用于在分析中包括额外的红噪声参数。有三个可能的取值：

```bash
0: Include No Additional Red Noise parameters
1: Include Red Noise using method vHL2013
2: Include Red Noise using method L2013
3: Fit a power law to the frequencies used in model 2 rather than allowing the power to vary independently
```

4.DM
`incDM = 0`
在分析中包含额外的 DM 参数的标志。支持三个可能的值：

```bash
0: Include No Additional DM parameters
1: Include DM using method Lee et al 2013
2: Include DM using method Lentati et al 2013
3: Fit a power law to the frequencies used in model 2 rather than allowing the power to vary independently
```

`numCoeff =3;`
如果将 incRED 或 incDM 设置为2，即要使用 L2013 的红噪声/DM模型，则`numCoeff`参数设置应包括模型中的频率数量。如果观测时间跨越T，则包括的频率将为`n/T`，其中`n`从1到`numCoeff`。

### Priors

`customPriors = 0;`
TempoNest 中的先验可以通过不同的方式处理，对应不同的控制级别。最简单的方法只需要为每个“类”设置一个先验，例如对所有定时模型参数使用一个先验，对所有 EFAC 使用一个先验等。这可以通过设置`customPriors = 0`来实现。然而，如果需要，所有先验都可以手动设置，使用`TempoNestParams.c`文件中的`setTNPriors`函数来实现，当`customPriors = 1`时可以访问该函数。下面我们将解释这两种不同的方法。

1. Simple Priors (customPriors == 0)
`FitSig = 4;`
如果将`customPriors`设置为零，则 TempoNest 将使用时序模型的初始 Tempo2 评估来定义时序模型参数的先验分布。它使用参数`FitSig`，该参数定义 Tempo2 返回的误差的倍数应包括在以最佳拟合值为中心的先验范围内。例如，对于`FitSig = 4`，时序模型参数的先验范围将为： `m0±（4×mϵ）` 其中`m0`是 Tempo2 返回的时序模型参数`m`的最佳拟合值，`mϵ`是其不确定性。

请注意，Tempo2 评估的时序模型显示的不确定性乘以拟合的减少卡方根号。TempoNest 将重新除以减少的卡方根号并使用该值，然后乘以 FitSig。

如果将`doMax`设置为1，则仅使用初始 Tempo2 评估来设置先验范围的宽度，而时序模型参数`m0`的中心值由最大似然值设置。

```bash
EFACPrior[0] = 0.1; 
EFACPrior[1] = 10;
```

随机参数的先验分布处理方式不同，对于每种类型的参数（EFAC，EQUAD 等），都有一个包含两个元素的数组，其中定义了先验分布的起始点和终止点。在上面的示例中，所有EFAC的先验分布被设置为在0.1和10之间均匀变化。

```bash
EQUADPrior[0] = −10;
EQUADPrior[1] = 0;
```

EQUAD 参数的先验概率在对数空间中是均匀的，因此白噪声中EQUAD贡献的振幅将从10^-10延伸到10^0。

```bash
AlphaPrior[0] = 2;
AlphaPrior[1] = 6;
AmpPrior[0] = -20;
AmpPrior[1] = -10;
```

vHL2013 的幂律红噪声模型的先验由两部分组成。Alpha（负的谱指数）的先验在本例中在2到6之间均匀变化（因此谱指数在-2到-6之间变化），而振幅的先验在对数空间中是均匀的，与 EQUAD 参数一样，在这个例子中从10^(-20)到10^(-10)。

```bash
DMAlphaPrior[0] = 2;
DMAlphaPrior[1] = 6;
DMAmpPrior[0] = -20;
DMAmpPrior[1] = -10;
```

DM 模型的幂律先验与红噪声模型的先验相同。这里，Alpha（负谱指数）的先验在本例中均匀变化，从2到6（因此谱指数从-2到-6变化），而振幅的先验在对数空间中均匀变化，例如从10^-20到10^-10。

2.Custom Priors (customPriors == 1)
当 customPriors 设置为1时，TempoNest 将读取`defaultparameters.conf`中与两个数组 TempoPriors 和 Dpriors 相关的任何行，这两个数组将在下面解释。

TempoPriors 包含有关定时模型参数和要包括在定时模型中的任何跳变的信息。先前已经说明，如果 Tempo2 执行了定时模型参数的初始评估，那么对于每个模型参数`m`，它将返回一个最佳拟合值`m0`和一个误差`mϵ`。TempoPriors 数组允许您在 Tempo2 无法收敛于正确解决方案的情况下手动设置此信息。 TempoPriors 数组中参数的顺序将与 TempoNest 启动时显示的参数的顺序相同（请参见示例部分）。这与 Tempo2 读取参数的顺序相同，因此用户必须知道哪个数组元素对应于哪个定时模型参数，并设置适当的值。

TempoNest 将在执行分析之前打印出它使用的先验概率，以便进行简单的合理性检查。 请注意，您不需要手动设置所有先验概率。如果您只想手动指定一个定时模型参数的先验概率，并希望 Tempo2 通过 FitSig 参数设置其余的参数，则只需要指定 TempoPriors 数组的相应元素。但是，如果 Tempo2 无法收敛，则必须手动设置所有定时模型先验概率。

Dpriors 数组包含 MultiNest 从中绘制样本的所有模型参数（定时模型和随机）的双精度先验概率。此数组中参数的顺序将是定时模型参数，然后是任何跳变，接下来是 EFAC、EQUAD，最后是红噪声和 DM 参数（请参见函数本身中的注释）。 例如，如果 FitSig 已设置为4，则与定时模型参数`m`对应的 DPriors 数组的元素将为：

```bash
Dpriors[m][0] = -4;
Dpriors[m][1] = 4;
```

## MULTINEST OPTIONS

如果需要的话，可以在`MultiNestParams.c`文件中找到一些 MultiNest 使用的可调参数。对于大多数情况，可以使用默认设置而不用担心，然而对于低维问题，可以通过设置以下参数来获得显著的加速。对于大维数问题或需要探索的先验空间较大的问题，可能需要调整这些设置以获得最佳性能。

`IS = 0；`
IS 将标志设置为使用重要嵌套抽样。将其设置为1可以获得更准确的证据值。当对大维数问题使用常数效率模式时，这将特别必要。

`modal = 1；`
modal 将标志设置为允许 MultiNest 在数据中搜索多个模式（即解决方案）。1 = 多模态，0 = 单模态。几乎所有情况下只有一个模式是预期的，但在低信噪比示例中可能会检测到多个模式，因此默认设置为1。

`ceff = 0；`
ceff 将 MultiNest 设置为恒定效率模式。这会导致 MultiNest 调整抽样过程中使用的椭圆大小的速度以维持 efr 参数设置的效率。对于大维数问题（¿20dim）非常有用，但是如果不使用重要抽样，则证据的准确性会受到影响。

`nlive = 500；`
nlive 设置 MultiNest 使用的“活点”数量。活点越多，MultiNest 探索参数空间的深度就越高，但抽样过程的完成时间也就越长。根据问题的维度（这是抽样的参数数目，排除需要边缘化的参数），可以参考以下指南：

- 小于5维：100
- 5到10维之间：200
- 10到50之间：500
- 超过50：1000
这个值如果要探索的先验体积很大（即10^7倍的 Tempo2 返回的误差来确认返回的值是真正的峰值还是局部最小值），则可能需要进行调整。在这种情况下，大约1000可能就足够了，但这取决于您想要探索的具体大小，因此可能需要使用多个值来检查其一致性。

`efr = 0.1;`
efr 代表 MultiNest 的目标采样效率。这个数越低，MultiNest 就会越仔细地探索参数空间，但采样完成的时间也会越长。它应该采取的值取决于问题的维度以及是否需要精确估计证据，或者是否仅需要参数估计。作为大致指南：

- 少于10个维度：0.8（参数估计），0.3（证据评估）
- 10到20个维度之间：0.5（参数估计），0.1（证据评估）
- 20到50之间：0.1（参数估计），0.05（证据评估）
- 超过50个维度：0.05（参数估计），0.01（证据评估）

这些数字可能会随着经验的积累而进行调整，在大多数脉冲星定时问题中，后验概率只包含一个模式，因此可以将这个数设置得比这里建议的要高，但这些值代表使用的安全限制。

在恒定效率模式下，无论维度如何，这个值都必须设置得更低：对于D<50，设置为0.05；对于D>50，设置为0.01。

## SIMULATIONS

TempoNest 可以利用现有的 par 和 tim 文件，使用其中包含的时间模型参数和 TOAs 产生一组模拟数据。例如，对于参数文件`X.par`和tim文件`X.tim`，可以通过以下命令查看可用的模拟选项：
` ./TempoNest -sim -h -f X.par X.tim `
这些选项包括包括EFAC和EQUAD参数、红噪声和色散测量贡献等。随机数可以使用系统时钟（默认）或包括种子选项来生成。最后，模拟tim文件中的toa误差栏可以更新以反映更新的白噪声，也可以保持不变。 例如，要生成一组仅由时间文件中存在的TOA误差栏所描述的白噪声组成的模拟残差，可以输入：
`./TempoNest -sim -f J0030+0451.par J0030+0451.tim`
或者，要包括一个功率律红噪声信号，其谱指数为γ=-4.333，对数幅度为A=-13.301，同时重置TOA的误差栏以反映一个新的常量值，则输入如下：
`. / TempoNest -sim -inc red -redamp -13.301 -redindex 4.333 -efac 0 -equad -7 -updateefac -updateequad -f J0030+0451.par J0030+0451.tim`
在这两种情况下，将生成一个新文件`J0030+0451.simulate`，其中包含新的TOAs。然后可以使用tempo2插件plk查看结果，在Tempo2拟合后的残差分别显示在图1的顶部和底部。

## OUTPUT

TempoNest的输出可以分为两类，一类是由MultiNest直接生成的文件，另一类是在分析完成后由TempoNest生成的文件。前者在MultiNest的README文件中有描述，但为了完整起见，我们将在下面包括该描述，然后是TempoNest输出的概要。

1. MultiNest Output
进度监控： MultiNest在每500次迭代后会生成physlive.dat和ev.dat文件，可以用来监控进度。这两个文件的格式和内容如下：`[root] physlive.dat`此文件包含当前的活动点集。它有ndim+2列。前ndim列是ndim参数值。ndim+1列是对数似然值，最后一列是节点编号（用于聚类）。
![image](img\TempoNest手册翻译\Pasted%20image%2020230331123049.png)
![image](img\TempoNest手册翻译\Pasted%20image%2020230331123105.png)

`[root]ev.dat` 该文件包含被拒绝的点的集合。它有ndim+3列。前ndim列是ndim参数值。第ndim+1列是对数似然值，ndim+2列是对数（先验概率）和最后一列是节点编号（用于聚类）。

后验文件： 在算法的每5000次迭代和采样结束后，MultiNest会在由用户提供的根目录下生成五个后验样本文件，如下所示：

`[root].txt` 与getdist兼容，具有2+ndim列。列包括样本概率，-2*对数似然值，参数值。样本概率是样本先验概率乘以其似然性，并由证据归一化。

`[root]postseparate.dat`仅在mmodal设置为T时才会创建此文件。局部对数证据值大于Ztol的模式的后验样本，由2个空行分隔。格式与[root] .txt文件相同。

`[root] stats.dat`包含全局对数证据、其误差和局部对数证据以及参数均值和标准差以及每个具有局部对数证据> Ztol的模式的最佳拟合和MAP参数。

`[root] postequalweights.dat` 包含等权后验样本。列包括参数值，后验对数似然值。

`[root] summary.txt`该文件中每个模式一行，每行有ndim*4+2个值。每行的列包括平均参数值、参数的标准差、最佳拟合（最大似然）参数值、MAP（最大后验概率）参数值、局部对数证据和最大对数似然值。

2.TempoNest Output
`[root] paramnames`包含模型中包括的参数标签列表。

`[root] designMatrix.txt`包含最大似然时间模型参数的设计矩阵元素。第一行包含TOAs的数量和每个TOA设计矩阵中的参数数量。第二行列出了脉冲星的最大似然RA和DEC的弧度，接下来的NTOA × Nparams行是设计矩阵中的元素。

`[root] res.dat`每行有3列，长度为NTOA。第一列是MJD中心时刻，第二列是从分析中减去最大似然时间模型后的残差，第三列是秒为单位的TOA误差栏。

## EXAMPLES

在接下来的部分，我们将提供三个不同示例的操作步骤，展示如何使用TempoNest进行分析。第一个示例将与Tempo2执行相同的分析，数据仅包含白噪声和定时模型。第二个示例将引入红噪声，并使用幂律红噪声模型在联合参数空间的最大似然位置上边缘化定时模型进行分析。最后，在第三个示例中，我们将分析一个仅包含八个TOAs的数据集，Tempo2无法收敛到解决方案，手动设置先验以探索参数空间。

1. Example 1 : Timing Model and White Noise
在 `Examples/Example1` 文件夹中，您会找到两个文件，`J0030+0451.par` 和 `J0030+0451.tim`。如果您查看 par 文件，您会发现有七个参数被标记为`Tempo2` 需要拟合的参数。它们是源位置 `RA` 和 `Dec`、脉冲频率和自转 `F0` 和`F1`、脉冲星的视运动 `PMRA` 和 `PMDEC`，以及视差 `PX`。列出的这些参数值是已经注入到模拟中的值。.tim 文件包含脉冲到达时间（TOAs）和噪声水平，在这个例子中所有的 TOAs 噪声水平都设置为 10^-8 秒。这是模拟中使用的正确噪声水平。TempoNest 应该已经配置好可以在这个例子上运行，因此 `defaultparameters.conf`文件中的参数设置如下：

```bash
numTemp2its = 1;
doLinearFit = 0;
doMax = 0;
incEFAC = 0;
incEQUAD = 0;
incRED = 0;
FitSig = 4;
```

因此，我们将先设置先验范围为Tempo2返回的初始拟合值的±4σ，并且仅包括定时模型进行分析。可以在Example1文件夹中使用以下命令运行TempoNest：
`tempo2 −gr temponest −f J0030+0451.par J0030+0451.tim`
当TempoNest首次运行时，您将看到Tempo2执行拟合过程的输出。然后MultiNest将开始运行。当MultiNest退出时，您应该看到类似于表I的输出，此时采样已完成，结果应在输出文件夹中以“Example1-J0030 + 0451-”和文件类型命名。 TempoNest包括一个简单的Python绘图工具，允许您可视化拟合中包含的任何模型参数的后验概率分布。它位于源目录中与其他TempoNest文件一起，并称为`triplot.py`。可以使用以下语法从命令行中使用它：
`python triplot.py -f Examples/Example1/results/White-J0030+0451`
![image](img\TempoNest手册翻译\Pasted%20image%2020230331123714.png)
表格I. TempoNest计算的定时模型参数估计值与Tempo2的最佳拟合值相比。在这个例子中只包括白噪声和定时模型，因此参数估计非常一致。
同时，你可以通过运行来绘制类似于图2的图形，该图显示了拟合中包含的参数的一维和二维边缘后验概率分布，即该参数取特定值的概率。
2. Example 2: Timing Model and Red Noise - Power Law Model
第二个例子的tim和par文件位于Examples/Example2文件夹中。这里除了白噪声和定时模型之外，还增加了一个具有log振幅为-13.301和谱指数为-4.333的红噪声功率谱。

对于第二个例子，我们需要修改`TempoNest-Params.c`文件中的一些参数。特别是，我们现在需要更改根文件名，并指定我们希望将功率律红噪声组件包括在我们的模型中（`incRED=1`）。此外，我们希望完全边缘化定时模型（`doTimeMargin=2`），但希望在联合参数空间的最大似然点处执行定时模型的线性化（即在包括红噪声时的定时模型的最大似然解），而不是使用初始的Tempo2拟合（`doMax=1`）。
![image](img\TempoNest手册翻译\Pasted%20image%2020230331124025.png)
下面列出了设置。

```bash
numTemp2its = 10;
doLinearFit = 0; doMax = 1;
incEFAC = 0;
incEQUAD = 0;
incRED = 1;
doTimeMargin = 2;
doJumpMargin = 0;
customPriors = 0;
```

![image](img\TempoNest手册翻译\Pasted%20image%2020230331124127.png)
表II。例2中基于时间模型的解析边际化红噪声频谱的随机参数估计。

```bash
FitSig=10;
AlphaPrior[0]=1.1;
AlphaPrior[1]=6.1;
AmpPrior[0]=-15.0;
AmpPrior[1]=-12.0;
```

与以往一样，TempoNest将通过执行初始的Tempo2拟合来开始。然后，它不会立即开始使用MultiNest进行分析，而是执行最大似然分析。这样做的最终输出结果应该类似于下面所示的结果：

```bash
Max RAJ: 0.13289447885714545498
Max DECJ: 0.084841141721966739074
Max F0: 205.5306960882738139
Max F1: -4.3053954092399696773e-16
Max PMRA: -5.4603973577427478113
Max PMDEC: -1.7107901244736347262
Max PX: 4.1427207799322711577
Max Red Amp: -13.30511142
Max Red Index: 3.78608546
```

然后，TempoNest将对时间模型参数进行边际化，并对红噪声频谱进行贝叶斯分析，其结果类似于表II中的结果。然后，我们可以使用以下方法绘制随机参数的后验分布：
![image](img\TempoNest手册翻译\Pasted%20image%2020230331124358.png)
图3. 例2的红噪声振幅和谱指数的一维和二维后验分布。
`python triplot.py -f Examples/Example2/results/Red-J0030+0451`
这将显示一个类似于图3的图像。
3. Example 3: Red Noise and DM
例3包含一个更复杂的例子，其中数据中同时存在红噪声和DM变化，以及一个EFAC。首先使用`incDM=1`，`incRED=1`，包括 EFAC，拟合红噪声和DM幂律模型。然后，使用`incDM=3`，`incRED=3`，使用不同数量的Fourier系数进行比较。仅使用10或20个系数的情况下，后一个模型的约束将更差，但随着系数数量的增加，两者应该收敛到类似的值。
最后，尝试混合使用`incRED=3`和`incDM=2`或反之亦然的模型。各个系数的后验分布将非常非高斯分布，在这种情况下，证据将明显支持两个幂律模型。这就结束了TempoNest的示例。可以使用模拟例程生成自己的数据，并尝试在贝叶斯分析中使用不同的模型。如果您有任何问题，请随时发送电子邮件至[ltl21@cam.ac.uk](mailto:ltl21@cam.ac.uk)。（注意，此邮箱根据我的观察已经失效）
