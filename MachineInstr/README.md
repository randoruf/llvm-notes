# MachineInstr

## 推荐资料

https://llvm.org/devmtg/2017-10/slides/Braun-Welcome%20to%20the%20Back%20End.pdf

## 易错点

- When iterating `MachineInstr`, there are some registers that will not be emitted but sit there. After the register allocation phase, those registers should have the `<imp>` flag. 
  - Read https://llvm.org/devmtg/2017-10/slides/Braun-Welcome%20to%20the%20Back%20End.pdf
- 

