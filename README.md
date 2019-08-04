# antiOllvm
anti ollvm like flat/bcf/sub

这是我个人做的一些关于去OLLVM混淆相关的研究，会不定期放出一些最新的进展和思路。

利用符号执行的思路，目前在Android arm32的so代码上面能够有效的解决[孤挺花](https://github.com/GoSSIP-SJTU/Armariris)版本的平坦化混淆。

最重要的一点在于能够很好的patch指令，修复后的so代码能够直接利用ida的F5插件得到伪代码。

不过目前代码还在实验阶段，在识别真实块和还原控制流以及patch修复方面还有很长的适配、兼容工作需要完成。

Good luck for me~



#### 去平坦化思路

一共用了三个类即三个模块去处理：`XCFGMgr`、`DSEFlow`、`Patcher`

- XCFGMgr

  对二进制文件做一些预处理，识别并提取出序言块、分发器、真实块、无用块、结束块等。预处理主要是对一些干扰指令进行nop等操作，避免下一步动态执行的时候造成干扰。我这里主要nop了一些`it`条件执行指令。其实在整个区混淆中，对真实块的识别相当重要，这一步会直接影响到最终恢复的结果。针对OLLVM5.0之前的版本来说，如果满足标准的分发器模型那么对真实块的识别十分准确，不过其他变异版本就不能很好地识别。这里下一步可以根据指令特征来识别真实块或者无用块。

- DSEFlow

  这一步的输入就是上一步的输出，将整理好的各种块传入，进行动态符号执行，其实就是模拟执行加上约束求解器进行运算，传入一个真实块就能找到他的下一个真实块。如果包含分支的时候，就hook判断的真值然后强制走两个分支。这里有个问题在于，对于孤挺花版本的来说，很多状态值是在序言块之中初始化的，若按照公开的动态执行，你得到的控制流肯定是有问题的。这里我的解决方案是每次在寻找一个真实块的下一个真实块的时候都会执行一遍序言块，这样保证所有的状态量在内存中都是初始化的。

- Patcher

  关于在ARM架构下的指令修复，目前还没有公开的解决方案。大多数人这一步都没有去做，因为找到真实的控制流其实去平坦化就完成了。不过我们最终目的是需要理解原始代码的意义，如果不能用IDA的F5功能那将毫无意义（~~太菜了，直接阅读汇编的大佬请坐~~）。

  在这块上面需要对OLLVM的混淆模型十分清楚，在分析孤挺花版本的混淆代码的时候，发现真实块有以下几种类型，或者说以下几种特点。大分类有两种：带分支和不带分支

  - 不带分支(状态量只有一个值)

    状态量通过寄存器更新，寄存器常量值在序言块中初始化

    ```assembly
    MOV             R0, R5 # R5为一个常量
    B               loc_11824 # 跳转到分发器
    ```

    状态量直接通过常量值更新

    ```assembly
    MOV             R0, #0x929E2EDC # 常量
    B               loc_11824 # 跳转到分发器
    ```

  - 带分支(状态量有两个值)

    最标准的一种，其中一个值在之前采用常量或者寄存器更新，另一个值通过常量更新，然后跳转到分发器

    ```assembly
    MOV           R0, #0xDD4441F9  # 常量
    ...
    ITT LT
    MOVLTW        R0, #0xAD0C  # 常量更新
    MOVTLT        R0, #0x4D73
    B             loc_1182C # 跳转到分发器
    ```

    一个值通过寄存器更新，另一个值也通过寄存器更新

    ```assembly
    MOV             R0, R8
    ...
    IT NE
    MOVNE           R0, R5
    B               loc_1182C
    ```

    一个值通过条件执行更新，另一个值通过寄存器更新

    ```assembly
    ITT GT
    MOVGTW       	 	R0, #0x743
    MOVTGT        	R0, #0x95CB
    ...
    IT EQ
    MOVEQ         	R0, R10
    B              	loc_11646
    ```

    其实可以把三种都可以看成一种，在带分支的真实块的时候，肯定状态量肯定有两个值，只是更新的条件或者方式不同，抓住这个点就行，然后再根据此来patch。

  **还有一个重要的注意点在于，patch的时候，条件跳转的地址大小有限。如果跳转的地址相差太大就会存在指令空间不够的情况。这时候我采用的策略是拆分为两条跳转指令，其中一条为条件跳转到当前的后面第二条指令位置，接着就是一条直接跳转指令，这样就能够既不破坏原始执行流又能跳转任意地址。**

  ```python
  # 根据指令编码计算跳转的偏移
  B_OPRAND = (int((target_addr_1-patch_addr-4)/2) + 0x100) & 0xff 
  
  # 判断是否指令空间足够，不够则分为两条指令
  if abs(int((target_addr_1-patch_addr-4)/2)) >= 128:
      is_need_split = True
      # patch的当前地址
      file_offset = last_instr.address - base_addr - 2*2
      patch_addr = last_instr.address - 2*2
      # 目标跳转地址
      target_addr_1 = childs[0]
  
      self.patch_data[file_offset] = 0x0
      self.patch_data[file_offset+1] = B_OPCODE
      file_offset += 2
      data = [0x0, B_OPCODE]
  else:
      # 将条件跳转与偏移写入数据并将文件偏移移动到False分支
      self.patch_data[file_offset] = B_OPRAND
      self.patch_data[file_offset+1] = B_OPCODE
      file_offset += 2
      data = [B_OPRAND, B_OPCODE]
  ```

  
