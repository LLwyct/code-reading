<font size=3>
[返回index](./index.md)

这一步主要是生成查询，`generateQuery()`

# AcasParser中的的重要成员

AcasParser.h
```cpp
private:
    AcasNeuralNetwork _acasNeuralNetwork;
    Map<NodeIndex, unsigned> _nodeToB;
    Map<NodeIndex, unsigned> _nodeToF;

以及定义了一个结构体
struct NodeIndex
{
    public:
        NodeIndex( unsigned layer, unsigned node );
        bool operator<( const NodeIndex &other ) const;
        bool operator==( const NodeIndex &other ) const
        {
            return _layer == other._layer && _node == other._node;
        }

        unsigned _layer;
        unsigned _node;
};
```

# 重要函数generateQuery

```cpp
void AcasParser::generateQuery( InputQuery &inputQuery )
{
    // First encode the actual network
    // _acasNeuralNetwork doesn't count the input layer, so add 1
    unsigned numberOfLayers = _acasNeuralNetwork.getNumLayers() + 1;
    unsigned inputLayerSize = _acasNeuralNetwork.getLayerSize( 0 );
    unsigned outputLayerSize = _acasNeuralNetwork.getLayerSize( numberOfLayers - 1 );

    unsigned numberOfInternalNodes = 0;
    // 获取隐藏层全部节点数，因为是全连接ReLU，获取隐藏层节点数，就是额外生成节点数，他这里默认隐藏层节点数就是ReLU节点数
    for ( unsigned i = 1; i < numberOfLayers - 1; ++i )
        numberOfInternalNodes += _acasNeuralNetwork.getLayerSize( i );

    printf( "Number of layers: %u. Input layer size: %u. Output layer size: %u. Number of ReLUs: %u\n",
            numberOfLayers, inputLayerSize, outputLayerSize, numberOfInternalNodes );

    // The total number of variables required for the encoding is computed as follows:
    //   1. Each input node appears once
    //   2. Each internal node has a B variable and an F variable
    //   3. Each output node appears once
    // 这里结算要被编码的总节点数，输入输出层计算一次，隐藏层的每个relu节点拆成bf，因此要乘二，在tiny2中共有5+5+50*2=110个节点
    unsigned numberOfVariables = inputLayerSize + ( 2 * numberOfInternalNodes ) + outputLayerSize;
    printf( "Total number of variables: %u\n", numberOfVariables );

    inputQuery.setNumberOfVariables( numberOfVariables );

    // Next, we want to map each node to its corresponding
    // variables. We group variables according to this order: f's from
    // layer i, b's from layer i+1, and repeat.

    
    /**
     * 很重要的一步，提供所有变量到索引的映射，填充 map<nodeIndex, unsigned> _nodeToB, _nodeToF;
     * 其中nodeIndex为 <layer, index>，说人话就是第layer层的第index节点(没被拆节点之前的layerNum)
     * 对于输入层，默认为F节点，对于输出层默认为B节点，对于隐藏层，每个节点即属于B又属于F(因为被拆开了)
     * ReLU(b) = f
     * _nodeToB(<1, 0>) --Relu--> _nodeToF(<1,0>)
     */
     对于最小网络 1 2 1
     F  B    F  B
        o -> o
     o          o
        o -> o


     F:
        (0,0):0
     B:
        (1,0):1
        (1,1):2
     F:
        (1,0):3
        (1,1):4
     B:
        (2,0):5
    一共算上externalnode有6个节点，根据提供的每一个<layer, index>在B或F下，都能返回对应的索引，没毛病

    unsigned currentIndex = 0;
    // numberOfLayers层数，在tiny2中为3
    for ( unsigned i = 1; i < numberOfLayers; ++i )
    {
        unsigned previousLayerSize = _acasNeuralNetwork.getLayerSize( i - 1 );
        unsigned currentLayerSize = _acasNeuralNetwork.getLayerSize( i );

        // First add the F variables from layer i-1
        for ( unsigned j = 0; j < previousLayerSize; ++j )
        {
            _nodeToF[NodeIndex( i - 1, j )] = currentIndex;
            ++currentIndex;
        }

        // Now add the B variables from layer i
        for ( unsigned j = 0; j < currentLayerSize; ++j )
        {
            _nodeToB[NodeIndex( i, j )] = currentIndex;
            ++currentIndex;
        }
    }

    // Now we set the variable bounds. Input bounds are
    // given as part of the network. B variables are
    // unbounded, and F variables are non-negative.
    // 添加默认边界，比如F一定要大于等于0
    for ( unsigned i = 0; i < inputLayerSize; ++i )
    {
        double min, max;
        _acasNeuralNetwork.getInputRange( i, min, max );

        inputQuery.setLowerBound( _nodeToF[NodeIndex(0, i)], min );
        inputQuery.setUpperBound( _nodeToF[NodeIndex(0, i)], max );
    }

    for ( const auto &fNode : _nodeToF )
    {
        // Be careful not to override the bounds for the input layer
        if ( fNode.first._layer != 0 )
        {
            inputQuery.setLowerBound( fNode.second, 0.0 );
            inputQuery.setUpperBound( fNode.second, FloatUtils::infinity() );
        }
    }

    for ( const auto &fNode : _nodeToB )
    {
        inputQuery.setLowerBound( fNode.second, FloatUtils::negativeInfinity() );
        inputQuery.setUpperBound( fNode.second, FloatUtils::infinity() );
    }

    // Next come the actual equations
    // 在最小样例(121)中numberOfLayers为3
    这里是给所有的B节点添加equation，在例子121中需要添加的节点有1，2，5
    对于每一个目标节点，系数必为-1，scala必为bias的取反，详情如下sum - b + fs = -bias
    -V1 + V0 = -0
    -V2 - V1 = -0
    -V5 + V3 + V4 = -0
    目前能看懂，符合预期
    for ( unsigned layer = 0; layer < numberOfLayers - 1; ++layer )
    {
        unsigned targetLayerSize = _acasNeuralNetwork.getLayerSize( layer + 1 );
        for ( unsigned target = 0; target < targetLayerSize; ++target )
        {
            // This will represent the equation:
            // sum - b + fs = -bias
            Equation equation;

            // The b variable
            unsigned bVar = _nodeToB[NodeIndex(layer + 1, target)];
            equation.addAddend( -1.0, bVar );

            // The f variables from the previous layer
            for ( unsigned source = 0; source < _acasNeuralNetwork.getLayerSize( layer ); ++source )
            {
                unsigned fVar = _nodeToF[NodeIndex(layer, source)];
                equation.addAddend( _acasNeuralNetwork.getWeight( layer, source, target ), fVar );
            }

            // The bias
            equation.setScalar( -_acasNeuralNetwork.getBias( layer + 1, target ) );

            // Add the equation to the input query
            inputQuery.addEquation( equation );
        }
    }

    // Add the ReLU constraints
    for ( unsigned i = 1; i < numberOfLayers - 1; ++i )
    {
        unsigned currentLayerSize = _acasNeuralNetwork.getLayerSize( i );

        for ( unsigned j = 0; j < currentLayerSize; ++j )
        {
            unsigned b = _nodeToB[NodeIndex(i, j)];
            unsigned f = _nodeToF[NodeIndex(i, j)];
            /**
             * 很重要的类，未细看
             */
            PiecewiseLinearConstraint *relu = new ReluConstraint( b, f );
                
            // 添加到inputQuery的_plConstrants里
            inputQuery.addPiecewiseLinearConstraint( relu );
        }
    }

    // Mark the input and output variables
    再次标记input、output变量，便于快速访问？不知道这一步的目的是什么，已经有了nodeToF和nodeToB了。
    这里是<nodeindex, index>,index是重新从0排序的，nodeindex是上面1-5的unsigned，相当于如果给出变量的index，能直接给出变量的顺序，比如在121模型中，给出5直接返回0
    for ( unsigned i = 0; i < inputLayerSize; ++i )
        inputQuery.markInputVariable( _nodeToF[NodeIndex( 0, i )], i );
    
    for ( unsigned i = 0; i < outputLayerSize; ++i )
        inputQuery.markOutputVariable( _nodeToB[NodeIndex( numberOfLayers - 1, i )], i );
}
```