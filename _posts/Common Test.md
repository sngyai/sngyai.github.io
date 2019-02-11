# 简介
Common Test 是一款用于自动化测试的软件，它适用于：

* **任何系统的黑盒测试** 通过SNMP，HTTP，CORBA，Telnet等

* **Erlang/OTP程序的白盒测试** 在测试用例中直接调用目标API

Common Test还集成了[cover](http://erlang.org/doc/man/cover.html)，cover是一个代码覆盖率分析的工具。

Common Test能够自动运行Test Suite并将测试结果打印到HTML格式的日志文件中，而无需测试人员值守。Common Test还可以通过OTP event manager将测试进度或测试结果发送到event handler中，这样一来，用户就可以将他们自己的程序（如日志、存储、监控等）集成到Common Test中来。

Common Test提供了多种有用的库，以满足不同的测试需求。例如，通过Test Specification支持灵活的测试声明，此外，它还支持并行运行的多个独立测试会话（到不同的目标系统）的集中配置和控制。

# Common Test基础

测试用例可以单独执行，也可以分批执行。 Common Test还具有分布式测试模式，具有中央控制和记录功能。使用这个功能，可以在一个公共会话中独立测试多个系统。例如，在运行自动化大规模回归测试时，这很有用。

在黑盒测试场景中，基于Common Test的测试程序通过HTTP，Telnet，SNMP等连接到目标系统。 Common Test提供了其中一些协议的实现和接口封装（其中大多数协议作为OTP中的独立组件和应用程序存在）。封装简化了配置并能够打印出更为详尽的日志。 Common Test的支持模块化扩展。不过，使用Common Test进行测试时，使用任何Erlang/OTP组件是一项简单的任务，而无需使用Common Test封装，就像调用Erlang函数一样简单。 

Common Test支持许多与目标系统无关的接口，例如Telnet和FTP。它们可以用作用于控制设备，生成负载等，可以是定制化的也可以直接拿来使用。

Common Test对于白盒测试Erlang代码（例如，模块测试）也是一个非常有用的工具，因为测试程序可以直接调用导出的Erlang函数。实现基本Test Suite和执行简单测试所需的开销非常小。对于黑盒测试Erlang软件，可以使用Erlang RPC和标准O＆M接口。

单个测试用例可以并行处理到一个或多个目标系统的多个连接，以执行测试所需的操作。并行处理多个连接是Common Test的主要优势之一，这主要归功于Erlang运行时系统中对并发的有效支持，我们可以充分利用这点优势。

## Test Suite组织
Test Suite位于`test`目录，每个Test Suite可以有独立的数据目录。通常这些目录和文件与其他源码一样，都是通过Git或Subversion进行版本控制的。

## 支持库
支持库包含许多对Test Suite实用的函数。除了Common Test支持库以及Erlang/OTP提供的库和应用之外，还有一些用户自定义的支持库。

## Test Suite和Test Case
测试是通过运行Test Case或者Test Suite（Test Case的集合）来完成的，Test Suite是通过名为`<suite_name>_SUITE.erl`的Erlang模块来实现的，模块包含了许多Test Case，一个Test Case是一个用来测试一个或多个事物的Erlang函数。Test Case是Common Test测试服务器时处理的最小单元。

测试用例的子集，叫做Test Case Group，一个Test Case Group可以有运行属性与之关联，运行属性可以指定Test Case Group中的Test Case是否以随机、顺序、并行执行以及Test Case Group是否要重复执行，Test Case Group可以嵌套（即一个Test Case Group除了Test Case之外，还可以包含子Test Case Group）。

除了Test Case和Test Case Group外，Test Suite还可以包含配置函数。这些函数是用来设定（和验证）被测系统的环境、状态的，需要测试程序正确执行。例如：发起一个到被测系统的连接，初始化数据库，运行安装脚本等。配置函数可以在每个Test Suite、每个Test Case Group以及每个单独的Test Case中执行。

Test Suite模块必须与Common Test测试服务定义的回调接口保持一致。

无论Test Case给调用这返回的结果是什么，只要返回了，就认为它是成功的。
不过有一些返回值是有特殊含义的：

* `{skip,Reason}` 该Test Case被跳过
* `{comment,Comment}` 为该Test Case在log中打印一条标注
* `{save_config,Config}` 让Common Test将配置传递给下一个Test Case


# 编写Test Suite

`ct`模块提供了编写测试用例的主要接口，主要包含：
* 打印输出日志的函数
* 读取配置数据的函数
* 结束一个测试用例，并返回错误原因的函数
* 向HTML总览页面添加评注的函数

`Common Test`应用同样包含了其他名为`ct_<component>`的模块，它们提供了各种支持，主要简化了通信协议的使用，这些通讯协议包括RPC，SNMP，FTP，Telnet等等。

## Test Suite的初始化与结束  
每个Test Suite模块，都可以包含配置函数（也可以不包含）`init_per_suite/1` 和 `end_per_suite/1`。如果定义了`init`，那么也必须要定义`end`。

如果定义了`init_per_suite`， 它将会在所有的测试用例执行之前被调用。它一般包含了SUITE中所有测试用例的通用初始化，这些初始化仅执行一次。建议使用`init_per_suite`来设置和验证被测系统（SUT）或Common Test主机节点的状态和环境，以便SUITE中的测试用例正确执行。以下是初始化配置时的操作的例子：
* 打开到被测系统（SUT）的连接。
* 初始化数据库
* 执行安装脚本

`end_per_suite`在测试套件执行的最后阶段被调用（在最后一个测试用例完成之后）。 该函数用于清理`init_per_suite`中的变动。

## Test Case的初始化与结束
每个Test Suite模块，都可以包含配置函数（也可以不包含）`init_per_testcase/2 ` 和 `end_per_testcase/2`。如果定义了`init`，那么也必须要定义`end`。

如果定义了`init_per_testcase`， 它将会在每个测试用例执行之前被调用。它通常包含为每个测试用例完成的必要的初始化（类似于SUITE中的`init_per_suite`）。

每次测试用例执行完成之后会调用`end_per_testcase/2`，用于清理`init_per_testcase`中的变动。


## 测试用例
测试服务关心的最小单元就是测试用例，每个测试用例里可以测试很多东西，例如：对同一接口函数使用不同参数多次调用。

我们可以选择在一个测试用例中放置或多或少的测试，需要记住以下几点：
* 许多小型测试用例往往导致额外的，可能是重复的代码，以及由于初始化和清理的大量开销而导致测试执行缓慢。避免重复的代码，例如，使用常见的帮助功能。 否则，生成的SUITE变得难以阅读和理解，并且维护起来很麻烦。

* 较大的测试用例使得如果失败则更难找出错误。此外，当发生错误时，大部分测试代码可能会被跳过。

* 当测试用例变得过大和宽泛时，可读性和可维护性会受到影响。 无法确定生成的日志文件是否能够很好地反映所执行的测试数量。

## 测试用例组
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

其中各项`Properties`含义如下：
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

以下示例展示了如何在深层嵌套中使用属性覆盖：

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


# 如何执行Common Test

## 命令行或Erlang Shell（比较原始的）

1. 假定测试的SUITE模块位于当前目录下
```
$ ct_run -dir .
```
指定测试的SUITE：
```
 $ ct_run -suite check_log_SUITE
```

2. 如要使用Erlang Shell运行Common Test：
```erl
 1> ct:run_test([{dir, "."}]).
```
指定测试的SUITE：
```erl
 1> ct:run_test([{suite, "check_log_SUITE"}]).
```

## 使用 Test Specifications

指定测试内容最灵活的方式是使用Test Specifications，它由一系列的Erlang Term组成。这些Erlang Term通常是在一个或者多个文本文件中声明的，但也可以以列表的形式传递给Common Test，这些Erlang Term 有两种类型：用于配置的和用于Test Spec的:

* **用于配置的Erlang Term**

通常用于指定`ct_run`启动标识位(flag)或`ct:run_test/1`的启动参数（例如配置文件、包含目录、日志级别等），它能够覆盖用于Test Specification的Erlang Term中的配置：

```
 {merge_tests, Bool}.

 {define, Constant, Value}.

 {specs, InclSpecsOption, TestSpecs}.

 {node, NodeAlias, Node}.

 {init, InitOptions}.
 {init, [NodeAlias], InitOptions}.

 {label, Label}.
 {label, NodeRefs, Label}.

 {verbosity, VerbosityLevels}.
 {verbosity, NodeRefs, VerbosityLevels}.

 {stylesheet, CSSFile}.
 {stylesheet, NodeRefs, CSSFile}.

 {silent_connections, ConnTypes}.
 {silent_connections, NodeRefs, ConnTypes}.

 {multiply_timetraps, N}.
 {multiply_timetraps, NodeRefs, N}.

 {scale_timetraps, Bool}.
 {scale_timetraps, NodeRefs, Bool}.

 {cover, CoverSpecFile}.
 {cover, NodeRefs, CoverSpecFile}.

 {cover_stop, Bool}.
 {cover_stop, NodeRefs, Bool}.

 {include, IncludeDirs}.
 {include, NodeRefs, IncludeDirs}.

 {auto_compile, Bool},
 {auto_compile, NodeRefs, Bool},

 {abort_if_missing_suites, Bool},
 {abort_if_missing_suites, NodeRefs, Bool},

 {config, ConfigFiles}.
 {config, ConfigDir, ConfigBaseNames}.
 {config, NodeRefs, ConfigFiles}.
 {config, NodeRefs, ConfigDir, ConfigBaseNames}.

 {userconfig, {CallbackModule, ConfigStrings}}.
 {userconfig, NodeRefs, {CallbackModule, ConfigStrings}}.

 {logdir, LogDir}.                                        
 {logdir, NodeRefs, LogDir}.

 {logopts, LogOpts}.
 {logopts, NodeRefs, LogOpts}.

 {create_priv_dir, PrivDirOption}.
 {create_priv_dir, NodeRefs, PrivDirOption}.

 {event_handler, EventHandlers}.
 {event_handler, NodeRefs, EventHandlers}.
 {event_handler, EventHandlers, InitArgs}.
 {event_handler, NodeRefs, EventHandlers, InitArgs}.

 {ct_hooks, CTHModules}.
 {ct_hooks, NodeRefs, CTHModules}.

 {enable_builtin_hooks, Bool}.

 {basic_html, Bool}.
 {basic_html, NodeRefs, Bool}.

 {esc_chars, Bool}.
 {esc_chars, NodeRefs, Bool}.

 {release_shell, Bool}.
```

* **用于Test Specification的Erlang Term**

通常用于指定要运行哪些测试，以及以何种顺序执行这些测试。 它可以指定一到多个测试套件（Test Suite），一到多个测试用例组（Test Case Group），以及同一个Test Suite中的一个到多个测试用例组中的一到多个测试用例：

```
 {suites, Dir, Suites}.                                
 {suites, NodeRefs, Dir, Suites}.

 {groups, Dir, Suite, Groups}.
 {groups, NodeRefs, Dir, Suite, Groups}.

 {groups, Dir, Suite, Groups, {cases,Cases}}.
 {groups, NodeRefs, Dir, Suite, Groups, {cases,Cases}}.

 {cases, Dir, Suite, Cases}.                           
 {cases, NodeRefs, Dir, Suite, Cases}.

 {skip_suites, Dir, Suites, Comment}.
 {skip_suites, NodeRefs, Dir, Suites, Comment}.

 {skip_groups, Dir, Suite, GroupNames, Comment}.
 {skip_groups, NodeRefs, Dir, Suite, GroupNames, Comment}.

 {skip_cases, Dir, Suite, Cases, Comment}.
 {skip_cases, NodeRefs, Dir, Suite, Cases, Comment}.
```

注：关于其中的类型说明参见[Test Specification Syntax](http://erlang.org/doc/apps/common_test/run_test_chapter.html#test-specification-syntax)


## 使用Erlang.mk

Erlang.mk自动发现和运行Common Test套件。它要求Test Suite文件名称以 *_SUITE.erl* 结尾，并且要放到 *$TEST_DIR* 目录下（默认为 *test/* ）
### 配置
1. `CT_OPTS`用于配置Common Test参数，例如
```
CT_OPTS = -cover test/ct.cover.spec -erl_args -pa ${DIR}/../*/ebin -name emqttd_ct@127.0.0.1
```
2. `CT_SUITES` 参数用于覆盖Erlang.mk自动发现的那些Test Suite，由于Erlang.mk会自动发现Test Suite，所以一般情况下不用设置这个参数，套件的名称为 *_SUITE.erl* 之前的部分，如文件名为 *http_SUITE.erl*，那么Test Suite名为`http`：
```
CT_SUITES = http ws
```

3. `CT_LOGS_DIR`用于设置HTML日志文件将要存放到哪里，默认为 *logs/* 

```
CT_LOGS_DIR = ct_output_log_dir
```

### 使用

运行所有测试（包含CT）
```
$ make tests
```

运行所有测试和静态检查（包含CT）
```
$ make check
```

也可以单独运行CT
```
$ make ct
```

指定test SUITE，例如要执行 *test/http_SUITE.erl* ：
```
$ make ct-http
```

要执行`http_SUITE`中的`http_compress`测试组：
```
$ make ct-http t=http_compress
```

要执行`http_SUITE`中的`http_compress`测试组中的`headers_dupe`测试用例：
```
$ make ct-http t=http_compress:headers_dupe
```

如果test SUITE中的测试用例没有分组，则用`c`来指定测试用例：
```
$ make ct-http c=headers_dupe
```

在一个综合性项目（umbrella project）中，需要使用`-C`参数来指定目录：
```
$ make -C apps/my_app ct-http t=my_group:my_case
```

这种用法对依赖中的Test Suite同样适用，例如依赖中的`cowboy`：
```
$ make -C deps/cowboy ct-http t=http_compress
```
