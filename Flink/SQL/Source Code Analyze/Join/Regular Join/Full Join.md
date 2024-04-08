### Logic & Physical Plan

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
StreamExecJoin-->|Inner or Outer Join|StreamingJoinOperator
```
## 核心逻辑流程图

## Resources
1. [[Change Log 原理与实现]]
2. [[Regular Join核心基础类]]
3. [[Regular Join核心逻辑&注意事项]]
4. [[Stream SQL Regular Join Example]]
5. [[Join Use Case]]