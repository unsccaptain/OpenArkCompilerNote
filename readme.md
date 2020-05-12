## 方舟编译器源码笔记
边看源码边做笔记。方舟编译器做的原来越专业了，再不写笔记就要看不懂了。  
这不是教程，只是源码解读，需要有编译原理基础。和理论有关的会直接贴来源，不会详细解释。

## 目录
+ [第一章：编译和调试](./Chapter1-Build&Debug.md)
+ [第二章：编译架构](./Chapter2-Architecture.md)
+ [第三章：编译流程驱动](./Chapter3-DriverDesign.md)
+ [第四章：IR设计](./Chapter4-IRDesign.md)
+ 第五章：函数级优化
    + [第一节：执行流程](./Chapter5-FunctionPhase-Section1-Launch.md)
    + [第二节：SSATable](Chapter5-FunctionPhase-Section3-me_ssa_tab.md)
    + [第三节：死代码消除](./Chapter5-FunctionPhase-Section2-me_dse.md)
    + [第四节：支配树](./Chapter5-FunctionPhase-Section4-me_dominance.md)
    + [第五节：SSA构造](./Chapter5-FunctionPhase-Section5-me_ssa.md)
    + [第六节：循环识别](./Chapter5-FunctionPhase-Section6-me_loop_analysis.md)