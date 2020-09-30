[返回](./index.md)


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

    /* 记录每一行、每一列非零元素的数目
    */
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
    /**
    统计每行、每列非零元素的数量
    */
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
        // 找到一个只有一个元素的行
        unsigned singletonRow = m;
        for ( unsigned i = numTriangularRows; i < m; ++i )
        {
            if ( nnzInRow[i] == 1 )
            {
                singletonRow = i;
                break;
            }
        }

        // 如果找到了
        if ( singletonRow < m )
        {
            // Have a singleton row! Swap it to the top and update counters
            /*
            numTriangularRows指向当前已构成的下三角矩阵的最下面一行的下一行
            就是找到一个只有一个元素的行，然后和当前下三角矩阵的最下面一行的下一行换，以构成一个更大的下三角矩阵。
            */
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
        // 找不到只有一个元素的行，找一个最密集的列
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
            /**
            把未构成下三角的首行，以及其下面所有非零元素化为0
            比如目前最密集的列是第col列，且未构成任何下三角。则numTriangularRows=0，因此：
            matrix[0][col]
            matrix[1][col]
            matrix[2][col]
            matrix[3][col]
            matrix[4][col]
            都将置为0.
            因此对应行的nnzInRow 要减一。
            */
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

关于最后为什么要转换为下三角矩阵，其实这里转换为上三角矩阵也是可以的，主要是要形式一个阶梯型的矩阵，在阶梯上的每一个元素所在的列的原本的列可以构成线性无关方程组，这些线性无关的列就可以组成基变量。而对于基础案例中m=5,n=6，势必是要多出一列的，而这列就会被舍弃。

假如最后`numTriangularRows < m`,说明找不到一个满秩的线性无关组（阶梯数=m），那么基变量是凑不齐的，会在之后的函数中添加辅助变量来补齐基变量。