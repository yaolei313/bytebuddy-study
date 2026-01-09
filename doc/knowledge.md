# bytebuddy
流程
	“匹配 (Matching) -> 加载字节码 (Locating) -> 解析与修改 (Transforming) -> 加载 (Loading)”  

### 第一阶段：类型匹配 (Matching)

这是工作流的起点，ByteBuddy 需要决定哪些类需要被“动手术”。
	
- **相关参数：**
  - **`.ignore(ElementMatcher)`**: 最先执行。符合此条件的类直接跳过，完全不处理。
  - **`.type(ElementMatcher)`**: 在未被 ignore 的类中，筛选出目标类（如你的 `jakarta.servlet.http.HttpServlet`）。
- **底层逻辑：** 当 JVM 加载一个类时，ByteBuddy 会拦截 `ClassFileTransformer.transform` 调用，并运行这些 Matcher。

------

### 第二阶段：定位原始字节码 (Locating)

确定了要修改类 A，并且要使用 Advice 类 B 来增强它。此时，ByteBuddy 必须拿到 A 和 B 的原始二进制流。
	
- **相关参数：**

  - **`.with(AgentBuilder.LocationStrategy)`**: 定义了**去哪里找**类文件。
  - **`ClassFileLocator`**: 具体的执行者。

- **在你的代码中：**

  Java

  ```
  .with(AgentBuilder.LocationStrategy.ForClassLoader.STRONG
      .withFallbackTo(ClassFileLocator.ForClassLoader.of(AgentClassLoader), ...))
  ```

  这告诉 ByteBuddy：如果 `AppClassLoader` 找不到 Advice 类，就去 `AgentClassLoader` 里找。这一步**只读取字节流，不触发类加载**。

------

### 第三阶段：解析与策略制定 (Strategy)

拿到字节码后，ByteBuddy 需要决定用什么方式来改写它。

- **相关参数：**
  - **`.with(AgentBuilder.TypeStrategy)`**:
    - `REDEFINE`: 完全重写类。
    - `REBASE`: 保存原始方法，改名后添加新逻辑（最安全）。
    - `DECORATE` (你使用的): 仅进行装饰。
  - **`.with(AgentBuilder.PoolStrategy)`**: 定义如何缓存已解析的类信息。`Default.FAST` 使用较轻量的缓存。
  - **`.with(AgentBuilder.DescriptionStrategy)`**: 定义如何解析类的元数据（如注解、字段）。`HYBRID` 会优先从层级结构中找，找不到再看源码。

------

### 第四阶段：转换与织入 (Transforming)

这是最核心的一步，将 Advice 的逻辑真正合入目标类。

- **相关参数：**
  - **`.transform(Transformer)`**: 你定义的 Lambda 表达式。
  - **`AgentBuilder.Transformer.ForAdvice()`**: 专门用于 Advice 模式的转换器。
- **底层逻辑：**
  1. ByteBuddy 使用其内部的 ASM 解析器读取 Advice 类（如 `Servlet5Advice`）。
  2. **报错点**：它会扫描类上的 `@OnMethodEnter`。如果解析器在这一步发现注解不匹配，就会报 `No advice defined`。
  3. 一旦匹配成功，它将 Advice 方法中的字节码指令（Opcode）拷贝到目标方法的开头或结尾。

------

### 第五阶段：初始化与执行 (Initialization)

字节码改写完了，但在目标类运行前，可能需要处理一些静态变量或辅助类。

- **相关参数：**
  - **`.with(AgentBuilder.InitializationStrategy)`**:
    - `NoOp.INSTANCE` (你使用的): 表示不需要在目标类中添加特殊的初始化逻辑（如静态构造块的修改）。
  - **`.with(AgentBuilder.RedefinitionStrategy)`**:
    - `RETRANSFORMATION` (你使用的): 允许对**已经加载到 JVM 里的类**进行重新转换。这在 Agent 晚挂载（Attach）时非常重要。

------

### 第六阶段：安装与监听 (Installing & Listening)

最后，将这个 Transformer 注册到 JVM 的 `Instrumentation` 接口中。

- **相关参数：**
  - **`.installOn(Instrumentation)`**: 最后的启动指令。
  - **`.with(AgentBuilder.Listener)`**: 你的 `TransformListener`。它会捕获整个过程中发生的错误。
- **报错现象**：由于转换是异步或在类加载时发生的，报错不会直接抛给用户，而是通过 Listener 的 `onError` 方法输出日志。

------

### 
