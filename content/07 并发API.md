> ## 35 使用std::async替代std::thread
* 异步运行函数的一种选择是，创建一个std::thread来运行，这称为基于线程的（thread-based）方法
```cpp
int doAsyncWork();
std::thread t(doAsyncWork);
```
* 另一种是使用std::async，这称为基于任务的（task-based）方法，传递给std::async的函数对象就是一个任务（task）
```cpp
auto fut = std::async(doAsyncWork); // "fut" for "future"
```
* 使用std::async通常比std::thread更好。如果函数有返回值，std::thread无法直接获取该值，而std::async返回的std::future提供了get函数。如果函数抛出异常，get函数能访问异常，而std::thread会调用std::terminate终止程序
* std::async有着更高的抽象，它避开了线程管理的细节。在并发的C++软件中，线程有三种含义：
  * hardware thread是实际执行计算的线程，计算机体系结构中会为每个CPU内核提供一个或多个硬件线程
  * software thread（OS thread或system thread）是操作系统实现跨进程管理，并执行硬件线程调度的线程
  * std::thread是C++进程中的对象，用作底层software thread的handle
* 软件线程是一种有限资源，如果试图创建的线程超出系统所能提供的数量，就会抛出std::system_error异常。这在任何时候都是确定的，即使要运行的函数不能抛异常
```cpp
int doAsyncWork() noexcept;
std::thread t(doAsyncWork); // 若无线程可用，仍会抛出异常
```
* 解决这个问题的一个方法是在当前线程中运行函数，但这会导致负载不均衡，而且如果当前线程是一个GUI线程，将导致无法响应。另一个方法是等待已存在的软件线程完成工作后再新建std::thread，但一种可能的问题是，已存在的软件线程在等待函数执行某个动作
* 即使没有用完线程也可能发生oversubscription的问题，即准备运行（非阻塞）的软件线程数量超过了硬件线程。此时，线程调度器会为软件线程在硬件线程上分配CPU时间片。当一个线程的时间片用完，另一个线程启动时，就会发生语境切换。这种语境切换会增加系统的线程管理开销，尤其是调度器切换到不同的CPU core上的硬件线程时会产生巨大开销。此时，软件线程通常不会命中CPU cache（即它们几乎不含有对该软件线程有用的数据和指令），CPU core运行的新软件线程还会污染cache上为旧线程准备的数据，旧线程曾在该CPU core上运行过，并很可能再次被调度到此处运行
* 避免oversubscription很困难，因为软件线程和硬件线程的最佳比例取决于软件线程变为可运行状态的频率，而这是会动态变化的，比如一个程序从I/O密集型转换计算密集型。软件线程和硬件线程的最佳比例也依赖于语境切换的成本和使用CPU cache的命中率，而硬件线程的数量和CPU cache的细节（如大小、速度）又依赖于计算机体系结构，因此即使在一个平台上避免了oversubscription也不能保证在另一个平台上同样有效
* 使用std::async则可以把oversubscription的问题丢给别人解决
```cpp
auto fut = std::async(doAsyncWork); // 由标准库的实现者负责线程管理
```
* 这个调用把线程管理的责任转交给了标准库实现。如果申请的软件线程多于系统可提供的，系统不保证会创建一个新的软件线程。相反，它允许调度器把函数运行在请求函数结果的线程中（如对fut调用了get或wait的线程）
* 即使使用std::async，GUI线程的响应性也仍然存在问题，因为调度器无法得知哪个线程迫切需要响应。这种情况下，可以将std::async的启动策略设定为std::lanuch::async，这样可以保证函数会在另一个线程中运行
* std::async分担了手动管理线程的负担，并提供了检查异步执行函数的结果的方式，但仍有几种不常见的情况直接使用线程更合适：
  * 需要访问底层线程实现的API：C++并发API通常会基于系统的底层API（如pthread或Windows线程库）实现，std::thread提供了native_handle成员函数，它返回实现定义的底层线程handle，而std::async返回的std::future则没有对应功能
  * 需要为应用优化线程用法：比如开发一个服务器软件，运行时的profile已知并作为唯一的主进程部署在硬件特性固定的机器上
  * 需要实现超出C++并发API的线程技术：比如在C++未提供线程池实现的平台上实现线程池

> ## 36 需要异步则指定std::launch::async
* std::async有两种标准启动策略：
  * std::launch::async：函数必须异步运行，即运行在不同的线程上
  * std::launch::deferred：函数只在返回的std::future调用get或wait时运行。即执行会推迟，调用get或wait时函数会同步运行，调用方会阻塞至函数运行结束
* std::async的默认启动策略并不是两者之一，而是对两者进行或运算的结果。下面两个调用等价
```cpp
auto fut1 = std::async(f); // 意义同下
auto fut2 = std::async(std::launch::async | std::launch::deferred, f);
```
* 默认启动策略允许异步或同步运行函数，这种灵活性使得std::async和标准库的线程管理组件能负责线程的创建和销毁、负载均衡以及避免oversubscription
* 但默认启动std::async会触发一些潜在问题，比如给定线程t执行如下语句
```cpp
auto fut = std::async(f);
```
* 潜在的问题有：
  * 无法预知f和t是否会并发运行，因为f可能被调度为推迟运行
  * 无法预知f运行的线程是否与调用fut的get或wait的线程不同，如果调用get或wait的线程是t，就说明无法预知f是否会运行在与t不同的某线程上
  * 甚至很可能无法预知f是否会运行，因为无法保证在程序的每条路径上，fut的get或wait会被调用
* 默认启动策略在调度上的灵活性会在使用thread_local变量时导致混淆，这意味着如果f读写此thread-local
storage（TLS）时，无法预知哪个线程的局部变量将被访问：
```cpp
auto fut = std::async(f); // f的TLS可能和一个独立线程相关
                          // 但也可能和调用fut的get或wait的线程相关
```
* 它也会影响使用timeout的wait-based循环，因为对任务调用wait_for或wait_until会产生std::launch::deferred值。这意味着以下循环看似最终会终止，但实际可能永远运行
```cpp
using namespace std::literals; // C++14的duration suffixes

void f() // f睡眠1秒后返回
{
    std::this_thread::sleep_for(1s);
}
auto fut = std::async(f);
while (fut.wait_for(100ms) != std::future_status::ready)
{ // 循环至f运行完成，但这可能永远不会发生
    …
}
```
* 如果选用了std::launch::async启动策略，f和调用std::async的线程并发执行，则没有问题。但如果f被推迟执行，则fut.wait_for总会返回std::future_status::deferred，于是循环永远不会终止
* 这类bug在开发和单元测试时很容易被忽略，只有在运行负载很重时才会被发现。解决方法很简单，检查std::async返回的future，确定任务是否被推迟。但没有直接检查是否推迟的方法，替代的手法是，先调用一个timeout-based函数，比如wait_for，这并不表示想等待任何事，而只是为了查看返回值是否为std::future_status::deferred
```cpp
auto fut = std::async(f);
if (fut.wait_for(0s) == std::future_status::deferred) // 任务被推迟
{
    … // 使用fut的wait或get异步调用f
}
else // 任务未被推迟
{
    while (fut.wait_for(100ms) != std::future_status::ready)
    {
        … // 任务未被推迟也未就绪，则做并发工作直至结束
    }
    … // fut准备就绪
}
```
* 综上，std::async使用默认启动策略能正常工作需要满足以下所有条件：
  * 任务不需要与调用get或wait的线程并发执行
  * 读写哪个线程的thread_local变量没有影响
  * 要么保证调用get或wait，要么接受任务可能永远不执行
  * 使用wait_for或wait_until的代码会考虑任务被推迟的可能
* 只要一点不满足，就可能意味着想确保异步执行任务，这只需要指定启动策略为std::launch::async
```cpp
auto fut = std::async(std::launch::async, f); // 异步执行f
```
* 如果一个函数行为和std::async相同，但自动使用std::launch::async启动策略，将会是一个很方便的工具，而这个函数并不难写，下面是C++11版本的实现
```cpp
template<typename F, typename... Ts>
inline
std::future<typename std::result_of<F(Ts...)>::type>
reallyAsync(F&& f, Ts&&... params) // 返回异步调用f需要的future
{
    return std::async(std::launch::async,
        std::forward<F>(f),
        std::forward<Ts>(params)...);
}
```
* 这个函数的用法和std::async一样
```cpp
auto fut = reallyAsync(f); // 异步运行f，如果std::async抛出异常则reallyAsync也抛出异常
```
* C++14中，对返回值使用类型推断可以简化代码
```cpp
template<typename F, typename... Ts>
inline
auto // C++14
reallyAsync(F&& f, Ts&&... params)
{
    return std::async(std::launch::async,
        std::forward<F>(f),
        std::forward<Ts>(params)...);
}
``` 

> ## 37 让std::thread对象在所有路径上不可合并（unjoinable）
* 每个std::thread对象都处于可合并或不可合并的状态。一个可合并的std::thread对应于一个底层异步运行的线程，若底层线程处于阻塞、等待调度或已运行结束的状态，则此std::thread可合并，若不处于这种状态则std::thread不可合并。不可合并的std::thread包括：
  * 默认构造的std::thread：此时没有要运行的函数，因此没有对应的底层运行线程
  * 已移动的std::thread：移动操作导致一个std::thread的底层线程被用于另一个std::thread
  * 已join的std::thread
  * 已detach的std::thread
* std::thread能否合并的重要性在于：如果可合并的线程对象的析构函数被调用，则程序的执行将终止。比如有一个函数doWork，参数为一个筛选器函数filter和一个最大值maxVal，doWork检查计算条件全部成立后，对筛选器选出的0到最大值之间的值进行计算。筛选和检查计算条件应当并发处理
* 优先会选择std::async，但假设要设置筛选操作的线程的优先级，这要求使用线程的底层handle，而这只能通过std::thread的API获取，因此只能使用std::thread
```cpp
constexpr auto tenMillion = 10'000'000; // C++14允许单引号分隔数字增强可读性
bool doWork(std::function<bool(int)> filter, int maxVal = tenMillion)
{
    std::vector<int> goodVals; // 筛选出的值
    std::thread t(
        [&filter, maxVal, &goodVals] // 遍历goodVals
        {
            for (auto i = 0; i <= maxVal; ++i)
            { if (filter(i)) goodVals.push_back(i); }
        });
    auto nh = t.native_handle(); // 使用底层handle设置优先级
    …
    if (conditionsAreSatisfied())
    {
        t.join(); // 让t结束执行
        performComputation(goodVals);
        return true; // 计算已执行
    }
    return false; // 计算未执行
}
```
* 如果conditionsAreSatisfied()返回true则不会产生问题，但如果返回false或抛出异常，则在doWork要结束并调用t的析构函数时，t是可合并状态，这将导致程序终止。std::thread的析构函数做出这种行为的原因是，另外两种选项问题更大：
  * 隐式join：此时std::thread的析构函数会等待它的底层异步线程完成运行，这听起来合理，但可能导致难以跟踪的性能异常。比如如果conditionsAreSatisfied()已经返回false，doWork仍会等待筛选器遍历所有值，这是违反直觉的
  * 隐式detach：此时std::thread的析构函数会分离std::thread对象和底层线程的连接，而底层线层会继续运行。这听起来也很合理，但导致的调试问题更加致命。比如，doWork内的goodVals是通过引用捕获的局部变量，它会在lambda中被修改（通过调用push_back），如果lambda异步运行时conditionsAreSatisfied()返回false，doWork将直接返回，局部变量被销毁，doWork栈帧被弹出，但线程仍然在doWork的调用方运行。在调用方的doWork之后的语句中，某个时刻调用其他函数，可能某个函数将使用一部分doWork栈帧用过的内存。假设这个函数是f，f运行时，doWork的lambda仍在异步运行，lambda在原先的栈上对goodVals调用push_back，修改了之前goodVals所在的内存，这在f看来，栈帧上的内存内容就出现了莫名其妙的变化
* 标准委员会认识到销毁一个可合并的线程十分可怕，所以规定可合并的线程的析构函数将导致程序终止，以此来规避这件事。这样，std::thread的使用者就要保证从它定义的作用域所经过的所有路径上，都处于不可合并的状态。但覆盖所有路径很复杂，这包括正常遍历所有作用域，以及通过return、continue、break、goto或异常跳出作用域
* 想在每条路径上执行某个动作，最常用的方法就是在局部对象的析构函数上执行此动作，这样的对象就是RAII对象，比如STL容器、智能指针、std::fstream对象。std::thread对象没有对应的标准RAII类，但自己写一个并不难
```cpp
class ThreadRAII {
public:
    enum class DtorAction { join, detach };
    ThreadRAII(std::thread&& t, DtorAction a) // std::thread对象是不可拷贝的
    : action(a), t(std::move(t)) {}
    ~ThreadRAII()
    {
        if (t.joinable())
        {
            if (action == DtorAction::join) t.join();
            else t.detach();
        }
    }
    // 析构函数会抑制合成移动操作，因此需要显式声明
    ThreadRAII(ThreadRAII&&) = default;
    ThreadRAII& operator=(ThreadRAII&&) = default;
    std::thread& get() { return t; }
private:
    DtorAction action;
    std::thread t;
};
```
* 在doWork中使用ThreadRAII
```cpp
bool doWork(std::function<bool(int)> filter, int maxVal = tenMillion)
{
    std::vector<int> goodVals;
    ThreadRAII t( // 使用RAII对象
        std::thread(
            [&filter, maxVal, &goodVals]
            {
                for (auto i = 0; i <= maxVal; ++i)
                { if (filter(i)) goodVals.push_back(i); }
            }),
        ThreadRAII::DtorAction::join // RAII action
    );
    auto nh = t.get().native_handle();
    …
    if (conditionsAreSatisfied())
    {
        t.get().join();
        performComputation(goodVals);
        return true;
    }
    return false;
}
```
* 这里选择在ThreadRAII析构函数中对异步执行线程调用join，虽然join会导致性能异常，但比起detach会导致未定义行为的调试噩梦，以及只使用std::thread导致的程序终止，性能异常是权衡之下的最佳选择

> ## 38 线程handle的析构函数的不同行为
* 可合并的线程对应一个底层系统线程，未推迟的任务的future和系统线程也有类似的关系，因此可以认为std::thread和future相当于系统线程的handle
* 销毁可合并的std::thread对象会导致程序终止，另外两个选择（隐式join和隐式detach）则更糟。销毁future有时表现为隐式join，有时表现为隐式detach，有时表现为既不隐式join也不隐式detach，但它不会导致程序终止。这套线程handle行为的不同表现是值得需要思考的
* 想象future处于信道的一端，callee通过借助一个std::promise对象把结果传给caller，caller用一个future来读取结果

![](https://upload-images.jianshu.io/upload_images/5587614-988a3d33b1ceb7ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 考虑问题，callee的结果存储在哪？caller调用get之前，callee可能已经执行完毕，因此结果不可能存储在callee的std::promise对象中。但结果也不可能存储在caller的future中，因为可能会用std::future创建std::shared_future，而std::shared_future在原始的std::future析构后仍然可以复制。因此结果只能存储在外部某个位置，这个位置称为shared state

![](https://upload-images.jianshu.io/upload_images/5587614-532d9df6e30a7b55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* shared state通常用堆上的对象表示，但类型、接口和具体实现由标准库作者决定。shared state很重要，它决定了future的析构函数的行为：
  * 通过std::aysnc创建的一个非推迟任务的future中，最后一个引用shared state的，析构函数会保持阻塞，直到任务完成。本质上，这样一个future的析构函数是对异步运行的底层线程执行了一次隐式join
  * 其他所有future的析构函数只是简单地析构future对象。对底层异步运行的任务，这相当于对线程执行了一次隐式detach。对于被推迟的任务来说，如果这是最后一个future，就意味着被推迟的任务将不会再运行
* 这些规则看似复杂，但本质就是一个正常行为和一个特殊行为。正常行为是future的析构函数会销毁future对象，它不join或detach任何东西，也没有运行任何东西，它只是销毁 future的成员变量。不过实际上它确实多做了一件事，就是减少了一次shared state中的引用计数，shared state由caller的future和callee的std::promise共同操控。引用计数让库得知何时能销毁shared state
* 特殊行为只在future满足以下所有条件时发生：
  * future引用的shared state由调用std::async创建
  * 任务的启动策略是std::launch::async，这可能是运行时系统选择的，或调用std::async时指定的
  * 这个future是最后一个引用shared state的future。对于std::future这点总成立，而对于std::shared_future，如果析构时不是最后一个引用shared state的future，则会被正常地析构
* 只有满足以上所有条件，future的析构函数才会表现出特殊行为：阻塞直到异步运行的任务结束。效果上，这相当于对运行着std::async创建的任务的线程执行了一次隐式join。特别制定这个规则的原因是，标准委员会想避免隐式detach相关的问题，但又不想对可合并的线程一样直接让程序终止，于是妥协的结果就是执行一次隐式join
* future没有提供API来判断shared state是否产生于std::async的调用，因此future对象无法得知自己是否会在析构函数中阻塞到异步任务执行结束，这有一些有趣的推论
```cpp
// this container might block in its dtor, because one or more contained futures
// could refer to a shared state for a non-deferred task launched via std::async
std::vector<std::future<void>> futs;

class Widget { // Widget类型对象可能会在析构函数中阻塞
public:
    ...
private:
    std::shared_future<double> fut;
};
```
* 如果有办法知道给定的future不满足触发特殊析构行为的条件（比如通过分析程序逻辑）， 即可断定future不会阻塞在析构函数中。例如，只有在std::async调用时出现的shared state才有表现特殊行为的资格，但还有其他方法可以创建出shared state，比如std::packaged_task。std::packaged_task类型对象会准备一个函数（或其他可调用对象）用于异步执行，并把结果放在shared state中。引用此shared state的future可以通过std::packaged_task的get_future成员获取结果
```cpp
int calcValue();
std::packaged_task<int()> pt(calcValue);
auto fut = pt.get_future(); // 获取pt的future
```
* 此时可知fut没有引用由std::async调用产生的shared state，所以它的析构函数将会表现出正常行为
* std::packaged_task类型对象一旦被创建，就会被运行在线程上（也可以由std::async的调用而运行，但如果要用std::async运行任务，就没必要创建一个std::packaged_task，因为std::async能在调度任务前就做到std::packaged_task能做的任何事情）
* std::packaged_task不能被拷贝，所以把pt传递给std::thread构造函数时必须转换成右值
```cpp
std::thread t(std::move(pt)); // 在t上运行pt
```
* 由此可以看出一些future的正常析构行为，如果把这些语句放在同一个块中就更容易看出来
```cpp
{
    std::packaged_task<int()>
    pt(calcValue);
    auto fut = pt.get_future();
    std::thread t(std::move(pt));
    …
}
```
* 在省略号位置，t可能做的事有三种：
  * t不做任何事。此时t在作用域结束前是可合并的，这将导致程序终止
  * t执行了join操作。此时fut就不需要在析构时阻塞，因为调用的代码已经join过了
  * t执行了detach操作。此时fut就不需要在析构时detach，因为调用的代码已经detach过了
* 换句话说，当future所对应的shared state是由std::packaged_task产生的，通常不需要采用特殊析构策略，因为操纵std::thread的代码会在终止、join、detach之间做出决定，而std::packaged_task通常运行在此std::thread之上 

> ## 39 对一次性事件通信使用void future
* 让一个任务通知另一个异步任务发生了特定事件，这种功能是有用的，因为第二个任务可能在事件发生前无法推进。这个事件可能是某个数据结构完成了初始化，可能是完成了某个计算，可能是检测到了某个传感器的值
* 实现这种通知的一种明显方法是使用条件变量。检测条件的任务称为检测任务（detecting task），响应条件的任务称为响应任务（reacting task），响应任务会等待一个条件变量，而检测任务在事件发生时通知条件变量
```cpp
std::condition_variable cv; // 事件的条件变量
std::mutex m; // 使用cv时加上的互斥量
```
* 这里假设只有一个反应任务，如果有多个则要用notify_all替代notify_one
```cpp
… // 检测事件
cv.notify_one(); // 通知响应任务
```
* 响应任务在条件变量调用wait等待通知之前，必须用std::unique_lock锁定mutex
```cpp
… // 准备响应
{
    std::unique_lock<std::mutex> lk(m);
    cv.wait(lk); // 等待通知，但这里会出错的
    … // 响应事件（m被锁定）
} // 析构lk以解锁m
… // 继续等待响应（m已解锁）
```
* 这种方法的首要问题有时也叫代码异味（code smell），即代码虽然暂时可以运行，但有些东西看起来不太对劲。上例中，代码异味源于需要使用mutex。mutex是用来控制共享数据的访问的，但检测任务和响应任务之间不需要这样的媒介。比如，检测任务负责初始化一个全局数据结构，并转交给响应任务使用，如果检测任务之后从不访问该数据，并且响应任务在得知数据就绪之前也不会访问数据，则程序逻辑上这两个任务就会互相阻止对方访问，完全不需要使用mutex
* 此外，还有两个问题需要关心：
  * 如果检测任务在响应任务调用wait之前就通知了条件变量，则响应任务将错过通知而失去响应
  * 响应任务无法应对虚假唤醒。线程API的一个事实是，即使没有通知条件变量，等待该条件变量的代码也可能被唤醒。处理这个问题的适当方法是，确认等待的条件已经发生，并将其作为唤醒后的首个动作。C++的条件变量API做到这点很简单，它允许将测试等待条件的lambda传递给wait
```cpp
cv.wait(lk,
    []{ return whether the event has occurred; });
```
* 要利用这个优点，就要求响应任务能确定等待的条件是否成立。但考虑的上述场景中，由检测线程负责识别等待的条件的发生，响应线程可能无法确定等待的事件是否发生，这正是它等待这个条件变量的原因
* 使用条件变量进行任务间的通信适用于许多场景，但现在这个问题似乎并不适用。许多开发者的一个trick是使用共享的bool标志位，它默认为false，当检测线程识别出查找的事件时会设置该标志位
```cpp
std::atomic<bool> flag(false); // 共享的bool标志位
… // 检测事件
flag = true; // 通知响应任务
```
* 响应线程简单地轮询标志位，一旦看到标志位被设置，就知道它正在等待的事已发生
```cpp
… // prepare to react
while (!flag); // wait for event
… // react to event
```
* 这种方法很好，不需要mutex，也不会有虚假唤醒，唯一的缺点是响应任务的轮询开销大。任务等待标志位被设置时，实际应该被阻塞，但它仍在运行，因此它占用了另一个任务本应该可以利用的硬件线程，而且在每次开始运行和时间片结束时都会有语境切换的成本。它还可能让一个本来可以关掉以节省电量的core保持运行。而阻塞的任务不会有这些开销，这也是条件变量的优点
* 常用的手法是结合条件变量和标志位，标志位表示关注的事件是否发生了，但在访问这个标志位前要通过一个mutex来同步，因为mutex会阻止并发访问该标志位。因为不是并发访问，该标志位不需要使用std::atomic，简单的bool就行了
```cpp
// 检测任务
std::condition_variable cv;
std::mutex m;
bool flag(false);
… // 检测事件
{
    std::lock_guard<std::mutex> g(m);
    flag = true; // 通知响应任务（part 1）
}
cv.notify_one(); // 通知响应任务（part 2）

// 响应任务
… // 准备响应
{
    std::unique_lock<std::mutex> lk(m);
    cv.wait(lk, [] { return flag; }); // 使用lambda防止虚假唤醒
    … // 对事件进行响应（m被锁定）
}
…
```
* 这个方法可以避免之前的问题，但还有一点代码异味，因为检测任务和响应任务的沟通方式很奇怪。检测任务通知条件变量以告诉响应任务等待的事件可能已经发生，但响应任务还要检查标志位才能确定。这个方法可行，但是不够干净利落
* 另一种方法是让响应任务等待检测任务设置的future。虽然检测任务和响应任务并非之前提过的callee-caller的关系，但std::promise到future的通信不仅限于caller-callee，而可以用于任何需要将信息从一处传输到另一处的地方
* 实际上，这里并没有需要传输的数据，对响应任务唯一有意义的事情只是future是否已被设置，因此std::promise和future只需要指定void类型
```cpp
std::promise<void> p;
… // 检测事件
p.set_value(); // 通知响应任务
… // 准备响应
p.get_future().wait();
… // 对事件进行响应
```
* 这种方法有标志位的优点，避免了mutex、虚假唤醒等问题。也有条件变量的优点，调用wait后响应任务被阻塞，等待时不消耗系统资源
* 但这种方法也有自身的缺陷，std::promise和future之间是shared state，而shared state通常是动态分配的，这可能导致堆上的分配和回收成本
* 最重要的一点是，std::promise对象只能设置一次，它和future之间的通信信道是一次性的，不能重复使用，而条件变量和标志位的方法都可以用来进行多次通信（条件变量可被重复通知，标志位可被清除并重设）
* 一次性约束带来的限制并没有想象的那么大，假设要创建暂停状态的系统线程，且只想暂停线程一次，用void future就是一个合理的选择
```cpp
std::promise<void> p;
void react(); // 响应任务的函数
void detect() // 检测任务的函数
{
    std::thread t([] // 创建线程
    {
        p.get_future().wait(); // 暂停t直到future被设置
        react();
    });
    … // 这里t处于暂停状态
    p.set_value(); // 取消暂停t（于是将调用react）
    … // 其他工作
    t.join(); // make t unjoinable
}
```
* 使t在所有detect的出向路径上都不可合并很重要，因此使用之前提及的ThreadRAII是可取的，于是可能会想出如下代码
```cpp
void detect()
{
    ThreadRAII tr(
        std::thread([]
        {
            p.get_future().wait();
            react();
        }),
        ThreadRAII::DtorAction::join // risky! (see below)
    );
    … // tr中的线程在此处暂停
    p.set_value(); // tr中的线程在此取消暂停
    …
}
```
* 这段代码并没有看上去那么安全，问题在于第一个省略号的位置，即线程暂停处，如果抛出异常，set_value将永远不会在p上调用，于是lambda中的wait将永远不会返回，这将导致运行lambda的线程永远不会完成，这是个问题，因为RAII对象配置的析构函数中会对线程执行join。简而言之，如果第一个省略号的位置抛出异常，函数将失去响应，因为tr的析构函数永远不会完成
* 下面不使用ThreadRAII，实现针对不止一个的响应任务执行先暂停再取消暂停的功能。这个实现不难，关键在于在响应代码中使用std::shared_future替代std::future，唯一的微妙之处是每个响应线程都需要自己的std::shared_future来引用shared state，因此响应线程的lambda中用值捕获std::shared_future
```cpp
std::promise<void> p;
void detect()
{
    auto sf = p.get_future().share(); // sf为std::shared_future<void>
    std::vector<std::thread> vt; // 响应线程的容器
    for (int i = 0; i < threadsToRun; ++i)
    {
        vt.emplace_back([sf]{ sf.wait(); react(); });
    }
    … // 如果此处抛异常则detect将失去响应
    p.set_value(); // 让所有线程取消暂停
    …
    for (auto& t : vt)
    {
        t.join(); // make all threads unjoinable
    }
}
```

> ## 40 对并发使用std::atomic，对特殊内存使用volatile
* volatile本来和并发程序设计毫无关系，但有时会和std::atomic混淆
* std::atomic提供的操作可以保证被其他线程视为原子的，如同这些操作处于mutex保护的临界区一样，但实际上这些操作通常会用特殊的机器指令实现，这比mutex更高效。考虑以下代码：
```cpp
std::atomic<int> ai(0);
ai = 10;
std::cout << ai;
++ai;
--ai;
```
* 这些语句的运行期间，其他读取ai的线程可能只会看到0、10、11的取值，而不会有其他可能，但如果使用volatile则不能提供任何保证
```cpp
volatile int vi(0);
vi = 10;
std::cout << vi;
++vi;
--vi;
```
* 运行这段代码时，如果其他线程正在读vi的值，可能会看到任何职，比如-12、68、4090727等任意值，这样的代码存在数据竞争，会出现未定义行为
```cpp
std::atomic<int> ac(0);
volatile int vc(0);
// Thread 1
++ac; ++vc;
// Thread 2
++ac; ++vc;
```
* 上述代码中，两个线程都完成后，ac必定为2，而vc不一定是2，因为它的自增可能不以原子方式发生。自增是RMW（read-modify-write）操作，包括读取vc值，自增读取值，再把结果写回vc，因此两次vc的自增可能会交错进行：
  * 线程1读取vc值为0
  * 线程2读取vc值为0
  * 线程1自增vc为1，并写回vc
  * 线程2自增vc为1，并写回vc
* 这样vc虽然进行了两次自增，但最终值是1，并且这不是唯一可能的结果，因为vc涉及会导致未定义行为的数据竞争，最终值一般无法预测
* 使用RMW操作不是唯一并发时std::atomic能成功而volatile会失败的情况。假设一个任务负责计算另一个任务需要的值，计算完成就会把值传给第二个任务，计算完成的标志可以用std::atomic<bool>表示
```cpp
std::atomic<bool> valAvailable(false);
auto imptValue = computeImportantValue(); // 计算值
valAvailable = true; // 通知其他任务值已可用
```
* 这段代码中，显而易见的是imptValue会在valAvailable之前赋值，这很重要。编译器或底层硬件对于不相关的赋值会重新排序以提高代码运行速度，std::atomic则可以限制编译器和底层硬件的重新排序以保证赋值顺序。而如果使用volatile，就不会限制代码的重新排序
```cpp
volatile bool valAvailable(false);
auto imptValue = computeImportantValue();
valAvailable = true; // 其他线程可能认为此赋值在imptValue之前
```
* 可见volatile对并发编程没用，它的真正用处是告诉编译器，正在处理的内存的行为不是常规行为。常规内存的特征是，如果把一个值写到内存某个位置，值会保留在那里，直到被覆盖。因此，对于如下代码：
```cpp
int x = 42;
auto y = x;
y = x;
```
* 编译器可以消除对y的赋值来优化代码，因为它和y的初始化形成了冗余
* 常规内存还有一个特征，如果把一个值写到内存某个位置但从不读取，然后再次写入该位置，则第一次的写入可被消除。因此，对于如下代码：
```cpp
x = 10;
x = 20;
```
* 编译器可以消除第一个操作，这意味着如果有以下代码：
```cpp
auto y = x;
y = x;
x = 10;
x = 20;
```
* 编译器将优化成：
```cpp
auto y = x;
x = 20;
```
* 这种冗余的代码不会直接被写出来，但往往会隐含在大量代码之中
* 这种优化只在常规内存中合法，特殊内存则不适用。最常见的特殊内存是用于memory-mapped I/O的内存，这种内存中的位置实际上用于与外部设备（如外部传感器、显示器、打印机、网络端口）通信，而非用于读写常规内存（即RAM）。此时再考虑看似冗余的读取操作：
```cpp
auto y = x;
y = x;
```
* x可能对应于温度传感器的值，因此x的第二次读取操作并非多余，因为此时温度可能已经改变。看似冗余的写入操作也是同理：
```cpp
x = 10;
x = 20;
```
* x可能对应于无线电发射器的控制端口，因此有可能是通过代码向无线电发送不同的命令
* volatile的用处就是告诉编译器，正在处理的是特殊内存，不要对此内存上的操作执行任何优化。因此如果x对应于特殊内存，应该用volatile修饰
```cpp
volatile int x;
```
* 特殊内存的行为也解释了std::atomic不适合特殊内存工作的原因。巧合的是，如下语句在x是std::atomic时是错误的：
```cpp
std::atomic<int> x;
auto y = x; // 错误
y = x; // 错误
```
* 原因是std::atomic的拷贝操作被delete了。这个设计是合理的，假设std::atomic能被拷贝，因为x是std::atomic，则y的类型被推断为std::atomic，为了使从x构造y是原子操作，编译器必须生成在单一原子操作中读取x并写入y的代码，而硬件通常无法实现此操作
* 从x中取值再放入y是可实现的，这需要调用std::atomic的成员函数load和store
```cpp
std::atomic<int> y(x.load());
y.store(x.load());
```
* 上述代码也表明，无法将这两条语句结合为单一的原子操作。给定上述代码，编译器可以通过将x的值存储在寄存器中，而不是读取两次，从而达到优化的效果
```cpp
register = x.load(); // 将x读入寄存器
std::atomic<int> y(register); // 用寄存器值初始化y
y.store(register); // 将寄存器值存入y
```
* 这样x的读取操作只执行了一次，这是在处理特殊内存时需要避免的那种优化（这种优化在volatile变量上不被允许）
* 总之，std::atomic对并发编程有用，对访问特殊内存没用，volatile对访问特殊内存有用，对并发编程没用。两者的用处不同，因此也可以一起使用
```cpp
volatile std::atomic<int> vai; // 对val的操作是原子的且不可优化
```
* 如果val对应于一个多线程同时访问的memory-mapped I/O位置，就可能是有用的
* 一些开发者更喜欢用std::atomic的load和store函数，即使没有必要，因为这能显式说明源码中的变量不是常规的，不过访问std::atomic会比非std::atomic慢。从正确性角度看，如果想通过某个变量来输出信息到其他线程，但没看见调用store，就意味着该变量没有声明为std::atomic，但它本应该这样做。这在很大程度上只是一个风格问题，与在std::atomic和volatile之间选择的性质完全不同
