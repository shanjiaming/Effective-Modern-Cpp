[《Effective Modern C++》](https://learning.oreilly.com/library/view/effective-modern-c/9781491908419/)主要讲述了C++11/14新特性的用法，并深入地解析了其原理。

## [1. 类型推断](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/01%20%E7%B1%BB%E5%9E%8B%E6%8E%A8%E6%96%AD.md)
 * 01 理解模板类型推断
 * 02 理解auto类型推断
 * 03 理解decltype
 * 04 查看推断类型的方法

## [2. auto](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/02%20auto.md)
* 05 使用auto替代显式类型声明
* 06 auto推断出非预期类型时，使用explicitly typed initializer手法

## [3. 转向现代C++](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/03%20%E8%BD%AC%E5%90%91%E7%8E%B0%E4%BB%A3C%2B%2B.md)
* 07 创建对象时注意区分()和{}
* 08 使用nullptr替代0和NULL
* 09 使用别名声明替代typedef
* 10 使用限定作用域enum替代非限定作用域enum
* 11 使用=delete替代private未定义函数
* 12 将要重写的函数声明为override
* 13 使用const_iterator替代iterator
* 14 将不抛异常的函数声明为noexcept
* 15 尽可能使用constexpr
* 16 保证const成员函数线程安全
* 17 理解特殊成员函数的生成机制

## [4. 智能指针](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/04%20%E6%99%BA%E8%83%BD%E6%8C%87%E9%92%88.md)
* 18 对专属所有权的资源管理使用std::unique_ptr
* 19 对共享所有权的资源管理使用std::shared_ptr
* 20 使用std::weak_ptr替代可能空悬的std::shared_ptr
* 21 使用std::make_unique和std::make_shared替代new
* 22 使用pimpl手法时，将特殊成员函数定义在实现文件中

## [5. 右值引用、移动语义和完美转发](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/05%20%E5%8F%B3%E5%80%BC%E5%BC%95%E7%94%A8%E3%80%81%E7%A7%BB%E5%8A%A8%E8%AF%AD%E4%B9%89%E5%92%8C%E5%AE%8C%E7%BE%8E%E8%BD%AC%E5%8F%91.md)
* 23 理解std::move和std::forward
* 24 区分转发引用和右值引用
* 25 对右值引用使用std::move，对转发引用使用std::forward
* 26 避免重载转发引用
* 27 重载转发引用的替代方案
* 28 理解引用折叠
* 29 移动操作缺乏优势的情形：不存在、高成本、不可用
* 30 完美转发的失败情形

## [6. lambda表达式](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/06%20lambda%E8%A1%A8%E8%BE%BE%E5%BC%8F.md)
* 31 避免默认捕获模式
* 32 使用初始化捕获将对象移入闭包
* 33 对auto&&类型形参使用decltype来std::forward
* 34 使用lambda替代std::bind

## [7. 并发API](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/07%20%E5%B9%B6%E5%8F%91API.md)
* 35 使用std::async替代std::thread
* 36 需要异步则指定std::launch::async
* 37 让std::thread对象在所有路径上不可合并（unjoinable）
* 38 注意多变的线程句柄析构函数行为
* 39 对一次性事件通信使用void future
* 40 对并发使用std::atomic，对特殊内存使用volatile

## [8. 其他轻微调整](https://github.com/downdemo/Effective-Modern-Cpp/tree/master/content/08%20%E5%85%B6%E4%BB%96%E8%BD%BB%E5%BE%AE%E8%B0%83%E6%95%B4.md)
* 41 对于可拷贝的形参，如果移动成本低且一定会被拷贝则考虑传值
* 42 使用emplace操作替代insert操作
