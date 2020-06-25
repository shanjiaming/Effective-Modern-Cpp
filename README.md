*[Effective Modern C++](https://learning.oreilly.com/library/view/effective-modern-c/9781491908419/)* 主要讲述了 C++11/14 新特性的用法，阐述从两方面展开，一是原理解析，二是注意事项。如 auto 的原理使用的是模板推断机制，进而介绍模板的推断机制，再由此引出 auto 在使用上应该注意的问题。不同读者在这本书中可以各取所需，如果希望了解 C++11/14 的方方面面，则可以抽丝剥茧地深究原理，如果希望短时间内快速上手 C++11/14 工程实践，则只需略读原理但牢记结论，使用时绕开易产生问题的用法即可。此为个人笔记，条款非直译，而是个人理解的更贴近作者意图的表述。

## [1. 类型推断](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/01%20%E7%B1%BB%E5%9E%8B%E6%8E%A8%E6%96%AD.md)

* 01 模板类型推断机制
* 02 [auto](https://en.cppreference.com/w/cpp/language/auto) 类型推断机制
* 03 [decltype](https://en.cppreference.com/w/cpp/language/decltype)
* 04 查看推断类型的方法

## [2. auto](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/02%20auto.md)

* 05 用 [auto](https://en.cppreference.com/w/cpp/language/auto) 替代显式类型声明
* 06 [auto](https://en.cppreference.com/w/cpp/language/auto) 推断出非预期类型时，先强制转换出预期类型

## [3. 转向现代 C++](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/03%20%E8%BD%AC%E5%90%91%E7%8E%B0%E4%BB%A3C%2B%2B.md)

* 07 创建对象时注意区分 () 和 {}
* 08 用 [nullptr](https://en.cppreference.com/w/cpp/language/nullptr) 替代 0 和 [NULL](https://en.cppreference.com/w/cpp/types/NULL)
* 09 用 [using 别名声明](https://en.cppreference.com/w/cpp/language/type_alias)替代 [typedef](https://en.cppreference.com/w/cpp/language/typedef)
* 10 用 [enum class](https://en.cppreference.com/w/cpp/language/enum#Scoped_enumerations) 替代 [enum](https://en.cppreference.com/w/cpp/language/enum#Unscoped_enumeration)
* 11 用 =delete 替代 private 作用域来禁用函数
* 12 用 [override](https://en.cppreference.com/w/cpp/language/override) 标记被重写的虚函数
* 13 用 [std::cbegin](https://en.cppreference.com/w/cpp/iterator/begin) 和 [std::cend](https://en.cppreference.com/w/cpp/iterator/end) 获取 const_iterator
* 14 用 [noexcept](https://en.cppreference.com/w/cpp/language/noexcept_spec) 标记不抛异常的函数
* 15 用 [constexpr](https://en.cppreference.com/w/cpp/language/constexpr) 表示编译期常量
* 16 用 [std::mutex](https://en.cppreference.com/w/cpp/thread/mutex) 或 [std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic) 保证 const 成员函数线程安全
* 17 特殊成员函数的隐式合成与抑制机制

## [4. 智能指针](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/04%20%E6%99%BA%E8%83%BD%E6%8C%87%E9%92%88.md)

* 18 用 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 管理所有权唯一的资源
* 19 用 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 管理所有权可共享的资源
* 20 用 [std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr) 观测 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 的内部状态
* 21 用 [std::make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique)（[std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)） 创建 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)（[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)）
* 22 用 [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) 实现 [pimpl 手法](https://en.cppreference.com/w/cpp/language/pimpl)必须在 .cpp 文件中提供析构函数定义

## [5. 右值引用、移动语义和完美转发](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/05%20%E5%8F%B3%E5%80%BC%E5%BC%95%E7%94%A8%E3%80%81%E7%A7%BB%E5%8A%A8%E8%AF%AD%E4%B9%89%E5%92%8C%E5%AE%8C%E7%BE%8E%E8%BD%AC%E5%8F%91.md)

* 23 [std::move](https://en.cppreference.com/w/cpp/utility/move) 和 [std::forward](https://en.cppreference.com/w/cpp/utility/forward) 只是一种强制类型转换
* 24 转发引用与右值引用的区别
* 25 对右值引用使用 [std::move](https://en.cppreference.com/w/cpp/utility/move)，对转发引用使用 [std::forward](https://en.cppreference.com/w/cpp/utility/forward)
* 26 避免重载使用转发引用的函数
* 27 重载转发引用的替代方案
* 28 引用折叠
* 29 移动不比拷贝快的情况
* 30 无法完美转发的类型

## [6. lambda 表达式](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/06%20lambda%E8%A1%A8%E8%BE%BE%E5%BC%8F.md)

* 31 捕获的潜在问题
* 32 用初始化捕获将对象移入闭包
* 33 用 [decltype](https://en.cppreference.com/w/cpp/language/decltype) 获取 auto&& 参数类型以 [std::forward](https://en.cppreference.com/w/cpp/utility/forward)
* 34 用 lambda 替代 [std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)

## [7. 并发 API](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/07%20%E5%B9%B6%E5%8F%91API.md)

* 35 用 [std::async](https://en.cppreference.com/w/cpp/thread/async) 替代 [std::thread](https://en.cppreference.com/w/cpp/thread/thread)
* 36 用 [std::launch::async](https://en.cppreference.com/w/cpp/thread/launch) 指定异步求值
* 37 RAII 线程管理
* 38 [std::future](https://en.cppreference.com/w/cpp/thread/future) 的析构行为
* 39 用 [std::promise](https://en.cppreference.com/w/cpp/thread/promise) 和 [std::future](https://en.cppreference.com/w/cpp/thread/future) 之间的通信实现一次性通知
* 40 [std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic) 提供原子操作，volatile 禁止优化内存

## [8. 其他轻微调整](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/08%20%E5%85%B6%E4%BB%96%E8%BD%BB%E5%BE%AE%E8%B0%83%E6%95%B4.md)

* 41 对于可拷贝的形参，如果移动成本低且一定会被拷贝则考虑传值
* 42 用 emplace 操作替代 insert 操作
