# Java 代码评审提示词（基于《Java 编码规范及最佳实践\_V2.0》）

## 一、代码健壮性检查

### （一）入参与返回值校验

**入参校验**

模块 API 方法需使用注解（@NotNull、@NotEmpty、@Size 等）声明入参规则。

模块内公共方法需通过`Assert`工具类校验入参非空（如`Assert.notNull()`）。

含`@Nullable`注解的入参，使用前必须进行判空处理。

入参或返回值为`Optional`类型时，禁止声明`@Nullable`注解。

**返回值处理**

处理`Optional`返回值时，需通过`orElseThrow()`或`isPresent()`校验存在性。

集合 / 数组返回值使用前，需通过`CollectionUtils.isEmpty()`或判空处理。

声明为`@Nullable`的返回值，使用前必须判空。

公共方法应避免返回`NULL`，集合 / 数组需返回空对象（如`Collections.emptyList()`）而非`NULL`，且内部元素不得为`NULL`。

### （二）异常与边界处理

**空指针与越界**

包装类型需显式转换为基本类型（如`Integer.intValue()`），禁止直接参与数值运算导致空指针风险。

必须使用`Objects.equals()`比较对象，避免`NullPointerException`。

数组转`Stream`前必须校验非空，防止空数组转换异常。

遍历集合时若需增删元素，必须使用`Iterator`的`remove()`方法，禁止直接调用集合的`remove()`方法引发并发修改异常。

**控制流兜底**

`switch-case`必须包含`default`分支，每个`case`需以`break`结束（除非合并逻辑）。

多分支`if-else`必须包含兜底`else`分支，即使逻辑上不会执行。

### （三）其他健壮性规则

**数据库与 IO 操作**

JPA 查询单条记录需使用`findById()`或`Optional`返回值，禁止使用`getOne()`。

IO 资源必须通过`try-with-resources`或`finally`关闭（如`IoUtil.closeQuietly()`），避免内存泄漏。

**数据类型**

禁止使用`float/double`进行精确计算，改用`BigDecimal`或整型。

数据库 Entity/DTO 的基本属性必须使用包装类型（如`Integer`），模块 API 的入参与返回值同理（`exists`/`count`开头的函数除外）。

## 二、代码高性能检查

### （一）基础性能优化

**条件判断**

多条件判断时，需优先执行低代价操作（如属性判断先于数据库查询）。

相同的数据库 / RPC 查询在同一业务流中只能执行一次，避免重复调用。

**循环优化**

循环体内禁止单条查询关联表数据或执行 RPC 调用，需改用批量操作。

循环体内必须使用`StringBuilder`进行字符串拼接，禁止使用`+`运算符。

循环体内打印日志时，仅允许 DEBUG 级别，且需通过`LOGGER.isDebugEnabled()`预处理。

### （二）集合与锁

**集合操作**

`Map`遍历需使用`entrySet()`，禁止通过`keySet()`+`get()`遍历。

集合转数组需使用`toArray(new T[size])`，数组转集合需使用`Arrays.asList()`（注意其不可增删特性）。

**并发控制**

高并发场景需使用细粒度锁（如 Guava `Striped`），禁止使用`Guava Interners`或`String.interner()`。

全局变量 / 单例对象属性必须使用并发安全类型（如`ConcurrentHashMap`、`AtomicInteger`），未声明`@ThreadSafe`的类在属性变更时需加锁。

## 三、代码安全性检查

### （一）并发与数据安全

**线程安全**

全局变量 / 单例对象属性若为非线程安全类型，必须通过加锁保证线程安全；若类声明`@ThreadSafe`，则允许使用非安全类型。

数据库更新必须使用乐观锁（如版本号控制），边界在 Service 层。

**数据安全**

日期格式化必须统一使用`DateTimeFormatter`，禁止使用过时的`SimpleDateFormat`。

### （二）异常处理

**异常捕获与抛出**

禁止捕获`Error`及其子类，以及`IllegalStateException`、`NullPointerException`等运行时异常（需通过前置校验避免）。

仅允许主动抛出`BusinessException`及特定运行时异常（如`IllegalArgumentException`、`UnsupportedOperationException`）。

`catch`异常后必须进行处理（记录日志、抛出或转换），禁止空捕获。

**异常处理结构**

同一个方法中禁止嵌套`try`关键字，`try`代码块代码不得超过 20 行，`finally`中禁止出现`return`语句。

禁止在`catch`中通过`instanceof`或强转处理异常，需按异常类型分别捕获（框架开发例外）。

## 四、日志规范检查

**日志记录规则**

公有方法调用失败后（如 IO 操作返回`false`、捕获异常）必须立即记录日志，包含完整堆栈信息（如`LOGGER.error(..., e)`）。

日志参数禁止包含数据状态变更操作（如`i++`），仅允许`toJson()`、`toString()`等简单方法调用。

INFO 及以下级别日志必须通过`if (LOGGER.isDebugEnabled())`预处理，避免性能损耗。

## 五、代码可读性检查

### （一）命名规范

**标识符规则**

包名、类名、方法名、变量名必须符合驼峰 / 全大写规则，禁止使用拼音、中文或混合命名，数组定义中`[]`需放在变量名前（如`String[] args`）。

测试类以`Test`结尾，测试方法以`test`开头；异常类以`Exception`结尾，抽象类以`Abstract`开头，设计模式需在类名中体现（如`OrderFactory`）。

常量必须全大写并使用`下划线分隔`（如`DEFAULT_PAGE_SIZE`），`long`类型赋值需使用大写`L`（如`2L`）。

**语义清晰性**

方法名必须为动词短语（如`validateInput()`），布尔方法以`is`/`has`开头且仅用于状态判断（如`isEmpty()`），`check`/`validate`开头的函数仅用于数据校验，失败时抛出`BusinessException`。

### （二）代码风格

**格式与结构**

左大括号前必须有空格（如`if (condition) {`），单行代码块必须用`{}`包裹，代码缩进为 4 个空格。

方法长度不得超过 50 行，`if/else`/`while`/`for`块不得超过 20 行，lambda 表达式不得超过 5 行，连续代码超过 10 行需用空行分隔。

运算符号左右必须加空格（如`int i = 1;`），禁止级联调用超过两层（`stream`/`StringBuilder`等 API 除外）。

**复杂度控制**

`if/while/for`判断条件不得超过两个，超过时需拆分为独立方法；三目运算符禁止包含复杂语句，需用`if-else`替代。

### （三）注释规范

**文档注释**

类、接口、POJO 必须使用`/** ... */`注释，说明功能、设计意图及版权信息；废弃接口需在注释中说明替代方案（`@Deprecated`+ 使用说明）。

**代码注释**

方法内部注释需解释 “为什么” 而非 “做什么”，无用代码必须删除，禁止注释保留。

使用`TODO`/`FIXME`标记待处理问题，`TODO`需明确后续计划，`FIXME`需优先处理。

禁止使用魔法数字，必须定义有意义的常量（如`public static final int PAGE_SIZE = 10;`）。

## 六、工具类使用规范

### （一）优先级与禁用规则

**工具类选择**

优先使用 SK-Java 工具类（如`com.ruijie.rcos.sk``.base.util.StringUtils`），其次为 JDK、Spring、Guava、Apache Commons 工具类，遵循优先级顺序。

建议使用 Guava 的`Lists/Maps/Sets`工厂方法构造固定长度集合类。

**禁用工具类**

严禁使用第三方包内同名工具类（如`org.springframework.cglib.core.CollectionUtils`、`org.junit.Assert`），避免冲突。

## 七、其他规范

**事务边界**

数据库事务内（tx 包）只能编写数据库处理逻辑，禁止混合业务逻辑。

**哈希与集合**

作为`HashMap`键或`HashSet`元素的对象必须同时重写`equals()`和`hashCode()`。

禁止修改集合类返回值，如需合并集合需创建新集合或使用`Stream.concat()`，仅`fill`/`put`/`convert`语义函数允许修改入参。

## 评审总结

请对照以上检查点进行代码评审，重点关注【强制】规范（必须严格遵守），【建议】规范作为最佳实践优先采纳。确保代码在健壮性、高性能、安全性上符合要求，同时具备良好的可读性和可维护性。对于违反规范的部分，需评估影响并记录整改计划，确保代码质量达标。