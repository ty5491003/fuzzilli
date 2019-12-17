# Fuzzilli

一个以覆盖率为指导的动态语言解释器，它基于一个定制的中介语言“FuzzIL”，该语言能被变异和被翻译成JS代码.

## Usage

基本使用步骤如下：

1. Download the source code for one of the supported JavaScript engines (currently [JavaScriptCore](https://github.com/WebKit/webkit), [Spidermonkey](https://github.com/mozilla/gecko-dev), and [v8](https://github.com/v8/v8)).
2. Apply the corresponding patch from the [Targets/](Targets/) directory. Also see the README.md in that directory.
3. Compile the engine with coverage instrumentation (requires clang >= 4.0) as described in the README.
4. Compile the fuzzer: `swift build [-c release]`.
5. Run the fuzzer: `swift run [-c release] FuzzilliCli --profile=<profile> [other cli options] /path/to/jsshell`. See also `swift run FuzzilliCli --help`.

### Hacking

检查 [main.swift](Sources/FuzzilliCli/main.swift) 来看一个Fuzzilli库的使用例，并且了解各种配置.

之后，看一下[Fuzzer.swift](Sources/Fuzzilli/Fuzzer.swift)来了解高层的fuzzing逻辑.

## Concept

当对core interpreter（例如JIT编译器） bugs进行fuzzing时，生成器程序的语义正确性是一个问题. 它与其他大多数的场景不同，比如对runtime APIs进行fuzzing，通过把生成的代码包裹在try-catch中就可以很容易地解决语义正确性问题. 实现可接受的语义正确样本率有不同的可能性，其中之一是一种变异方法，要求其语料库中的所有样本都是语义有效的. 在这种情况下，每个突变只有很小的机会将有效样本变成无效样本. 

为了实现一个基于变异的JS fuzzer，针对JS代码的mutations必须被定义. 定制的中间语言(IL)被定义为可以更直接地执行对程序的控制和数据流进行更改的语言，而不是对程序的AST或其他语法元素进行更改. 这个IL代码在之后会被翻译成JS代码来执行. 关于IL的概览如下：

    v0 <− LoadInt '0'
    v1 <− LoadInt '10'
    v2 <− LoadInt '1'
    v3 <− Phi v0
    BeginFor v0, '<', v1, '+', v2 −> v4
       v6 <− BinaryOperation v3, '+', v4
       Copy v3, v6
    EndFor
    v7 <− LoadString 'Result: '
    v8 <− BinaryOperation v7, '+', v3
    v9 <− LoadGlobal 'console'
    v10 <− CallMethod v9, 'log', [v8]

上面的IL代码可以被翻译为下面的JS代码：

    const v0 = 0;
    const v1 = 10;
    const v2 = 1;
    let v3 = v0;
    for (let v4 = v0; v4 < v1; v4 = v4 + v2) {
        const v6 = v3 + v4;
        v3 = v6;
    }
    const v7 = "Result: ";
    const v8 = v7 + v3;
    const v9 = console;
    const v10 = v9.log(v8);

或者转化为下面的JS代码，通过内联中介解释器：

    let v3 = 0;
    for (let v4 = 0; v4 < 10; v4++) {
        v3 = v3 + v4;
    }
    console.log("Result: " + v3);

FuzzIL有以下特性：
* FuzzIL程序是一个简单的指令列表（集合）.
* 一条FuzzIL指令就是一个操作，它包含input和output变量，可能还有一个或多个参数(在上面的符号中用单引号括起来).
* 指令的输入永远是变量，并且没有直接数值（immediate values）.
* IL代码遵循SSA格式：所有变量只assign一次. 然而，由一个 `Phi` 操作产生的变量可以在之后通过一个 `Copy` 操作来 reassign.
* 所有变量在使用前必须先定义.


在执行下列程序后，可以产生一系列突变：

* [InputMutator](Sources/Fuzzilli/Mutators/InputMutator.swift): 一个简单的数据流突变，其中一条指令的input值会被替换成另一个不同的.
* [CombineMutator](Sources/Fuzzilli/Mutators/CombineMutator.swift) and [SpliceMutator](Sources/Fuzzilli/Mutators/SpliceMutator.swift): 它们通过将一个程序的一部分插入另一个程序来组合多个程序.
* [InsertionMutator](Sources/Fuzzilli/Mutators/InsertionMutator.swift): 从现有程序的随机位置生成新代码，从[predefined code generators](Sources/Fuzzilli/Core/CodeGenerators.swift)列表中.
* [OperationMutator](Sources/Fuzzilli/Mutators/OperationMutator.swift): 突变一个操作的参数，比如，替换掉一个integer constant为另外一个.
* and many more...

## Implementation

这个fuzzer主要由 [Swift](https://swift.org/) 实现, 部分(比如coverage measurements, socket interactions等)由 C 实现.

### Architecture

A fuzzer instance (implemented in [Fuzzer.swift](Sources/Fuzzilli/Fuzzer.swift)) 由以下主要组件构成:

* [FuzzerCore](Sources/Fuzzilli/Core/FuzzerCore.swift)(Sources/Fuzzilli/Core/FuzzerCore.swift): 通过使用[mutations](Sources/Fuzzilli/Mutators) 从现有程序中产生新程序. 之后执行该生成用例并评估.
* [ScriptRunner](Sources/Fuzzilli/Execution)(Sources/Fuzzilli/Execution): 执行目标语言的程序.
* [Corpus](Sources/Fuzzilli/Core/Corpus.swift)(Sources/Fuzzilli/Core/Corpus.swift): 存储有趣的例子，并将它们提供给core fuzzer.
* [Environment](Sources/Fuzzilli/Core/JavaScriptEnvironment.swift)(Sources/Fuzzilli/Core/JavaScriptEnvironment.swift): has knowledge of the runtime environment, e.g. the available builtins, property names, and methods.
* [Minimizer](Sources/Fuzzilli/Minimization/Minimizer.swift)(Sources/Fuzzilli/Minimization/Minimizer.swift): 最小化crashing和有趣的例子（用例精简？）.
* [Evaluator](Sources/Fuzzilli/Evaluation)(Sources/Fuzzilli/Evaluation): 评估一个用例是否是“有趣的”，通过某些指标，比如代码覆盖率.
* [Lifter](Sources/Fuzzilli/Lifting)(Sources/Fuzzilli/Lifting): 翻译一个FuzzIL程序为目标语言 (JavaScript).

此外,许多可用的模块(可选):

* [Statistics](Sources/Fuzzilli/Modules/Statistics.swift)(Sources/Fuzzilli/Modules/Statistics.swift): gathers various pieces of statistical information.
* [NetworkWorker/NetworkMaster](Sources/Fuzzilli/Modules/NetworkSync.swift)(Sources/Fuzzilli/Modules/NetworkSync.swift): synchronize multiple instances over the network.
* [ThreadWorker/ThreadMaster](Sources/Fuzzilli/Modules/ThreadSync.swift)(Sources/Fuzzilli/Modules/ThreadSync.swift): synchronize multiple instances within the same process.
* [Storage](Sources/Fuzzilli/Modules/Storage.swift)(Sources/Fuzzilli/Modules/Storage.swift): stores crashing programs to disk.

本fuzzer是事件驱动的，类与类之间的大部分交互都是通过事件. 事件被分派，例如由于一个崩溃或一个有趣的程序被发现，一个新程序被执行，一个日志消息被生成等等. 到[Events.swift](Sources/Fuzzilli/Core/Events.swift) 可以查看完整的事件列表. 事件机制有效地解耦了fuzzer的各个组件，使得实现附加模块变得容易.

A FuzzIL program can be built up using a [ProgramBuilder](Sources/Fuzzilli/Core/ProgramBuilder.swift) instance. A ProgramBuilder 提供方法来创建和添加新的指令、从另一个程序添加指令、检索现有变量、查询当前位置的执行上下文（比如是否处在一个循环中）等.

### Execution

fuzzilli支持不同的模式来执行目标引擎：

* [Forkserver](Sources/Fuzzilli/Execution/Forkserver.swift): 和 [afl](http://lcamtuf.coredump.cx/afl/) 相同, 这将在流程初始化(部分)完成后停止子流程中的执行，然后为每个生成用例派生一个新的子流程.
* [REPRL (read-eval-print-reset-loop)](Sources/Fuzzilli/Execution/REPRL.swift): 在这种模式下，目标引擎被修改为通过某个IPC通道接受脚本，执行它，然后重置它的内部状态并等待下一个脚本. 这种模式往往更快.


### Scalability 可拓展性

There is one fuzzer instance per target process. This enables synchronous execution of programs and thereby simplifies the implementation of various algorithms such as consecutive mutations and minimization. Moreover, it avoids the need to implement thread-safe access to internal state, e.g. the corpus. Each fuzzer instance has its own dedicated [OperationQueue](https://developer.apple.com/documentation/foundation/operationqueue), conceptually corresponding to a single thread. Every interaction with a fuzzer instance must then happen on the instance’s queue. This guarantees thread-safety as the queue is serial. For more details see [the docs](Docs/ProcessingModel.md).



To scale, fuzzer instances can become workers, in which case they report newly found interesting samples and crashes to a master instance. In turn, the master instances also synchronize their corpus with the workers. Communication between masters and workers can happen in different ways, each implemented as a module:

* [Inter-thread communication](Sources/Fuzzilli/Modules/ThreadSync.swift): synchronize instances in the same process by enqueuing tasks to the other fuzzer’s DispatchQueue.
* Inter-process communication (TODO): synchronize instances over an IPC channel.
* [Inter-machine communication](Sources/Fuzzilli/Modules/NetworkSync.swift): synchronize instances over a simple TCP-based protocol.

This design allows the fuzzer to scale to many cores on a single machine as well as to many different machines. As one master instance can quickly become overloaded if too many workers send programs to it, it is also possible to configure multiple tiers of master instances, e.g. one master instance, 16 intermediate masters connected to the master, and 256 workers connected to the intermediate masters.

## Resources 资源

Further resources about this fuzzer:

* A [presentation](https://saelo.github.io/presentations/offensivecon_19_fuzzilli.pdf) about Fuzzilli given at Offensive Con 2019.
* The [master's thesis](https://saelo.github.io/papers/thesis.pdf) for which the initial implementation was done.

## Bug Showcase BUG列表

The following is a list of some of the bugs found with the help of Fuzzilli. Only bugs with security impact are included in the list. Special thanks to all users of Fuzzilli who have reported bugs found by it!

#### WebKit/JavaScriptCore

* [Issue 185328](https://bugs.webkit.org/show_bug.cgi?id=185328): DFG Compiler uses incorrect output register for NumberIsInteger operation
* [CVE-2018-4299](https://www.zerodayinitiative.com/advisories/ZDI-18-1081/): performProxyCall leaks internal object to script
* [CVE-2018-4359](https://bugs.webkit.org/show_bug.cgi?id=187451): compileMathIC produces incorrect machine code
* [CVE-2019-8518](https://bugs.chromium.org/p/project-zero/issues/detail?id=1775): OOB access in FTL JIT due to LICM moving array access before the bounds check
* [CVE-2019-8558](https://bugs.chromium.org/p/project-zero/issues/detail?id=1783): CodeBlock UaF due to dangling Watchpoints
* [CVE-2019-8611](https://bugs.chromium.org/p/project-zero/issues/detail?id=1788): AIR optimization incorrectly removes assignment to register
* [CVE-2019-8623](https://bugs.chromium.org/p/project-zero/issues/detail?id=1789): Loop-invariant code motion (LICM) in DFG JIT leaves stack variable uninitialized
* [CVE-2019-8622](https://bugs.chromium.org/p/project-zero/issues/detail?id=1802): DFG's doesGC() is incorrect about the HasIndexedProperty operation's behaviour on StringObjects
* [CVE-2019-8671](https://bugs.chromium.org/p/project-zero/issues/detail?id=1822): DFG: Loop-invariant code motion (LICM) leaves object property access unguarded
* [CVE-2019-8672](https://bugs.chromium.org/p/project-zero/issues/detail?id=1825): JSValue use-after-free in ValueProfiles
* [CVE-2019-8678](https://bugs.webkit.org/show_bug.cgi?id=198259): JSC fails to run haveABadTime() when some prototypes are modified, leading to type confusions
* [CVE-2019-8685](https://bugs.webkit.org/show_bug.cgi?id=197691): JSPropertyNameEnumerator uses wrong structure IDs
* [CVE-2019-8765](https://bugs.chromium.org/p/project-zero/issues/detail?id=1915): GetterSetter type confusion during DFG compilation
* [CVE-2019-8820](https://bugs.chromium.org/p/project-zero/issues/detail?id=1924): Type confusion during bailout when reconstructing arguments objects

#### Gecko/Spidermonkey

* [CVE-2018-12386](https://ssd-disclosure.com/archives/3765/ssd-advisory-firefox-javascript-type-confusion-rce): IonMonkey register allocation bug leads to type confusions
* [CVE-2019-9791](https://bugs.chromium.org/p/project-zero/issues/detail?id=1791): IonMonkey's type inference is incorrect for constructors entered via OSR
* [CVE-2019-9792](https://bugs.chromium.org/p/project-zero/issues/detail?id=1794): IonMonkey leaks JS\_OPTIMIZED\_OUT magic value to script
* [CVE-2019-9816](https://bugs.chromium.org/p/project-zero/issues/detail?id=1808): unexpected ObjectGroup in ObjectGroupDispatch operation
* [CVE-2019-9813](https://bugs.chromium.org/p/project-zero/issues/detail?id=1810): IonMonkey compiled code fails to update inferred property types, leading to type confusions
* [CVE-2019-11707](https://bugs.chromium.org/p/project-zero/issues/detail?id=1820): IonMonkey incorrectly predicts return type of Array.prototype.pop, leading to type confusions

#### Chromium/v8

* [Issue 939316](https://bugs.chromium.org/p/project-zero/issues/detail?id=1799): Turbofan may read a Map pointer out-of-bounds when optimizing Reflect.construct
* [Issue 944062](https://bugs.chromium.org/p/project-zero/issues/detail?id=1809): JSCallReducer::ReduceArrayIndexOfIncludes fails to insert Map checks
* [CVE-2019-5831](https://bugs.chromium.org/p/chromium/issues/detail?id=950328): Incorrect map processing in V8
* [Issue 944865](https://bugs.chromium.org/p/chromium/issues/detail?id=944865): Invalid value representation in V8
* [CVE-2019-5841](https://bugs.chromium.org/p/chromium/issues/detail?id=969588): Bug in inlining heuristic
* [CVE-2019-5847](https://bugs.chromium.org/p/chromium/issues/detail?id=972921): V8 sealed/frozen elements cause crash
* [CVE-2019-5853](https://bugs.chromium.org/p/chromium/issues/detail?id=976627): Memory corruption in regexp length check
* [Issue 992914](https://bugs.chromium.org/p/project-zero/issues/detail?id=1923): Map migration doesn't respect element kinds, leading to type confusion

## Disclaimer

This is not an officially supported Google product.
