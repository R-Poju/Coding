std::lock_guard 和 unique_lock 的区别



std::lock_guard

行为：构造时立即锁定互斥量，析构时自动解锁。声明周期内锁的状态不可变（始终持有锁）

特点：轻量级设计，无额外开销。不可手动解锁或重新锁定。不可复制或移动（C++11起不可移动）

适用场景：简单的临界区保护（进入作用域加锁，退出自动解锁）



```
{
    std::lock_guard<std::mutex> lock(mutex_); // 构造时锁定
    // 临界区操作...
} // 析构时自动解锁
```



std::unique_lock

行为：提供更灵活的锁管理：可延迟锁定（构造时不加锁）。可手动解锁/重新锁定。支持所有权转移（通过移动语义）

特点：可与条件变量配合使用（std::condition_variable::wait要求传入unique_lock）。支持尝试锁定（try_lock()）、超时锁定（try_lock_for()）。因功能复杂，开销略高于lock_guard

适用场景：需要精细控制锁的场景（如条件变量、跨作用域传递锁所有权、读写锁分离等）

```
{	
	// 延迟锁定（构造时不加锁）
	std::unique_lock<std::mutex> lock(mutex_, std::defer_lock);

	// 手动控制锁定
	lock.lock();
	// 临界区操作...
	lock.unlock(); // 提前解锁

	// 再次锁定
	lock.lock();

} // 析构时若仍持有锁则自动解锁
```



条件变量必须使用unique_lock



如何选择？

当只需简单自动加解锁，追求极致性能时，用 std::lock_guard

当需要灵活控制锁（延迟锁定、手动解锁、条件变量、所有权转移）时，用 std::lock_guard















