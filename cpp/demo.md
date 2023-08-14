# const char*/ char const*/ char* const
```char const* ptr/ const char* ptr``` 指向一个char类型的常量，不允许通过该指针去修改变量
```char* const pointer```不能修改ptr指针，但是可以修改指针指向的内容
# 起因
测试gtest的断言``ASSERT_THROW(func, char*)``时，``throw "xxxx"``显示和预期的类型不匹配，最后改成"const char*"才通过测试
