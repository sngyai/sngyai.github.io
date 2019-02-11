# 给编写Test Suite作者的帮助：
`ct`模块提供了编写测试用例的主要接口，主要包含：
* 打印输出日志的函数
* 读取配置数据的函数
* 结束一个测试用例，并返回错误原因的函数
* 向HTML总览页面添加评注的函数

`Common Test`应用同样包含了其他名为`ct_<component>`的模块，它们提供了各种支持，主要简化了通信协议的使用，这些通讯协议包括RPC，SNMP，FTP，Telnet等等。

# Test Suite
一个Test Suite是一个包含测试用例的普通Erlang模块，建议如下：
* 模块名称形如`*_SUITE.erl`，否则，Common Test中的目录和自动编译功能无法找到它（至少默认不会）。
* 所有的Test Suite模块都要包含`ct.hrl`。
* 所有的Test Suite模块都要导出`all/0`，该函数返回模块中所有需要执行的Test Case和Test Case Group的列表。
* 一些必要的Common Test回调函数需要实现。

# Test Suite的初始化与结束  
每个Test Suite模块，都可以包含配置函数（也可以不包含）`init_per_suite/1` 和 `end_per_suite/1`。如果定义了`init`，那么也必须要定义`end`。

如果定义了`init_per_suite`， 它将会在所有的测试用例执行之前被调用。它一般包含了SUITE中所有测试用例的通用初始化，这些初始化仅执行一次。建议使用`init_per_suite`来设置和验证被测系统（SUT）或Common Test主机节点的状态和环境，以便SUITE中的测试用例正确执行。以下是初始化配置时的操作的例子：
* 打开到被测系统（SUT）的连接。
* 初始化数据库
* 执行安装脚本

`end_per_suite`在测试套件执行的最后阶段被调用（在最后一个测试用例完成之后）。 该函数用于清理`init_per_suite`中的变动。

# Test Case的初始化与结束
每个Test Suite模块，都可以包含配置函数（也可以不包含）`init_per_testcase/2 ` 和 `end_per_testcase/2`。如果定义了`init`，那么也必须要定义`end`。

如果定义了`init_per_testcase`， 它将会在每个测试用例执行之前被调用。它通常包含为每个测试用例完成的必要的初始化（类似于SUITE中的`init_per_suite`）。

每次测试用例执行完成之后会调用`end_per_testcase/2`，用于清理`init_per_testcase`中的变动。


# 测试用例
测试服务关心的最小单元就是测试用例，每个测试用例里可以测试很多东西，例如：对同一接口函数使用不同参数多次调用。

作者可以选择在一个测试用例中放置或多或少的测试，需要记住以下几点：
* 许多小型测试用例往往导致额外的，可能是重复的代码，以及由于初始化和清理的大量开销而导致测试执行缓慢。避免重复的代码，例如，使用常见的帮助功能。 否则，生成的SUITE变得难以阅读和理解，并且维护起来很麻烦。

* 较大的测试用例使得如果失败则更难找出错误。此外，当发生错误时，大部分测试代码可能会被跳过。

* 当测试用例变得过大和宽泛时，可读性和可维护性会受到影响。 无法确定生成的日志文件是否能够很好地反映所执行的测试数量。


# 测试用例信息函数

# Test Suite信息函数

# 测试用例组
测试用例组是一组共享配置功能和执行属性的测试用例。 测试用例组由函数`group/0`根据以下语法定义：

```erl
 groups() -> GroupDefs

 Types:

 GroupDefs = [GroupDef]
 GroupDef = {GroupName,Properties,GroupsAndTestCases}
 GroupName = atom()
 GroupsAndTestCases = [GroupDef | {group,GroupName} | TestCase]
 TestCase = atom()
```

GroupName是测试用例组的名称，而且在Test Suite模块中必须是唯一的，组可以通过在另一个组的`GroupsAndTestCases`列表中包含一个组的定义来进行嵌套，`Properties`是组的执行属性列表。可能的值如下：

```erl
 Properties = [parallel | sequence | Shuffle | {RepeatType,N}]
 Shuffle = shuffle | {shuffle,Seed}
 Seed = {integer(),integer(),integer()}
 RepeatType = repeat | repeat_until_all_ok | repeat_until_all_fail |
              repeat_until_any_ok | repeat_until_any_fail
 N = integer() | forever
```

释义：
* ***parallel*** `Common Test`并行执行组内的所有测试用例。
* ***sequence*** `Common Test`顺序执行组内的所有测试用例。
* ***shuffle***  `Common Test`以随机顺序执行组内的所有测试用例。
* ***repeat***   令`Common Test`重复执行组内测试用例指定次数，或直到任意测试用例执行失败或直到所有测试用例执行成功。

例子：
```
 groups() -> [{group1, [parallel], [test1a,test1b]},
              {group2, [shuffle,sequence], [test2a,test2b,test2c]}].
```
要指定各个测试用例组以何种顺序执行（同时也是为了照顾那些不属于任何组的测试用例），向`all/0`中添加格式为`{group,GroupName}`的元组。
例如：

```erl
 all() -> [testcase1, {group,group1}, testcase2, {group,group2}].
```
可以通过在`all/0`中添加一个格式为`{group,GroupName,Properties}`的组元组来指定执行属性，这些`Properties`会覆盖`groups/0`中定义的属性。这样一来，相同的测试集能够以不同的属性来执行，而无需对它们额外复制一份再加以修改。

如果一个测试用例组包含子测试用例组，同样能够以`{group,GroupName,Properties,SubGroups}`来指定执行属性，其中`SubGroups`是一个元组的列表，元组格式为`{GroupName,Properties}`或`{GroupName,Properties,SubGroups}`。
所有在`group/0`中定义，而又没有在`SubGroups`列表中指定的子测试用例组，使用它们之前预定义的属性来执行。

例子：

```
 groups() -> {tests1, [], [{tests2, [], [t2a,t2b]},
                           {tests3, [], [t31,t3b]}]}.
```

要让`tests2`与`tests1`执行两次，而每次tests2的属性都不同：

```erl
 all() ->
    [{group, tests1, default, [{tests2, [parallel]}]},
     {group, tests1, default, [{tests2, [shuffle,{repeat,10}]}]}].
```

对于如下配置也同样适用：

```
 all() ->
    [{group, tests1, default, [{tests2, [parallel]},
                               {tests3, default}]},
     {group, tests1, default, [{tests2, [shuffle,{repeat,10}]},
                               {tests3, default}]}].
```
其中，`default`表示要使用预定义的属性。

以下示例展示了如何在深层嵌套中使用覆盖属性：

```
 groups() ->
    [{tests1, [], [{group, tests2}]},
     {tests2, [], [{group, tests3}]},
     {tests3, [{repeat,2}], [t3a,t3b,t3c]}].

 all() ->
    [{group, tests1, default, 
      [{tests2, default,
        [{tests3, [parallel,{repeat,100}]}]}]}].
```

每个测试用例都由特定的Erlang进程执行。 该进程在测试用例启动时生成，测试用例结束时终止。 配置函数`init_per_testcase`和`end_per_testcase`在与测试用例相同的进程上执行。


# 并行属性和组嵌套

# 并行的测试用例与IO

# Test Case Group重复

# 测试用例组信息函数

# 初始化与结束配置的信息函数

# 数据和私有目录

# 运行环境

# 时间陷阱与超时

# 日志 —— 种类与冗长级别

# 非法的依赖


