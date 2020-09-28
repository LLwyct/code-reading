```cpp
void ConstraintMatrixAnalyzer::analyze( const double *matrix, unsigned m, unsigned n )
{
    freeMemoryIfNeeded();

    _m = m;
    _n = n;

    _matrix = new double[m * n];
    _work = new double[n];
    _rowHeaders = new unsigned[m];
    _columnHeaders = new unsigned[n];

    // Initialize the local copy of the matrix
    // 获得一份constraintMatrix的拷贝，不影响原数据
    memcpy( _matrix, matrix, sizeof(double) * m * n );

    // Initialize the row and column headers
    // 这里涉及到矩阵交换行和列，如果直接大段的移动数据过于低效，在这里采用rowHeader和columnHeader的方法，只交换Header储存的编号
    for ( unsigned i = 0; i < _m; ++i )
        _rowHeaders[i] = i;

    for ( unsigned i = 0; i < _n; ++i )
        _columnHeaders[i] = i;

    // 利用高斯消元法，把矩阵转换为阶梯型矩阵
    gaussianElimination();
}
```

```cpp
void ConstraintMatrixAnalyzer::gaussianElimination()
{
    /*
      We work column-by-column, until m columns are found
      that are non-zeroes, or until we run out of columns.
    */
    _eliminationStep = 0;

    while ( _eliminationStep < _m )
    {
        // Find largest pivot in active submatrix
        double largestPivot = 0.0;
        unsigned bestRow = 0, bestColumn = 0;
        /**
         * 找出系数矩阵中的最大值，并记录所在的行和列，以它为基进行变换
         */
        for ( unsigned i = _eliminationStep; i < _m; ++i )
        {
            for ( unsigned j = _eliminationStep; j < _n; ++j )
            {
                double contender = FloatUtils::abs( _matrix[_rowHeaders[i]*_n + _columnHeaders[j]] );
                // gt(x, y): x > y ? true : false
                if ( FloatUtils::gt( contender, largestPivot ) )
                {
                    largestPivot = contender;
                    bestRow = i;
                    bestColumn = j;
                }
            }
        }

        // If we couldn't find a non-zero pivot, elimination is complete
        if ( FloatUtils::isZero( largestPivot ) )
            return;

        // Move the pivot row to the top
        if ( bestRow != _eliminationStep )
            swapRows( bestRow, _eliminationStep );

        // Move the pivot column to the top
        if ( bestColumn != _eliminationStep )
            swapColumns( bestColumn, _eliminationStep );

        // Eliminate the rows below the pivot row
        double invPivotEntry = 1 / _matrix[_rowHeaders[_eliminationStep]*_n + _columnHeaders[_eliminationStep]];

        /**
         * 5 4 3 2 1
         * 3 4 2 2 1
         * 
         * 现在这个基就是5，invPivotEntry就是1/5
         * factor就是 -1/5
         * 直接把3置为0，剩下的变为
         * 5 4 3 2 1
         * 0 4 2 2 1
         * 然后把第二行依次 4 = 4 + 4*factor；2 = 2 + 3*factor；2 = 2 + 2*factor；1 = 1 + 1*factor；
         * 变为
         * 5 4 3 2 1
         * 0 ? ? ? ?
         * 明天继续
         */
        for ( unsigned i = _eliminationStep + 1; i < _m; ++i )
        {
            double factor =
                -_matrix[_rowHeaders[i]*_n + _columnHeaders[_eliminationStep]] * invPivotEntry ;
            _matrix[_rowHeaders[i]*_n + _columnHeaders[_eliminationStep]] = 0;

            for ( unsigned j = _eliminationStep + 1; j < _n; ++j )
                _matrix[_rowHeaders[i]*_n + _columnHeaders[j]] +=
                    (factor * _matrix[_rowHeaders[_eliminationStep]*_n + _columnHeaders[j]]);
        }
        /*
        std::cout<<"The reduction times "<<_eliminationStep<<std::endl;
        for (int iii = 0; iii < (int)_m; ++iii) {
            for (int jjj = 0; jjj < (int)_n; ++jjj) {
                // std::cout << _matrix[_rowHeaders[iii] * _n + _columnHeaders[jjj]] << " ";
                printf("%2.0f ", _matrix[_rowHeaders[iii] * _n + _columnHeaders[jjj]]);
            }
            std::cout<<std::endl;
        }*/
        _independentColumns.append( _columnHeaders[_eliminationStep] );
        ++_eliminationStep;
    }

    dumpMatrix( "Elimination finished" );
}
```
[回到上级](./index.md)