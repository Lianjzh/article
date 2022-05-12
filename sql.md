

一条SQL的处理过程如下：

![spark_catalyst_1](images/spark_catalyst_1.jpeg)

* 首先会通过解析器，将其解析为一个抽象语法树(AST)，这叫做UnresolvedRelation LogicalPlan。

* 之后进入analysis阶段，可以将其分为几个子阶段
    * `Hints` 比如BroadcastJoinHints处理
    * `Simple Sanity Check` 简单check，比如检查SQL中的Function是否存在
    * `Substitution` 对SQL中的一些进行替换，比如如果union 只有一个child，则取消union
    * `Resolution` 对SQL中的一些信息进行绑定，这样就是Resolved LogicalPlan
    * `Post-Hoc Resolution resolution`之后的操作，默认是空，用户可以自己注入
    * 之后还有其他阶段，所以analysis阶段，不止resolution

* 接下来进行optimization阶段，使用Rule对LogicalPlan 进行优化，得到Optimized LogicalPlan

* 之后是通过使用`SparkStrategy`将`LogicalPlan`转换为可执行的物理计划`SparkPlan`。

* 之后进行codeGen

LogicalPlan是逻辑计划，SparkPlan是物理计划，两者都是QueryPlan的子类。

    abstract class SparkPlan extends QueryPlan[SparkPlan] with Logging with Serializable

    abstract class LogicalPlan
    extends QueryPlan[LogicalPlan]
    with LogicalPlanStats
    with QueryPlanConstraints
    with Logging

Rule不会发生质变，即logical plan=>logical plan，spark plan=>spark plan，Strategy会发生质变，即logical plan=>physical plans。

## TreeNode
一个表达式或者QueryPlan都是一个TreeNode。

    abstract class Expression extends TreeNode[Expression]


## 1、Parser
    class SparkSqlParser(conf: SQLConf) extends AbstractSqlParser

    abstract class AbstractSqlParser extends ParserInterface with Logging

    trait ParserInterface

例子：
1. PlanParserSuite中的test("simple select query")

## 2、Analyzer
    class Analyzer(
    catalog: SessionCatalog,
    conf: SQLConf,
    maxIterations: Int)
    extends RuleExecutor[LogicalPlan] with CheckAnalysis

分析阶段的规则基本都在Analyzer的batches里面列举，然后调用execute方法生效。

    object EliminateUnions extends Rule[LogicalPlan] {
    def apply(plan: LogicalPlan): LogicalPlan = plan transform {
    case Union(children) if children.size == 1 => children.head
    }
    }

例子
1. AnalysisSuite中的test("Eliminate the unnecessary union")

## 3、Optimizer
    abstract class Optimizer(sessionCatalog: SessionCatalog)
    extends RuleExecutor[LogicalPlan]

优化阶段的规则基本都在Optimizer的batches里面列举，然后调用execute方法生效。

## 4、WSCG
在生成了RDD后，RDD算子中执行的代码会通过Code Generate自动生成代码来执行。

CodeGenerator类

    def generate(expressions: InType, inputSchema: Seq[Attribute]): OutType =
    generate(bind(expressions, inputSchema))

    def generate(expressions: InType): OutType = create(canonicalize(expressions))

    org.apache.spark.sql.catalyst.expressions.codegen.GenerateUnsafeProjection.create

使用Janino来将source code编译成class

FunctionRegistry是内部函数注册，UDFRegistration内部通过FunctionRegistry注册函数。

例子
1. ConstantFoldingSuite

## 实践
借助SparkSessionExtensions进行扩展解析器和优化器，借助SparkSession的experimental.extraStrategies扩展策略。

1、扩展解析器，遇到select *提示必须指定列
2、扩展优化器
3、扩展策略

后续：
1、sql parser、ast以及anltr
