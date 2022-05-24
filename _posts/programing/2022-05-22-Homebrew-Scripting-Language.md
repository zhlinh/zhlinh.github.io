---
layout: article
title: "Homebrew Scripting Language"
categories: programing
tags: [compilers, reading]
toc: false
image:
    teaser: programing/2022-05-22-Homebrew-Scripting-Language/teaser.jpg

date: 2022-05-22
---

「两周自制脚本语言」可以说是一本入门书，但通常说两周或者一周之类的很可能就会偏离经典书籍或必读书籍的行列。
当然总体来说还是值得一读的，了解基本的词法分析、语法分析脉络，构建AST并解释执行，
使用HashMap<String, Object>来实现环境的记录，然后逐步实现嵌套的作用域，以及后续的性能优化

---

## 1 语言处理器和本书的架构

- 源代码
  - **词法**分析（形成简短字符串排列）
  - **语法**分析（形成抽象语法树）
    - **【编译器】**代码生成（形成其他语言如机器语言的程序）
    - **【解释器】**执行（形成程序执行的结果）

## 2 设计程序设计语言

- stone语句体必须使用大括号，避免dangling-else的问题。java中的dangling-else匹配最近的一个if

```java
if (x > 0)
	if (y > 0)
		return x + y
else  // dangling-else
	return -x
```

## 3 分割单词

read方法将返回一个新的单词

peak(i)方法将返回read即将返回单词之后的第i个单词

如果单词都已读取，read和peak方法将返回Token.EOF，用于表示程序结束

每个单词都是一个Token对象，Token对象的子类有NumToken、IdToken和StrToken

## 4 用于表示程序的对象

抽象语法树(AST, Abstract Syntax Tree)是一种用于表示程序结构的树形结构。构造抽象语法树的过程称为语法分析，仍然属于语言处理器的前端

BNF(Backus-Naur form, 巴科斯范式)与正则表达式类似，都用于表达某种模式，以检查序列的内容

``` shell
factor:            NUMBER | "(" expression ")"
term:              factor { ("*" | "/") factor }
expression:        term { ("+" | "-") term }

其中
{ pat }       表示pat至少重复0次
[ pat ]       表示pat出现0次或1次

BNF的特征为具有循环结构的递归定义，例如，expression还可以写成
expression:        term | expression ("+" | "-") term
```

## 5 设计语法分析器

```c++
BNF巴科斯范式
primary:           "(" expr ")" | NUMBER | IDENTIFIER | STRING
factor:            "-" primary | primary
expr:              factor { OP factor }
block:             "{" [ statement ] { (";" | EOL) [ statement ] } "}"
simple:            expr
statement:         "if" expr block [ "else" block ] | "while" expr block | simple
program:           [ statement ] (";" | EOL)
```

rule方法创建Parser对象

number方法，identifier方法，string方法，token方法，sep方法，ast方法，or方法，expression方法

```java
// 保留字
HashSet<String> reserved = new HashSet<String>();
// 运算符
Operators oprators = new Operators();
Parser expr0 = rule()
Parser primary = rule(PrimaryExpr.class)
                 .or(rule().sep("(").ast(expr0).sep(")"),
                     rule().number(NumberLiteral.class),
                     rule().identifier(Name.class, reserved),
                     rule().string(StringLiteral.class));
Parser factor = rule().or(rule(NegativeExpr.class).sep("-").ast(primary), primary);
Parser expr = expr0.exprssion(BinaryExpr.class, factor, operators);

Parser statement0 = rule();
Parser block = rule(BlockStmnt.class)
  						 .sep("{").option(statement0)
               .repeat(rule().sep(";", Token.EOL).option(statement0))
               .sep("}");
Parser simple = rule(PrimaryExpr.class).ast(expr);
Parser statement = statement0.or(rule(IfStmnt.class).sep("if").ast(expr).ast(block)
                                                    .option(rule().sep("else").ast(block)),
                                 rule(WhileStmnt.class).sep("while").ast(expr).ast(block),
                                 simple);
Paser program = rule().or(statement, rule(NullStmnt.class)).sep(";", Token.EOL);

public BasicParser() {
	reserved.add(";");
  reserved.add("}");
  reserved.add(Token.EOL);

  // 1表示优先级，越小优先级越高
  // RIGHT表示右结合，(x = (y = 3))
  operators.add("=", 1, Operators.RIGHT);
  // LEFT表示左结合，((x == y) == 3)
  operators.add("==", 2, Operators.LEFT);
  operators.add(">", 2, Operators.LEFT);
  operators.add("<", 2, Operators.LEFT);
  operators.add("+", 3, Operators.LEFT);
  operators.add("-", 3, Operators.LEFT);
  operators.add("*", 4, Operators.LEFT);
  operators.add("/", 4, Operators.LEFT);
  operators.add("%", 4, Operators.LEFT);
}
```

## 6 通过解释器执行程序

解释器的基本实现原理：从语法分析得到的抽象语法树的根节点开始遍历该树直至叶节点，并计算各节点的内容

```java
// 简化版本
// 通常来说env是记录变量名和值对应关系的哈希表
public Object eval(Environment env) {
	Object left = left().eval(env);
  Object right = right().eval(env);
  return (Integer)left + (Integer)right;
}
```

## 7 添加函数功能

```
param : IDENTIFIER
params : param { "," param }
```

动态作用域：outer字段指向调用者的callerEnv环境

静态作用域：outer字段指向def语句的执行环境，java及大多数语言采用的是静态作用域

| 环境         | 作用                                                         |
| ------------ | ------------------------------------------------------------ |
| newEnv       | 调用函数时新创建的环境。用于记录函数参数和函数内部的局部变量 |
| newEnv.outer | 常用于记录全局变量                                           |
| callerEnv    | 函数调用语句所处的环境。用于计算实参                         |

## 8 关联Java语言

Stone语言添加调用Java语言写的函数的功能，称为原生函数。类似于java调用c语言等其他一些语言写成的函数

```java
public class NativeFunction {
	protected Method method;
	protected String name;
	protected int numParams;
	public NativeFunction(String n, Method m) {
		name = n;
		method = m;
		numParams = m.getParameterTypes().length;
	}
	public Object invoke(Object[] args, AStree tree) {
		try {
			return method.invoke(null, args);
		} catch (Exception e) {
			throw new StoneException("bad native function call:" + name, tree);
		}
	}
}

public class Natives {
	protected void appendNatives(Environment env) {
		append(env, "print", Natives.class, "print", Object.class);
		append(env, "currentTime", Natives.class, "currentTime");
	}

	protected void append(Environment env, String name, Class<?> clazz, String methodName,
	                      Class<?> ...params) {
		Method m;
		try {
			m = clazz.getMethod(methodName, params):
		} catch (Exception e) {
			throw new StoneException("cannot find a native function: " + methodName);
		}
		env.put(name, new NativeFunction(methodName, m);
	}
}
```

## 9 设计面向对象语言

```shell
# 基于闭包作扩展，添加ClassParser
member : def | simple
classs_body : "{" [ member ] { (";" | EOL) [ member ] } "}"
defclass : "class" IDENTIFIER [ "extends" IDENTIFIER ] class_body
postfix : "." IDENTIFIER | "(" [ args ] ")"
program : [ defclass | def | statement ] (";" | EOL)


public class ClassParser extends ClosureParser {
	Parser member = rule().or(def, simple);
	Parser class_body = rule(ClassBody.class).sep("{").option(member)
											.repeat(rule().sep(";", Token.EOL).option(member))
											.sep("}");
	Parser defclass = rule(ClassStmnt.class).sep("class").identifier(reserved)
										.option(rule().sep("extends").identifier(reserved))
										.ast(class_body)
	public ClassParser() {
		postfix.insertChoice(rule(Dot.class).sep(".")identifier(reserved));
		program.insertChoice(defclass):
	}
}
```

## 10 无法割舍的数组

```shell
# postfix的作用主要是支持数组下标，即数组的元素是另一个数组的元素
elements : expr { "," expr }
primary : ( "[" [ elements ] "]" | "(" expr ")" | NUMBER | IDENTIFIER | STRING )
          { postfix }
postfix : "(" [ args ] ")" | "[" expr "]"
```

## 11. 优化变量读写性能

Name对象保存数组元素的下标（index），并记录环境的所处层数（nest）

```java
public class ArrayEnv implements Enviroment {
  protected object[] values;
  protected Enviroment outer;
  public Object get(int nest, int index) {
    if (nest == 0) {
      return values[index];
    } else if (outer == null) {
      return null;
    } else {
      return ((EnvEx2) outer).get(nest - 1, index);
    }
  }
  public Object get(String name) {
    error(name);
    return null;
  }
  private void error(String name) {
    throw new StoneException("cannot access by name:" + name);
  }
}
```

## 12. 优化对象操作性能

对象的字段值和方法的定义无需通过哈希表查找名称来获取，只需通过编号就能直接在数组中查找对应的数据

利用内联缓存优化性能，先判断对象的类型，如果和上一次的相同，则直接上使用上次的查找结果，减少因查找保存位置而造成的性能下降

## 13. 设计中间代码解释器

之前的Stone语言处理器会一边遍历抽象语法树的节点，一边执行程序

抽象语法树的遍历会是一种很大的性能负担，所以我们需要事先将抽象语法树转换成中间代码，称为中间代码解释器或者虚拟机。中间代码可以保存在`内存`也可以保存在`文件`，Java语言选择了后者，并将中间代码称为Java二进制代码。如果是保存成`文件`，生成并保存中间代码的过程称为编译

编译器和解释器唯一的区别在于，之后它不是通过eval方法执行程序，而是将抽象语法树转换成机器语言，并以文件形式保存

中间代码解释器称为虚拟机，处理的中间语言称为虚拟机器语言。虚拟机由若干寄存器和内存组成。内存分为四个区域，分别是栈区、堆区、程序代码区和文字常量区。虚拟机器语言保存在程序代码区，字符串字面量保存在文字常量区

寄存器包括通用寄存器、pc（程序计数器）、fp（帧指针）、sp（栈指针）和ret（函数调用）五个寄存器。

```
# 引用局部变量的值
move 3 r1      # 当然一般机器语言是写成这样的格式 move r1 3(fp)
move r1 3

# 引用全局变量的值
gmove 3 r1    # 这里指的是将堆区第3个元素的值复制到寄存器r1中
gmove r1 3
```

调用惯例指的是：函数调用所需的实参和返回值的传递方式，以及栈帧的传递方式

## 14. 为Stone语言添加静态类型支持以优化性能

**类型声明和推论**，如果语句没有指定数据类型，变量的类型将取决于类型推论的结果。例如`def inc(n: Int): Int {n + 1}`

```
type_tag : ":" IDENTIFIER
variable : "var" IDENTIFIER [ type_tag ] "=" expr
param    : IDENTIFIER [ type_tag ]
def      : "def" IDENTIFIER param_list [type_tag ] block
statement: variable | "if" ... | "while" ... | simple
```

此外还有**类型检查**，在对赋值表达式进行类型检查时，右侧表达式的类型只要和左侧的变量类型相同，或是它的子类即可

## 15. 手动设计词法分析器

正则表达式能够表达非常复杂的字符串模式匹配逻辑，可以由有限状态机来实现。但有部分嵌套不能够用正则实现，例如注释嵌套注释

```
# 此时可能只能匹配到ruby后面的*/，不能匹配到scala后面的*/
/* java /* ruby */ scala */
```

## 16. 语法分析方式

正则表达式用于字符串的匹配，BNF用于单词序列的匹配。如果将字符串的每个字符视为一个单词，则正则表达式和BNF的本质相同。注意需要包含终止符。此外，BNF还支持递归定义，这个是和正则的不同

```
[0-9][0-9]*
digit: "0" | "1" | "2" | ... | "8" | "9"
number: digit | digit number
```

使用自动工具将BNF语法转换为语法分析器，例如yaac，基于的是LALR语法分析。其他的语法分析有LR(Left-to-right，Rightmost derivation)语法分析，是自底向上的分析器；LL(Left-to-right, Leftmost derivation)语法分析，是利用递归的自顶向上的语法分析器。LL语法分析比LR的简单，但分析能力较弱

大部分语法分析器都试图从前往后逐一读取单词序列，并依次进行【规约(reduce)】。所以每读取一个单词后，有三种结果：该单词为终止符，则规约该模式；该单词为某种模式的起始部分；该单词处于某种模式的中段。其中，后两种结果称为【移进(shift)】

移进和规约冲突可通过运算符优先级或者更复杂的语法规则进行分析

## 17. Parser库的内部结构

Parser类的parse方法将根据构造的模式执行语法分析。它主要采用LL语法分析方式，并根据需要，部分使用了算符优先分析法。\

比如其中一个分支是左括号、表达式和右括号，另一个分支是Number，则可以构造为：

```
Parser factor = rule().or(rule().sep("(").ast(expr).sep(")")),  rule().number());
```

## 18. GluonJ的使用方法

修改器需要在编译后，执行程序前，将修改器应用于相关的类。可以通过启动程序来实现这一处理。准备一个main方法，调用由GlouonJ提供的Loader类中的run方法。

例如BasicInterpreter是包含了原本main方法的类，我们将它应用BasicEvaluator修改器

```
public static void main(String[] args) throws Throwable {
	Loader.run(BasicInterpreter.class, args, BasicEvaluator.class);
}
```

## 19. 抽象语法树与设计模式

interpreter解释器模式

```
class Evaluator {
	int eval(ASTree t) {
		return t.eval()
	}
}

class BinaryExpr extends ASTree {
	int eval() {
		...
	}
}

class NumberLiteral Extends ASTree {
	int eval() {
		...
	}
}

```

Vistor访问者模式

```
class Evaluator {
	int eval(ASTree t) {
		EvalVistor v = new EvalVistor();
		t.accept(v);
		return v.result;
	}
}

Class NumberLiteral extends ASTree {
	// 由ASTree的子类接收(accept)指定的访问者(visitor)
	<T> T  accept(Vistor<T> v) {
	  // 再由访问者来visit不同对象
		v.visit(this);
	}
}

class BinaryExpr extends ASTree {
	<T> T accept(Vistor<T> v) {
		v.visit(this);
	}
}

// 通过范型可以让不同的Vistor返回不同类型，增加通用性
class EvalVistor implements Vistor<T> {
	T visit(NumberLiteral n) {
		...
	}

	T visit(BinaryExpr b) {
		...
	}
}

// 此处可以扩展多个Vistor，
// 即和解释器模式相比，将一维的扩展变成二维的扩展
class LookupVisitor implements Vistor {
	void visit(NumberLiteral n) {
		...
	}

	void visit(BinaryExpr b) {
		...
	}
}
```
