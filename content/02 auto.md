> ## 05 使用auto替代显式类型声明
* auto声明的变量必须初始化，因此使用auto可以避免忘记初始化的问题
```cpp
int x1; // 潜在的未初始化风险
auto x2; // 错误：必须初始化
auto x3 = 0; // OK
```
* 对于名称非常长的类型，如迭代器相关的一切类型
```cpp
template<typename It>
void f(It b, It e)
{
    while (b != e)
    {
        typename std::iterator_traits<It>::value_type currValue = *b;
        ...
    }
}
```
* 使用auto可以大大简化工作
```cpp
template<typename It>
void f(It b, It e)
{
    while (b != e)
    {
        auto currValue = *b;
        …
    }
}
```
* 并且auto可以用来表示只有编译器才知道的类型，比如lambda产生的闭包类型
```cpp
auto derefUPLess =
    [](const std::unique_ptr<A>& p1, const std::unique_ptr<A>& p2)
    { return *p1 < *p2; };
```
* C++14中，auto可以直接用于lambda的形参
```cpp
auto derefLess =
    [](const auto& p1, const auto& p2) { return *p1 < *p2; };
```
* 也许你认为不需要声明变量来持有闭包，因为可以用C++11的std::function对象来完成。比如声明一个能指向任何签名如下的可调用对象
```cpp
bool(const std::unique_ptr<Widget>&, const std::unique_ptr<Widget>&)
```
* 对象为std::function类型，声明如下
```cpp
std::function<bool(const std::unique_ptr<Widget>&, const std::unique_ptr<Widget>&)> func;
```
* 因此对于产生闭包的lambda，不用auto而用std::function则是
```cpp
std::function<bool(const std::unique_ptr<Widget>&, const std::unique_ptr<Widget>&)>
    derefUPLess =
        [](const std::unique_ptr<Widget>& p1, const std::unique_ptr<Widget>& p2)
        { return *p1 < *p2; };
```
* 除了冗长的语法外，std::function和auto不同的是，auto和闭包类型一致，内存量和闭包相同，而std::function是类模板，它声明的变量是std::function的一个实例，有一个固定的大小，而这个大小对其持有的闭包不一定够用，这样的话std::function会分配堆上的内存以存储闭包。一般来说，std::function对象比auto变量占用更多的内存。此外，编译器一般会限制内联，这样std::function调用闭包会比auto慢
* auto还可以避免简写类型的问题，比如如下看似习以为常的写法
```cpp
std::vector<int> v;
…
unsigned sz = v.size();
```
* v.size()的类型实际是std::vector<int>::size_type，但std::vector<int>::size_type只规定为无符号整型，因此很多人认为写成unsigned就够了。在32位Windows上，unsigned和std::vector<int>::size_type尺寸一样，但在64位Windows上，unsigned是32位，而std::vector<int>::size_type是64位，这样32位Windows上运行正常的代码移植到64位上就会出现问题。使用auto就可以避免这个问题
* 再比如下面看似合理的代码也有潜在问题
```cpp
std::unordered_map<std::string, int> m;
…
for (const std::pair<std::string, int>& p : m)
{
    …
}
```
* std::unordered_map的key是const，因此哈希表中的std::pair应该是std::pair<const std::string, int>，因此对上述代码，编译器要将std::pair<const std::string, int>转为std::pair<std::string, int>，这样对容器的每个对象都要拷贝出一个临时对象，再把p绑定到其上，每次迭代结束再析构，如果期间对p取地址，得到的也是临时对象的指针。使用auto就可以避免这个问题
* 使用auto最需要关心的也许是代码可读性上，但不必过于担心，auto是可选项而非必选项，如果显式类型声明能让代码更清晰或有其他好处，依然可以用显式类型声明，此外IDE的类型提示也能缓解不能直接看出对象类型的问题

> ## 06 auto推断出非预期类型时，使用explicitly typed initializer手法
* 假设一个函数返回vector<bool>，其中每个bool元素相当于一个判断标记
```cpp
std::vector<bool> features(const Widget& w);
```
* 假设第五个元素代表是否为高优先级，则可以有如下代码
```cpp
Widget w;
bool highPriority = features(w)[5];
processWidget(w, highPriority);
```
* 但如果把显式声明改为auto则会出现非预期行为
```cpp
auto highPriority = features(w)[5];
processWidget(w, highPriority); // 未定义行为
```
* 原因在于std::vector<bool>的operator[]的返回类型不是bool，而是std::vector<bool>::reference
* std::vector<bool>::reference产生的原因是std::vector<bool>做过特化，用压缩形式表示持有的bool元素，每个元素用一个bit表示
* 这给std::vector<bool>的operator[]带来了问题，因为std::vector<T>的operator[]返回T&，而C++不允许bit的引用，不能返回bool&，因此需要一个表现得像bool&的对象，实现原理是std::vector<bool>::reference做了一个到bool（不是bool&）的隐式转换
```cpp
bool highPriority = features(w)[5];
```
* 上述右侧返回的std::vector<bool>::reference可以隐式转换为bool，因此能正常运行。而如果用auto替代显式声明，highPriority不一定指向std::vector<bool>的第五个bit，这取决于std::vector<bool>::reference的具体实现
* 一种实现是让对象含有一个指向一个机器字（machine word）的指针，机器字持有被引用的bit，以及这个bit对应字的偏移（offset）
* 因为highPriority是std::vector<bool>::reference对象的一个拷贝，所以它有一个指向operator[]返回的临时量的机器字的指针以及偏移。表达式结束处临时对象被析构，结果highPriority含有一个空悬指针，最终导致调用时出现未定义行为
```cpp
processWidget(w, highPriority); // 未定义行为
```
* std::vector<bool>::reference是一个代理类（proxy class）的例子，代理类就是模拟或扩展其他类型的类，比如std::shared_ptr和std::unique_ptr是一眼能看出的代理类，而还有一些不明显的代码类如std::vector<bool>::reference和std::bitset::reference
* 同属于代理类的还有另一些C++库中的类，它们采用了表达式模板的技术。这类库是为了提高数值计算代码的效率而开发的，比如给定一个Matrix类和它的对象
```cpp
Matrix sum = m1 + m2 + m3 + m4;
```
* Matrix对象的operator+返回的是结果的代理而非结果本身，则表达式的计算会高效得多，因此两个Matrix对象的operator+会返回一个代理类对象，如Sum<Matrix, Matrix>而不是一个Matrix对象，同std::vector<bool>::reference一样，代理类会有一个对应的隐式类型转换
* 一般auto和隐性代理类不能和平相处，所以要避免写出如下代码
```cpp
auto someVar = expression of "invisible" proxy class type;
```
* 但并非要为了代理类放弃auto，auto本身不是问题，问题是没有推断出预期的类型，解决方案是强制进行另一次类型转换，这种方法称为the explicitly typed initializer idiom
```cpp
auto highPriority = static_cast<bool>(features(w)[5]);
auto sum = static_cast<Matrix>(m1 + m2 + m3 + m4);
```
* 这种手法不局限于代理类的转换，也可以用于任何希望推断类型不同于表达式类型的情况
```cpp
double calcEpsilon(); // 计算差值的函数
auto ep = static_cast<float>(calcEpsilon());
```
