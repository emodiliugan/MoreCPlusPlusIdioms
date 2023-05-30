# Erase-Remove

##目的

从STL容器中删除元素，以减小其大小。

##别名

Remove-Erase

##动机

std::remove算法并不会从容器中删除元素！它只是将不需要删除的元素移动到容器的前面，将末尾的内容留在容器中，这是因为std::remove算法仅使用一对前向迭代器（迭代器对惯用语）和前向迭代器的通用概念无法知道如何从任意数据结构中删除数据元素。只有容器成员函数才能消除容器元素，因为只有成员才知道内部数据结构的细节。使用Erase-Remove惯用语可以真正从容器中删除数据元素。

举个例子(参照effective STL第32条)：

调用remove之前v的布局如下：

![image-20230529151538564](C:\Users\zoumingwh\AppData\Roaming\Typora\typora-user-images\image-20230529151538564.png)

如果我们将remove的返回值保存在一个新的迭代器对象newEnd中

```
vector<int>::iterator newEnd(remove(v.begin(),v.end(),99));
```

那么，在调用remove后，v的布局如下：

![image-20230529151801760](C:\Users\zoumingwh\AppData\Roaming\Typora\typora-user-images\image-20230529151801760.png)

##解决方案及示例

std::remove算法返回一个迭代器，指向“未使用”元素的范围的开头。它不会更改容器的end（）迭代器，也不会更改容器的大小。成员erase函数可按以下惯用方式从容器中真正删除成员。

```
template<typename T>
inline void remove(std::vector<T> & v, const T & item)
{
	v.erase(std::remove(v.begin(), v.end(), item), v.end());
}

std::vector<int> v;
// fill it up somehow
remove(v, 99);// 真正删除值为99的所有元素
```

这里v.end()的评估顺序和std::remove的调用顺序并不重要，因为std::remove算法不会以任何方式更改end()迭代器。

##已知的应用

[boost_remove_erase](http://www.boost.org/doc/libs/1_43_0/libs/range/doc/html/range/reference/
algorithms/new/remove_erase.html)

##相关的惯用法

Effective STL, Item 32 - Scott Meyers

##参考