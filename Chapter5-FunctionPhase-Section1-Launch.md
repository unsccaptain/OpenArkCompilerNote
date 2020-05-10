&emsp;&emsp;MeFuncPhaseManager负责函数级别phase的驱动。  
&emsp;&emsp;在DriverRunner中，当所有需要执行的phase被选出来以后，会在AddPhases方法中通过InterleavedManager的AddPhases方法将所有的phase添加到对应的phase-manager中：
``` cpp
void DriverRunner::AddPhases(InterleavedManager& mgr, const std::vector<std::string>& phases,
	const PhaseManager& phaseManager) const {
    ...
	else if (type == typeid(MeFuncPhaseManager)) {
		mgr.AddPhases(phases, false, timePhases, genMeMpl);
	}
    ...
}
```
&emsp;&emsp;InterleavedManager中：
``` cpp
void InterleavedManager::AddPhases(const std::vector<std::string> &phases, bool isModulePhase, bool timePhases,
                                   bool genMpl) {
    // 创建MeFuncPhaseManager对象
    auto *fpm = GetMemPool()->New<MeFuncPhaseManager>(GetMemPool(), mirModule, mrm);
    // 注册所有定义过的phase
    fpm->RegisterFuncPhases();
    ...
    // 添加在DriverRunner中筛选出来的phase
    fpm->AddPhasesNoDefault(phases);
    // 添加到phase manager sequence中
    phaseManagers.push_back(fpm);
}
```
&emsp;&emsp;所以在这里，首先调用RegisterFuncPhases注册所有支持的phase。RegisterFuncPhases中的代码是用宏定义来生成的。
``` cpp
void MeFuncPhaseManager::RegisterFuncPhases() {
  // register all Funcphases defined in me_phases.def
#define FUNCTPHASE(id, mePhase)                                               \
  do {                                                                        \
    void *buf = GetMemAllocator()->GetMemPool()->Malloc(sizeof(mePhase(id))); \
    CHECK_FATAL(buf != nullptr, "null ptr check");                            \
    RegisterPhase(id, *(new (buf) mePhase(id)));                              \
  } while (0);
#define FUNCAPHASE(id, mePhase)                                                    \
  do {                                                                             \
    void *buf = GetMemAllocator()->GetMemPool()->Malloc(sizeof(mePhase(id)));      \
    CHECK_FATAL(buf != nullptr, "null ptr check");                                 \
    RegisterPhase(id, *(new (buf) mePhase(id)));                                   \
    arFuncManager.AddAnalysisPhase(id, (static_cast<MeFuncPhase*>(GetPhase(id)))); \
  } while (0);
#include "me_phases.def"
#undef FUNCTPHASE
#undef FUNCAPHASE
}
```
&emsp;&emsp;其中me_phases中对phase的定义：
``` cpp
FUNCAPHASE(MeFuncPhase_DOMINANCE, MeDoDominance)
...
```
&emsp;&emsp;这是一个编译器程序中常用的技巧，因为词法单元，语法结构或者IR等会有十分庞大的定义，这时可以将定义放在def文件中，利用宏来自动的生成这些定义对应的代码，这样十分有利与阅读和维护。这种方法本身也就类似于一个DSL，利用宏来解析DSL批量生成对应的代码。  
&emsp;&emsp;在这里，FUNCTPHASE的第一个参数是phase的id，第二个参数是phase的实现类。phase分为两种，一种是analysis-phase，用FUNCTPHASE来生成，另一种则是transform-phase，用FUNCAPHASE来生成。analysis-phase需要导出接口来让其他phase进行查询，transform-phase则不需要这样的借口。这样经过FUNCTPHASE和FUNCAPHASE的宏替换，就可以生成添加所有phase的代码。  
&emsp;&emsp;在一般的编译器中都会有两类pass，一种是分析型的，用于源程序的结构和数据进行分析，比如Loop分析或者AA分析，并将分析结果保存起来让其他的pass可以查询，另一种是转换型，一般是优化，比如死代码删除或常数折叠，但也可能只是方便其他的pass去分析而没有优化作用，比如phi插入或者关键边消除，这种pass一般都会改变程序结构或者添加/删除指令。
&emsp;&emsp;然后在PhaseManager的AddPhasesNoDefault中，将所有的phase添加到phase sequence中。
``` cpp
// 调用父类的AddPhase方法将phase添加到phaseSequences中
void MeFuncPhaseManager::AddPhasesNoDefault(const std::vector<std::string> &phases) {
  for (size_t i = 0; i < phases.size(); ++i) {
    PhaseManager::AddPhase(phases[i]);
  }
  ASSERT(phases.size() == GetPhaseSequence()->size(), "invalid phase name");
}

void AddPhase(const std::string &pname) {
    for (auto it = RegPhaseBegin(); it != RegPhaseEnd(); ++it) {
      if (GetPhaseName(it) == pname) {
        phaseSequences.push_back(GetPhaseId(it));
        phaseTimers.push_back(0);
        return;
      }
    }
    CHECK_FATAL(false, "%s is not a valid phase name", pname.c_str());
}
```
&emsp;&emsp;PhaseManager的子类会通过各自实现的Run方法来执行所有注册进来的phase，MeFuncPhaseManager也实现了自己的Run方法。下面我们来看MeFuncPhaseManager的Run函数实现MeFuncPhaseManager::Run。  
&emsp;&emsp;第一步是将MirFunction转换为MeFunction，我们之前说过，MapleIR是有多层语义结构的，既有接近高级语言的指令，代码中称为MapleIR，也有语言无关指令,代码中称为Me（middle-end?），而中端优化肯定是基于语言无关指令的。同时，因为存在高级控制流指令，所以没法从高级的MapleIR直接构造控制流结构。所以在这里，我们需要将程序的结构拍平，将高级控制流指令转换为低级的控制流指令。这个过程叫做lower：
``` cpp
void MeFuncPhaseManager::Run(MIRFunction *mirFunc, uint64 rangeNum, const std::string &meInput) {
  ...
  func.Prepare(rangeNum);
  ...
}

void MeFunction::Prepare(unsigned long rangeNum) {
  // 首先对MIRFunction进行lower
  MIRLower mirLowerer(mirModule, CurFunction());
  ...
  ASSERT(CurFunction() != nullptr, "nullptr check");
  mirLowerer.LowerFunc(*CurFunction());

  // 生成所有BasicBlock
  CreateBasicBlocks();
  ...

  // 然后构造控制流图
  RemoveEhEdgesInSyncRegion();
  theCFG = memPool->New<MeCFG>(*this);
  theCFG->BuildMirCFG();
  if (MeOption::optLevel > mapleOption::kLevelZero) {
    theCFG->FixMirCFG();
  }
  ...
}
```
&emsp;&emsp;这一步在每一个函数进行分析优化之前都会做一次。它将if，while等结构转化为brtrue，brfalse这种，然后根据跳转/条件跳转和lable根据标准的基本块生成算法来生成所有的基本块，最后构造控制流图并进行一些初级的分析。  
&emsp;&emsp;下一步就是在MeFunction上执行所有的分析优化：
``` cpp
for (auto it = PhaseSequenceBegin(); it != PhaseSequenceEnd(); ++it, ++phaseIndex) {
  PhaseID id = GetPhaseId(it);
  auto *p = static_cast<MeFuncPhase*>(GetPhase(id));
  ...
  RunFuncPhase(&func, p);
  ...
  if (p->IsChangedCFG()) {
    changeCFGPhase = p;
    p->ClearChangeCFG();
    break;
  }
}
```
&emsp;&emsp;当发生控制流变化后会跳出分析过程，这里其实不是很理解。下面是段过程间分析的代码，不太清楚这里的逻辑。
RunFuncPhase的实现：
``` cpp
void MeFuncPhaseManager::RunFuncPhase(MeFunction *func, MeFuncPhase *phase) {
  ...
  AnalysisResult *analysisRes = nullptr;
  MePhaseID phaseID = phase->GetPhaseId();
  // 跳过空函数，但emit phase例外。
  if ((func->NumBBs() > 0) || (phaseID == MeFuncPhase_EMIT)) {
    analysisRes = phase->Run(func, &arFuncManager, modResMgr);
    phase->ReleaseMemPool(analysisRes == nullptr ? nullptr : analysisRes->GetMempool());
    phase->ClearString();
  }
  // 如果是analysis-phase的话，将分析结果保存起来。
  if (analysisRes != nullptr) {
    // if phase is an analysis Phase, add result to arm
    arFuncManager.AddResult(phase->GetPhaseId(), *func, *analysisRes);
  }
}
```
&emsp;&emsp;其实方舟目前的phase结构还比较简单，在LLVM中，类似于AA分析和Loop分析，他们自身也是一个pass manager，他们内部也有一套驱动逻辑，这样LLVM就形成的多层的pass结构，而且同一个pass可能会反复执行多次。