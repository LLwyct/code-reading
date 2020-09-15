- [1. Marabou 执行流程](#1-marabou-执行流程)
  - [1.1. main.cpp](#11-maincpp)
  - [1.2. Marabou.run()](#12-marabourun)
  - [1.3. prepareInputQuery()](#13-prepareinputquery)
  - [1.4. propertyparser()](#14-propertyparser)
  - [1.5. processSingleLine（）](#15-processsingleline)
  - [1.6. inputQuery](#16-inputquery)
  - [1.7. run.solveQuery()](#17-runsolvequery)
  - [1.8. processInputQuery()](#18-processinputquery)
  - [1.9. invokePreprocesser()](#19-invokepreprocesser)
  - [1.10. createConstraintMatrix()](#110-createconstraintmatrix)
  - [1.11. removeRedundantEquations](#111-removeredundantequations)
  - [1.12. selectInitialVariablesForBasis()](#112-selectinitialvariablesforbasis)
  - [1.13. initializeTableau()](#113-initializetableau)
  - [1.14. engine.solve()](#114-enginesolve)

# 1. Marabou 执行流程

## 1.1. main.cpp
主函数没什么好看的，只调用了 `run()` 函数。
- DNC：分治模式，论文中提到如果开启了这个速度会快，但好像没有细说。
```c++
if ( options->getBool( Options::DNC_MODE ) )
    DnCMarabou().run();
else
    Marabou( options->getInt( Options::VERBOSITY ) ).run();
```

## 1.2. Marabou.run()

没什么好说的就执行了两个函数，接下来重点看这两个函数
- [1.3. prepareInputQuery()](#13-prepareinputquery)
- [1.7. run.solveQuery()](#17-runsolvequery)
```c++
function run () {
    prepareInputQuery();
    solveQuery();
}
```

## 1.3. prepareInputQuery()

准备输入查询

1. 提取`inputQueryFilePath`文件中的查询，这里参考[loadQuery.md](loadQuery.md)，其中涉及到了[inputquery类](./InputQuery.md)
2. 提取`networkFilePath`文件中的网络，这里又分三步
   1. 构造AcasParser, `new AcasParser()`

        用AcasParser去解析神经网络nnet文件。AcasParser中有一个私有成员`AcasNeuralNetwork _acasNeuralNetwork`
        
        而AcasNeuralNetwork有一个私有成员，`AcasNnet *_network`
        
        调用关系如下：AcasParser -> AcasNeuralNetwork -> AcasNnet

        从头顺一遍，顶层代码是在Marabou.cpp中的
        ```cpp
        void Marabou::prepareInputQuery() {

            _acasParser = new AcasParser( networkFilePath );
            
        }
        然后调用AcasParser的构造函数
        AcasParser::AcasParser( const String &path )
            : _acasNeuralNetwork( path )
        {
        }
        然后实则调用AcasNeuralNetwork的构造函数
        AcasNeuralNetwork::AcasNeuralNetwork( const String &path )
             : _network( NULL )
         {
             _network = load_network( path.ascii() );
         }
        最终调用AcasNnet.cpp中的load_network函数，生成network
        AcasNnet *load_network(const char* filename) {
            pass
        }
        ```

        因此这一步其实真正做的是执行`load_network(const char* filename)`。

        [详情看这里AcasNnet.cpp](./AcasNnet.md)
   2. 生成查询, `generateQuery()`

        这一步执行了_acasParser的`generateQuery(_inputQuery)`

        这一步的主要操作是构建了nodeToB和nodeToF,并且生成了初始约束，生成了Equations以及ReLU约束。
        [AcasParser](AcasParser.md)
   3. 构造网络级推理, `constructNetworkLevelReasoner()`

        ```cpp
        _inputQuery.constructNetworkLevelReasoner();
        ```
        [constructNetworkLevelReasoner()](./InputQueryconstructNetworkLevelReasoner.md)
3. 解析`propertyFilePath`文件中的属性, [GOTO](#14-propertyparser)
4. 提取选项
5. 结束，准备`sloveQuery()`，[GOTO](#12-marabourun)
```c++
void Marabou::prepareInputQuery()
{
    String inputQueryFilePath = Options::get()->getString( Options::INPUT_QUERY_FILE_PATH );
    if ( inputQueryFilePath.length() > 0 )
    {
        /*
          Step 1: extract the query
          提取查询
        */
        _inputQuery = QueryLoader::loadQuery( inputQueryFilePath );
    }
    else
    {
        /*
          Step 1: extract the network
          提取网络
        */
        /**
         * 用AcasParser去解析神经网络nnet文件
         * AcasParser的私有成员，AcasNeuralNetwork _acasNeuralNetwork;
         * 而AcasNeuralNetwork有一个私有成员，AcasNnet *_network;
         * 调用关系如下：AcasParser -> AcasNeuralNetwork -> AcasNnet
         */

        // For now, assume the network is given in ACAS format
        _acasParser = new AcasParser( networkFilePath );
        _acasParser->generateQuery( _inputQuery );
        _inputQuery.constructNetworkLevelReasoner();
            

        /*
          Step 2: extract the property in question
        */
        String propertyFilePath = Options::get()->getString( Options::PROPERTY_FILE_PATH );
        if ( propertyFilePath != "" )
        {
            PropertyParser().parse( propertyFilePath, _inputQuery );
        }

        /*
          Step 3: extract options
        */
        _engine.setConstraintViolationThreshold( splitThreshold );
    }

    String queryDumpFilePath = Options::get()->getString( Options::QUERY_DUMP_FILE );
    if ( queryDumpFilePath.length() > 0 )
    {
        _inputQuery.saveQuery( queryDumpFilePath );
        exit( 0 );
    }
}
```
这里的第一步是提取网络
再大概看看第二步提取属性是咋做的,具体就是 `PropertyParser().parse( propertyFilePath, _inputQuery )`这个函数

`propertyparser()`这个类的作用：

>This class reads a property from a text file, and stores the property's constraints within an InputQuery object. Currently, properties can involve the input variables (X's) and the output variables (Y's) of a neural network.

## 1.4. propertyparser()

这一部分的工作主要是根据属性约束去更新变量的上下界
比如在nnet中x0大于等于-1，小于等于1。这里用属性文件的x0大于0.5去更新下界。

```c++
void PropertyParser::parse( const String &propertyFilePath, InputQuery &inputQuery )
{
    File propertyFile( propertyFilePath );
    propertyFile.open( File::MODE_READ );

    try
    {
        while ( true )
        {
            String line = propertyFile.readLine().trim();
            //读取一行，关键代码
            processSingleLine( line, inputQuery );
        }
    }
    catch ( const CommonError &e )
    {
        // A "READ_FAILED" is how we know we're out of lines
        if ( e.getCode() != CommonError::READ_FAILED )
            throw e;
    }
}
```

理解约束怎么转换为输入的关键,代码较长。

 ## 1.5. processSingleLine（）

```c++
void PropertyParser::processSingleLine( const String &line, InputQuery &inputQuery )
{
    // 自己实现的，在Mstring.cpp中，按" "分割
    List<String> tokens = line.tokenize( " " );
    // 转化为token的list，例如x0<=0.5,转化为["x0","<=","0.5"]
    if ( tokens.size() < 3 )
        throw InputParserError( InputParserError::UNEXPECTED_INPUT, line.ascii() );

    auto it = tokens.rbegin();//反向迭代
    if ( !isScalar( *it ) )//如果不是标量（即实数）
    {
        Stringf message( "Right handside must be scalar in the line: %s", line.ascii() );
        throw InputParserError( InputParserError::UNEXPECTED_INPUT, message.ascii() );
    }
    // 获取scalar 0.5
    double scalar = extractScalar( *it );//RHS
    ++it;
    //判断符号类型
    Equation::EquationType type = extractRelationSymbol( *it );
    ++it;

    // Now extract the addends. In the special case where we only have
    // one addend, we add this equation as a bound. Otherwise, we add
    // as an equation.
    if ( tokens.size() == 3 )
    {
        // Special case: add as a bound
        String token = (*it).trim();

        bool inputVariable = token.contains( "x" );
        bool outputVariable = token.contains( "y" );
        bool hiddenVariable = token.contains( "h" );

        // Make sure that we have identified precisely one kind of variable
        unsigned variableKindSanity = 0;
        if ( inputVariable ) ++variableKindSanity;
        if ( outputVariable ) ++variableKindSanity;
        if ( hiddenVariable ) ++variableKindSanity;

        if ( variableKindSanity != 1 )
            throw InputParserError( InputParserError::UNEXPECTED_INPUT, token.ascii() );

        // Determine the index (in input query terms) of the variable whose
        // bound is being set.

        unsigned variable = 0;
        List<String> subTokens;
        if ( inputVariable )
        {
            // 拿出x的下标
            subTokens = token.tokenize( "x" );

            if ( subTokens.size() != 1 )
                throw InputParserError( InputParserError::UNEXPECTED_INPUT, token.ascii() );

            unsigned justIndex = atoi( subTokens.rbegin()->ascii() );//获取下标
			//inputQuery  ???  是啥
            ASSERT( justIndex < inputQuery.getNumInputVariables() );
            variable = inputQuery.inputVariableByIndex( justIndex );
        }
        else if ( outputVariable )
        {
            subTokens = token.tokenize( "y" );

            if ( subTokens.size() != 1 )
                throw InputParserError( InputParserError::UNEXPECTED_INPUT, token.ascii() );

            unsigned justIndex = atoi( subTokens.rbegin()->ascii() );

            ASSERT( justIndex < inputQuery.getNumOutputVariables() );
            variable = inputQuery.outputVariableByIndex( justIndex );
        }
        else if ( hiddenVariable )
        {
            // These variables are of the form h_2_5
            subTokens = token.tokenize( "_" );

            if ( subTokens.size() != 3 )
                throw InputParserError( InputParserError::UNEXPECTED_INPUT, token.ascii() );

            auto subToken = subTokens.begin();
            ++subToken;
            unsigned layerIndex = 2 * atoi( subToken->ascii() ) - 1;
            ++subToken;
            unsigned nodeIndex = atoi( subToken->ascii() );
			//未看，推理有关的
            NLR::NetworkLevelReasoner *nlr = inputQuery.getNetworkLevelReasoner();
            if ( !nlr )
                throw InputParserError( InputParserError::NETWORK_LEVEL_REASONING_DISABLED );

            if ( nlr->getNumberOfLayers() < layerIndex )
                throw InputParserError( InputParserError::HIDDEN_VARIABLE_DOESNT_EXIST_IN_NLR );

            const NLR::Layer *layer = nlr->getLayer( layerIndex );
            if ( layer->getSize() < nodeIndex || !layer->neuronHasVariable( nodeIndex ) )
                throw InputParserError( InputParserError::HIDDEN_VARIABLE_DOESNT_EXIST_IN_NLR );

            variable = layer->neuronToVariable( nodeIndex );
        }

        if ( type == Equation::GE )
        {
            // 如果属性问价的下界大于初始化的下界，那么更新
            if ( inputQuery.getLowerBound( variable ) < scalar )
                inputQuery.setLowerBound( variable, scalar );
        }
        else if ( type == Equation::LE )
        {
            // 如果属性文件的上界大于初始化的上界，那么更新
            if ( inputQuery.getUpperBound( variable ) > scalar )
                inputQuery.setUpperBound( variable, scalar );
        }
        else
        {
            ASSERT( type == Equation::EQ );
            // 如果属性文件是等于号，那么将上下界设为同一个数
            if ( inputQuery.getLowerBound( variable ) < scalar )
                inputQuery.setLowerBound( variable, scalar );
            if ( inputQuery.getUpperBound( variable ) > scalar )
                inputQuery.setUpperBound( variable, scalar );
        }
    }
    else
    {
        // Normal case: add as an equation
        Equation equation( type );
        equation.setScalar( scalar );

        while ( it != tokens.rend() )
        {
            String token = (*it).trim();

            bool inputVariable = token.contains( "x" );
            bool outputVariable = token.contains( "y" );

            if ( !( inputVariable ^ outputVariable ) )
                throw InputParserError( InputParserError::UNEXPECTED_INPUT, token.ascii() );

            List<String> subTokens;
            if ( inputVariable )
                subTokens = token.tokenize( "x" );
            else
                subTokens = token.tokenize( "y" );

            if ( subTokens.size() != 2 )
                throw InputParserError( InputParserError::UNEXPECTED_INPUT, token.ascii() );

            unsigned justIndex = atoi( subTokens.rbegin()->ascii() );
            unsigned variable;

            if ( inputVariable )
            {
                ASSERT( justIndex < inputQuery.getNumInputVariables() );
                variable = inputQuery.inputVariableByIndex( justIndex );
            }
            else
            {
                ASSERT( justIndex < inputQuery.getNumOutputVariables() );
                variable = inputQuery.outputVariableByIndex( justIndex );
            }

            String coefficientString = *subTokens.begin();
            double coefficient;
            if ( coefficientString == "+" )
                coefficient = 1;
            else if ( coefficientString == "-" )
                coefficient = -1;
            else
                coefficient = atof( coefficientString.ascii() );

            equation.addAddend( coefficient, variable );
            ++it;
        }

        inputQuery.addEquation( equation );
    }
}
```



## 1.6. inputQuery

[参考资料](./InputQuery.md)

## 1.7. run.solveQuery()

```c++
void Marabou::solveQuery()
{
    if ( _engine.processInputQuery( _inputQuery ) )
        _engine.solve( Options::get()->getInt( Options::TIMEOUT ) );

    if ( _engine.getExitCode() == Engine::SAT )
        _engine.extractSolution( _inputQuery );
}
```
主要包含两个步骤：
1. `_engine.processInputQuery( _inputQuery )`，[GOTO](#18-processinputquery)
2. `_engine.solve`


## 1.8. processInputQuery()

[调用预处理器](./invokePreprocesser.md)

[添加辅助变量](./makeAllEquationsEqualities.md)

[创建约束矩阵](#110-createconstraintmatrix)

[移除冗余等式](#111-removeredundantequations)

[选择基变量](#112-selectinitialvariablesforbasis)

[初始化单纯形表](#113-initializetableau)

```c++
//Process the input query and pass the needed information to the underlying tableau. Return false if query is found to be infeasible,true otherwise.
bool Engine::processInputQuery( InputQuery &inputQuery, bool preprocess )
{
    ENGINE_LOG( "processInputQuery starting\n" );

    struct timespec start = TimeUtils::sampleMicro();

    try
    {
        //将分段线性约束中的变量边界设定好
        /**
        这一部分的主要工作是给Relu变量notify上下界notifyLowerBound()
        原本的上下界是保存在inputquery中的直系私有属性_lowbounds _upbounds中的。
        这里添加到inputQuery的_plConstrant的lowbounds和upperbounds。
        根据字面意思，这里是添加了一个订阅者
        */
        informConstraintsOfInitialBounds( inputQuery );


        //调用预处理器，预处理器为preprocessor(),见下面1.9
        /**
        invoke这一步主要做了四件事，都很重要
        1. 在makeAllEquationsEqualities()函数中，把InputQuery的Equtions里的全部等式类型转化为EQ类型。
        在我自己给出的例子中，全都是EQ类型，因此不需要转化，全部continue了。
        -V1+V0 = -0
        -V2-V0 = -0
        -V5+V3+V4 = -0

        2.在addPlAuxiliaryEquations()函数中遍历_plconstrant，对于每一个约束添加辅助变量，把所有的ReLU约束转换为等式加入inputQuery的_equtions。

        Relu约束可描述为 _b -> _f,由于ReLU函数的性质，可以轻松知道 f >= b
        通过移项和添加辅助变量可转换为：
        f - b >= 0
        f - b - aux == 0 && aux >= 0
        其中aux即为新的辅助变量，把辅助变量加入变量组，并把等式f - b + aux == 0加入_equtions

        3. 在预处理数据的时候，很多信息都存放在inputQuery中，这里主战场已经来到了Engine上，因此这一步的操作是把inputQuery赋值给Engine的_preprocessedQuery，便于后续操作
        
        4. 返回处理后的InputQuery，_processor
        */
        invokePreprocessor( inputQuery, preprocess );
        if ( _verbosity > 0 )
            printInputBounds( inputQuery );
		//关键，怎么将约束啥的转化为矩阵。见下面
        double *constraintMatrix = createConstraintMatrix();
        //移除冗余等式，见下面
        removeRedundantEquations( constraintMatrix );

        // The equations have changed, recreate the constraint matrix
        delete[] constraintMatrix;
        constraintMatrix = createConstraintMatrix();
		
        List<unsigned> initialBasis;
        List<unsigned> basicRows;
        //没搞明白 ，干啥的？大概是确定初始基变量。
        selectInitialVariablesForBasis( constraintMatrix, initialBasis, basicRows );
        //将等式右边全部变为0，通过添加辅助变量
        addAuxiliaryVariables();
        //不懂
        augmentInitialBasisIfNeeded( initialBasis, basicRows );
		//不懂
        storeEquationsInDegradationChecker();

        // The equations have changed, recreate the constraint matrix
        delete[] constraintMatrix;
        constraintMatrix = createConstraintMatrix();

        initializeNetworkLevelReasoning();
        //重点！ 初始化单纯形表，见 下面
        initializeTableau( constraintMatrix, initialBasis );

        if ( GlobalConfiguration::WARM_START )
            warmStart();

        delete[] constraintMatrix;

        performMILPSolverBoundedTightening();

        struct timespec end = TimeUtils::sampleMicro();
        _statistics.setPreprocessingTime( TimeUtils::timePassed( start, end ) );
    }
    catch ( const InfeasibleQueryException & )
    {
        ENGINE_LOG( "processInputQuery done\n" );

        struct timespec end = TimeUtils::sampleMicro();
        _statistics.setPreprocessingTime( TimeUtils::timePassed( start, end ) );

        _exitCode = Engine::UNSAT;
        return false;
    }

    ENGINE_LOG( "processInputQuery done\n" );

    _smtCore.storeDebuggingSolution( _preprocessedQuery._debuggingSolution );
    return true;
}
```
[back](#18-processinputquery)
## 1.9. invokePreprocesser()
[invokePreprocesser详细资料](./invokePreprocesser.md)

## 1.10. createConstraintMatrix()

```c++
double *Engine::createConstraintMatrix()
{
    const List<Equation> &equations( _preprocessedQuery.getEquations() );
    unsigned m = equations.size();
    unsigned n = _preprocessedQuery.getNumberOfVariables();

    // Step 1: create a constraint matrix from the equations
    //用的是长度为n*m 的一维数组
    double *constraintMatrix = new double[n*m];
    if ( !constraintMatrix )
        throw MarabouError( MarabouError::ALLOCATION_FAILED, "Engine::constraintMatrix" );
    // 初始化为0.0
    std::fill_n( constraintMatrix, n*m, 0.0 );

    unsigned equationIndex = 0;
    for ( const auto &equation : equations )
    {
        if ( equation._type != Equation::EQ )
        {
            _exitCode = Engine::ERROR;
            throw MarabouError( MarabouError::NON_EQUALITY_INPUT_EQUATION_DISCOVERED );
        }

        for ( const auto &addend : equation._addends )
            constraintMatrix[equationIndex*n + addend._variable] = addend._coefficient;
        // 主要是添加addend的系数
        /*
         1  -1
        -1      -1
                     1  -1
        -1       1
                -1          -1
        */
        /*
        std::cout<<"The " << equationIndex << " times construct constraintMatrix:"<<std::endl;
        for (int iii = 0; iii < (int)m; ++iii) {
            for (int jjj = 0; jjj < (int)n; ++jjj) {
                std::cout << constraintMatrix[iii * n + jjj] << " ";
            }
            std::cout<<std::endl;
        }
        */
        ++equationIndex;
    }

    return constraintMatrix;
}
```
[back](#18-processinputquery)

## 1.11. removeRedundantEquations

```c++
void Engine::removeRedundantEquations( const double *constraintMatrix )
{
    const List<Equation> &equations( _preprocessedQuery.getEquations() );
    unsigned m = equations.size();
    unsigned n = _preprocessedQuery.getNumberOfVariables();

    // Step 1: analyze the matrix to identify redundant rows
    //关键代码，详情见ConstraintMatrixAnalyzer.cpp
    AutoConstraintMatrixAnalyzer analyzer;
    analyzer->analyze( constraintMatrix, m, n );

    ENGINE_LOG( Stringf( "Number of redundant rows: %u out of %u",
                  analyzer->getRedundantRows().size(), m ).ascii() );

    // Step 2: remove any equations corresponding to redundant rows
    Set<unsigned> redundantRows = analyzer->getRedundantRows();

    if ( !redundantRows.empty() )
    {
        _preprocessedQuery.removeEquationsByIndex( redundantRows );
        m = equations.size();
    }
}
```
[back](#18-processinputquery)

回到processInputQuery()中的selectInitialVariablesForBasis()

## 1.12. selectInitialVariablesForBasis()

```c++
//This method permutes rows and columns in the constraint matrix (prior to the addition of auxiliary variables), in order to obtain a set of column that constitue a lower triangular matrix. The variables corresponding to the columns of this matrix join the initial basis.(It is possible that not enough variables are obtained this way, in which case the initial basis will have to be augmented later).
//此方法对约束矩阵中的行和列进行置换（在添加辅助变量之前），以便获得构成下三角矩阵的一组列。 与该矩阵的列相对应的变量将加入初始基础（可能无法通过这种方式获得足够多的变量，在这种情况下，初始基础将不得不在以后增加）。
void Engine::selectInitialVariablesForBasis( const double *constraintMatrix, List<unsigned> &initialBasis, List<unsigned> &basicRows )
{
 	const List<Equation> &equations( _preprocessedQuery.getEquations() );

    unsigned m = equations.size();
    unsigned n = _preprocessedQuery.getNumberOfVariables();

    // Trivial case, or if a trivial basis is requested
    if ( ( m == 0 ) || ( n == 0 ) || GlobalConfiguration::ONLY_AUX_INITIAL_BASIS )
    {
        for ( unsigned i = 0; i < m; ++i )
            basicRows.append( i );

        return;
    }

    unsigned *nnzInRow = new unsigned[m];
    unsigned *nnzInColumn = new unsigned[n];

    std::fill_n( nnzInRow, m, 0 );
    std::fill_n( nnzInColumn, n, 0 );

    unsigned *columnOrdering = new unsigned[n];
    unsigned *rowOrdering = new unsigned[m];

    for ( unsigned i = 0; i < m; ++i )
        rowOrdering[i] = i;

    for ( unsigned i = 0; i < n; ++i )
        columnOrdering[i] = i;

    // Initialize the counters
    for ( unsigned i = 0; i < m; ++i )
    {
        for ( unsigned j = 0; j < n; ++j )
        {
            if ( !FloatUtils::isZero( constraintMatrix[i*n + j] ) )
            {
                ++nnzInRow[i];
                ++nnzInColumn[j];
            }
        }
    }

    DEBUG({
            for ( unsigned i = 0; i < m; ++i )
            {
                ASSERT( nnzInRow[i] > 0 );
            }
        });

    unsigned numExcluded = 0;
    unsigned numTriangularRows = 0;
    unsigned temp;

    while ( numExcluded + numTriangularRows < n )
    {
        // Do we have a singleton row?
        unsigned singletonRow = m;
        for ( unsigned i = numTriangularRows; i < m; ++i )
        {
            if ( nnzInRow[i] == 1 )
            {
                singletonRow = i;
                break;
            }
        }

        if ( singletonRow < m )
        {
            // Have a singleton row! Swap it to the top and update counters
            temp = rowOrdering[singletonRow];
            rowOrdering[singletonRow] = rowOrdering[numTriangularRows];
            rowOrdering[numTriangularRows] = temp;

            temp = nnzInRow[numTriangularRows];
            nnzInRow[numTriangularRows] = nnzInRow[singletonRow];
            nnzInRow[singletonRow] = temp;

            // Find the non-zero entry in the row and swap it to the diagonal
            DEBUG( bool foundNonZero = false );
            for ( unsigned i = numTriangularRows; i < n - numExcluded; ++i )
            {
                if ( !FloatUtils::isZero( constraintMatrix[rowOrdering[numTriangularRows] * n + columnOrdering[i]] ) )
                {
                    temp = columnOrdering[i];
                    columnOrdering[i] = columnOrdering[numTriangularRows];
                    columnOrdering[numTriangularRows] = temp;

                    temp = nnzInColumn[numTriangularRows];
                    nnzInColumn[numTriangularRows] = nnzInColumn[i];
                    nnzInColumn[i] = temp;

                    DEBUG( foundNonZero = true );
                    break;
                }
            }

            ASSERT( foundNonZero );

            // Remove all entries under the diagonal entry from the row counters
            for ( unsigned i = numTriangularRows + 1; i < m; ++i )
            {
                if ( !FloatUtils::isZero( constraintMatrix[rowOrdering[i] * n + columnOrdering[numTriangularRows]] ) )
                    --nnzInRow[i];
            }

            ++numTriangularRows;
        }
        else
        {
            // No singleton rows. Exclude the densest column
            unsigned maxDensity = nnzInColumn[numTriangularRows];
            unsigned column = numTriangularRows;

            for ( unsigned i = numTriangularRows; i < n - numExcluded; ++i )
            {
                if ( nnzInColumn[i] > maxDensity )
                {
                    maxDensity = nnzInColumn[i];
                    column = i;
                }
            }

            // Update the row counters to account for the excluded column
            for ( unsigned i = numTriangularRows; i < m; ++i )
            {
                double element = constraintMatrix[rowOrdering[i]*n + columnOrdering[column]];
                if ( !FloatUtils::isZero( element ) )
                {
                    ASSERT( nnzInRow[i] > 1 );
                    --nnzInRow[i];
                }
            }

            columnOrdering[column] = columnOrdering[n - 1 - numExcluded];
            nnzInColumn[column] = nnzInColumn[n - 1 - numExcluded];
            ++numExcluded;
        }
    }

    // Final basis: diagonalized columns + non-diagonalized rows
    List<unsigned> result;

    for ( unsigned i = 0; i < numTriangularRows; ++i )
    {
        initialBasis.append( columnOrdering[i] );
    }

    for ( unsigned i = numTriangularRows; i < m; ++i )
    {
        basicRows.append( rowOrdering[i] );
    }

    // Cleanup
    delete[] nnzInRow;
    delete[] nnzInColumn;
    delete[] columnOrdering;
    delete[] rowOrdering;
}

```
[back](#18-processinputquery)

## 1.13. initializeTableau()

和tableua.cpp连接起来

```c++
void Engine::initializeTableau( const double *constraintMatrix, const List<unsigned> &initialBasis )
{
    const List<Equation> &equations( _preprocessedQuery.getEquations() );
    unsigned m = equations.size();
    unsigned n = _preprocessedQuery.getNumberOfVariables();
	//定义一个CSR  
     //见 tableua.cpp
    _tableau->setDimensions( m, n );

    adjustWorkMemorySize();

    unsigned equationIndex = 0;
    for ( const auto &equation : equations )
    {
        _tableau->setRightHandSide( equationIndex, equation._scalar );
        ++equationIndex;
    }

    // Populate constriant matrix
    //见 tableua.cpp
    //转换为csr
    _tableau->setConstraintMatrix( constraintMatrix );

    for ( unsigned i = 0; i < n; ++i )
    {
        _tableau->setLowerBound( i, _preprocessedQuery.getLowerBound( i ) );
        _tableau->setUpperBound( i, _preprocessedQuery.getUpperBound( i ) );
    }
	//没看，边界缩紧有关。
    _tableau->registerToWatchAllVariables( _rowBoundTightener );
    _tableau->registerResizeWatcher( _rowBoundTightener );

    _tableau->registerToWatchAllVariables( _constraintBoundTightener );
    _tableau->registerResizeWatcher( _constraintBoundTightener );

    _rowBoundTightener->setDimensions();
    _constraintBoundTightener->setDimensions();

    // Register the constraint bound tightener to all the PL constraints
    for ( auto &plConstraint : _preprocessedQuery.getPiecewiseLinearConstraints() )
        plConstraint->registerConstraintBoundTightener( _constraintBoundTightener );

    _plConstraints = _preprocessedQuery.getPiecewiseLinearConstraints();
    for ( const auto &constraint : _plConstraints )
    {
        constraint->registerAsWatcher( _tableau );
        constraint->setStatistics( &_statistics );
    }
	//初始化单纯形表中的基变量
    _tableau->initializeTableau( initialBasis );

    _costFunctionManager->initialize();
    _tableau->registerCostFunctionManager( _costFunctionManager );
    _activeEntryStrategy->initialize( _tableau );

    _statistics.setNumPlConstraints( _plConstraints.size() );
}
```
[back](#18-processinputquery)

接下来，就该

## 1.14. engine.solve()

另开一篇
