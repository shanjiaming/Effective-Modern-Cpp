* [std::function](https://en.cppreference.com/w/cpp/utility/functional/function)
```cpp
#include <functional>
#include <iostream>
 
struct Foo {
    Foo(int num) : num_(num) {}
    void print_add(int i) const { std::cout << num_+i << '\n'; }
    int num_;
};
 
void print_num(int i)
{
    std::cout << i << '\n';
}
 
struct PrintNum {
    void operator()(int i) const
    {
        std::cout << i << '\n';
    }
};
 
int main()
{
    // store a free function
    std::function<void(int)> f_display = print_num;
    f_display(-9);
 
    // store a lambda
    std::function<void()> f_display_42 = []() { print_num(42); };
    f_display_42();
 
    // store the result of a call to std::bind
    std::function<void()> f_display_31337 = std::bind(print_num, 31337);
    f_display_31337();
 
    // store a call to a member function
    std::function<void(const Foo&, int)> f_add_display = &Foo::print_add;
    const Foo foo(314159);
    f_add_display(foo, 1);
    f_add_display(314159, 1);
 
    // store a call to a data member accessor
    std::function<int(Foo const&)> f_num = &Foo::num_;
    std::cout << "num_: " << f_num(foo) << '\n';
 
    // store a call to a member function and object
    using std::placeholders::_1;
    std::function<void(int)> f_add_display2 = std::bind( &Foo::print_add, foo, _1 );
    f_add_display2(2);
 
    // store a call to a member function and object ptr
    std::function<void(int)> f_add_display3 = std::bind( &Foo::print_add, &foo, _1 );
    f_add_display3(3);
 
    // store a call to a function object
    std::function<void(int)> f_display_obj = PrintNum();
    f_display_obj(18);
}

// output
-9
42
31337
314160
314160
num_: 314159
314161
314162
18
```

* [std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)
```cpp
#include <random>
#include <iostream>
#include <memory>
#include <functional>
 
void f(int n1, int n2, int n3, const int& n4, int n5)
{
    std::cout << n1 << ' ' << n2 << ' ' << n3 << ' ' << n4 << ' ' << n5 << '\n';
}
 
int g(int n1)
{
    return n1;
}
 
struct Foo {
    void print_sum(int n1, int n2)
    {
        std::cout << n1+n2 << '\n';
    }
    int data = 10;
};
 
int main()
{
    using namespace std::placeholders;  // for _1, _2, _3...
 
    // demonstrates argument reordering and pass-by-reference
    int n = 7;
    // (_1 and _2 are from std::placeholders, and represent future
    // arguments that will be passed to f1)
    auto f1 = std::bind(f, _2, _1, 42, std::cref(n), n);
    n = 10;
    f1(1, 2, 1001); // 1 is bound by _1, 2 is bound by _2, 1001 is unused
                    // makes a call to f(2, 1, 42, n, 7)
 
    // nested bind subexpressions share the placeholders
    auto f2 = std::bind(f, _3, std::bind(g, _3), _3, 4, 5);
    f2(10, 11, 12); // makes a call to f(12, g(12), 12, 4, 5);
 
    // common use case: binding a RNG with a distribution
    std::default_random_engine e;
    std::uniform_int_distribution<> d(0, 10);
    auto rnd = std::bind(d, e); // a copy of e is stored in rnd
    for(int n=0; n<10; ++n)
        std::cout << rnd() << ' ';
    std::cout << '\n';
 
    // bind to a pointer to member function
    Foo foo;
    auto f3 = std::bind(&Foo::print_sum, &foo, 95, _1);
    f3(5);
 
    // bind to a pointer to data member
    auto f4 = std::bind(&Foo::data, _1);
    std::cout << f4(foo) << '\n';
 
    // smart pointers can be used to call members of the referenced objects, too
    std::cout << f4(std::make_shared<Foo>(foo)) << '\n'
              << f4(std::make_unique<Foo>(foo)) << '\n';
}

// output
2 1 42 10 7
12 12 12 4 5
1 5 0 2 0 8 2 2 10 8 // 这里是随机数，结果可能不同
100
10
10
10
```

> ## 31 避免默认捕获模式
* 默认的引用捕获可能导致空悬引用，默认的值捕获会让人误以为可以避免空悬引用的问题，以及闭包是独立的（实际上可能不是），这节将具体解释默认捕获模式的危害
* 引用捕获会导致闭包包含指向局部变量的引用，或定义lambda的作用域内的形参的引用。如果lambda创建的闭包超过了该局部变量或形参的生命期，闭包内的引用就会空悬
```cpp
using FilterContainer = std::vector<std::function<bool(int)>>;
FilterContainer filters;

void addDivisorFilter()
{
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();
    auto divisor = computeDivisor(calc1, calc2);
    filters.emplace_back( // 危险：对divisor的引用可能空悬
        [&](int value) { return value % divisor == 0; }
    );
}
```
* lambda中包含局部变量divisor的引用，该变量在emplace_back后，即函数addDivisorFilter结束时被销毁，于是emplace_back到容器的闭包对象包含了一个被销毁的引用，容器内将产生未定义行为
* 如果已知lambda的闭包会被立即使用（比如传递给STL算法）且不被拷贝，则引用捕获不存在空悬的风险。但这种安全也是暂时的，如果该lambda在其他语境中有用处（比如上述的添加到容器），并被复制粘贴到闭包生命期超过变量divisor生命期的语境中，仍会导致空悬，而且这样更难以分析divisor的生命期了
* 解决此问题的一种方法是采用值捕获模式，对于上述例子来说已经足够了
```cpp
filters.emplace_back( // 现在divisor不会空悬
    [=](int value) { return value % divisor == 0; }
);
```
* 但值捕获并不能完美地避免空悬，因为值捕获指针和引用捕获的情况类似。这有点匪夷所思，因为你会坚定地认为在自己的掌控下，智能指针才是最佳选择，使用原始指针的情况根本不会发生。但事实上，原始指针会在无意中被使用，甚至在眼前被delete而不被察觉
```cpp
class Widget {
public:
    … // ctors, etc.
    void addFilter() const; // add an entry to filters
private:
    int divisor; // used in Widget's filter
};

void Widget::addFilter() const
{
    filters.emplace_back(
        [=](int value) { return value % divisor == 0; }
    );
}
```
* 这段看似安全的代码实际错到离谱。捕获只能应用于lambda所在作用域内可见的non-static局部变量，而Widget::addFilter的函数体内，divisor是Widget类的数据成员而非局部变量，它不能被捕获。如果去掉=，取消默认的值捕获模式，代码就不能编译
```cpp
void Widget::addFilter() const
{
    filters.emplace_back(
        [](int value) { return value % divisor == 0; } // 错误：没有可捕获的divisor
    );
}
```
* 换成显式捕获divisor（不管是值捕获还是引用捕获），依然无法编译
```cpp
void Widget::addFilter() const
{
    filters.emplace_back(
        [divisor](int value) { return value % divisor == 0; } // 错误：没有可捕获的divisor
    );
}
```
* 实际发生的事不难推测，=捕获的并不是divisor，而是Widget的this指针，在成员函数中，编译器就会把divisor替换成this->divisor，于是使用了=的成员函数相当于
```cpp
void Widget::addFilter() const
{
    auto currentObjectPtr = this;
    filters.emplace_back(
    [currentObjectPtr](int value)
        { return value % currentObjectPtr->divisor == 0; }
    );
}
```
* 因此，lambda闭包与其中拷贝而来的this指针所属的Widget对象的生命期是绑定在一起的。下列代码就说明了值捕获产生空悬的情况，尽管还使用了智能指针
```cpp
using FilterContainer = std::vector<std::function<bool(int)>>;
FilterContainer filters;

void doSomeWork()
{
    auto pw = std::make_unique<Widget>();
    pw->addFilter();
    …
} // 销毁Widget：filters现在持有空悬指针！
```
* 这个特定问题的一个解决方案是，把成员变量拷贝到一个局部变量中，再捕获该局部变量
```cpp
void Widget::addFilter() const
{
    auto divisorCopy = divisor; // 拷贝数据成员
    filters.emplace_back(
    [divisorCopy](int value) // 捕获拷贝对象
        { return value % divisorCopy == 0; } // 使用拷贝对象
    );
}
```
* C++14中，捕获成员变量的一种更好的方法是使用广义lambda捕获（generalized lambda-capture），也叫初始化捕获（init-capture）
```cpp
void Widget::addFilter() const
{
    filters.emplace_back(
    [divisor = divisor](int value) // C++14：将divisor拷贝进闭包
        { return value % divisor == 0; } // 使用拷贝对象
    );
}
```
* 值捕获的另一个问题是，它似乎表明闭包是独立的，与闭包外的数据变化无关。事实并非如此，除了可以被捕获的局部变量外，lambda还会依赖于静态存储期的对象，这样的对象可以在lambda中使用，但不能捕获
```cpp
void addDivisorFilter()
{
    static auto calc1 = computeSomeValue1();
    static auto calc2 = computeSomeValue2();
    static auto divisor = computeDivisor(calc1, calc2);
    filters.emplace_back(
        [=](int value) // 未捕获任何东西
        { return value % divisor == 0; } // 指向static对象
    );
    ++divisor; // 意外修改divisor将导致每个lambda的行为不同
}
```

> ## 32 使用初始化捕获将对象移入闭包
* C++14中提供了初始化捕获，用于将对象移动进闭包
```cpp
class Widget { // some useful type
public:
    …
    bool isValidated() const;
    bool isProcessed() const;
    bool isArchived() const;
private:
    …
};

auto pw = std::make_unique<Widget>();
…
auto func =
    [pw = std::move(pw)] // 用std::move(pw)初始化闭包类的成员变量pw
    { return pw->isValidated() && pw->isArchived(); };
```
* 也可以直接由std::make_unique初始化
```cpp
auto func =
    [pw = std::make_unique<Widget>()]
    { return pw->isValidated() && pw->isArchived(); };
```
* C++11中不允许捕获一个表达式，但可以通过创建一个重载了operator()并支持移动构造的类实现同样目的
```cpp
class IsValAndArch {
public:
    using DataType = std::unique_ptr<Widget>;
    explicit IsValAndArch(DataType&& ptr)
    : pw(std::move(ptr)) {}
    bool operator()() const
    { return pw->isValidated() && pw->isArchived(); }
private:
    DataType pw;
};

auto func = IsValAndArch(std::make_unique<Widget>());
```
* 若要用lambda在C++11中模拟初始化捕获，首先把要捕获的对象移动到std::bind产生的函数对象中，随后给该lambda一个绑定到捕获对象的引用即可
```cpp
auto func = std::bind(
    [](const std::unique_ptr<Widget>& pw)
    { return pw->isValidated() && pw->isArchived(); },
    std::make_unique<Widget>()
);
```
* std::bind返回的对象称为bind对象。bind对象中包含传递给std::bind的所有实参的拷贝，对于每个左值实参，bind对象中的对应部分由拷贝构造，对于右值实参则是移动构造。上例中第二个实参是右值，采用的是移动构造，这正是把右值移动进bind对象的手法
```cpp
std::vector<double> data; // 要移动到闭包的对象
...
auto f = [data = std::move(data)] { ... }; // C++14：初始化捕获
auto f2 = std::bind( // C++11：模拟初始化捕获
    [] (const std::vector<double>& data) { ... }, std::move(data)
);
```
* 调用f2()时，由移动构造得到的data副本就会作为实参传递给std::bind中的lambda，lambda中对data的操作都会执行在移动而来的data上。虽然lambda的参数类型是左值引用，但传递而来的实参已经是移动得到的data了
* 默认情况下，lambda生成的闭包类的operator()默认为const，闭包中的所有成员变量也会为const。但移动而来的data副本不为const，为了防止data副本被修改，则必须将lambda形参声明为const
* 如果lambda的声明为mutable，则闭包中的operator()就不会为const，lambda形参也就不必声明为const
```cpp
auto f3 =
    std::bind( // C++11对可变lambda模拟初始化捕获
        [](std::vector<double>& data) mutable
        { ... },
        std::move(data)
);
```
* 因为bind对象的生命期和闭包相同，所以对bind对象中的对象和闭包中的对象可以用同样的手法处理

> ## 33 对auto&&类型形参使用decltype来std::forward
* C++14中，lambda的形参可以声明为auto，这种lambda称为泛型lambda（generic lambda）
* 泛型lambda闭包类中的operator()使用模板实现
```cpp
auto f = [](auto x){ return func(normalize(x)); };
// the closure class’s function call operator looks like this:
class SomeCompilerGeneratedClassName {
public:
    template<typename T>
    auto operator()(T x) const
    { return func(normalize(x)); }
    … // 闭包类的其他功能
};
```
* 上例中lambda对x唯一做的就是将其转发给normalize，更恰当的做法是使用完美转发，但修改之后便会发现问题
```cpp
auto f = [](auto&& x)
    { return func(normalize(std::forward<???>(x))); };
```
* x类型由auto推断，于是传递给std::forward的类型无法得知。此时可以想到用decltype判断传入的实参是左值还是右值
  * 如果传递给auto&&的实参是左值，则x为左值引用类型，decltype(x)为左值引用类型
  * 如果传递给auto&&的实参是右值，则x为右值引用类型，decltype(x)为右值引用类型
* 于是把decltype(x)传递给std::forward就能得到完美转发的泛型lambda
```cpp
auto f = [](auto&& x)
    { return func(normalize(std::forward<decltype(x)>(x))); };
```
* 稍作改动就可以得到接受多个形参的版本
```cpp
auto f = [](auto&&... args)
    { return func(normalize(std::forward<decltype(args)>(args)...)); };
```

> ## 34 使用lambda替代std::bind
## 概述
* 这节将阐述lambda比起std::bind的几个优点
  * lambda代码更简洁，std::bind的占位符的可读性差
  * 实参绑定的是std::bind的返回对象而非内部的函数
  * 有多个重载函数时，lambda可以根据参数数量选择正确的重载版本，而std::bind只能识别函数名，无法区分重载版本
  * 要解决重载问题，必须给std::bind指定要确定的重载版本的函数指针。此时，lambda闭包类的operator()采用的是能被编译器内联的常规的函数调用，而std::bind用编译器通常不会内联的函数指针调用，这样lambda的代码就比std::bind运行得更快
  * lambda可以指定值捕获和引用捕获，而std::bind总会按值拷贝实参，如果要按引用传递则需要用std::ref()指定实参
* C++14中没有需要使用std::bind的理由，C++11中也只有两个受限的使用场合
  * 移动捕获。C++11中没有提供移动捕获特性，但可以用std::bind和lambda来模拟
  * 多态函数对象。bind对象的operator()利用了完美转发，可以接受任何类型的实参。这在想要绑定一个模板化operator()的对象时是有用的
* 对于一个模板化operator()的类，用std::bind绑定其对象，则可以用任意类型实参调用
```cpp
class PolyWidget {
public:
    template<typename T>
    void operator()(const T& param);
    …
};
PolyWidget pw;
auto boundPW = std::bind(pw, _1);
boundPW(1930); // pass int to PolyWidget::operator()
boundPW(nullptr); // pass nullptr to PolyWidget::operator()
boundPW("Rosebud"); // pass string literal to PolyWidget::operator()
```
* C++11中的lambda无法达成上述效果，但C++14的泛型lambda可以轻松实现上述效果
```cpp
auto boundPW = [pw](const auto& x) { pw(x); };
```

## 详述
* 假设有一个设置声音警报的函数，要设计一个程序，程序的某处设定一小时后将发出持续30秒的警报，警报的声音由一个lambda确定
```cpp
using Time = std::chrono::steady_clock::time_point;
enum class Sound { Beep, Siren, Whistle };
using Duration = std::chrono::steady_clock::duration;
void setAlarm(Time t, Sound s, Duration d); // 设置声音警报的函数

auto setSoundL = [](Sound s)
{
    using namespace std::chrono;
    setAlarm(steady_clock::now() + hours(1), s, seconds(30));
};
```
* 用C++14的s、ms、h等标准后缀可以进一步简化代码
```cpp
auto setSoundL = [](Sound s)
{
    using namespace std::chrono;
    using namespace std::literals; // for C++14 suffixes
    setAlarm(steady_clock::now() + 1h, s, 30s);
};
```
* 如果用std::bind来实现此效果，除了要引入降低可读性的占位符外，还可能导致某些问题
```cpp
using namespace std::chrono;
using namespace std::literals;
using namespace std::placeholders; // needed for use of "_1"
auto setSoundB = std::bind(
    setAlarm,
    steady_clock::now() + 1h, // 不符合目的的做法
    _1,
    30s);
```
* 上述代码的问题在于，std::bind的调用中，`steady_clock::now() + 1h`作为实参被传递给std::bind而非setAlarm，因此表达式求值发生在调用std::bind的时刻，且求得的时间结果值保存在bind对象中，最终导致警报的启动时刻是调用std::bind而非setAlarm的时刻后的一小时
* std::bind延迟到setAlarm被调用时计算表达式值可以解决此问题，实现方法是在std::bind中再嵌套一个std::bind
```cpp
auto setSoundB = std::bind(
    setAlarm,
    std::bind(std::plus<>(), steady_clock::now(), 1h), // C++14
    _1,
    30s);
```
* C++11中，std::plus<type>中的类型不能省略，再加上不能使用标准后缀简化代码，最终结果就是
```cpp
using namespace std::chrono;
using namespace std::placeholders;

auto setSoundB = std::bind(
    setAlarm,
    std::bind(std::plus<steady_clock::time_point>(),
        steady_clock::now(),
        hours(1)),
    _1,
    seconds(30));
```
* 回顾C++14中的lambda的实现，如何选择已经毫无疑义
```cpp
auto setSoundL = [](Sound s)
{
    using namespace std::chrono;
    using namespace std::literals; // for C++14 suffixes
    setAlarm(steady_clock::now() + 1h, s, 30s);
};
```
* 此外，如果给setAlarm增加一个带音量形参的重载版本
```cpp
enum class Volume { Normal, Loud, LoudPlusPlus };
void setAlarm(Time t, Sound s, Duration d, Volume v);
```
* lambda仍能正确选择只有三个形参的setAlarm，而std::bind得知的信息仅有函数名，无法确定调用的重载版本，为解决此问题必须将setAlarm转换为目标重载版本的函数指针类型
```cpp
using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);
auto setSoundB = std::bind(
    static_cast<SetAlarm3ParamType>(setAlarm), // OK
    std::bind(std::plus<>(), steady_clock::now(), 1h),
    _1,
    30s);
```
* lambda的闭包类的operator()是正常的函数调用，可以被编译器内联，而函数指针发起的调用通常不会被内联，这意味着通过setSoundB调用setAlarm被完全内联的几率比通过setSoundL要低
```cpp
setSoundL(Sound::Siren); // body of setAlarm may well be inlined here
setSoundB(Sound::Siren); // body of setAlarm is less likely to be inlined here
```
* 因此，比起std::bind，lambda生成的代码可能运行得更快
* 上例中只是涉及一个函数调用，如果代码越复杂，lambda的优势就越明显，比如对于一个简单的判断值是否在区间内的lambda
```cpp
// C++14
auto betweenL =
    [lowVal, highVal] (const auto& val)
    { return lowVal <= val && val <= highVal; };
// C++11
auto betweenL =
    [lowVal, highVal] (int val)
    { return lowVal <= val && val <= highVal; };
```
* 用std::bind表达同样的意思就繁琐得多
```cpp
// C++14
using namespace std::placeholders;
auto betweenB = std::bind(
    std::logical_and<>(),
    std::bind(std::less_equal<>(), lowVal, _1),
    std::bind(std::less_equal<>(), _1, highVal));
// C++11
auto betweenB = std::bind(
    std::logical_and<bool>(),
    std::bind(std::less_equal<int>(), lowVal, _1),
    std::bind(std::less_equal<int>(), _1, highVal));
```
* lambda的实参传递方式更明显。假设有一个用来生成类对象压缩副本的函数，创建一个用于指定压缩等级的函数对象
```cpp
enum class CompLevel { Low, Normal, High }; // 压缩等级
Widget compress(const Widget& w, CompLevel lev);

Widget w;
using namespace std::placeholders;
auto compressRateB = std::bind(compress, w, _1);
```
* w是按值传递给bind对象的，如果要按引用传递
```cpp
auto compressRateB = std::bind(compress, std::ref(w), _1);
```
* 而对于lambda，传递方式清晰可见
```cpp
auto compressRateL =
    [w](CompLevel lev) // w按值捕获，lev按值传递
    { return compress(w, lev); };

compressRateL(CompLevel::High); // 实参按值传递
compressRateB(CompLevel::High); // 实参如何传递？
```
