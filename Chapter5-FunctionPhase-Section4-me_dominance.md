# me_dominance：支配性分析
&emsp;&emsp;这是一个分析型analysis，用于分析控制流图的支配性。源码位置位于src/maple_me/me_dominance.cpp
## 支配性：
&emsp;&emsp;定义：假设一个有向图存在一个入口节点entry和一个出口节点exit，图中存在两个点a，b，如果任意从entry到b的路径都会经过a，那么我们说a支配b，记为a dom b。具体看我的博客：[支配关系以及流图中的循环（一）](https://blog.csdn.net/yeshahayes/article/details/89021028)
&emsp;&emsp;关于支配性可以看论文《Algorithms for Finding Dominators in Directed Graphs》，这片总结了几种常见的支配树生成算法并分析了其效率。
## 实现
&emsp;&emsp;MeDoDominance只是简单的调用了Dominance的接口来初始化，具体的实现还是在Dominance类中：
``` cpp
AnalysisResult* MeDoDominance::Run(MeFunction* func, MeFuncResultMgr*, ModuleResultMgr*) {
  MemPool* memPool = NewMemPool();
  auto* dom = memPool->New<Dominance>(*memPool, *NewMemPool(), func->GetAllBBs(),
  *func->GetCommonEntryBB(), *func->GetCommonExitBB());
  // 生成后序遍历序列和逆后序序列
  dom->GenPostOrderID();
  // 计算支配关系（被支配节点到支配节点的映射）
  dom->ComputeDominance();
  // 计算支配边界
  dom->ComputeDomFrontiers();
  // 计算支配关系（支配节点到被支配节点的映射）
  dom->ComputeDomChildren();
  size_t num = 0;
  // 生成支配树的前序序列
  dom->ComputeDtPreorder(*func->GetCommonEntryBB(), num);
  dom->GetDtPreOrder().resize(num);
  // 生成支配树的dfs序
  dom->ComputeDtDfn();
  // 生成反向流图的后序和逆后序序列
  dom->PdomGenPostOrderID();
  // 计算反向图的支配关系（同上）
  dom->ComputePostDominance();
  // 计算反向图的支配边界
  dom->ComputePdomFrontiers();
  // 计算反向图的支配关系（同上）
  dom->ComputePdomChildren();
  num = 0;
  // 生成反向图支配树的前序序列
  dom->ComputePdtPreorder(*func->GetCommonExitBB(), num);
  dom->ResizePdtPreOrder(num);
  // 生成反向图支配树dfs序
  dom->ComputePdtDfn();
  return dom;
  }
```
## Dominance算法
&emsp;&emsp;没什么意外，和烧脑的LT算法或者SEMI-NCA相比，还是宁愿选择最简单的iterative算法。如果不是太奇葩的流图，效率不输于那些高大上的算法又容易理解。出自论文"A Simple, Fast Dominance Algorithm"  
&emsp;&emsp;计算支配关系：
``` cpp
// Figure 3 in "A Simple, Fast Dominance Algorithm" by Keith Cooper et al.
void Dominance::ComputeDominance() {
  doms[commonEntryBB.GetBBId()] = &commonEntryBB;
  bool changed;
  do {
    changed = false;
    // 在逆后序上进行遍历
    for (size_t i = 1; i < reversePostOrder.size(); ++i) {
      BB* bb = reversePostOrder[i];
      BB* pre = nullptr;
      if (CommonEntryBBIsPred(*bb) || bb->GetPred().empty()) {
        pre = &commonEntryBB;
      }
      else {
        pre = bb->GetPred(0);
      }
      size_t j = 1;
      // 遍历所有前继，找到第一个有迭代信息的节点
      while ((doms[pre->GetBBId()] == nullptr || pre == bb) && j < bb->GetPred().size()) {
        pre = bb->GetPred(j);
        ++j;
      }
      BB* newIDom = pre;
      // 遍历剩余的前继
      for (; j < bb->GetPred().size(); ++j) {
        pre = bb->GetPred(j);
        // 如果迭代信息发生变化
        if (doms[pre->GetBBId()] != nullptr && pre != bb) {
          // 执行交集运算找到最近祖先
          newIDom = Intersect(*pre, *newIDom);
        }
      }
      // 信息有所更新
      if (doms[bb->GetBBId()] != newIDom) {
        doms[bb->GetBBId()] = newIDom;
        changed = true;
      }
    }
  } while (changed);
}
```

## DominanceFrontier算法
&emsp;&emsp;出自同一片论文
``` cpp
// Figure 5 in "A Simple, Fast Dominance Algorithm" by Keith Cooper et al.
void Dominance::ComputeDomFrontiers() {
  for (const BB* bb : bbVec) {
    if (bb == nullptr || bb == &commonExitBB) {
      continue;
    }
    if (bb->GetPred().size() < kBBVectorInitialSize) {
      continue;
    }
    // 遍历所有前继
    for (BB* pre : bb->GetPred()) {
      BB* runner = pre;
      // runner不支配当前节点，意味有其他边汇入当前节点，那么当前节点就是runner的支配边界
      while (runner != doms[bb->GetBBId()] && runner != &commonEntryBB) {
        domFrontier[runner->GetBBId()].insert(bb->GetBBId());
        runner = doms[runner->GetBBId()];
      }
    }
  }
}
```

## A是否支配B
``` cpp
// true if b1 dominates b2
bool Dominance::Dominate(const BB& bb1, const BB& bb2) {
  // 任意节点支配自身
  if (&bb1 == &bb2) {
    return true;
  }
  if (doms[bb2.GetBBId()] == nullptr) {
    return false;
  }
  const BB* immediateDom = &bb2;
  // 沿着bb2在支配树中的位置向上查找，如果找到bb1，那么说明bb1 dom bb2
  do {
    if (immediateDom == nullptr) {
      return false;
    }
    immediateDom = doms[immediateDom->GetBBId()];
    if (immediateDom == &bb1) {
      return true;
    }
  } while (immediateDom != &commonEntryBB);
  return false;
}
```

## 扩展
&emsp;&emsp;这里方舟实现的不是十分完美，有兴趣的可以参考LLVM的实现。LLVM中计算dominance是在抽象的有向图上进行的，有向图的节点类型作为模板传到算法中。任何符合有向图特性的图数据结构只需要抽象出一个traits提供给dominance类，便可以计算出该图的支配树。这种方法可以很好的扩展到各种不同的图而不用重新写代码。后续我会在我的data-structure项目下实现类似的功能。