[回到首页](./index.md)

# 1. engine

- [1. engine](#1-engine)
  - [1.1. engine.solve()](#11-enginesolve)
  - [1.2. Engine::updateDirections()](#12-engineupdatedirections)
  - [1.3. 略](#13-略)

## 1.1. engine.solve()

代码很长

```c++
	//计算极性（更靠近正还是更靠近负）（active or inactive），分割时优先处理的方向
	updateDirections();

    //最重要的一个死循环
    while ( true )
    {
        try
        {   

			//eta 是否为空  基矩阵是否可用
            if ( _tableau->basisMatrixAvailable() )
                //收缩边界 通过xB = inv(B)*b - inv(B)*An
                explicitBasisBoundTightening();
			//分割过
            if ( splitJustPerformed )
            {
                do
                {		
                    //bst 收缩边界  依靠网络拓扑
                    performSymbolicBoundTightening();
                }        //遍历每一个约束，如果约束是active 并且状态固定，则应用split，并返回true。
                while ( applyAllValidConstraintCaseSplits() );//只有一个方向的分割 （active or inactive） getValidCaseSplits
                splitJustPerformed = false;
            }
            //上面的操作就是把分段线性约束当作普通线性约束来处理，后面如果发现不符合，再进行处理（smt）

            // Perform any SmtCore-initiated case splits
            //如果某一个约束违反约束的次数超过一定次数，则needToSplit 为true,并选择一个去修复
            if ( _smtCore.needToSplit() )
            {	//约束必须是 PHASE_NOT_FIXED，如果状态固定之前应该已经处理
                //如果变为 inactive 则 忽略，这个分割是active 和 inactive两个方向都有，只不过两种顺序不同  getCaseSplits
                //但是每次只应用一种分割，另一种留着下次
                _smtCore.performSplit();
                splitJustPerformed = true;
                continue;
            }

            if ( !_tableau->allBoundsValid() )
            {
                // Some variable bounds are invalid, so the query is unsat
                throw InfeasibleQueryException();
            }

            if ( allVarsWithinBounds() )
            {
                // The linear portion of the problem has been solved.
                // Check the status of the PL constraints
                collectViolatedPlConstraints();

                // If all constraints are satisfied, we are possibly done
                if ( allPlConstraintsHold() )
                {
                    if ( _tableau->getBasicAssignmentStatus() !=
                         ITableau::BASIC_ASSIGNMENT_JUST_COMPUTED )
                    {
                        if ( _verbosity > 0 )
                        {
                            printf( "Before declaring sat, recomputing...\n" );
                        }
                        // Make sure that the assignment is precise before declaring success
                        _tableau->computeAssignment();
                        continue;
                    }
                    if ( _verbosity > 0 )
                    {
                        printf( "\nEngine::solve: sat assignment found\n" );
                        _statistics.print();
                    }
                    _exitCode = Engine::SAT;
                    return true;
                }

                // We have violated piecewise-linear constraints.
                //修复不符合的分段线性约束
                //现找到一个 不符合的（默认找修复次数最少的。如果不是，则找列表第一个），在报告给 smt ，记录次数， 最后尝试修复
                //修复有两种，一是smartFix （如果线性无关，则不行，就是possibleFix），并看是否存在不需要后续pivot的fix（找不是基变量的，并且赋值在范围内），存在则更新基变量（xB = inv(B)*b - inv(B)*An），并看成一次fake pivot，计算 changeColumn 更新基变量状态
                performConstraintFixingStep();

                // Finally, take this opporunity to tighten any bounds
                // and perform any valid case splits.
                
                //每隔 几次就进行一次边界收缩（利用约束矩阵，），频率通过参数设置
                //The cosntraint matrix A satisfies Ax = b.Each row is of the form:sum ci xi - b = 0

     			 //We first compute the lower and upper bounds for the expression sum ci xi - b
                tightenBoundsOnConstraintMatrix();
                //将 各种边界缩紧     应用Tightingening   两种：行 缩紧  和 约束缩紧
                applyAllBoundTightenings();
                // For debugging purposes
                checkBoundCompliancyWithDebugSolution();
				//和上面的do while{} 一样，有 的变量值和上下界都以改变，重新split和 收缩边界
                while ( applyAllValidConstraintCaseSplits() )
                    performSymbolicBoundTightening();

                continue;
            }

            // We have out-of-bounds variables.
            //上面修复了约束，会有变量超出了其上下界。执行单纯形方法，向变量都满足靠近
            performSimplexStep();
            continue;
        }
    }
```

## 1.2. Engine::updateDirections()

```c++
void Engine::updateDirections()
{
    if ( GlobalConfiguration::USE_POLARITY_BASED_DIRECTION_HEURISTICS )
        for ( const auto &constraint : _plConstraints )
            //是否支持对称边界缩紧
            if ( constraint->supportPolarity() &&
                 constraint->isActive() && !constraint->phaseFixed() )
                //更新修复和处理分割的方向
                constraint->updateDirection();
}
```
[回到顶部](#1-engine)

[回到首页](./index.md)
## 1.3. 略



