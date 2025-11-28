# 别名模型： 树状抽象域

某种意义上是LLVM的TBAA和 MLIR 的 scoped alias/noalias混合。它本身具有两可性，既是可以用作面向上层编译器和程序员的hint接口，也可以用作编译器做别名分析时依赖的专属类型系统(Domain-Specific Type System)。不过本规范仅仅指定它用作Hint接口，是否成为优化器的某一个实现真正采用的专属类型系统则不做规定。它是一套依赖集合与格运算的框架。

## 抽象域

别名分析的基础概念是抽象域，抽象域可以并不意味着一块连续的内存，而仅是一种抽象的概念。比如全体堆中内存构成的域或者全体双精度浮点数指针指向构成的域，这个两个域并意味着两块连续的内存，一个指针可以同时属于两个域。域是一种特殊的集合，有从属关系，可以进行相交相并的运算。本文中，相交非空被简称为相交。

在本模型中，全体域都从属一个树状关系，编译器自动为这颗树生成统一的虚拟根。树状关系自动表达了从属，一个节点对应必然属于它的父节点。属于关系是一种偏序，是自反传递的。所以全部的抽象域构成了一个半格。我们设计的核心因素之一就是，并不假定一个父节点下属的全体子节点相交非空或者说它们是否互斥。举个例子，虚拟根节点，可以下属类型子树和域子树。类型子树和域子树，可以是相交的，并不互斥。一个指针可以同时是双精度浮点的堆中指针。


## 节点属性与相交关系

在树状抽象域模型中，每个**抽象域**称为**节点**。每个节点必须通过**互斥属性**定义其与兄弟节点的基本关系。互斥属性分为两类：

- **互斥（Exclusive）**：表示节点与兄弟节点互不相交
- **相交（Inclusive）**：表示节点与兄弟节点可能相交

### 相交关系判定规则

两个节点A和B的相交关系由它们的互斥属性和**局部优先级**共同决定：

1. **相同属性节点**：
   - 两个互斥节点：必然不相交（N）
   - 两个相交节点：必然相交（T）

2. **不同属性节点**（一个互斥，一个相交）：
   - 如果相交节点的优先级 > 互斥节点的优先级，则相交（T）
   - 如果优先级相同或互斥节点优先级更高，则不相交（N）

### 提升规则（关键算法）

当需要判断任意两个节点A和B的相交关系时：

1. 如果A和B是**兄弟节点**（同一父节点）：
   - 直接应用上述基于属性和优先级的相交规则

2. 如果A和B**不是兄弟节点**：
   - 寻找它们深度最小的**公共祖先节点C**
   - 在C的直接子节点中，找到A的祖先节点D和B的祖先节点E
   - D和E是兄弟节点，它们的相交结果即为A和B的相交结果

### 全集规则

假定每个节点都有其独占部分，不属于任何兄弟节点。因此：
- 兄弟节点的并集等于父节点，**当且仅当**并集包含了该父节点的所有直接子节点

## 抽象域树的构建

### 语法定义

使用 `#AliasTree` 指令构建抽象域树节点：
```
#AliasTree(name_id, parent_id, attri_id, priority)
```

**参数说明**：
- `name_id`：节点唯一标识符，全树范围内不得重名
- `parent_id`：父节点标识符，必须引用已定义的节点
- `attri_id`：互斥属性，取值为：
  - `Exclusive`：互斥属性
  - `Inclusive`：相交属性
- `priority`：局部优先级，无符号整数字面量，需用反引号包裹

**特殊节点**：
- 根节点名为 `AliasRoot`
- 用户定义节点不得与上述特殊节点重名

### 构建示例

```plaintext
# 第一层：类型和作用域域
#AliasTree(type, AliasRoot, Inclusive, `3`)
#AliasTree(scope, AliasRoot, Inclusive, `3`)

# 类型子树
#AliasTree(int, type, Exclusive, `1`)
#AliasTree(float, type, Exclusive, `1`)
#AliasTree(void, type, Inclusive, `3`)

# 整型子树
#AliasTree(int64, int, Exclusive, `1`)
#AliasTree(intDefault, int, Inclusive, `3`)
#AliasTree(int32, int, Exclusive, `1`)

# 浮点型子树
#AliasTree(float64, float, Exclusive, `1`)
#AliasTree(floatDefault, float, Inclusive, `3`)
#AliasTree(float32, float, Exclusive, `1`)

# 作用域子树
#AliasTree(stack, scope, Exclusive, `1`) 
#AliasTree(heap, scope, Exclusive, `1`)

# 特殊限制域
#AliasTree(restrict, AliasRoot, Exclusive, `3`)
```



### 规定指针从属的域

指针所属的抽象域通过特定的工具链注释来规定。该注释以通用符号 `-!` 开头，后接关键字 `alias`（在Vacuum的语法规范中它实际是个标识符），最后用方括号 `[]` 包裹描述域的规则体。完整的格式为 `-!alias[rules]`，其中方括号是强制性的。

规则体采用类C语言的集合运算语法来定义指针所属的虚拟集合，支持使用小括号 `()` 明确运算优先级：
*   **与运算** `&&` 表示集合的**交集**。
*   **或运算** `||` 表示集合的**并集**。
*   **非运算** `!` 应用于单个节点，表示该节点在**整个抽象域树的根节点（AliasRoot）** 中的补集（请注意，不是其直接父节点）。

运算符优先级与C语言规定一致。以下是指针声明的示例：

```c
ptr-!alias[ (!int && type) && (!heap && scope) ] ptr_1;
ptr-!alias[ (int && stack) || (float && heap) ] ptr_2;

ptr-!alias[ int ] ptr3;
ptr-!alias[ stack ] ptr4;
```


下表是C语言中这三个逻辑运算符的优先级和结合性。

| 运算符 | 名称 | 优先级 | 结合性 |
| :--- | :--- | :---: | :---: |
| **`!`** | 逻辑非 | 最高 | 从右到左 |
| **`&&`** | 逻辑与 | 居中 | 从左到右 |
| **```\|\|```** | 逻辑或 | 最低 | 从左到右 |


您提出了一个重要的补充要求。我来重新组织这部分内容，明确禁止在任何地方使用`AliasRootRoot`。

## 空集的表示与构造

空集是本别名模型中的一个基础概念，表示不与任何其他指针存在别名关系的特殊集合。系统提供两种构造空集的方式：

**隐式构造**：两个默认规则下互斥的集合，它们的交集即为空集。例如，根据互斥属性表，任何集合与"强互斥"集合的交集都为空集。在别名判断中，如果一个指针的域被定义为两个互斥域的"与"操作（如 `int && float`），且根据规则它们的交集为空，则该指针的从属域为空集。

**显式构造**：通过根节点的补集显式指定空集，语法为 `-!alias[!AliasRoot]`。这是系统推荐的显式空集表示方法，提供了明确的语义表达。

## 根节点默认规则与安全设计

基于工程安全性的考虑，系统采用了以下保守性规则：

- **空规则体处理**：当规则体为空（即 `-!alias[]`）时，该指针被解释为从属于**根节点（AliasRoot）**，即全集。这一设计虽然与集合论的基础直觉（空规则体对应空集）相反，但确保了不会因标注疏忽导致危险的激进优化。

- **未标注规则**：如果指针声明中**完全省略** `-!alias[...]` 注解，编译器同样应默认该指针从属于根节点，遵循相同的安全保守原则。

## 根节点的使用规范与限制

### 允许的使用方式
根节点 `AliasRoot` 可以在规则中显式使用，特别是通过其补集 `!AliasRoot` 来表示空集。例如：
```c
ptr-!alias[!AliasRoot] non_aliasing_ptr;  // 明确声明为空集
ptr-!alias[AliasRoot] conservative_ptr;  // 显式声明为根节点（与空规则体等效）
```

## 通过集合运算判断指针别名


**基本规定** 两个变量所属的域是无交的，那么两个指针不会重名。

如果由于编译器运行时限或者说编译器策略有限，两个确实无交的集合如果无法证明是无交的，那么它会保守的判断两个无交的集合是可能相交的，可能出现别名。确实无交，但是编译器出于自身原因没有证明无交的，误认为相交不会引起错误，仅仅失去优化可能。

上述的例子中`ptr1` 所属的集合和`ptr2`无交。但是`ptr3`和 `ptr4`相交，可能存在别名。

一个指针如果从属虚拟集合为空集，那么它和任意其它指针所属的域都是无交的，换言之是无别名的。一个指针如果从属的虚拟集合为根节点，那么它除了和从属虚拟集合为空集指针相交为空之外，和任意其指针有交集，进而有别名。根集合的非运算指定为空集，根集合为这个上半格系统的全集。

# 运算规则

## 1. 基本代数定律

抽象域的集合运算遵循标准布尔代数的基本定律，这些定律确保了表达式化简和变换的正确性。

**符号定义：**
- 设 $A$, $B$, $C$ 为任意抽象域集合
- $\cup$ 表示并集运算（对应 `||`），$\cap$ 表示交集运算（对应 `&&`）
- $\neg$ 表示补集运算（相对于根节点 AliasRoot，对应 `!`）

**三大基本定律：**

- **结合律**：在连续进行同种运算时，分组方式不影响结果
  - 并集结合律：$A \cup (B \cup C) = (A \cup B) \cup C$
  - 交集结合律：$A \cap (B \cap C) = (A \cap B) \cap C$

- **交换律**：运算数的顺序可以任意交换
  - 并集交换律：$A \cup B = B \cup A$
  - 交集交换律：$A \cap B = B \cap A$

- **分配律**：一种运算对另一种运算的分配关系
  - 交集对并集分配：$A \cap (B \cup C) = (A \cap B) \cup (A \cap C)$
  - 并集对交集分配：$A \cup (B \cap C) = (A \cup B) \cap (A \cup C)$

**实际应用示例：**
假设我们要分析表达式 `(int && stack) || (float && heap)`：
- 应用分配律可重写为：`(int || float) && (int || heap) && (stack || float) && (stack || heap)`
- 这种变换可能帮助识别某些明显的空集情况，比如如果已知 `int || float` 为空，则整个表达式为空

## 2. 德摩根定律

德摩根定律描述了补集、并集和交集之间的转换关系，是表达式化简的重要工具。

**形式化描述：**
- $\neg(A \cup B) = \neg A \cap \neg B$
- $\neg(A \cap B) = \neg A \cup \neg B$

**文字说明：**
并集的补集等于各补集的交集，交集的补集等于各补集的并集。这允许我们将补集运算"推入"括号内部，从而简化复杂表达式的分析。

**实际应用示例：**
```
!(int || float) = !int && !float
!(int && stack) = !int || !stack
```
在分析指针 `ptr-!alias[!(int||float)]` 时，可以将其转换为 `ptr-!alias[!int && !float]`，可能更容易进行后续分析。

## 3. 升链规则（保守近似规则）

升链规则是别名分析中保守性的核心保证，它允许在信息不完整时进行安全但可能不够精确的判断。

**形式化描述：**
如果 $B \subseteq A$（$B$ 是 $A$ 的子集），那么对于任意集合 $C$：
- 若 $A \cap C = \emptyset \Rightarrow B \cap C = \emptyset$（确定无别名，可用于优化）
- 若 $B \cap C \neq \emptyset$ 并不必然推导出 $A \cap C \neq \emptyset$，但是编译器可以选择认为 $A \cap C \neq \emptyset$，这是一种保守近似，它会丧失优化机会但是不会引起错误。

**实际应用示例：**
假设有域树结构：$\text{type} \supseteq \text{int} \supseteq \text{int32}$，即 $\text{int32} \subseteq \text{int} \subseteq \text{type}$
- 要判断指针 $p$（属于 $\text{int32}$）和 $q$（属于 $\text{stack}$）是否可能有别名
- 如果我们能证明 $\text{int} \cap \text{stack} = \emptyset$，则由于 $\text{int32} \subseteq \text{int}$，可确定 $\text{int32} \cap \text{stack} = \emptyset$（无别名），
- 但是如果按照我们的”建立抽象域构成的树一节“中的例子的设定 $\text{int} \cap \text{stack} \neq \emptyset$。此时认为 $\text{int32} \cap \text{stack}$ 非空。这一认定并不源自保守近似，而是实际上来自提升规则带来的强制规定。保守规则其实更多的用于补集的计算。



这种保守性确保了安全性：宁可错过优化机会，也不进行错误的优化。

## 4. 补集计算规则

补集规则处理复杂集合运算的近似计算，特别是在兄弟节点覆盖不完整的情况下。

**符号定义：**
- 设 $P$ 为父节点，$S_1, S_2, \ldots, S_n$ 为其子节点
- 设 $U = S_1 \cup S_2 \cup \ldots \cup S_n$ 为某些子节点的并集

**补集计算规则：**
- 如果 $U = P$（子节点覆盖整个父节点），则 $\neg U = \neg P$
- 如果 $U \neq P$（子节点未完全覆盖父节点），则可以选择使用保守规则 $\neg U \subseteq \neg P \cup (P \cap \neg U) \subseteq \neg P \cup (\text{兄弟节点中不在}U\text{的节点})$

**实际应用示例：**
假设 type 节点有三个子节点：`int`, `float`, `void`
- 情况1：如果 $\text{int} \cup \text{float} \cup \text{void} = \text{type}$（完全覆盖）
  - 则 $!(\text{int} \parallel \text{float}\parallel \text{void}) = !\text{type} $
- 情况2：如果 $\text{int} \cup \text{float} \neq \text{type}$（未完全覆盖）
  - 则 $!(\text{int} \parallel \text{float}) \subseteq !\text{type} \parallel \text{void}$（保守上界近似，只有在`void`强互斥的时候等号成立）

# 别名运算算法流程简例

## 步骤1：交集计算
给定两个指针的域表达式 $E_1$ 和 $E_2$，计算它们的交集表达式：
$$E = E_1 \cap E_2$$

**示例：**
- $E_1 = \text{int} \cap \text{stack}$, $E_2 = \text{float} \cap \text{heap}$
- $E = (\text{int} \cap \text{stack}) \cap (\text{float} \cap \text{heap}) = \text{int} \cap \text{stack} \cap \text{float} \cap \text{heap}$

## 步骤2：转换为析取范式（DNF）
将交集表达式转换为多个合取项的并集形式，便于逐个分析。

**DNF 形式：** $E = (T_{11} \cap T_{12} \cap \ldots) \cup (T_{21} \cap T_{22} \cap \ldots) \cup \ldots$

**示例：**
表达式 $(\text{int} \parallel \text{float}) \&\& \text{stack}$ 可转换为 DNF：
$(\text{int} \cap \text{stack}) \parallel (\text{float} \cap \text{stack})$

## 步骤3：合取支空性判断
对每个合取支 $C = T_1 \cap T_2 \cap \ldots \cap T_k$ 进行分析：

**判断逻辑：**
- 对 $C$ 中每对项 $(T_i, T_j)$，基于互斥属性表检查 $T_i \cap T_j$ 是否为空
- 如果存在任意一对为空，则整个合取支 $C$ 为空
- 如果所有两两交集都不为空，保守认为 $C$ 可能非空

**关键难点：三集合相交问题**
已知：$A \cap B \neq \emptyset$, $B \cap C \neq \emptyset$, $A \cap C \neq \emptyset$
但无法确定：$A \cap B \cap C$ 是否为空
保守策略：在这种情况下认为可能非空，允许存在别名。我们例子中不需要面对这个问题。因为 stack 交 heap 是空的。

## 步骤4：整体别名判断
- 如果所有合取支都为空 $\Rightarrow$ 确定无别名（可安全优化）
- 如果任一合取支可能非空 $\Rightarrow$ 可能存在别名（需保守处理）




## 补充规则

本规范支持补充规则，补充规则的原则是编译器忽略完全忽略编译器和用户设定的补充规则不会引起错误。

### 额外无交假定

本规范支持补充规则，其核心原则是：编译器即使完全忽略用户设定的补充规则，也不会引起分析错误。

- **保守性保证**：在默认规则中相交非空的两个抽象域，可以通过额外规则将其相交关系设定为空。反之，在默认规则中已经相交为空的两个抽象域，**不允许**通过额外规则将其设置为相交非空。这一限制确保了补充规则只能进一步严格化别名分析，而不能放松安全性约束。

- **语法格式**：使用指令 `#AliasNoIntersection(id_1, id_2)` 来声明两个抽象域 `id_1` 和 `id_2` 无交集。

#### 示例

假设已有以下抽象域定义：
```
#aliasTree(type, AliasRoot, Inclusive, `3`)
#aliasTree(int, type, Exclusive, `3`)
#aliasTree(float, type, Exclusive, `3`)

#aliasTree(scope, AliasRoot, Inclusive, `3`)  
#aliasTree(stack, scope, Exclusive, `1`)
#aliasTree(heap, scope, Exclusive, `1`)
```

根据默认的互斥属性规则，`int` 和 `stack` 域可能存在交集（即可能存在既是整型又位于栈上的指针）。

如果程序特定地保证了“整型指针绝不会指向栈内存”，则可通过补充规则声明：
```
#AliasNoIntersection(int, stack)
```

**效果**：
- 声明后，编译器可据此优化，判定属于 `int` 域的指针与属于 `stack` 域的指针绝无别名。
- 即使编译器实现选择忽略此条 `#AliasNoIntersection` 规则，它仍会按照默认的 `Exclusive` 属性进行可能更保守的分析，这不会导致错误的优化，只是可能错过一些优化机会。

**错误示例**（违反规范）：
```
# 错误：不能将默认无交的域声明为有交
#AliasNoIntersection(int, float) # 此声明无效且应被编译器忽略或报错
```
因为 `int` 和 `float` 节点在父节点 `type` 下均为 `Exclusive` 属性，根据核心规则表，它们默认已无交。此类声明被禁止，以维护整个别名分析系统的可靠性与一致性。



## 错误处理

本规范要求编译器在处理别名模型定义时，应进行必要的静态检查。一旦发现以下错误，编译器应报告错误并中止编译过程，以确保程序语义的清晰和正确。

### 编译时错误

以下情况会触发编译时错误：

| 错误类型 | 触发条件 | 示例（错误） | 说明 |
| :--- | :--- | :--- | :--- |
| **节点重名** | 使用 `#AliasTree` 指令创建了同名的节点（包括根节点）。 | `#AliasTree(A, ...)`<br>`#AliasTree(A, ...)` | 节点标识符必须在整个树中唯一。 |
| **引用未定义父节点** | 在 `#AliasTree` 指令中，`parent_id` 指向一个未定义的节点。 | `#AliasTree(Child, UndefinedParent, ...)` | 父节点必须先行定义。 |
| **非法互斥属性** | 在 `#AliasTree` 指令中，`attri_id` 不是四个预定义的属性标识符之一。 | `#AliasTree(N, P, InvalidAttr)` | 属性必须是 `Exclusive`, `Inclusive` 之一。 |
| **指针域表达式语法错误** | `-!alias[...]` 内的集合表达式存在语法错误，如括号不匹配、使用了未定义的节点标识符或运算符错误。 | `-!alias[ (int && stack ] ptr;` （括号不匹配）<br>`-!alias[ unknownNode ] ptr;` （节点未定义） | 表达式必须符合规定的类C语法。 |
| **补充规则冲突** | 使用 `#AliasNoIntersection` 试图将默认规则中已无交的两个域声明为无交。 | `#AliasNoIntersection(int, float)` （假设它们默认已弱互斥） | 补充规则只能收紧约束，不能放松。 |



### 小结：编译器何时会报错？

总结来说，编译器在处理本别名模型时，会在以下两类主要情况下报错：

1.  **模型定义不合法**：包括但不限于抽象域树的结构性问题（如重名、未定义引用）和规则本身的语法错误。这类错误在编译初期即可发现，必须由程序员修正。
2.  **规则逻辑冲突**：试图通过补充规则放松默认的、更严格的别名关系（例如，强制让两个互斥的域相交）。这违反了系统设计的“保守性”基本原则，编译器不应接受。

通过以上的错误处理规定，可以确保别名模型信息在源头上是正确和一致的，为后续的别名分析和优化奠定可靠的基础。



## 文法规则规范

### 1. 抽象域树定义文法

#### 程序结构
```
Program ::= (AliasTreeDefinition | AliasNoIntersection)*
```

#### 抽象域节点定义
```
AliasTreeDefinition ::= "#AliasTree" "(" NodeName "," ParentNode "," Attribute "," Priority ")"

NodeName      ::= Identifier
ParentNode    ::= Identifier
Attribute     ::= "Exclusive" | "Inclusive"
Priority      ::= "`" UnsignedInteger "`"
```

**语义说明**：
- `NodeName` 必须在整个抽象域树中唯一
- `ParentNode` 必须引用已定义的节点（除根节点 `AliasRoot` 外）
- `Priority` 为无符号整数，仅在**同一父节点下的兄弟节点之间**具有比较意义
- 根节点 `AliasRoot` 由系统预定义，不需要也不允许用户定义

#### 补充无交规则
```
AliasNoIntersection ::= "#AliasNoIntersection" "(" NodeName "," NodeName ")"
```

**语义约束**：
- 两个 `NodeName` 必须引用已定义的节点
- 该规则只能将默认规则中**可能相交**的节点声明为无交
- 不能将默认规则中**已经无交**的节点声明为有交（此类声明将被忽略或报错）

### 2. 指针别名注解文法

#### 变量声明语法
```
VariableDeclaration ::= TypeSpecifier AliasAnnotation? Identifier 

AliasAnnotation ::= "-!alias" "[" DomainExpression "]"
                  | "-!alias" "[" "]"  // 空规则体：表示根节点 AliasRoot
```

**语义说明**：
- 注解以 `-!alias` 开头，后接用方括号包裹的域表达式
- 空规则体 `[]` 表示该指针属于根节点（最保守的别名假设）
- 完全省略注解的指针默认属于根节点（安全性优先原则）

#### 域表达式语法
```
DomainExpression ::= DomainTerm
                   | DomainExpression "||" DomainTerm        // 并集运算

DomainTerm       ::= DomainFactor
                   | DomainTerm "&&" DomainFactor            // 交集运算

DomainFactor     ::= NodeReference
                   | "!" DomainFactor                        // 补集运算
                   | "(" DomainExpression ")"                // 括号分组

NodeReference    ::= Identifier  // 必须引用已定义的抽象域节点
```

#### 运算符优先级与结合性
按优先级从高到低排列：

| 运算符 | 名称 | 优先级 | 结合性 | 说明 |
|--------|------|--------|--------|------|
| `!` | 逻辑非 | 最高 | 从右到左 | 相对于 AliasRoot 的补集 |
| `&&` | 逻辑与 | 中等 | 从左到右 | 集合交集运算 |
| `||` | 逻辑或 | 最低 | 从左到右 | 集合并集运算 |

### 3. 完整语法示例

#### 抽象域定义示例
```plaintext
#AliasTree(type, AliasRoot, Inclusive, `3`)
#AliasTree(int, type, Exclusive, `1`)
#AliasTree(float, type, Exclusive, `1`)
#AliasTree(stack, AliasRoot, Exclusive, `2`)
#AliasNoIntersection(int, stack)
```

#### 指针声明示例
```plaintext
// 简单节点引用
ptr-!alias[int] ptr1;

// 交集运算
ptr-!alias[int && stack] ptr2;

// 并集运算  
ptr-!alias[(int && stack) || (float && heap)] ptr3;

// 补集运算
ptr-!alias[!int && type] ptr4;

// 空规则体（根节点）
ptr-!alias[] ptr5;

// 无注解（默认根节点）
ptr ptr6;
```

### 4. 语法约束与验证

#### 编译时检查
编译器必须在编译时验证以下约束：
1. **节点唯一性**：所有 `NodeName` 必须唯一
2. **父节点存在性**：`ParentNode` 必须已定义（根节点 `AliasRoot` 除外）
3. **节点引用有效性**：所有 `NodeReference` 必须指向已定义的节点
4. **表达式语法正确性**：括号匹配、运算符使用正确
5. **优先级数值有效性**：`Priority` 必须为无符号整数

#### 错误处理
违反上述约束应导致编译错误，确保别名分析模型的完整性和一致性。

这个重构后的文法规范更加严谨，明确了优先级的作用范围，保持了与语义规则的一致性，并提供了清晰的语法示例和约束条件。