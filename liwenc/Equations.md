# 1. Eqution.h

首先是一个枚举类型,分别代表等于、大于等于、小于等于
```cpp
class Equation {
public:
    enum EquationType {
        EQ = 0,
        GE = 1,
        LE = 2
    };
}
```
定义了一个`Addend`结构体

```cpp
struct Addend
    {
    public:
        Addend( double coefficient, unsigned variable );

        double _coefficient;
        unsigned _variable;

        bool operator==( const Addend &other ) const;
    };
```
以及`Eqution`中的一些私有成员

```cpp
List<Addend> _addends;
double _scalar;
EquationType _type;
```

# 2. Eqution.cpp
Eqution的构造函数

```cpp
// Eqution的构造函数1
Equation::Equation()
    : _scalar( 0 )
    , _type( Equation::EQ ) // 默认是EQ类型
{
}
// Eqution的构造函数2
Equation::Equation( EquationType type )
    : _scalar( 0 )
    , _type( type )
{
}

// Addend的构造函数
Equation::Addend::Addend( double coefficient, unsigned variable )
    : _coefficient( coefficient )
    , _variable( variable )
{
}
```

## 2.1. dump函数
就是把零碎的信息拼成一个等式，例如：$+x_1-5x_2+7x_3<=5$
- _coefficient即为系数
- _variable即为变量名
- _scalar为标量，在上例中为5

利用Stringf来格式化字符串，若小于0，则自动带符号，因此当系数为正时要额外加正号
```cpp
void Equation::dump( String &output ) const
{
    output = "";
    for ( const auto &addend : _addends )
    {
        if ( FloatUtils::isZero( addend._coefficient ) )
            continue;

        if ( FloatUtils::isPositive( addend._coefficient ) )
            output += String( "+" );

        output += Stringf( "%.2lfx%u ", addend._coefficient, addend._variable );
    }

    switch ( _type )
    {
    case Equation::GE:
        output += String( " >= " );
        break;

    case Equation::LE:
        output += String( " <= " );
        break;

    case Equation::EQ:
        output += String( " = " );
        break;
    }

    output += Stringf( "%.2lf\n", _scalar );
}
```