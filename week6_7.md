# Report Week 6 and 7

本周

1. 循环查找和不变量外提
2. 常量折叠

### 循环

循环的主要优化

1. 循环不变量外提 Loop invariant variable motion
2. 归纳变量消除 Induction variable elimination（未使用
3. 常数循环展开 Const Lopp Expansion （未使用

识别循环

1. 唯一的一个入口：header
2. 至少有一条路径通向header
3. 假定CFG中存在一条回边n → header，循环由n, header 和其他所有**不需要经过header就能到达n的节点**组成
    1. 注释：
    2. 这个定义比较形式，其实就是 header 和 n 之间的节点
    3. 再进一步的，**一个强连通分量就是一个循环**（size = 1 除外
    4. 回边就是指深度优先搜索中，后代指向其祖先的边

嵌套循环

如果CFG中两个循环的header相同，那么可以认为是一个循环

如果CFG中两个循环的header不同，则为以下情况之一

1. 两个循环不相交
2. 一个循环包含在另一个内（内循环和外循环）

## 源码

循环查找

```cpp
void LoopFind::run() {
  std::unordered_set<CFGnode *> reserved;
  std::unordered_set<std::unordered_set<CFGnode *> *> comps;
  std::unordered_set<CFGnode *> nodes;

  for (auto func : m_->getFunctions()) {
    if (func->getBasicBlocks().size() == 0)
      continue;

    buildCFG(func, nodes);
		while (stronglyConnComponents(nodes, comps)) {

      if (comps.size() == 0)
        break;
      for (auto set : comps) {
        auto base = findLoopBase(set, reserved);
			// store result
        auto BBset = new BBset_t;
        for (auto n : *set) {
          BBset->insert(n->BB);
        }
        loops.insert(BBset);
        base2Loop.insert({base->BB, BBset});
        loop2Base.insert({BBset, base->BB});
        for (auto BB : *BBset) {
          if (BB2Base.find(BB) == BB2Base.end())
            BB2Base.insert({BB, base->BB});
          else
            BB2Base[BB] = base->BB;
        }
			}
	}
}
```

循环不变量

run 方法：

对所有最内层循环，从内向外的逐层尝试外提

```cpp
void LoopInvariant::run() {
  finder = std::make_unique<LoopFind>(m_);
  finder->run();
  areaCount = 0;
  for (auto it = finder->begin(); it != finder->end(); it++) {
    if (!finder->isBaseLoop(*it))
      continue;
    auto loop = *it;
    do {
      invariant.clear();
      findInvariants(loop);
      if (invariant.size())
        moveInvariantsOut(loop);
      loop = finder->getParentLoop(loop);
    } while (loop);
  }
}
```

`moveInvariantsOut` 需要创立外提出的基本块`LoopInvariantArea` ，然后处理分支和Phi，更新前驱后继

Loopfind

算法以一个[有向图](https://zh.wikipedia.org/wiki/%E6%9C%89%E5%90%91%E5%9B%BE)作为输入，并按照所在的强连通分量给出其顶点集的一个[划分](https://zh.wikipedia.org/wiki/%E9%9B%86%E5%90%88%E5%8A%83%E5%88%86)。

算法的基本思想如下：任选一节点开始进行[深度优先搜索](https://zh.wikipedia.org/wiki/%E6%B7%B1%E5%BA%A6%E4%BC%98%E5%85%88%E6%90%9C%E7%B4%A2)（若深度优先搜索结束后仍有未访问的节点，则再从中任选一点再次进行）。搜索过程中已访问的节点不再访问。搜索树的若干子树构成了图的强连通分量。

节点按照被访问的顺序存入[堆栈](https://zh.wikipedia.org/wiki/%E5%A0%86%E7%96%8A)中。从搜索树的子树返回至一个节点时，检查该节点是否是某一强连通分量的根节点（lowlink==index）并将其从堆栈中删除。如果某节点是强连通分量的根，则在它之前出堆栈且还不属于其他强连通分量的节点构成了该节点所在的强连通分量。

```cpp
bool LoopFind::stronglyConnComponents(
    std::unordered_set<CFGnode *> &nodes,
    std::unordered_set<std::unordered_set<CFGnode *> *> &result) {
  indexCount = 0;
  stack.clear();
  for (auto n : nodes) {
    if (n->index == -1)
      traverse(n, result);
  }
  return result.size() != 0;
}

void LoopFind::traverse(
    CFGnode *n, std::unordered_set<std::unordered_set<CFGnode *> *> &result) {
  n->index = indexCount++;
  n->lowlink = n->index;
  stack.push_back(n);
  n->onStack = true;

  for (auto su : n->succs) {
    if (su->index == -1) {
      traverse(su, result);
      n->lowlink = std::min(su->lowlink, n->lowlink);
    } else if (su->onStack) {
      n->lowlink = std::min(su->index, n->lowlink);
    }
  }

  if (n->index == n->lowlink) {
    auto set = new std::unordered_set<CFGnode *>;
    CFGnode *tmp;
    do {
      tmp = stack.back();
      tmp->onStack = false;
      set->insert(tmp);
      stack.pop_back();
    } while (tmp != n);
    if (set->size() == 1)
      delete set;
    else
      result.insert(set);
  }
}
```

Double Loop 示例

划分结果

```cpp
find Loop	(3 6 10 18 13 ) delete 3
find Loop	(10 13 ) delete 10
```

### 常量折叠和常量传播

先来看常量传播之前的代码

MIR   if.out.svg

### 常量折叠

```cpp
class ConstFlod {
public:
  ConstFlod(Module *m) : module_(m) {}
	// 二元操作如加减乘除
  ConstantInt *compute(Instruction::OpID op, ConstantInt *v1, ConstantInt *v2);
  // 一元操作非和负
  ConstantInt *compute(Instruction::OpID op, ConstantInt *v1); // not neg
	// 二元比较操作
  ConstantInt *compute(CmpInst::CmpOp op, ConstantInt *v1, ConstantInt *v2);

private:
  Module *module_;
};
```

```cpp
ConstantInt *ConstFlod::compute(Instruction::OpID op, ConstantInt *v1,
                                ConstantInt *v2) {
  int lhs = v1->getValue();
  int rhs = v2->getValue();
  int ret;
  switch (op) {
  case Instruction::Add:
    ret = lhs + rhs;
    break;
  case Instruction::Sub:
    ret = lhs - rhs;
    break;
  case Instruction::Mul:
    ret = lhs * rhs;
    break;
  case Instruction::Div:
    ret = lhs / rhs;
    break;
  case Instruction::Rem:
    ret = lhs % rhs;
    break;
  case Instruction::And:
    ret = lhs & rhs;
    break;
  case Instruction::Or:
    ret = lhs | rhs;
    break;
  default:
    std::cerr << "error in const flod" << std::endl;
    exit_ifnot(_CantFindSuitableOp_compute_ConstFlod, false);
    break;
  }
  return ConstantInt::get(ret, module_);
}

ConstantInt *ConstFlod::compute(Instruction::OpID op, ConstantInt *v1) {
  int hs = v1->getValue();
  int ret;
  switch (op) {
  case Instruction::Not:
    ret = ~hs;
    break;
  case Instruction::Neg:
    ret = -hs;
    break;
  default:
    std::cerr << "error in const flod" << std::endl;
    exit_ifnot(_CantFindSuitableOp_compute_ConstFlod, false);
    break;
  }
  return ConstantInt::get(ret, module_);
}

ConstantInt *ConstFlod::compute(CmpInst::CmpOp op, ConstantInt *v1,
                                ConstantInt *v2) {
  int lhs = v1->getValue();
  int rhs = v2->getValue();
  int ret;
  switch (op) {
  case CmpInst::EQ:
    ret = lhs == rhs;
    break;
  case CmpInst::NE:
    ret = lhs != rhs;
    break;
  case CmpInst::GT:
    ret = lhs > rhs;
    break;
  case CmpInst::GE:
    ret = lhs >= rhs;
    break;
  case CmpInst::LE:
    ret = lhs <= rhs;
    break;
  case CmpInst::LT:
    ret = lhs < rhs;
    break;
  default:
    std::cerr << "error in const flod" << std::endl;
    exit_ifnot(_CantFindSuitableOp_compute_ConstFlod, false);
    break;
  }
  return ConstantInt::get(ret, module_);
}
```

源代码：二元运算

```cpp
if (instr->isAdd() || instr->isSub() || instr->isMul() || instr->isDiv() ||
        instr->isRem() || instr->isAnd() || instr->isOr()) {

      auto v1 = getConstVal(instr->getOperand(0));
      auto v2 = getConstVal(instr->getOperand(1));

			//getConstVal: auto const_v = dynamic_cast<ConstantInt *>(v);

      if (v1 && v2) {
        auto flod_const = flod_->compute(instr->getInstrType(), v1, v2);
        for (auto use : instr->getUseList()) {
					// (instr->lval)->getUseList: %31 
          dynamic_cast<User *>(use.val_)->setOperand(use.arg_no_, flod_const);
        }
        wait_delete.push_back(instr);
      }
    }
```

对于其他二元运算，一元运算和比较，都类似。

constant fold 以module即编译单元为单位

### 常量传播

```cpp
void BBConstPropagation::run() {
  for (auto func : m_->getFunctions()) {
    for (auto bb : func->getBasicBlocks()) {
      bb_ = bb;
			// 常量传播
      ConstPropagation();
    }
  }
	// 修改条件分支语句
	for (auto func : m_->getFunctions()) {
    for (auto bb : func->getBasicBlocks()) {
      builder_->SetInsertPoint(bb);
      if (bb->getTerminator()->isBr()) {
        auto br = bb->getTerminator();
        if (dynamic_cast<BranchInst *>(br)->isCondBr()) //发生变化的一定是条件br
        {
          auto cond = dynamic_cast<ConstantInt *>(br->getOperand(0));
          auto truebb = br->getOperand(1);
          auto falsebb = br->getOperand(2);
          if (cond) {
            if (cond->getValue() == 0) {  // 条件为假
              bb->deleteInstr(br);
              for (auto succ_bb : bb->getSuccBasicBlocks()) {
                succ_bb->removePreBasicBlock(bb);
                if (succ_bb == truebb) {  // truebb 不会被执行
                  for (auto instr : succ_bb->getInstructions()) {
                    if (instr->isPHI()) {
                      for (int i = 0; i < instr->getNumOperand(); i++) {
											// phi指令的操作数是%0 label1 %1 label2 %2
                        if (i % 2 == 1) { // 两个label
                          if (instr->getOperand(i) == bb) {
                            instr->removeOperand(i - 1, i);
                          }
                        }
                      }
                    }
                  }
                }
              }
              builder_->CreateBr(dynamic_cast<BasicBlock *>(falsebb));
							// 在bb末尾插入跳转到falsebb的Br指令
              bb->getSuccBasicBlocks().clear();
              bb->addSuccBasicBlock(dynamic_cast<BasicBlock *>(falsebb));  
							// bb只会跳转到falsebb
            }
```

### 循环不变量外提

[参考课程link](https://www.cs.cmu.edu/~aplatzer/course/Compilers11/17-loopinv.pdf)

如果在某个循环的计算中 d = %x op %y 满足如下条件（之一）则d是循环不变量：

1. x y是数字常量
2. x y 的定义在循环外
3. x y 是循环不变量（递归

主要前置问题：find loop 循环查找

当控制流图中有一条从n到d的边，并且d支配n，称这条边为回边 back edge

自然循环：从d所支配的节点中，能不经过d到达n的节点，构成自然循环

$$\{a : h \ge a, a \rightarrow s_1 \rightarrow s_2 \rightarrow \dots \rightarrow s_n \quad with s_i \neq h\}$$

循环头不能区分一个循环，因为一个循环头可能是多个回边跳转到它们自己所在的自然循环的目标地址，但是回边可以辨识自然循环

更重要的概念：内循环

循环优化的收益按照循环的level指数放大，所以要辨识出inner loop

局部性原理与循环transform（SIMD拓展

[参考link](https://www.cs.cmu.edu/~aplatzer/course/Compilers12/24-cacheloop.pdf)

源代码分析

```cpp
std::map<std::tuple<Value *, Value *, Instruction::OpID>, std::pair<int, Value *>>
    AEB; // string包含两个操作数的地址，int代表pos, value * 代表新变量。
std::map<std::tuple<Value *, int, Instruction::OpID>, std::pair<int, Value *>>
    AEB_rconst;
std::map<std::tuple<int, Value *, Instruction::OpID>, std::pair<int, Value *>>
    AEB_lconst;

std::map<std::tuple<Value *, Value *, int, Instruction::OpID>, Value *>
    load_val;
std::map<std::tuple<Value *, Value *, Instruction::OpID>, Value *> 
		gep_val;
std::map<std::tuple<Value *, int, Instruction::OpID>, Value *> 
		gep_val_const;

std::map<std::pair<std::string, Arglist>, Value *> call_val;

void BBCommonSubExper::run() {
  getEliminatableCall();
  for (auto func : m_->getFunctions()) {
    for (auto bb : func->getBasicBlocks()) {
      AEB.clear();
      AEB_rconst.clear();
      AEB_lconst.clear();
      load_val.clear();
      gep_val.clear();
      gep_val_const.clear();
      call_val.clear();
      if (bb->getPreBasicBlocks().size() == 1) {
        bb_ = bb->getPreBasicBlocks().front();
        CommonSubExperElimination();
        bb_ = bb;
        CommonSubExperElimination();
      } else {
        bb_ = bb;
        CommonSubExperElimination();
      }
    }
  }
}
```

操作数指令为例

pos: 指令的位置，即基本块内的第几条指令

```cpp
if (instr->isBinary()) {
      auto lhs = getBinaryLop(instr);
      auto rhs = getBinaryRop(instr);

      auto lhs_const = dynamic_cast<ConstantInt *>(lhs);
      auto rhs_const = dynamic_cast<ConstantInt *>(rhs);

      if (rhs_const) {
        if (AEB_rconst.find({lhs, rhs_const->getValue(),
                             instr->getInstrType()}) != AEB_rconst.end()) {
          auto iter = AEB_rconst.find(
              {lhs, rhs_const->getValue(), instr->getInstrType()});
          instr->replaceAllUseWith(iter->second.second);
          wait_delete.push_back(instr);
        } else {
          AEB_rconst.insert(
              {{lhs, rhs_const->getValue(), instr->getInstrType()},
               {pos, instr}});
        }
      } else if (lhs_const) {
        if (AEB_lconst.find({lhs_const->getValue(), rhs,
                             instr->getInstrType()}) != AEB_lconst.end()) {
          auto iter = AEB_lconst.find(
              {lhs_const->getValue(), rhs, instr->getInstrType()});
          instr->replaceAllUseWith(iter->second.second);
          wait_delete.push_back(instr);
        } else {
          AEB_lconst.insert(
              {{lhs_const->getValue(), rhs, instr->getInstrType()},
               {pos, instr}});
        }
      }
```

经过此Pass后

```cpp
<label>entry        ;preds: 
    i32 %16  = Rem i32 12 i32 3 
    Br <label> %17 
        ;succ: %17    
<label>17        ;preds: %20    %entry    
    i32 %30  = PHI i32 3 <label> %entry i32 %31 <label> %20 
    i32 %31  = PHI i32 %16 <label> %entry i32 %23 <label> %20 
    i1 %19  = CmpNE i32 %31 i32 0 
    Br i1 %19 <label> %20 <label> %24 
        ;succ: %24    %20    
<label>20        ;preds: %17    
    i32 %23  = Rem i32 %30 i32 %31 
    Br <label> %17 
        ;succ: %17    
<label>24        ;preds: %17    
    ret i32 0 
        ;succ: 
>>>>>>>>>>>> After pass 
16BBCommonSubExper
 <<<<<<<<<<<<

; Module name: 'SysY code'
source_filename: ""
define i32@getint()
define i32@getch()
define @putint(i32 %0 )
define @putch(i32 %0 )
define i32@getarray(i32* %0 )
define i32@putarray(i32 %0 , i32* %1 )
define @_sysy_starttime(i32 %0 )
define @_sysy_stoptime(i32 %0 )
define i32@__mtstart()
define @__mtend(i32 %0 )
define i32@main()
<label>entry        ;preds: 
    i32 %16  = Rem i32 12 i32 3 
    Br <label> %17 
        ;succ: %17    
<label>17        ;preds: %20    %entry    
    i32 %30  = PHI i32 3 <label> %entry i32 %31 <label> %20 
    i32 %31  = PHI i32 %16 <label> %entry i32 %23 <label> %20 
    i1 %19  = CmpNE i32 %31 i32 0 
    Br i1 %19 <label> %20 <label> %24 
        ;succ: %24    %20    
<label>20        ;preds: %17    
    i32 %23  = Rem i32 %30 i32 %31 
    Br <label> %17 
        ;succ: %17    
<label>24        ;preds: %17    
    ret i32 0 
        ;succ:
```

### 交换操作数位置

```cpp
void RefactorPartIns::run() {
  for (auto func : m_->getFunctions()) {
    if (func->getBasicBlocks().size() == 0)
      continue;
    for (auto BB : func->getBasicBlocks()) {
      for (auto ins : BB->getInstructions()) {
        refactor(ins);
      }
    }
  }
}
```

refactor inst的代码

```cpp
void RefactorPartIns::refactor(Instruction *ins) {
  auto &v = ins->getOperands();
  if (v.size() == 0 || !dynamic_cast<ConstantInt *>(v[0]))
    return;
  Value *tmp;
  switch (ins->getInstrType()) {
  case Instruction::OpID::PHI: {
    for (unsigned r = ins->getNumOperand() - 2, l = 0; l < r;) {
      if (dynamic_cast<ConstantInt *>(v[l]) == nullptr) {
        l += 2;
        continue;
      }
      if (dynamic_cast<ConstantInt *>(v[r]) != nullptr) {
        r -= 2;
        continue;
      }
      v[r]->removeUse(ins, r);
      v[r + 1]->removeUse(ins, r + 1);
      v[l]->removeUse(ins, l);
      v[l + 1]->removeUse(ins, l + 1);
      tmp = v[l];
      ins->setOperand(l, v[r]);
      ins->setOperand(r, tmp);
      tmp = v[l + 1];
      ins->setOperand(l + 1, v[r + 1]);
      ins->setOperand(r + 1, tmp);
      l += 2;
      r -= 2;
    }
    break;
  }
  case Instruction::OpID::Cmp: {
    exit_ifnot(_refactor_RefactorPartIns, ins->getNumOperand() == 2);
    tmp = v[0];
    ins->removeUseOfOps();
    ins->setOperand(0, v[1]);
    ins->setOperand(1, tmp);
    auto ins_cmp = dynamic_cast<CmpInst *>(ins);
    switch (ins_cmp->getCmpOp()) {
    case CmpInst::CmpOp::GE:
      ins_cmp->setCmpOp(CmpInst::CmpOp::LE);
      break;
    case CmpInst::CmpOp::GT:
      ins_cmp->setCmpOp(CmpInst::CmpOp::LT);
      break;
    case CmpInst::CmpOp::LE:
      ins_cmp->setCmpOp(CmpInst::CmpOp::GE);
      break;
    case CmpInst::CmpOp::LT:
      ins_cmp->setCmpOp(CmpInst::CmpOp::GT);
      break;
    default:
      break;
    }
    break;
  }
  case Instruction::OpID::Sub:
    tmp = v[0];
    ins->removeUseOfOps();
    ins->setOperand(0, v[1]);
    ins->setOperand(1, tmp);
    ins->setOpID(Instruction::OpID::RSub);
    break;
  case Instruction::OpID::Mul:
  case Instruction::OpID::Or:
  case Instruction::OpID::Add:
  case Instruction::OpID::And:
    tmp = v[0];
    ins->removeUseOfOps();
    ins->setOperand(0, v[1]);
    ins->setOperand(1, tmp);
    break;
  default:
    return;
  }
}
```

### LowerIR

LowerIR：将中层 IR 翻译到低层 IR 并在上面做一系列相关优化

- SplitGEP：将 GEP 指令拆分为子指令（Mul + Add）
- SplitRem：将 Rem 指令拆分成子指令（Div + Mul + Sub）
- FuseCmpBr：将比较指令和分支指令融合（这样比较结果就不占用寄存器）
- FuseMulAdd：将乘法和加法融合为乘加指令
- ConvertMulDivToShift：将部分常数乘除法替换成移位运算
- ConvertRemToAnd：将部分常数取模替换成逻辑与
- RemoveUnusedOp：消除不被用到的操作数

run方法：

```cpp
void LowerIR::run() {
  BBConstPropagation const_fold(m_);
  for (auto f : m_->getFunctions()) {
    for (auto bb : f->getBasicBlocks()) {
      fuseCmpBr(bb);
      splitGEP(bb);
      if (!disable_div_optimization)
        convertRemToAnd(bb);
      splitRem(bb);
    }
  }
  const_fold.run();
  for (auto f : m_->getFunctions()) {
    for (auto bb : f->getBasicBlocks()) {
      removeUnusedOp(bb);
    }
  }
  for (auto f : m_->getFunctions()) {
    for (auto bb : f->getBasicBlocks()) {
      fuseAddLoadStore(bb);
    }
  }
  for (auto f : m_->getFunctions()) {
    for (auto bb : f->getBasicBlocks()) {
      if (!disable_div_optimization)
        convertMulDivToShift(bb);
      fuseConstShift(bb);
      fuseShiftLoadStore(bb);
    }
  }
  for (auto f : m_->getFunctions()) {
    for (auto bb : f->getBasicBlocks()) {
      fuseMulAdd(bb);
    }
  }
  // add has three ops
  for (auto f : m_->getFunctions()) {
    for (auto bb : f->getBasicBlocks()) {
      fuseShiftArithmetic(bb);
    }
  }
  m_->setIRLevel(Module::LIR);
}
```

具体的优化拆分函数

`SplitREM` 对之前求余数的“伪”指令拆分为Div Mul Sub

$$a \space \text{mod}\space b= m*b +\text{remains} \space 其中\space m = a//b$$

```cpp
void LowerIR::splitRem(BasicBlock *bb) {
  auto &insts = bb->getInstructions();
  for (auto iter = insts.begin(); iter != insts.end();) {
    auto inst = *iter;
    if (inst->isRem()) {
      auto op1 = inst->getOperand(0);
      auto op2 = inst->getOperand(1);
      auto div = BinaryInst::createDiv(op1, op2);
      div->setParent(bb);
      auto mul = BinaryInst::createMul(div, op2);
      mul->setParent(bb);
      auto sub = BinaryInst::createSub(op1, mul);
      sub->setParent(bb);
      insts.insert(iter, div);
      insts.insert(iter, mul);
      insts.insert(iter, sub);
      inst->replaceAllUseWith(sub);
      inst->removeUseOfOps();
      iter = insts.erase(iter);
    } else
      ++iter;
  }
}
```

`SplitGEP` 对数组操作之前的“伪”指令GEP拆分为Mul+Add

```cpp
void LowerIR::splitGEP(BasicBlock *bb) {
  auto &insts = bb->getInstructions();
  for (auto iter = insts.begin(); iter != insts.end();) {
    auto inst = *iter;
    if (inst->isGEP() && (inst->getNumOperand() == 2)) {
      auto size = inst->getType()->getPointerElementType()->getSize();
      auto offset = BinaryInst::createMul(inst->getOperand(1),
                                          ConstantInt::get(size, m_));
      offset->setParent(bb);
      auto addaddr = BinaryInst::createAdd(inst->getOperand(0), offset);
      addaddr->setParent(bb);
      insts.insert(iter, offset);
      insts.insert(iter, addaddr);
      inst->replaceAllUseWith(addaddr);
      inst->removeUseOfOps();
      iter = insts.erase(iter);
    } else if (inst->isGEP()) {
      std::cerr << "GEP have more than 2 operands ???" << std::endl;
      exit_ifnot(_splitGEP_LowerIR, false);
    } else {
      ++iter;
    }
  }
}
```

`FuseCmpBr` 分支指令的第一个操作数是Cmp的左值，对形如

```cpp
  i1 %19  = CmpNE i32 %31 i32 0 
  Br i1 %19 <label> %20 <label> %24 
        ;succ: %24    %20    
```

且`Cmp` 的结果不再被继续使用，可以优化掉，具体来说，需要这些条件：

1. bb 的最后一条指令是条件跳转
2. 条件跳转的布尔结果从该bb内的一条Cmp指令计算得来
3. 该布尔结果不在其他地方被使用

```cpp
void LowerIR::fuseCmpBr(BasicBlock *bb) {
  auto t = bb->getTerminator();
  if (t->isBr()) {
    auto br = static_cast<BranchInst *>(t);
    if (br->isCondBr()) {
      auto inst = static_cast<Instruction *>(br->getOperand(0));
      if (inst->isCmp()) {
        auto cmp = static_cast<CmpInst *>(br->getOperand(0));
        if (cmp->getParent() == bb && cmp->getUseList().size() == 1) {
          br->fuseCmpInst();
          bb->deleteInstr(cmp);
          delete cmp;
        }
      }
    }
  }
}
void BranchInst::fuseCmpInst() {
  auto cmp = static_cast<CmpInst *>(getOperand(0));    // 推导：br的布尔值是cmp的左值
  auto if_true = getOperand(1);   // true bb
  auto if_false = getOperand(2);  // false bb
  removeUseOfOps();   setNumOps(4);
  setOperand(0, cmp->getOperand(0));
  setOperand(1, cmp->getOperand(1));
  setOperand(2, if_true);
  setOperand(3, if_false);
}
```

`fuseMulAdd` 对于需要先乘后加（尤其是数组元素地址 A[i] = A + i*4

对于形如 `offset = i*4 ; address = index + offset;` 可以把`i,4,index` 创建一个乘加指令

```cpp
void LowerIR::fuseMulAdd(BasicBlock *bb) {
  auto &insts = bb->getInstructions();
  for (auto iter = insts.begin(); iter != insts.end(); ++iter) {
    auto inst = *iter;
    if (inst->isAdd()) {
      auto op1 = dynamic_cast<Instruction *>(inst->getOperand(0));
      if (op1 && op1->isMul() && (op1->getParent() == bb) &&
          (op1->getUseList().size() == 1)) {
        auto muladd = MulAddInst::createMulAddInst(
            op1->getOperand(0), op1->getOperand(1), inst->getOperand(1));
        iter = insts.erase(iter);
        iter = insts.insert(iter, muladd);
        muladd->setParent(bb);
        bb->deleteInstr(op1);
        inst->replaceAllUseWith(muladd);
        inst->removeUseOfOps();
        continue;
      }
      auto op2 = dynamic_cast<Instruction *>(inst->getOperand(1));
      if (op2 && op2->isMul() && (op2->getParent() == bb) &&
          (op2->getUseList().size() == 1)) {
        auto muladd = MulAddInst::createMulAddInst(
            op2->getOperand(0), op2->getOperand(1), inst->getOperand(0));
        iter = insts.erase(iter);
        iter = insts.insert(iter, muladd);
        muladd->setParent(bb);
        bb->deleteInstr(op2);
        inst->replaceAllUseWith(muladd);
        inst->removeUseOfOps();
      }
    }
  }
}
```