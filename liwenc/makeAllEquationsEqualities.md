## 1.9. makeAllEquationsEqualities():

```c++
//思路同单纯性转换为标准型，添加辅助变量。
void Preprocessor::makeAllEquationsEqualities()
{
    for ( auto &equation : _preprocessed.getEquations() )
    {
        // 如果是等于类型则忽略
        if ( equation._type == Equation::EQ )
            continue;

        // 在现有变量数量的基础上，新变量的下标就是现有变量数量（因为从零开始的）
        unsigned auxVariable = _preprocessed.getNumberOfVariables();
        _preprocessed.setNumberOfVariables( auxVariable + 1 );

        // Auxiliary variables are always added with coefficient 1
        // 如果是大于等于，就设置上界为0，反之亦然
        if ( equation._type == Equation::GE )
            _preprocessed.setUpperBound( auxVariable, 0 );
        else
            _preprocessed.setLowerBound( auxVariable, 0 );

        equation._type = Equation::EQ;

        equation.addAddend( 1, auxVariable );
    }
}
```
[back invokePreprocesser](./invokePreprocesser.md)