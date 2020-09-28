## 1.9. preprocessor()

invoke这一步主要做了四件事，都很重要
1. 在makeAllEquationsEqualities()函数中，把InputQuery的Equtions里的全部等式类型转化为EQ类型。
在我自己给出的例子中，全都是EQ类型，因此不需要转化，全部continue了。
-V1+V0 = -0
-V2-V0 = -0
-V5+V3+V4 = -0

2. 在addPlAuxiliaryEquations()函数中遍历_plconstrant，对于每一个约束添加辅助变量，把所有的ReLU约束转换为等式加入inputQuery的_equtions。

    在这里，由于有两个ReLU函数，因此_equtions的长度从3增加到5

    Relu约束可描述为 _b -> _f,由于ReLU函数的性质，可以轻松知道 f >= b
    通过移项和添加辅助变量可转换为：
    f - b >= 0
    f - b - aux == 0 && aux >= 0
    其中aux即为新的辅助变量，把辅助变量加入变量组，并把等式f - b + aux == 0加入_equtions

3. 甚至还消除了冗余变量？
   
   ```cpp
   if ( attemptVariableElimination )
        eliminateVariables();
   ```
   
   这里存疑一下，

4. 在预处理数据的时候，很多信息都存放在inputQuery中，这里主战场已经来到了Engine上，因此这一步的操作是把inputQuery赋值给Engine的_preprocessedQuery，便于后续操作

5. 返回处理后的InputQuery，_processor

---

invokePreprocesser函数主要执行了下面这个函数


`
_preprocessedQuery = _preprocessor.preprocess(inputQuery, 一个Golbal参数);
`

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
        在这里，由于有两个ReLU函数，因此_equtions的长度从3增加到5
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

    /**
    收集迄今为止所有使用过的变量到一个set<unsigned> usedVariables里，
    但似乎最终目的不是为了这个，而是为了得到所有上界等于下界的变量，并储存到
    Map<unsigned, double> _fixedVariables中

    最终目的是：所有的FixedValue和MergeValue都要被消除，这一步是为了收集Fixed和Merged变量。
    */
    collectFixedValues();
    separateMergedAndFixed();

    /**
    尝试消除Fixed和Merged变量
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

### 函数eliminateVariables()
---

目的是找到，属于Fixed或Merge变量，但是不属于输入输出变量的变量，要把这些变量消除

目前已经看到的，从NLR中的nerouToVariables消除，从_equtions中消除。

如果一个eqution的全部addend被消除，那么这个eqution也会被消除。

让分段线性约束知道任何已消除的变量，如果约束过时，则删除它们本身。

---

依我目前的理解，就是消除上下界都为0的变量（或是上下界统一的变量？这样可以视为一个常数），在简单例子中，经过推算，x4和x6是上下界都为0的变量，因此要消除，因此x5和x7要向前推成x4和x5（从x0开始计算）。


举例：原_eqution[2]中，存在
$$
-x5 + x3 + x4 = -0
$$
移项
$$
-x5 + x3 = -0 - x4
$$
因为x4等于0
$$
-x5 + x3 = -0
$$
x5下标向前推1
$$
-x4 + x3 = -0
$$

```cpp
while ( equation != equations.end() )
{
     // Each equation is of the form sum(addends) = scalar. So, any fixed variable
     // needs to be subtracted from the scalar. Merged variables should have already
     // been removed, so we don't care about them
     List<Equation::Addend>::iterator addend = equation->_addends.begin();
     while ( addend != equation->_addends.end() )
     {
         ASSERT( !_mergedVariables.exists( addend->_variable ) );

          // 该变量需要消除，移项、乘系数、改scalar
         if ( _fixedVariables.exists( addend->_variable ) )
         {
             // Addend has to go...
             double constant = _fixedVariables.at( addend->_variable ) * addend->_coefficient;
             equation->_scalar -= constant;
             addend = equation->_addends.erase( addend );
         }
         // 不需要消除，更新下标
         else
         {
             // Adjust the addend's variable index
             addend->_variable = _oldIndexToNewIndex.at( addend->_variable );
             ++addend;
         }
     }

     // If all the addends have been removed, we remove the entire equation.
     // Overwise, we are done here.
     // 如果一个等式的addends全部被删，则删除该等式。
     if ( equation->_addends.empty() )
     {
         if ( _statistics )
             _statistics->ppIncNumEquationsRemoved();

         // No addends left, scalar should be 0
         if ( !FloatUtils::isZero( equation->_scalar ) )
             throw InfeasibleQueryException();
         else
             equation = equations.erase( equation );
     }
     else
         ++equation;
}
```