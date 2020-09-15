## 1.9. preprocessor()

invoke这一步主要做了四件事，都很重要
1. 在makeAllEquationsEqualities()函数中，把InputQuery的Equtions里的全部等式类型转化为EQ类型。
在我自己给出的例子中，全都是EQ类型，因此不需要转化，全部continue了。
-V1+V0 = -0
-V2-V0 = -0
-V5+V3+V4 = -0

2. 在addPlAuxiliaryEquations()函数中遍历_plconstrant，对于每一个约束添加辅助变量，把所有的ReLU约束转换为等式加入inputQuery的_equtions。

Relu约束可描述为 _b -> _f,由于ReLU函数的性质，可以轻松知道 f >= b
通过移项和添加辅助变量可转换为：
f - b >= 0
f - b - aux == 0 && aux >= 0
其中aux即为新的辅助变量，把辅助变量加入变量组，并把等式f - b + aux == 0加入_equtions

3. 在预处理数据的时候，很多信息都存放在inputQuery中，这里主战场已经来到了Engine上，因此这一步的操作是把inputQuery赋值给Engine的_preprocessedQuery，便于后续操作

4. 返回处理后的InputQuery，_processor

---

invokePreprocesser函数主要执行了下面这个函数


```cpp

_preprocessedQuery = _preprocessor.preprocess(inputQuery, GlobalConfiguration::PREPROCESSOR_ELIMINATE_VARIABLES);
```

```cpp
InputQuery Preprocessor::preprocess( const InputQuery &query, bool attemptVariableElimination )
{
    _preprocessed = query;

    /*
      Next, make sure all equations are of type EQUALITY. If not, turn them
      into one.
    */
    //见下面。
    /**
    上面提到的第一步，将所有等式类型化为EQ类型

    1. 在makeAllEquationsEqualities()函数中，把InputQuery的Equtions里的全部等式类型转化为EQ类型。
        在我自己给出的例子中，全都是EQ类型，因此不需要转化，全部continue了。
        -V1+V0 = -0
        -V2-V0 = -0
        -V5+V3+V4 = -0
    */
    makeAllEquationsEqualities();
```
[makeAllEquationsEqualities详细资料](./makeAllEquationsEqualities.md)
```cpp
    /*
      Attempt to construct a network level reasonor
    */
    _preprocessed.constructNetworkLevelReasoner();

    /*
      Collect input and output variables
    */
    for ( const auto &var : _preprocessed.getInputVariables() )
        _inputOutputVariables.insert( var );
    for ( const auto &var : _preprocessed.getOutputVariables() )
        _inputOutputVariables.insert( var );

    /*
      Initial work: if needed, have the PL constraints add their additional
      equations to the pool.
    */

    /**
    把ReLU约束转化为EQ等式并添加辅助变量：
    2.在addPlAuxiliaryEquations()函数中遍历_plconstrant，对于每一个约束添加辅助变量，把所有的ReLU约束转换为等式加入inputQuery的_equtions。

        Relu约束可描述为 _b -> _f,由于ReLU函数的性质，可以轻松知道 f >= b
        通过移项和添加辅助变量可转换为：
        f - b >= 0
        f - b - aux == 0 && aux >= 0
        其中aux即为新的辅助变量，把辅助变量加入变量组，并把等式f - b + aux == 0加入_equtions
    */
    if ( GlobalConfiguration::PREPROCESSOR_PL_CONSTRAINTS_ADD_AUX_EQUATIONS )
        addPlAuxiliaryEquations();

    /*
      Do the preprocessing steps:

      Until saturation:
        1. Tighten bounds using equations
        2. Tighten bounds using pl constraints

      Then, eliminate fixed variables.
    */

    /**
    1. 进行边界束紧
    比如在121的例子里，有-X1 + X0 = 0,并且我们的property_try.txt文件指明，x0属于[0.5, 1],那么可借此把
    X1的上下界也约束在[0.5, 1]里
    */
    bool continueTightening = true;
    while ( continueTightening )
    {
        /**
        很长，但是很重要，而且绝大部分不难看懂，关于利用Equation进行边界束紧
        */
        continueTightening = processEquations(); 
        /**
        很短，和processEquations有一些类似，关于利用Constraint进行边界束紧
        */
        continueTightening = processConstraints() || continueTightening;
        if ( attemptVariableElimination )
            continueTightening = processIdenticalVariables() || continueTightening;

        if ( _statistics )
            _statistics->ppIncNumTighteningIterations();
    }

    collectFixedValues();
    separateMergedAndFixed();

    /**
    尝试消除变量
    */
    if ( attemptVariableElimination )
        eliminateVariables();

    /**
    1. 返回处理后的InputQuery
    */
    return _preprocessed;
}
```
[进入processEquations().md](./processEquations.md)

[返回index.md](./index.md)