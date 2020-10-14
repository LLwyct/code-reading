```cpp
// 非常重要的部分
bool Preprocessor::processEquations()
{
    enum {
        ZERO = 0,
        POSITIVE = 1,
        NEGATIVE = 2,
        INFINITE = 3,
    };

    bool tighterBoundFound = false;

    List<Equation> &equations( _preprocessed.getEquations() );
    List<Equation>::iterator equation = equations.begin();

    while ( equation != equations.end() )
    {
        // The equation is of the form sum (ci * xi) - b ? 0
        Equation::EquationType type = equation->_type;

        unsigned maxVar = equation->_addends.begin()->_variable;
        for ( const auto &addend : equation->_addends )
        {
            if ( addend._variable > maxVar )
                maxVar = addend._variable;
        }

        ++maxVar;

        double *ciTimesLb = new double[maxVar];
        double *ciTimesUb = new double[maxVar];
        char *ciSign = new char[maxVar];

        Set<unsigned> excludedFromLB;
        Set<unsigned> excludedFromUB;

        unsigned xi;
        double xiLB;
        double xiUB;
        double ci;
        double lowerBound;
        double upperBound;
        bool validLb;
        bool validUb;

        // The first goal is to compute the LB and UB of: sum (ci * xi) - b
        // For this we first identify unbounded variables
        /*
         * 第一个目标是计算以下各项的LB和UB：sum（ci * xi）-b.为此，我们首先确定无界变量
         */
        double auxLb = -equation->_scalar;
        double auxUb = -equation->_scalar;
        for ( const auto &addend : equation->_addends )
        {
            ci = addend._coefficient;
            xi = addend._variable;

            if ( FloatUtils::isZero( ci ) )
            {
                ciSign[xi] = ZERO;
                ciTimesLb[xi] = 0;
                ciTimesUb[xi] = 0;
                continue;
            }

            ciSign[xi] = ci > 0 ? POSITIVE : NEGATIVE;

            xiLB = _preprocessed.getLowerBound( xi );
            xiUB = _preprocessed.getUpperBound( xi );

            if ( FloatUtils::isFinite( xiLB ) )
            {
                ciTimesLb[xi] = ci * xiLB;
                if ( ciSign[xi] == POSITIVE )
                    auxLb += ciTimesLb[xi];
                else
                    auxUb += ciTimesLb[xi];
            }
            else
            {
                if ( ci > 0 )
                    excludedFromLB.insert( xi );
                else
                    excludedFromUB.insert( xi );
            }

            if ( FloatUtils::isFinite( xiUB ) )
            {
                ciTimesUb[xi] = ci * xiUB;
                if ( ciSign[xi] == POSITIVE )
                    auxUb += ciTimesUb[xi];
                else
                    auxLb += ciTimesUb[xi];
            }
            else
            {
                if ( ci > 0 )
                    excludedFromUB.insert( xi );
                else
                    excludedFromLB.insert( xi );
            }
        }

        // Now, go over each addend in sum (ci * xi) - b ? 0, and see what can be done
        // 现在，遍历 sum(ci*xi)-b?0 的每个Addend。，看看能做什么
        for ( const auto &addend : equation->_addends )
        {
            ci = addend._coefficient;
            xi = addend._variable;

            // If ci = 0, nothing to do.
            if ( ciSign[xi] == ZERO )
                continue;

            /*
              The expression for xi is:

                   xi ? ( -1/ci ) * ( sum_{j\neqi} ( cj * xj ) - b )

              We use the previously computed auxLb and auxUb and adjust them because
              xi is removed from the sum. We also need to pay attention to the sign of ci,
              and to the presence of infinite bounds.

              Assuming "?" stands for equality, we can compute a LB if:
                1. ci is negative, and no vars except xi were excluded from the auxLb
                2. ci is positive, and no vars except xi were excluded from the auxUb

              And vice-versa for UB.

              In case "?" is GE or LE, only one direction can be computed.
            */
            if ( ciSign[xi] == NEGATIVE )
            {
                /**
                 * 判断是否可以缩紧下界
                 */
                validLb =
                    ( ( type == Equation::LE ) || ( type == Equation::EQ ) )
                    &&
                    ( excludedFromLB.empty() ||
                      ( excludedFromLB.size() == 1 && excludedFromLB.exists( xi ) ) );
                /**
                 * 判断是否可以缩紧上界
                 */
                validUb =
                    ( ( type == Equation::GE ) || ( type == Equation::EQ ) )
                    &&
                    ( excludedFromUB.empty() ||
                      ( excludedFromUB.size() == 1 && excludedFromUB.exists( xi ) ) );
            }
            else
            {
                /**
                 * 判断是否可以缩紧下界
                 */
                validLb =
                    ( ( type == Equation::GE ) || ( type == Equation::EQ ) )
                    &&
                    ( excludedFromUB.empty() ||
                      ( excludedFromUB.size() == 1 && excludedFromUB.exists( xi ) ) );
                /**
                 * 判断是否可以缩紧shang界
                 */
                validUb =
                    ( ( type == Equation::LE ) || ( type == Equation::EQ ) )
                    &&
                    ( excludedFromLB.empty() ||
                      ( excludedFromLB.size() == 1 && excludedFromLB.exists( xi ) ) );
            }

            // Now compute the actual bounds and see if they are tighter
            /**
             * 如果可以缩紧下界，判断系数类型，如果系数为正，则上下界对应关系颠倒(移项后为负)，否则上界对应上界，下界对应下界
             */
            if ( validLb )
            {
                if ( ciSign[xi] == NEGATIVE )
                {
                    lowerBound = auxLb;
                    /**
                     * 如果被排除，这里要先减掉一个值，其实这里没太懂
                     */
                    if ( !excludedFromLB.exists( xi ) )
                        lowerBound -= ciTimesUb[xi];
                }
                else
                {
                    lowerBound = auxUb;
                    if ( !excludedFromUB.exists( xi ) )
                        lowerBound -= ciTimesUb[xi];
                }

                lowerBound /= -ci;
                /**
                 * 判断新的下界是不是比原下界小，真则更新并标记tighterboundFound为真
                 */
                if ( FloatUtils::gt( lowerBound, _preprocessed.getLowerBound( xi ) ) )
                {
                    tighterBoundFound = true;
                    _preprocessed.setLowerBound( xi, lowerBound );
                }
            }
            // 与上面对应，不再解释
            if ( validUb )
            {
                if ( ciSign[xi] == NEGATIVE )
                {
                    upperBound = auxUb;
                    if ( !excludedFromUB.exists( xi ) )
                        upperBound -= ciTimesLb[xi];
                }
                else
                {
                    upperBound = auxLb;
                    if ( !excludedFromLB.exists( xi ) )
                        upperBound -= ciTimesLb[xi];
                }

                upperBound /= -ci;

                if ( FloatUtils::lt( upperBound, _preprocessed.getUpperBound( xi ) ) )
                {
                     tighterBoundFound = true;
                    _preprocessed.setUpperBound( xi, upperBound );
                }
            }
            /**
             * lb应该小于ub，若lb-ub>1e-5, 这里1e-5视为0. return true, and throw the Exception
             */
            if ( FloatUtils::gt( _preprocessed.getLowerBound( xi ),
                                 _preprocessed.getUpperBound( xi ),
                                 GlobalConfiguration::PREPROCESSOR_ALMOST_FIXED_THRESHOLD ) )
            {
                delete[] ciTimesLb;
                delete[] ciTimesUb;
                delete[] ciSign;

                throw InfeasibleQueryException();
            }
        }

        delete[] ciTimesLb;
        delete[] ciTimesUb;
        delete[] ciSign;

        /*
          Next, do another sweep over the equation.
          Look for almost-fixed variables and fix them, and remove the equation
          entirely if it has nothing left to contribute.
        */
        /*
         * 接下来，再次扫描方程式。 寻找几乎固定的变量并将其固定，如果没有任何贡献可将其完全删除。
         */
        bool allFixed = true;
        for ( const auto &addend : equation->_addends )
        {
            unsigned var = addend._variable;
            double lb = _preprocessed.getLowerBound( var );
            double ub = _preprocessed.getUpperBound( var );

            if ( FloatUtils::areEqual( lb, ub, GlobalConfiguration::PREPROCESSOR_ALMOST_FIXED_THRESHOLD ) )
                _preprocessed.setUpperBound( var, _preprocessed.getLowerBound( var ) );
            else
                allFixed = false;
        }

        if ( !allFixed )
        {
            ++equation;
        }
        else
        {
            double sum = 0;
            for ( const auto &addend : equation->_addends )
                sum += addend._coefficient * _preprocessed.getLowerBound( addend._variable );

            if ( FloatUtils::areDisequal( sum, equation->_scalar, GlobalConfiguration::PREPROCESSOR_ALMOST_FIXED_THRESHOLD ) )
            {
                throw InfeasibleQueryException();
            }
            equation = equations.erase( equation );
        }
    }

    return tighterBoundFound;
}
```

[返回 preprocessor()](./invokePreprocesser.md)