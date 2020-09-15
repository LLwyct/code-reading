```cpp
bool InputQuery::constructNetworkLevelReasoner()
{
    INPUT_QUERY_LOG( "PP: constructing an NLR... " );

    if ( _networkLevelReasoner )
        delete _networkLevelReasoner;
    NLR::NetworkLevelReasoner *nlr = new NLR::NetworkLevelReasoner;

    Map<unsigned, unsigned> handledVariableToLayer;

    // First, put all the input neurons in layer 0
    List<unsigned> inputs = getInputVariables();
    nlr->addLayer( 0, NLR::Layer::INPUT, inputs.size() );
    unsigned index = 0;
    for ( const auto &inputVariable : inputs )
    {
        nlr->setNeuronVariable( NLR::NeuronIndex( 0, index ), inputVariable );
        handledVariableToLayer[inputVariable] = 0;
        ++index;
    }

    unsigned newLayerIndex = 1;
    // Now, repeatedly attempt to construct addditional layers
    while ( constructWeighedSumLayer( nlr, handledVariableToLayer, newLayerIndex ) ||
            constructReluLayer( nlr, handledVariableToLayer, newLayerIndex ) ||
            constructAbsoluteValueLayer( nlr, handledVariableToLayer, newLayerIndex ) ||
            constructSignLayer( nlr, handledVariableToLayer, newLayerIndex ) )
    {
        ++newLayerIndex;
    }

    bool success = ( newLayerIndex > 1 );

    if ( success )
    {
        unsigned count = 0;
        for ( unsigned i = 0; i < nlr->getNumberOfLayers(); ++i )
            count += nlr->getLayer( i )->getSize();

        INPUT_QUERY_LOG( Stringf( "successful. Constructed %u layers with %u neurons (out of %u)\n",
                      newLayerIndex,
                      count,
                      getNumberOfVariables() ).ascii() );

        把局部变量nlr的地址赋值给inputQuery的私有成员_networkLevelReasoner
        _networkLevelReasoner = nlr;
    }
    else
    {
        INPUT_QUERY_LOG( "unsuccessful\n" );
        delete nlr;
    }

    return success;
}
```