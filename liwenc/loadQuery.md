这一部分主要介绍loadQuery()函数

```cpp
_inputQuery = QueryLoader::loadQuery( inputQueryFilePath );
```

loadQuery函数：

- 参数：查询的文件路径
- 返回值：[InputQuery对象](./InputQuery.md)


这一部分的主要内容是初始化inputQuery，看一看inputQuery的主要成员。
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



参考：[InputQuery对象](inputQuery.md)

---

`loadQuery()`主要分为六个部分
- 处理input变量
- 处理output变量
- set lower bounds
- set upper bounds
- 构造并添加等式`inputQuery.addEquation(equation)`
- 构造并添加约束`inputQuery.addPiecewiseLinearConstraint(constraint)`

```cpp
InputQuery QueryLoader::loadQuery( const String &fileName )
{
    if ( !IFile::exists( fileName ) )
        pass

    InputQuery inputQuery;
    AutoFile input( fileName );
    input->open( IFile::MODE_READ );

    unsigned numVars = atoi( input->readLine().trim().ascii() );
    unsigned numLowerBounds = atoi( input->readLine().trim().ascii() );
    unsigned numUpperBounds = atoi( input->readLine().trim().ascii() );
    unsigned numEquations = atoi( input->readLine().trim().ascii() );
    unsigned numConstraints = atoi( input->readLine().trim().ascii() );

    inputQuery.setNumberOfVariables( numVars );

    // Input Variables
    unsigned numInputVars = atoi( input->readLine().trim().ascii() );
    for ( unsigned i = 0; i < numInputVars; ++i )
    {
        String line = input->readLine();
        List<String> tokens = line.tokenize( "," );
        auto it = tokens.begin();
        unsigned inputIndex = atoi( it->ascii() );
        it++;
        unsigned variable = atoi( it->ascii() );
        it++;
        inputQuery.markInputVariable( variable, inputIndex );
        // 这里的markOutputVariable也很重要，各种Index在之后会经常用。
        /**
        void InputQuery::markInputVariable( unsigned variable, unsigned inputIndex )
        {
            _variableToInputIndex[variable] = inputIndex;
            _inputIndexToVariable[inputIndex] = variable;
        }*/
    }

    // Output Variables
    unsigned numOutputVars = atoi( input->readLine().trim().ascii() );
    for ( unsigned i = 0; i < numOutputVars; ++i )
    {
        String line = input->readLine();
        List<String> tokens = line.tokenize( "," );
        auto it = tokens.begin();
        unsigned outputIndex = atoi( it->ascii() );
        it++;
        unsigned variable = atoi( it->ascii() );
        it++;
        inputQuery.markOutputVariable( variable, outputIndex );
    }

    // Lower Bounds
    for ( unsigned i = 0; i < numLowerBounds; ++i )
    {
        String line = input->readLine();
        List<String> tokens = line.tokenize( "," );

        // format: <var, lb>

        auto it = tokens.begin();
        unsigned varToBound = atoi( it->ascii() );
        ++it;

        double lb = atof( it->ascii() );
        ++it;

        inputQuery.setLowerBound( varToBound, lb );
    }

    // Upper Bounds
    for ( unsigned i = 0; i < numUpperBounds; ++i )
    {
        String line = input->readLine();
        List<String> tokens = line.tokenize( "," );

        // format: <var, ub>
        auto it = tokens.begin();
        unsigned varToBound = atoi( it->ascii() );
        ++it;

        double ub = atof( it->ascii() );
        ++it;

        inputQuery.setUpperBound( varToBound, ub );
    }
```
参考[Equations.cpp](Equations.md)
```cpp
    // Equations
    for( unsigned i = 0; i < numEquations; ++i )
    {
        String line = input->readLine();
        List<String> tokens = line.tokenize( "," );
        auto it = tokens.begin();
        // Skip equation number
        ++it;
        int eqType = atoi( it->ascii() );
        ++it;
        double eqScalar = atof( it->ascii() );
        Equation::EquationType type = Equation::EQ;
        switch ( eqType )
        {
        case 0:
            type = Equation::EQ;
            break;

        case 1:
            type = Equation::GE;
            break;

        case 2:
            type = Equation::LE;
            break;

        default:
            // Throw exception
            throw MarabouError();
            break;
        }
        Equation equation( type );
        equation.setScalar( eqScalar );
        while ( ++it != tokens.end() )
        {
            int varNo = atoi( it->ascii() );
            ++it;
            double coeff = atof( it->ascii() );
            equation.addAddend( coeff, varNo );
        }
        inputQuery.addEquation( equation );
    }

    // Constraints
    for ( unsigned i = 0; i < numConstraints; ++i )
    {
        String line = input->readLine();
        List<String> tokens = line.tokenize( "," );
        auto it = tokens.begin();
        // Skip constraint number
        ++it;
        String coType = *it;
        String serializeConstraint;
        // include type in serializeConstraint as well
        while ( it != tokens.end() ) {
            serializeConstraint += *it + String( "," );
            it++;
        }
        serializeConstraint = serializeConstraint.substring( 0, serializeConstraint.length() - 1 );
        PiecewiseLinearConstraint *constraint = NULL;
        // 支持模块化添加不同类型的约束，现在支持relu和max
        if ( coType == "relu" ) {
            constraint = new ReluConstraint( serializeConstraint );
        }
        else if ( coType == "max" ) {
            constraint = new MaxConstraint( serializeConstraint );
        }
        else {
            throw MarabouError;
        }
        inputQuery.addPiecewiseLinearConstraint( constraint );
    }
    return inputQuery;
}
```

继承自`ITableau::VariableWatcher`的`PiecewiseLinearConstraint`类，是各种分段线性约束的基类，从这里继承的子类有：
- ReluConstraint
- DisjuctionConstraint
- MaxConstraint
- AbsoluteValueConstraint

[返回](./index.md)