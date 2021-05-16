# Report Week 2

本周周报

1. 阅读了SSA之前的MIR代码
2. 测试求阶乘函数的所有Pass之后的代码
3. 看了HIRToMIR，SimplifyCFG和死代码删除

接下来的打算

1. 看函数优化和内联部分
2. 看`Mem2RegPass`和常量传播，公共子表达式部分

SSA分配寄存器前，此阶段无需要计算的简化（如常量折叠和传播），主要是 control-flow 和 变量定义的死代码删除，此阶段标记为MIR_MEM

### **HIRToMIR**

`HIRToMIR` 继承自Transform类

回顾一下BaseBlock的继承关系，HIR阶段BaseBlock用Type来标识其的类型

```cpp
class BaseBlock : public Value {
public:
  enum BlockType {Basic,  If,  While};

  bool isBasicBlock() const { return block_ty_ == Basic; }
  bool isIfBlock() const { return block_ty_ == If; }
  bool isWhileBlock() const { return block_ty_ == While; }
  //...
}
```

HIR层级上保留了IF和WHILE

```xml
; BaseBlock #1
<label>entry
    i32* %1  = Alloca
    Store i32 %0 i32* %1

; BaseBlock #2
IF :
    Cond:
    <label>3
        i32 %4  = Load i32* %1
        i1 %5  = CmpEQ i32 %4 i32 0
    Then:
    <label>6
        ret i32 1
    Else:
<label>7
    ret i32 0
```

到了MIR之后比较符合Basic Block的定义，给每个label添加了前驱后继：

- 控制流只能从第一条语句进入基本块
- 基本块最后一条指令只能是Br分支指令或Ret返回指令
- 中间不能有跳转

```xml
<label>entry        ;preds:
    i32* %1  = Alloca
    Store i32 %0 i32* %1
    Br <label> %3
        ;succ: %3
<label>3        ;preds: %entry
    i32 %4  = Load i32* %1
    i1 %5  = CmpEQ i32 %4 i32 0
    Br i1 %5 <label> %6 <label> %7
        ;succ: %7    %6
<label>6        ;preds: %3
    ret i32 1
        ;succ:
<label>7        ;preds: %3
    ret i32 0
        ;succ:
```

run方法

```cpp
void HIRToMIR::run() {
  for (auto func : m_->getFunctions()) {
    auto basebbs = func->getBaseBlocks();
    //auto &basicbbs = func->getBasicBlocks(); 注释这条不影响执行
    BasicBlock *next_bb = nullptr;
    for (auto iter = basebbs.rbegin(); iter != basebbs.rend(); ++iter) {
      auto basebb = *iter;
      next_bb = genBaseBlock(basebb, next_bb, nullptr, nullptr, func);
    }
    func->getBaseBlocks().clear();
  }
  m_->setIRLevel(Module::MIR_MEM);
}
```

`HIRToMIR.cc`剩余部分都是`genBaseBlock()`函数：

根据BaseBlock的类别，把If类BaseBlock的三类If cond，Then，Else部分转化成Basic Block并设置前驱以及后继

### **SimplifyCFG**

`SimplifyCFG` 类包含的方法：

```cpp
class SimplifyCFG : public Transform {
public:
  SimplifyCFG(Module *m) : Transform(m) {}
  ~SimplifyCFG(){};
  void run() override;
  void RemoveNoPredecessorBB();
  void MergeSinglePredecessorBB();
  void EliminateSinglePredecessorPhi();
  void EliminateSingleUnCondBrBB();
  void RemoveSelfLoopBB();
};
```

看Simplify CFG的代码，`run` 对module内的所有函数，把几个上述优化方法执行了一遍。

顺序：

注：优化可能会改变程序的IR形式，使之包含无用的控制流。如果编译器包括可能产生无用控制流（作为一种副效应）的优化，那么其中也应该包括一趟对应的处理，即通过消除无用控制流来简化CFG。

1. `RemoveNoPredecessorBB()`
    1. 如果某basic block不是`entryBB`且无前驱，标记该bb为待删除
    2. 并对其后继的所有succbb，在它们的前驱列表中移除bb
    3. 并且对于后继bb的所有指令，如果是PHI指令，因为这条指令少了个待选择的基本块，修改
2. `EliminateSinglePredecessorPhi()`
    1. 如果bb的前驱仅有一个或者它就是`entry`块
    2. 对bb中所有指令，如果是Phi指令
    3. 检查这个指令的`uselist`，直接设置为前驱的那个`value`
3. `MergeSinglePredecessorBB()`
    1. （自上而下地）对每个bb，如果其仅有1个前驱基本块prebb
    2. 若前驱基本块的最后一条指令br不是跳转指令，可以报错结束编译
    3. 若prebb的最后一条指令br满足`br->getNumOperand() == 1`，即仅有一个跳转分支，可以删除br，把bb的指令挪到bb的前驱中
    4. 更新bb后继的前驱
    5. 更新对bb的所有使用为指向prebb
4. `EliminateSingleUnCondBrBB()`
5. `RemoveSelfLoopBB()`
    1. 如果前驱基本块仅有一个而且这个前驱基本块就是自身，标记为删除
    2. selfloop一般不会出现在IR builder的结果中，只会由优化导致

### **死代码删除**

Dead Code

- 无用的，对给定操作来说，如果没有其他操作使用其结果，或使用其结果的所有指令本身都是死代码，那么这个操作是无用的（useless）
- 不可达的，对于给定操作来说，如果没有有效的控制流路径包含该操作，则其是不可达（unreachable）

死代码删除在所有 Pass 中和SimplifyCFG都多次出现——有些优化可能会导致不可达基本块或者死代码

```cpp
// FunctionOptimization
PM.addPass<FunctionOptimization>();
PM.addPass<DeadCodeEliminate>();
// Inline
PM.addPass<FunctionInline>();
PM.addPass<GlobalVariableLocal>();
PM.addPass<DeadCodeEliminate>();
PM.addPass<SimplifyCFG>();
// SSA
// 常量传播后删除
PM.addPass<SimplifyCFG>();
PM.addPass<BBConstPropagation>();
PM.addPass<DeadCodeEliminate>();
```

举例：

```c
int function() {
  a = 2;
  if (a > 5) {
    return(5);  //dead code
  }
  return(a);
}
```

经过常量折叠（计算常量表达式）传播

```c
int function() {
  a = 2;
  if (0) {
    return(5);  //dead code
  }
  return(a);
}
```

就能简化为

```c
int function() {
  a = 2;
  return(a);
}
```

run方法：

```cpp
void DeadCodeEliminate::run() {
  deleteDeadFunc(m_);  // 删除不可达函数
  detectNotSideEffectFunc(m_);   //检测模块中无副作用的函数
  for (auto func : m_->getFunctions()) {
    if (func != m_->getMainFunction())
      deleteDeadRet(func);       //删除除了main函数外的无效return语句
  }
  for (auto func : m_->getFunctions()) {
    deleteDeadInst(func);    //
    deleteDeadStore(func);   // 删除没有被使用的本地变量Alloca
    deleteDeadInst(func);    // 
  }
}
```

删除不会被访问的函数，检测无副作用的函数，然后对于所有函数，如果不是main函数，删除无效的Ret语句，如在第一个有效Ret之后的所有Ret，不会被访问到的Ret，然后对于所有函数，删除不会被访问的指令和不被使用的local赋值语句

注：每条指令都有4个操作数

```cpp
Value *getRVal() { return this->getOperand(0); }
Value *getLVal() { return this->getOperand(1); }
Value *getOffset() { return this->getOperand(2); }
Value *getShift() { return this->getOperand(3); }
```

1. 检测无副作用的函数：首先排除空函数和main函数，它们不在考虑范围之内；在MIR_MEM 层级，如果有全局的存储指令，调用指令，认为函数有副作用，把它们添加到一个无序集合中
2. 删除无用函数：若函数的use list为空`func->getUseList().empty()`且不是main函数即可删除
3. 删除无用指令：若指令的use list为空且不为Store，Call，Ret，Br之一则删除
4. 删除无用函数：在module层级，对其所有函数，若use list为空且不是main函数，删除
5. 删除不使用的返回值：对函数的uselist中的每个use，若`!use.val_->getUseList().empty()`无需检查，如果不成立则检查函数中所有指令，如果有ret删除
6. 删除无用存储

```cpp
void DeadCodeEliminate::deleteDeadStore(Function *func) {
  for (auto bb : func->getBasicBlocks()) {
    std::unordered_set<StoreInst *> pre_stores;
    std::vector<Instruction *> wait_remove;

    for (auto inst : bb->getInstructions()) {
      if (inst->isStore()) {
        auto new_store = static_cast<StoreInst *>(inst);
        StoreInst *pre_store = nullptr;
        for (auto pre : pre_stores) {
          if (isEqualStorePtr(pre, new_store)) {
					// 如果有一个在该赋值指令之前的赋值指令pre的
					// isEqualStorePtr 检查两个赋值语句左值是否相同
            wait_remove.push_back(pre);
            pre_store = pre;
            break;
          }
        }
        if (pre_store)
          pre_stores.erase(pre_store); // 删除最后一个pre_store
        pre_stores.insert(new_store);

      } else if (inst->isLoad()) {
        auto load = static_cast<LoadInst *>(inst);
        std::vector<StoreInst *> removes;
        for (auto pre : pre_stores) {
          if (isStrictEqualStoreLoadPtr(pre, load)) {
            // 替换使用此load的value的所有use
            load->replaceAllUseWith(pre->getRVal());
            wait_remove.push_back(load);
          } else if (isEqualStoreLoadPtr(pre, load)) {
            removes.push_back(pre);
          }
        }
        for (auto r : removes)
          pre_stores.erase(r);
      } else if (inst->isCall()) {
        auto call = static_cast<CallInst *>(inst);
        if (isSideEffectFunc(call->getFunction())) {  //递归
          pre_stores.clear();
        }
      }
    }
    for (auto inst : wait_remove) {
      bb->deleteInstr(inst);
      store_counter++;
    }
  }
}
```

utility functions:

1. `getFirstAddr()` ：对value类对象v，动态类型转换为Instruction类对象inst，返回其第一条指令：如果该指令为Alloca则返回inst；或者是GetElementPointer（递归其Rval）；或者是Load返回v；或者该指令的操作符有指针（递归其解引用的对象）
2. `isLocalStore(value *)` : 对该store指令的左值即要存储的变量调用 `getFirstAdd` 如果是Alloca，返回true
3. `isSideEffectAndCall(Instruction *inst)` :  返回指令是否为Store，Call，Ret，Br之一
4. `isSideEffect(Instruction *inst)` : 对比上个函数增加了对被调用 function 是否有副作用的判断
5. `isEqualStorePtr` 检查两个store指令的左值，偏移量（如果有）是否相同

### Function inline

可以修改内联的优化程度

```cpp
class FunctionInline : public Transform {
public:
  const bool ONLY_INLINE_SMALL_FUNC = false;
  const bool INLINE_BB_NUM_MAX = 4;
  const bool NOT_INLINE_MULTILEVEL_LOOP_FUNC = true;
  const int INLINE_LOOP_LEVEL_MAX = 1;
  const bool INLINE_RECURSIVE = false;
	//...
}
```

### 常量传播

首先是 constant fold 工具函数，常量折叠有三个范围

1. 对于二元算数或逻辑运算，创建值为结果的新常量代替两个操作数和一个操作符
    1. ADD SUB MUL DIV REM, ADD OR
2. 对于一元算数或逻辑运算，创建值为结果的新常量代替一个操作数和一个操作符（求负求非
    1. NEG NOT
3. 对于二元比较运算，使用0或1代表逻辑的真或假
    1. EQ NE GT LT GE LE