# std::chrono

长久以来，C++标准库都缺了一样重要的工具：Time library，这种情况终于在C++标准委员会指定C++11时得到了改变。那就是`chrono`，`chrono`是一个优雅的，轻量级，功能强大，强类型的时间库。

chrono的特点如下

* 轻量级，比如`chrono::duration`实际就是一个`long long`类型的整数而已。
* 强类型，比如`seconds`就是`seconds`
* 支持自定义

