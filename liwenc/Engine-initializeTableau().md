```cpp
initializeTableau( const double *constraintMatrix, const List<unsigned> &initialBasis )
{
	1、初始化_tableau，设置m、n的值，定义属性
	   	_tableau->setDimensions( m, n );

	2、清除_work记忆,清空_work
	  	adjustWorkMemorySize();

	3、方程组等式右边（b向量）设为零	
	   	unsigned equationIndex = 0;
	   	for ( const auto &equation : equations )
    	   {
        	   _tableau->setRightHandSide( equationIndex, equation._scalar );
        	   ++equationIndex;
    	   }
	
	4、记录矩阵的非零值
	   	_tableau->setConstraintMatrix( constraintMatrix );
	   例：
	   对矩阵
	   2 0 3 
	   5 6 0 
	   0 2 1 
	   按行保存到_sparseRowsOfA数组中
		_sparseRowsOfA[0]:Entry((0,2)),Entry((2,3))
		_sparseRowsOfA[1]:Entry((0,5)),Entry((1,6))
		_sparseRowsOfA[2]:Entry((1,1)),Entry((2,1))
	   按列保存到_sparseColumnsOfA数组中
		_sparseColumnsOfA[0]:Entry((0,2)),Entry((1,5))
		_sparseColumnsOfA[1]:Entry((1,6)),Entry((2,2))
		_sparseColumnsOfA[2]:Entry((0,3)),Entry((2,1))

	5、保存watcher:_rowBoundTightener，_constraintBoundTightener到_globalWatchers列表
	   保存watcher:_rowBoundTightener，_constraintBoundTightener到_resizeWatchers列表
	   	_tableau->registerToWatchAllVariables( _rowBoundTightener ); 
		_tableau->registerResizeWatcher( _rowBoundTightener );
	 	_tableau->registerToWatchAllVariables( _constraintBoundTightener );
	   	_tableau->registerResizeWatcher( _constraintBoundTightener );
	
	6、初始化_rowBoundTightener,_constraintBoundTightener,设置m,n的值，定义属性
	   	_rowBoundTightener->setDimensions();
    	_constraintBoundTightener->setDimensions();

	7、_constraintBoundTightener赋值给plConstraint的_constraintBoundTightener属性
	   for ( auto &plConstraint : _preprocessedQuery.getPiecewiseLinearConstraints() )
	      plConstraint->registerConstraintBoundTightener( _constraintBoundTightener );

	8、对每个变量，记录和它有关的约束，可以通过变量查看和它有关的约束，约束是一个watcher
	   _statistics赋值给每个约束的_statistics属性，_statistics保存运行过程的一些数据，如运行时间，迭代次数
	   _plConstraints = _preprocessedQuery.getPiecewiseLinearConstraints();
    	   for ( const auto &constraint : _plConstraints )
    	   {
	       constraint->registerAsWatcher( _tableau );  
	       constraint->setStatistics( &_statistics );
    	   }
	
	9、计算基本解
	   _tableau->initializeTableau( initialBasis )
	
	  设网络对应的矩阵为A，对A进行行变换，列变换，计算基本解
	   (1)、非基向量对应变量的值设置为定值，设为它的最小值，将他们移到方程组右边，改变b向量的值;(此时，矩阵包含基向量，b向量)
	   (2)、使用simplex算法，对基向量组成的矩阵B选择枢轴元素;
	        循环[1]、[2]，直到找到所有的枢轴元素
	        [1]、选择枢轴元素，3个选择依据
		  a、找只有一个元素的行，若存在，这个元素作为枢轴元素，不存在，进行下一种选择依据
		  b、找只有一个元素的列，若存在，这个元素作为枢轴元素，不存在，进行下一种选择依据
		  c、Markowitz规则，
	             寻找min(cost) = min( _numURowElements[uRow] - 1 ) * ( _numUColumnElements[uColumn] - 1 )
		     花费相同时，取绝对值大的元素

		     cost计算示例:
		     矩阵
		     2 0 3
	   	     5 6 3
	   	     0 2 1
		     对1行1列的元素6,_numURowElements[uRow]-1=1 , _numUColumnElements[uColumn]-1=1,所以cost=1*1=1
	        [2]、枢轴变换
		   设选择i行j列元素作为枢轴元素，对除第i行的所有行进行枢轴变换，假设对第k行进行变换
		   Bk = Bk-(Ai/Aij)*Bkj
		   Bk表示第k行的元素，
		   Bi表示第i行的元素，
		   Bij表示第i行,第j列的元素，	
		   Bkj表示第k行,第j列的元素，
		 
		   枢轴变换示例：
		   矩阵
		   2 0 3 
	   	   5 6 3 
	   	   0 2 1 
		   i=1,j=1,Bij=6;
		   k=2,Bkj=2;
		   (0 2 1 0) = (0 2 1 0)-(5 6 3 0)/6*2
	  (3)、对矩阵A进行行变换，列变换，使得基变量对应的矩阵块变为单位矩阵;
	  (4)、b向量的值即为所求基本解。

	  基本解计算示例：
	  (1)非基变量设为定值后，矩阵A变为：
	     2 0 3 3
	     5 6 3 4
	     0 2 1 5
	  (2)矩阵B选枢轴元素，作枢轴变换
	     变换后的矩阵：
	     0     0     3
	     5     6     3
	     -5/3  0     0 
	     枢轴元素为 （1,1）（2,0）（0,2）
	  (3)矩阵A变为
	       y2 y1 y3  b
	    x2 1   0  0  19/3
	    x3 0   1  0  -3
	    x1 0   0  1  1/3
	  (4)基本解（y1,y2,y3）=(-3,19/3,1/3),其余非基向量对应变量的值为1中设置的定值
	
	10、初始化_costFunctionManager，m,n赋值，定义属性
	    _costFunctionManager->initialize();
	11、_costFunctionManager赋值给_tableau的_costFunctionManager属性
	    _tableau->registerCostFunctionManager( _costFunctionManager );
	12、初始化_activeEntryStrategy，m,n赋值，定义属性
	    _activeEntryStrategy->initialize( _tableau );
		
	13、_statistics中设置relu约束的数量
	    _statistics.setNumPlConstraints( _plConstraints.size() );                
		
}
```
