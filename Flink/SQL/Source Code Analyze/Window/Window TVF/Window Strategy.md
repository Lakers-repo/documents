## 继承关系
![[Window Strategy.png]]

## Example
`FlinkStreamRuleSets`
```Java
/** RuleSet to do physical optimize for stream */  
val PHYSICAL_OPT_RULES: RuleSet = RuleSets.ofList(  
  // window TVFs  
  StreamPhysicalWindowTableFunctionRule.INSTANCE,  
  StreamPhysicalWindowAggregateRule.INSTANCE,  
  PullUpWindowTableFunctionIntoWindowAggregateRule.INSTANCE,  
  ExpandWindowTableFunctionTransposeRule.INSTANCE,  
  StreamPhysicalWindowRankRule.INSTANCE,  
  StreamPhysicalWindowDeduplicateRule.INSTANCE,  
)
```

- Window TVF Aggregate
```Java
StreamPhysicalWindowTableFunctionRule -> new StreamPhysicalWindowTableFunction() -> TimeAttributeWindowingStrategy

StreamPhysicalWindowAggregateRule -> new StreamPhysicalWindowAggregate() -> WindowAttachedWindowingStrategy

PullUpWindowTableFunctionIntoWindowAggregateRule -> new StreamPhysicalWindowAggregate() -> TimeAttributeWindowingStrategy
```
