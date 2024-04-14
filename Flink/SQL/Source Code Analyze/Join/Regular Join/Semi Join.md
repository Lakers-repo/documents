### Logic & Physical Plan & Execution

- Physical Logical Optimized
```mermaid
flowchart LR
FlinkLogicalJoin-->|StreamPhysicalJoinRule|StreamPhysicalJoin
```
- Transform to Exec Plan
```mermaid
flowchart LR
StreamPhysicalJoin-->StreamExecJoin
```
- Stream Operator
```mermaid
flowchart LR
StreamExecJoin-->|Semi or Anti Join|StreamingSemiAntiJoinOperator
```
## 核心逻辑流程图

## Resources
1. [[Change Log 原理与实现]]
2. [[Regular Join核心基础类]]
3. [[Regular Join核心逻辑&注意事项]]
4. [[Stream SQL Regular Join Example]]
5. [[Join Use Case]]