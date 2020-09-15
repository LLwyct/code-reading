- [上一级：queryLoader](loadQuery.md)
- [上一级：index](index.md)

---
inputQuery是定义在Marabou.h中的私有成员。

# 重要成员
```cpp
private:
    unsigned _numberOfVariables;
    List<Equation> _equations;
    Map<unsigned, double> _lowerBounds;
    Map<unsigned, double> _upperBounds;
    List<PiecewiseLinearConstraint *> _plConstraints;
    Map<unsigned, double> _solution;
public:
    Map<unsigned, unsigned> _variableToInputIndex;
    Map<unsigned, unsigned> _inputIndexToVariable;
    Map<unsigned, unsigned> _variableToOutputIndex;
    Map<unsigned, unsigned> _outputIndexToVariable;
    /**
     * 知道要检查的网络拓扑的对象，可以用于各种操作，例如对基于拓扑的边界缩紧进行网络评估。
     */
    NLR::NetworkLevelReasoner *_networkLevelReasoner;
```
