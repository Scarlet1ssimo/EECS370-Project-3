# Project 3 Writeup

懂的都懂

## Staged Registers

| IF/ID     | ID/EX      | EX/MEM         | MEM/WB      | WB/END      |
| --------- | ---------- | -------------- | ----------- | ----------- |
| `instr`   | `instr`    | `instr`        | `instr`     | `instr`     |
| `pcPlus1` | `pcPlus1`  | `branchTarget` | `writeData` | `writeData` |
|           | `readRegA` | `aluResult`    |
|           | `readRegB` | `readRegB`     |
|           | `offset`   |

## Operations Review

| opcode | effect                                 | `10987654321098765432109876543210` |
| ------ | -------------------------------------- | ---------------------------------- |
| add    | `rgD = rgA + rgB`                      | `_______000rgArgB_____________rgD` |
| nor    | `rgD = rgA nor rgB`                    | `_______001rgArgB_____________rgD` |
| lw     | `rgB = [rgA+offset]`                   | `_______010rgArgB--offset-Field--` |
| sw     | `[rgA+offset] = rgB`                   | `_______011rgArgB--offset-Field--` |
| beq    | `PC = (rgA==rgB) ? PC+1+offset : PC+1` | `_______100rgArgB--offset-Field--` |
| halt   |                                        | `_______110______________________` |
| noop   |                                        | `_______111______________________` |

## Baseline

### Data Hazard (Detect and Forward)

Hazard Detection and Forward

Compare regA regB with destination reg in IDEX/EXMEM/MEMWB with opcode (add, nor, lw).

**Stall once if `lw` followed by a `sw` and a hazard detected. (LW hazard)**

### Control Hazard (Branch Prediction with Assume not Equal)

Hazard Detection:

Once take the branch, change first three instructions into noop.

### Happy Reference

1. read instruction from instrMem[PC] and pass
2. calculate and pass PC+1
3. get PC as PC+1 or branchTarget based on ALUresult & instr in EXMEM

4. pass instr
5. pass PC+1
6. read RegA 
7. read RegB
8. decode offset

9. pass instr
10. get the **correct first ALU operand** from somewhere (LIDEX.readRegA, LEXMEM.aluResult, LMEMWB.writeData, or LWBEND.writeData)
11. get the **correct second ALU operand** from somewhere (LIDEX.readRegB, LIDEX.offset, LEXMEM.aluResult, LMEMWB.writeData, or LWBEND.writeData)
12. calculate aluResult as add
13. calculate aluResult as nor
14. calculate aluResult as eq
15. calculate branchTarget according to eq
16. pass a correct valB from somewhere

17. pass instr
18. pass writeData as ALUresult
19. pass writeData as dataMem[aluResult]
20. write valB in dataMem[aluResult]

21. pass instr
22. pass writeData
23. save writeData to reg[dest]


| op        | IF                                | ID                       | EX               | MEM   | WB       |
| --------- | --------------------------------- | ------------------------ | ---------------- | ----- | -------- |
| add       | 1,2,3                             | 4,5,6,7,8                | 9,10,11,12,15,16 | 17,18 | 21,22,23 |
| nor       | 1,2,3                             | 4,5,6,7,8                | 9,10,11,13,15,16 | 17,18 | 21,22,23 |
| lw        | 1,2,3                             | 4,5,6,7,8                | 9,10,11,12,15,16 | 17,19 | 21,22,23 |
| sw        | 1,2,3                             | 4,5,6,7,8                | 9,10,11,12,15,16 | 17,20 | 21,22    |
| beq       | 1,2,3                             | 4,5,6,7,8                | 9,10,11,14,15,16 | 17,18 | 21,22    |
| halt      | 1,2,3                             | 4,5,6,7,8                | 9,15,16          | 17    | 21       |
| noop      | 1,2,3                             | 4,5,6,7,8                | 9,15,16          | 17    | 21       |
| beq take  | noop the first three instructions |
| LW hazard | do nothing                        | noop current instruction |



## Tips for working smartly

0. Scrutinize lecture slides first. 
1. Write a function to find the correct value of a register (doing detection and forwarding).
2. Make a MACRO for repetitive work.
3. Compare your memory and register state with a correct simulator (in project1) after HALT.

## Selftest Script

```shell
#!/bin/bash

# WSL or Linux only!

# This file will automatically test your project on all *.as in current directory with properly configured:
# 1. classic.c for the simulator in project 1
# 2. simulator.c and classic.c should output the content in memory and regfiles to `output` and `output.ans, respectively (do it under -DTEST flag).
# 3. run with ./

gcc simulator.c -Wall -Wextra -Wpedantic -g -o simulator -DTEST
gcc classic.c -Wall -Wextra -Wpedantic -g -o classic -DTEST

allas=$(ls *.as)
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color


for i in $allas; do
    ./assembler $i $i.mc
    ./simulator $i.mc > $i.out
    k1=$?
    ./classic $i.mc > $i.ans
    k2=$?
    diff output output.ans # memory and regfiles printed to file 

    if (( k1==0 && k2==0 && $? == 0 )); then
        echo -e "$GREEN$i passed.$NC"
        rm $i.out
        rm $i.ans
        rm $i.mc  # uncomment for further debuging
    else
        echo -e "$RED$i failed.$NC"
    fi
done
```
