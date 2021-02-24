# SVA简单语法总结
## 第一章 SVA介绍
**断言是设计属性的描述**

断言又被称为“监视器”“检验器”

作为一种调试技术手段，用在设计和验证中

**为什么使用SVA**

1. verilog是一种过程语言
2. verilog是一种冗长的语言
3. 由于verilog的过程性，因此在一定情况下，verilog的检验器甚至可能无法捕捉到所有被触发事件
4. verilog没有提供内嵌的的机制来提供功能覆盖率数据

**采用SVA的原因**

1. sva是一种描述性语言，可以很好地描述时序
2. 提供了内嵌函数来测试特定的情况，
3. 提供了一些构造来自动收集功能覆盖率数据

 **sva检验失败时会自动打印相关信息**

**systemverilog是基于事件的执行模式，在每个time slot，许多事件按照安排的顺序发生。这个事件的列表按照安排的顺序发生** 
### 断言的评估和执行
1. 预备（preponed) : 采样断言变量，而且信号和变量的值不能改变，这样确保在time slot开始时采样到稳定的值
2. 观察（obversed）： 对所有的属性表达式求值
3. 响应（reactive）：调度评估属性成功或失败的代码

>> 关于time slot以及事件调度详细请参看《systemverilog 3.1a LRM》

***断言分类*** 
1. 并发断言
   `a_cc: assert property(@(posedge clk) not (a && b));`
    1. 基于时钟周期
    2. 在时钟边沿根据调用的变量的采样值计算测试表达式
    3. **变量的采样在预备阶段完成，表达式的计算在调度器的观察阶段完成**
    4. 可以在procedural block、module 、interface、program中出现
    5. 可以在静态和动态验证中出现
2. 及时断言
    1. 基于模拟事件的语义
    2. 测试表达式的求值就像在过程块中的其他verilog表达式一样**本质不是时序相关，而是立即求值**
    3. 必须出现在过程块中
    4. 只能用于动态模拟

    >    ` always_comb begin
    >       a_ia : assert ( a&&b );
    >  end 
   `

**区分即时断言和并发断言的关键词是property**

**SVA使用sequence来表示事件**
` sequence name_of_sequence ;
    <test expression>;
endsequence `

**SVA使用property来表示更加复杂的有序行为**
` property name_of_property ;
    <test expression>;
endproperty `

**SVA使用assert来检查属性** 
` assertion_name : assert property(property_name); `
### 边缘表达式
**SVA内嵌了边缘表达式，以便用户监视信号从一个时钟周期到另一个时钟周期的跳变**这使得用户可以检验边沿敏感信号
1. `$rose(boolean expression or signal name );`
2. `$fell(boolean expression or signal name );`
3. `$stable(boolean expression or signal name );`
### 逻辑关系的序列
### 序列表达式 
**在序列定义中定义形参，相同的序列可以被重用到设计中具有相似行为的信号上** 
``` 
    sequence s3_lib(a,b);
        a || b ;
    endsequence  
```
*一些在设计中常见的通常的属性可以被开发成一个库以便于重用。比如，one-hot状态机检查，等效性检查等都适合放在这样的检验器库中*
### 时序关系的序列
有时间，我们关心的是检查需要几个时钟周期才能完成的事件，也就是所谓的时序检查   
在SVA中，时钟周期延迟用##来表示，如##3 表示3个时钟周期
### SVA中的时钟定义
一个sequence或者property在模拟中并不起什么作用，它们必须别assert才能发挥作用
在sequence、property、assert中都可以定义时钟
建议：**在property中定义时钟，保持sequence独立与时钟**是一种良好的编码风格
断言一个序列并不一定需要定义一个独立的属性。因为断言语句调用属性，在断言的语句中可以直接调用被检查的表达式
*注意时钟定义重复的情况* 
### 禁止属性 
使用关键词**not** 
### action block 执行块
可以使用断言陈述的之执行块（action block）来打印自定义的成功或失败信息.
``` 
assertion_name : 
    assert property(property_name)
        <success message>;
    else 
        <fail message>;
```
**可以在action block中实现控制模拟环境收集功能覆盖率数据等功能**
### 蕴含操作符(implication)
被定义为：当检查的起始点不是有效时，忽略这次检查
蕴含等效于一个if-then 结构。蕴含的左边叫作**先行算子(antecedent)**，右边叫作**后续算子”(consequent)**。先行算子是约束条件。当先行算子成功时，后续算子才会被计算。如果先行算子不成功，那么整个属性就默认地被认为成功。这叫作**空成功 (vacuous success)**。蕴含结构只能被用在属性定义中，不能在序列中使用
1. 交叠蕴含（overlapped implication) 
    * 交叠蕴含用符号|->表示。如果先行算子匹配，在**同一个时钟周期**计算后续算子表达式
2. 非交叠蕴含(Non-overlapped implication)
    * 非交叠蕴含用符号|=>表示。如果先行算子匹配，那么在**下一个时钟周期**计算后续算子表达式。后续算子表达式的计算总是有一个时钟周期的延迟
#### 后续算子带固定延迟的蕴含
``` systemverilog
property p10;
    @(posedge clk) a |-> ##2 b;
endproperty
```

**先行算子可以使用序列的定义**也就是对先行算子的限制不是特别严格

### SVA检验器的时序窗口
SVA允许使用时序窗口来匹配后续算子（时序窗口表达式左手边的值必须小于右手边的值,左手边的值可以是0。如果它是0，表示后续算子必须在先行算子成功的那个时钟边沿开始计算。
``` systemverilog
property p12;
    @(posedge clk) (a && b) |-> ##[1:3] c;
endproperty
a12 ： assert property(p12);
```
每声明一个时序窗口，就会在每个时钟沿上触发多个线程来检查所有可能的成功。
在任何一个时钟上升沿只能有一个有效地开始，但是可以有很多有效的结束
### 重叠的时序窗口
先行算子和后续算子可以从同一个时钟边沿开始
### 无限的时序窗口
在时序窗口的窗口上限可以用符号"$"”定义，这表明时序没有上限。这叫作**可能性(eventuality)**运算符。检验器不停地检查表达式是否成功直到模拟结束

会对性能产生巨大的影响，**不推荐使用**
如果有一个有效地开始，但是后续算子在模拟结束前依然无效，则这些检查被报告为**未完成检验（incomplete check）**

### ended结构
将多个序列以序列的起始点作为同步点，来组合成时间上连续的检查。SVA提供了另一种使用**序列的结束点**作为同步点的连接机制,这种机制通过给序列名字追加关键字ended来表示
关键词“ended”保存了一个布尔值，值的真假取决于序列是否在特定的时钟边沿匹配检验。这个s.ended的布尔值只有在相同时钟周期有效。
### 使用参数的SVA检验器
可以在SVA检验器中使用参数
### 使用选择运算符的SVA检验器
```? :  ```  
___
- [ ] **使用true表达式的SVA检验器**
可以在时间上延长SVA检验器

### $past构造
可以得到信号在几个时钟周期之前的值。在默认情况下，它提供信号在前一个时钟周期的值。
``` systemverilog
$past(signal_name,number of clock cycles);
```
#### 带门控时钟的$past构造
``` systemverilog
$past(signal_name,number of clock cycles,gating signal);
```
只有当门控时钟信号的值为真时才检查后续算子的情况

## 重复算子
1. 连续重复（consecutive repetition）
    允许用户表明信号或者序列将在指定数量的时钟周期内都连续地匹配,信号的每次匹配之间都有一个时钟周期的隐藏延迟
    ``` systemverilog
    signal or sequence [*n];
    //a[*3] 等价于 a ##1 a ##1 a
    ```
    可用于序列、延迟窗口、可以和可能性运算符结合
2. 跟随重复（go to repetition）
    允许用户表明一个表达式将匹配达到指定的次数，而且不一定在连续的时钟周期上发生,这些匹配可以是间歇性的，而且不一定在连续的时钟周期上发生，跟随重复的主要要求是被检验重复的表达式的最后一个匹配应该发生在整个序列匹配结束之前
    ``` systemverilog
    signal [->n]
    start ##1 a[->3] ##1 stop ;
    /*
        这个序列需要信号“a”的匹配(即信号“a”的第三次，也就是最后一次重复的匹配)正好发生在“stop”成功之前。换句话说，信号“stop”在序列的最后一个时钟周期匹配，而且在前一个时钟周期，信号“a”有一次匹配。
    */
    ```
3. 非连续重复（non-consecutive repetition）
    与跟随重复相似，除了它并不要求信号的最后一次重复匹配发生在整个序列匹配前的那个时钟周期
    ``` systemverilog
    signal or sequence [=n];
    ```
**在跟随重复和非连续重复中只允许使用表达式，不能使用序列**
### "and"构造
and可以逻辑地组合两个序列。两个序列必须具有相同的起始点，但是可以具有不同的结束点。检验的起始点是第一个序列成功时的起始点，而检验的结束点是使得属性最终成功的另一序列成功时的点

### "intersect"构造
两个序列必须在相同时刻开始且在相同时刻结束,即两个序列的长度必须相等
### "or"构造
### "first_match"构造
任何时候使用了逻辑运算符(如“and”和“or”)的序列中指定了时间窗，就有可能出现同一个检验具有多个匹配的情况。“first_match”构造可以确保只用第一次序列匹配，而丢弃其他的匹配
### "throughout"构造
implication是允许定义前提条件的一项技术,含**只在时钟边沿检验前提条件一次**，然后就开始检验后续算子部分，因此它**不检测先行算子是否一直保持为真**。
``` systemverilog
    (expression) throughout (sequence definition)
```
**在整个检查过程中，expression应该保持为真**  

### "within"构造
within构造允许在一个序列中定义另一个序列
```seq1 within seq2 ```
这表示seq1在seq2的开始到结束的范围内发生，且序列seq2的开始匹配点必须在seq1的开始匹配点之前发生，序列seq1的结束匹配点必须在seq2的结束匹配点之前结束

### 内建的系统函数

1. $onehot(expression)  换句话说，就是在任意给定的时钟沿，表达式只有一位为高。
2.  $onehot0(expression) 检验表达式满足“zero one-hot”，换句话说，就是在任意给定的时钟沿，表达式只有一位为高或者没有任何位为高
3. $isunknown(expression)—— 检验表达式的任何位是否是X或者Z。
4. $countones(expression)—— 计算向量中为高的位的数量


### disable iff 构造
在某些设计情况中，如果一些条件为真，则我们不想执行检验。换句话说，这就像是一个异步的复位，使得检验在当前时刻不工作。SVA提供了关键词“disable iff”来实现这种检验器的异步复位 
``` systemverilog 
disable iff (expression) < property definition>
```
当property definition为真时，我们不想检验expression
### 使用intersect控制序列的长度
``` systemverilog
property p35;
(@(posedge clk) 1[*2:5] intersect
                (a ##[1:$] b ##[1:$] c));
endproperty

a35： assert property(p35);
```
这可以使用带1[*2:5]的intersect运算符来加以约束。这个intersect的定义检查从序列的有效开始点(信号“a”为高)，到序列成功的结束点(信号“c”为高)，一共经过2~5个时钟周期 

### 在属性中使用形参
可以用定义形参的方式来重用一些常用的属性
SVA 允许使用属性的形参来定义时钟。这样，属性可以应用在使用不同时钟的相似设计模块中
``` systemverilog
property arb (a，b，c，d);
    @(posedge clk) 
    ($fell(a) ##[2:5] $fell(b)) |->
    ##1 ($fell(c) && $fell(d)) ##0
    (!c&&!d) [*4] ##1 (c&&d) ##1 b;
endproperty
```

### 嵌套的蕴含
``` systemverilog
`define free (a && b && c && d)
property p_nest;
    @(posedge clk) $fell(a) |->
    ##1 (!b && !c && !d) |->
    ##[6：10]
    `free;
endproperty

a_nest： assert property(p_nest);
```

### 在蕴含中使用if/else
SVA 允许在使用蕴含的属性的后续算子中使用“if/else”语句。
``` systemverilog
property p_if_else;
@(posedge clk)
    ($fell(start) ##1 (a||b)) |->
    if(a)
        (c[->2] ##1 e)
    else
          (d[->2] ##1 f);
endproperty
                
a_if_else： assert property(p_if_else);
```
### SVA中的多时钟定义
SVA 允许序列或者属性使用多个时钟定义来采样独立的信号或者子序列。SVA 会自动地同步不同信号或子序列使用的时钟域。  
**当在一个序列中使用了多个时钟信号时，只允许使用“##1”延迟构造**
**使用“##0”会产生混淆，即在信号“a”匹配后究竟哪个时钟信号才是最近的时钟。这将引起竞争，因此不允许使用。使用##2也不允许，因为不可能同步到时钟“clk2”的最近的上升沿。**  
**禁止在两个不同时钟驱动的序列之间使用交叠蕴含运算符。因为先行算子的结束和后续算子的开始重叠，可能引起竞争的情况，这是非法的。**  
可以使用非交叠蕴含(|=>)