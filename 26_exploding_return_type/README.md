# 爆炸式返回类型(Exploding return type)

##目的

用于通过错误代码或抛出异常来返回的一种惯用法

##别名
##动机

这种惯用法的好处是可以让函数或者类方法根据不同的情况返回不同的类型，从而提高程序的灵活性和扩展性，降低耦合性，提高自适应能力 。例如，可以根据错误码或者异常来返回不同的类型，从而让调用者可以选择合适的处理方式。另外，可以利用C++11的移动语义来提高性能和避免不必要的拷贝。

##解决方案及示例

```cpp
// 返回错误代码或抛出异常的函数
int foo()
{
  // Do something
  if (error_condition)
  {
    // Return an error code
    return -1;
  }
  else if (exception_condition)
  {
    // Throw an exception
    throw std::runtime_error("Something bad happened");
  }
  else
  {
    // Return a normal value
    return 0;
  }
}

//使用这个惯用法的调用函数
void bar()
{
  try
  {
    // 调用foo()并检查错误代码
    if (foo() == -1)
    {
      // 处理错误情况
      std::cerr << "Error occurred\n";
    }
    else
    {
      //处理正常情况
      std::cout << "Success\n";
    }
  }
  catch (const std::exception& e)
  {
    //处理异常情况
    std::cerr << "Exception occurred: " << e.what() << "\n";
  }
}
```

##已知的应用
##相关的惯用法
##参考

[Choose Your Poison](http://accu.org/content/conf2007/Alexandrescu-Choose_Your_Poison.pdf)

[Overload no. 53](https://en.wikibooks.org/wiki/Category%3A)