# me_ssa：SSA构造算法
&emsp;&emsp;这是一个转换型analysis，用于进行SSA的构造。源码位置位于src/maple_me/me_ssa.cpp

## SSA构造算法
&emsp;&emsp;在前面的me_ssa_tab中，生成了SSA的所有节点信息，但这时候并不是SSA，因为所有的变量仍然使用的是初始的版本（version 0），def语句也没有更新版本。所以我们需要进行SSA构造。这里的构造算法是标准的PHI插入算法。算法的思想就是，对于每一个变量，找到其所有的def块，如果有多个def块，那么就在这些def块的迭代支配边界上插入这个变量的phi节点。所有phi节点都插入后，进行rename更新每个use的版本。一般来说还会通过活跃性分析类进行剪枝优化。  
&emsp;&emsp;具体理论可以看《SSA BOOK》,不过这本书也是几篇论文拼拼凑凑而成的，所以也可以从它的reference找到具体的论文。

## MeDoSSA
&emsp;&emsp;这个phase通过条用MeSSA的run方法来执行SSA构造。
``` cpp
AnalysisResult *MeDoSSA::Run(MeFunction *func, MeFuncResultMgr *funcResMgr, ModuleResultMgr*) {
  ...
  auto *ssa = ssaMp->New<MeSSA>(*func, *dom, *ssaMp, DEBUGFUNC(func));
  ssa->BuildSSA();
  ssa->VerifySSA();
  ...
}
```

## MeSSA
&emsp;&emsp;主要实现了BuildSSA，就刚刚说的两步走：phi-insertion和rename。
``` cpp
void MeSSA::BuildSSA() {
  InsertPhiNode();
  InitRenameStack(func->GetMeSSATab()->GetOriginalStTable(), func->GetAllBBs().size(),
                  func->GetMeSSATab()->GetVersionStTable());
  // recurse down dominator tree in pre-order traversal
  const MapleSet<BBId> &children = dom->GetDomChildren(func->GetCommonEntryBB()->GetBBId());
  for (const auto &child : children) {
    RenameBB(*func->GetBBFromID(child));
  }
}
```

## 插入PHI节点
#### 收集所有def块

``` cpp
void MeSSA::InsertPhiNode() {
  std::map<OStIdx, std::set<BBId>> ost2DefBBs;
  // 收集所有变量的def节点
  CollectDefBBs(ost2DefBBs);
  ...
```
&emsp;&emsp;在刚刚ssa-tab的创建SSA的过程中，已经将每个语句可能会def什么变量都分析好了，现在只需要遍历函数中的所有语句，将该语句所属基本块加入到对应变量的def块列表中就好了。当然这里有针对JAVA的处理逻辑，不太熟悉JAVA不太清楚其原理。  
&emsp;&emsp;方舟中一共有两种def，一共是mustdef表示对一个变量清晰的def
``` cpp
void MeSSA::CollectDefBBs(std::map<OStIdx, std::set<BBId>> &ostDefBBs) {
  // to prevent valid_end from being called repeatedly, don't modify the definition of eIt
  for (auto bIt = func->valid_begin(), eIt = func->valid_end(); bIt != eIt; ++bIt) {
    auto *bb = *bIt;
    for (auto &stmt : bb->GetStmtNodes()) {
      if (!kOpcodeInfo.HasSSADef(stmt.GetOpCode())) {
        continue;
      }
      // 遍历maydef的def列表，将该基本块加入到def的变量的defbb列表中
      MapleMap<OStIdx, MayDefNode> &mayDefs = GetSSATab()->GetStmtsSSAPart().GetMayDefNodesOf(stmt);
      for (auto iter = mayDefs.begin(); iter != mayDefs.end(); ++iter) {
        const OriginalSt *ost = func->GetMeSSATab()->GetOriginalStFromID(iter->first);
        // 这里的以及下面的!ost->IsFinal() || func->GetMirFunc()->IsConstructor()是针对JAVA的处理逻辑
        if (ost != nullptr && (!ost->IsFinal() || func->GetMirFunc()->IsConstructor())) {
          ostDefBBs[iter->first].insert(bb->GetBBId());
        } else if (stmt.GetOpCode() == OP_intrinsiccallwithtype) {
          ...
        }
      }
      // 回想一下me_ssa_tab中的处理，dassgin有maydef列表也有一个mustdef
      // 这里就是处理那个mustdef
      if (stmt.GetOpCode() == OP_dassign || stmt.GetOpCode() == OP_maydassign) {
        VersionSt *vst = GetSSATab()->GetStmtsSSAPart().GetAssignedVarOf(stmt);
        OriginalSt *ost = vst->GetOrigSt();
        if (ost != nullptr && (!ost->IsFinal() || func->GetMirFunc()->IsConstructor())) {
          ostDefBBs[vst->GetOrigIdx()].insert(bb->GetBBId());
        }
      }
      // Needs to handle mustDef in callassigned stmt
      if (!kOpcodeInfo.IsCallAssigned(stmt.GetOpCode())) {
        continue;
      }
      // 遍历mustdef的def列表，将该基本块加入到def的变量的defbb列表中
      MapleVector<MustDefNode> &mustDefs = GetSSATab()->GetStmtsSSAPart().GetMustDefNodesOf(stmt);
      for (auto iter = mustDefs.begin(); iter != mustDefs.end(); ++iter) {
        OriginalSt *ost = iter->GetResult()->GetOrigSt();
        if (ost != nullptr && (!ost->IsFinal() || func->GetMirFunc()->IsConstructor())) {
          ostDefBBs[ost->GetIndex()].insert(bb->GetBBId());
        }
      }
    }
  }
}
```
#### 插入PHI指令
&emsp;&emsp;每个变量的所有def块都收集完了，然后就需要找到这些def块的迭代支配边界IDF，在IDF的开头插入变量的PHI节点。不过方舟这里好像没有做活跃性分析，不知道是暂时没有还是在这种设计中没必要。
``` cpp
void MeSSA::InsertPhiNode() {
  std::map<OStIdx, std::set<BBId>> ost2DefBBs;
  CollectDefBBs(ost2DefBBs);
  OriginalStTable *otable = &func->GetMeSSATab()->GetOriginalStTable();
  // 遍历所有的变量
  for (size_t i = 1; i < otable->Size(); ++i) {
    OriginalSt *ost = otable->GetOriginalStFromID(OStIdx(i));
    VersionSt *vst = func->GetMeSSATab()->GetVersionStTable().GetVersionStFromID(ost->GetZeroVersionIndex(), true);
    // 没有def块，可能是在构造函数中def或者是被final修饰
    // 其实这里还可以过滤三种情况
    //   1，只有一个def块
    //   2，唯一的def无法支配use
    //   3，def和use在同一个基本块中
    if (ost2DefBBs[ost->GetIndex()].empty()) {
      continue;
    }
    // volatile变量没有SSA格式，因为volatile必须要从内存读写，不能保存在寄存器中。
    // 所以自然不能被转换为SSA的posude register
    if (ost->IsVolatile()) {
      continue;
    }
    std::deque<BB*> workList;
    // 将defBB加入到工作队列中
    for (auto it = ost2DefBBs[ost->GetIndex()].begin(); it != ost2DefBBs[ost->GetIndex()].end(); ++it) {
      BB *defBB = func->GetAllBBs()[*it];
      if (defBB != nullptr) {
        workList.push_back(defBB);
      }
    }
    // 计算迭代支配边界并插入PHI节点
    // 这里效率应该很差，迭代支配边界一直在重复计算。
    while (!workList.empty()) {
      BB *defBB = workList.front();
      workList.pop_front();
      MapleSet<BBId> &dfs = dom->GetDomFrontier(defBB->GetBBId());
      for (auto bbID : dfs) {
        BB *dfBB = func->GetBBFromID(bbID);
        // 如果dfBB的PHI列表中没有该变量的PHI，那么就插入
        if (dfBB->PhiofVerStInserted(*vst) == nullptr) {
          workList.push_back(dfBB);
          dfBB->InsertPhi(&func->GetAlloc(), vst);
        }
      }
    }
  }
}
```

#### 重命名
&emsp;&emsp;重命名有两种算法，一种是在控制流图上进行，这种很抽象很麻烦，LLVM就是这种，我是看不太懂，这里就不说了。另一种是在支配树上，这种既高效又容易理解，这里就介绍一下这种算法。  
&emsp;&emsp;首先还是每个变量都维护一个版本栈，这个不管什么方法都要的。然后开始在支配树上进行dfs，我们可以观察到，对于每一个use，如果有一个def存在于它的祖先结点（支配树）并且在这个def到这个use之间没有其他的def，那么use的版本就是这个def生成的版本。当dfs时，每遇到一个def，就向版本栈追加一个版本。每遇到一个use，就找到这个变量的版本栈，如果栈顶元素所在的基本块支配这个use，那么就直接使用栈顶元素的版本，否则不断弹出栈顶元素直到栈顶元素支配use所在的基本块。因为是在支配树上进行dfs，我们总能保证弹出的def不再会有其他use。证明略（有时间再写，论文上应该也会有）。  
&emsp;&emsp;但方舟实现略有不同，他会在每个节点的dfs过程结束时直接将当前dfs过程对栈的更新进行还原，这样实现简单但可能会有点效率问题。我后面会在博客里介绍上一段中的做法。  
&emsp;&emsp;下面看看方舟的实现：
``` cpp
void MeSSA::RenameBB(BB &bb) {
  // 该BB中的phi已经被Rename了（树上的dfs应该不会有重复访问吧？）
  if (GetBBRenamed(bb.GetBBId())) {
    return;
  }
  SetBBRenamed(bb.GetBBId(), true);

  // record stack size for variable versions before processing rename. It is used for stack pop up.
  std::vector<uint32> oriStackSize(GetVstStacks().size());
  // 获取所有变量当前的最新版本
  for (size_t i = 1; i < GetVstStacks().size(); ++i) {
    oriStackSize[i] = GetVstStack(i)->size();
  }
  // 重命名所有PHI节点，产生新版本
  RenamePhi(bb);
  for (auto &stmt : bb.GetStmtNodes()) {
    // 重命名use
    RenameUses(stmt);
    // 重命名def，产生新版本
    RenameDefs(stmt, bb);
    // 重命名must def，产生新版本
    RenameMustDefs(stmt, bb);
  }
  // 更新后集节点的use
  RenamePhiUseInSucc(bb);
  // 在dom-tree上DFS
  const MapleSet<BBId> &children = dom->GetDomChildren(bb.GetBBId());
  for (const BBId &child : children) {
    RenameBB(*func->GetBBFromID(child));
  }
  // dfs结束后再次回到当前节点，弹出之前更新的version
  for (size_t i = 1; i < GetVstStacks().size(); ++i) {
    while (GetVstStack(i)->size() > oriStackSize[i]) {
      PopVersionSt(i);
    }
  }
}
```
&emsp;&emsp;看下具体Rename的过程，首先是RenamePhi:
``` cpp
void SSA:: RenamePhi(BB &bb) {
  for (auto phiIt = bb.GetPhiList().begin(); phiIt != bb.GetPhiList().end(); ++phiIt) {
    VersionSt *vSym = (*phiIt).second.GetResult();
    // 已经rename了
    if (vSym->GetVersion() > 0) {
      return;
    }
    // 创建新版本，里面会有新版本入栈操作
    VersionSt *newVersionSym = CreateNewVersion(*vSym, bb);
    // 用新版本取代老版本
    (*phiIt).second.SetResult(*newVersionSym);
    ...
  }
}
```
&emsp;&emsp;RenameDef:
``` cpp
void SSA::RenameDefs(StmtNode &stmt, BB &defBB) {
  Opcode opcode = stmt.GetOpCode();
  AccessSSANodes *theSSAPart = ssaTab->GetStmtsSSAPart().SSAPartOf(stmt);
  if (kOpcodeInfo.AssignActualVar(opcode)) {
    // 创建新版本，里面会有新版本入栈操作
    VersionSt *newVersionSym = CreateNewVersion(*theSSAPart->GetSSAVar(), defBB);
    ...
    // 更新为新版本的变量
    theSSAPart->SetSSAVar(*newVersionSym);
  }
  // 更新maydef，基本差不多
}
```
&emsp;&emsp;RenameMustDef和RenameDef也差不多。  
&emsp;&emsp;RenameMayUses：
``` cpp
void SSA::RenameMayUses(BaseNode &node) {
  MapleMap<OStIdx, MayUseNode> &mayUseList = ssaTab->GetStmtsSSAPart().GetMayUseNodesOf(static_cast<StmtNode&>(node));
  MapleMap<OStIdx, MayUseNode>::iterator it = mayUseList.begin();
  // 对所有的MyayUseNode更新其中SSA变量为对应原始变量的版本栈栈顶版本
  for (; it != mayUseList.end(); ++it) {
    MayUseNode &mayUse = it->second;
    VersionSt *vSym = mayUse.GetOpnd();
    mayUse.SetOpnd(vstStacks.at(vSym->GetOrigIdx())->top());
  }
}
```
&emsp;&emsp;Rename过程还是比较简单的，就是对不同指令的处理会有些繁琐。

### 总结
&emsp;&emsp;基本是标准的SSA的Rename算法，有些地方可以剪枝优化的没有做，代码还是比较好理解的。