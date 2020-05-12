# me_loop_analysis：循环识别
&emsp;&emsp;这是一个分析型phase，用于识别控制流中的循环。源码位于src/maple_me/src/me_loop_analysis.cpp。

## 循环识别
&emsp;&emsp;编译过程中的循环识别比较常用的是tarjan论文《Testing flow graph reducibility》中的自底向上识别。首先找到回边，然后从回边起始节点开始沿着reverse cfg向上搜索直到碰到循环头。但这只适用于可规约流图，对于不可规约流图很可能有其他的入口，这样就碰不到循环头了。在实际的高级语言程序中可能会碰到一些不可规约流图。所以可以简单的判断回边起始节点是否被循环头所支配，这样可以把不可规约循环过滤掉。实际程序中很少会有不可规约循环，一般优化意义也不大。  关于这些流图术语看我博客：[流图的分类与识别](https://blog.csdn.net/yeshahayes/article/details/88712047)  
&emsp;&emsp;关于循环识别，在《Identifying Loops In Almost Linear Time》中归纳了几种经典的算法，都可以识别不可规约循环，包括Havlak基于tarjan算法改进的利用并查集搜索循环嵌套关系，以及在DJ图上进行的循环识别算法，它会利用DJ图上的强联通分量。此外还有篇国内的《A New Algorithm for Identifying Loops in Decompilation》，里面的算法也很有意思。  
&emsp;&emsp;关于LLVM如何识别循环看我博客：[LLVM-Analysis-LoopInfo](blog.csdn.net/yeshahayes/article/details/97233940)。

## MeDOMeLoop
``` cpp
AnalysisResult *MeDoMeLoop::Run(MeFunction *func, MeFuncResultMgr *m,
                                ModuleResultMgr *) {
  auto *dom = static_cast<Dominance *>(
      m->GetAnalysisResult(MeFuncPhase_DOMINANCE, func));
  ...
  IdentifyLoops *identLoops =
      meLoopMp->New<IdentifyLoops>(meLoopMp, *func, dom);
  identLoops->ProcessBB(func->GetCommonEntryBB());
  ...
}
```
&emsp;&emsp;可以看到，主要在IdentifyLoops的ProcessBB中实现循环识别。同时还需要依赖于Dominance信息。

## ProcessBB
``` cpp 
// process each BB in preorder traversal of dominator tree
void IdentifyLoops::ProcessBB(BB *bb) {
  if (bb == nullptr || bb == func.GetCommonExitBB()) {
    return;
  }
  /// 遍历一个节点的所有前继
  for (BB *pred : bb->GetPred()) {
    /// 当前节点可以支配前继节点
    if (dominance->Dominate(*bb, *pred)) {
      /// 在这里创建一个loop，当前节点是循环头，被支配的前继是循环尾
      LoopDesc *loop = CreateLoopDesc(*bb, *pred);
      /// 用一个工作队列来搜索循环内的所有节点
      std::list<BB *> bodyList;
      bodyList.push_back(pred);
      while (!bodyList.empty()) {
        BB *curr = bodyList.front();
        bodyList.pop_front();
        // skip bb or if it has already been dealt with
        if (curr == bb || loop->loopBBs.count(curr->GetBBId()) == 1) {
          continue;
        }
        /// 在循环中加入curBB
        loop->loopBBs.insert(curr->GetBBId());
        /// 为当前curBB设置所属循环
        SetLoopParent4BB(*curr, *loop);
        /// 将前继加入到工作队列
        for (BB *curPred : curr->GetPred()) {
          bodyList.push_back(curPred);
        }
      }
      /// 对循环头做同样的操作
      loop->loopBBs.insert(bb->GetBBId());
      SetLoopParent4BB(*bb, *loop);
    }
  }
  // 递归的处理支配树子节点
  const MapleSet<BBId> &domChildren = dominance->GetDomChildren(bb->GetBBId());
  for (auto bbIt = domChildren.begin(); bbIt != domChildren.end(); ++bbIt) {
    ProcessBB(func.GetAllBBs().at(*bbIt));
  }
}
```

## SetLoopParent4BB
&emsp;&emsp;这个方法用于设置循环的嵌套关系。由于遍历支配树是对支配树做BFS，因此最先遍历到的循环总是最大的那个循环，如果在循环头的支配域中找到了一个新的小循环，那么这个小循环肯定是大循环的子循环。
``` cpp
void IdentifyLoops::SetLoopParent4BB(const BB &bb, LoopDesc &loopDesc) {
  // 如果当前BB已经属于某个Loop了
  if (bbLoopParent[bb.GetBBId()] != nullptr) {
    if (loopDesc.parent == nullptr) {
      // 将当前loop的父loop指针设置为那个loop
      loopDesc.parent = bbLoopParent[bb.GetBBId()];
      // 计数循环深度
      loopDesc.nestDepth = loopDesc.parent->nestDepth + 1;
    }
  }
  // 建立BB到循环的映射，每找到一个更小的循环，这个映射就会被更新
  bbLoopParent[bb.GetBBId()] = &loopDesc;
}
```