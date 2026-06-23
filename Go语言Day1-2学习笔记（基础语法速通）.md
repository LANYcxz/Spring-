# Go 语言 Day 1-2 学习笔记：基础语法速通

> **学习阶段**：Phase 1 · Week 1 · Day 1-2
>
> **目标**：完成环境搭建，掌握 Go 基础语法，能编写包含完整错误处理的 Go 程序。
>
> **参考背景**：已有 Java + Spring Boot 分布式开发经验。

---

## 一、环境搭建

### 1.1 安装 Go

```bash
# macOS 用 Homebrew 安装
brew install go

# 验证安装
go version
# 期望输出：go version go1.22.x darwin/arm64
```

### 1.2 关键环境配置

```bash
# 查看全部环境变量
go env

# 开启 Go Modules（现代项目必须）
go env -w GO111MODULE=on

# 设置国内镜像代理（大陆网络必须，否则依赖极慢）
go env -w GOPROXY=https://goproxy.cn,direct
```

### 1.3 创建第一个项目

```bash
mkdir go-learning && cd go-learning

# 初始化 Go Module（类比 mvn archetype:generate）
go mod init github.com/yourname/go-learning
```

生成的 `go.mod`（类比 `pom.xml`）：

```
module github.com/yourname/go-learning

go 1.22
```

---

## 二、Hello World 解剖

```go
package main   // ① 每个 Go 文件必须声明所属包

import "fmt"   // ② 导入标准库（类比 Java import）

// ③ main 函数是程序入口（类比 public static void main(String[] args)）
func main() {
    fmt.Println("Hello, Go!")
}
```

**运行方式：**

```bash
go run main.go        # 直接运行，不生成二进制（类比 java Main）
go build -o app .     # 编译为可执行文件（类比 mvn package）
./app                 # 执行编译产物
```

### ⚡ 与 Java 的第一印象差异

| 特性 | Java | Go |
|---|---|---|
| 入口函数签名 | `public static void main(String[] args)` | `func main()` |
| 语句结尾分号 | 必须 `;` | 不需要（编译器自动插入） |
| 类型位置 | `String name`（类型在前） | `name string`（类型在后） |
| 编译速度 | 慢 | 极快（秒级） |
| 运行方式 | 需要 JVM | 编译为原生二进制，直接执行 |

---

## 三、变量与类型系统

### 3.1 四种变量声明方式

```go
// ① 完整声明（类比 Java: String name = "Alice";）
var name string = "Alice"

// ② 类型推断
var age = 25

// ③ 短变量声明（最常用，仅限函数体内）
city := "Beijing"

// ④ 批量声明
var (
    firstName string = "John"
    lastName  string = "Doe"
    score     int    = 100
)
```

> ⚠️ **重要规则**：Go 声明的变量**必须被使用**，否则编译报错！
> ⚠️ **`:=` 只能在函数体内使用**，包级别变量只能用 `var`。

### 3.2 基础数据类型

```go
var i   int     = 42          // 平台相关（64 位系统 = int64）
var i32 int32   = 42
var i64 int64   = 42
var f32 float32 = 3.14
var f64 float64 = 3.14159     // 默认浮点类型
var b   bool    = true
var s   string  = "Hello, 世界" // UTF-8 字节序列（非 Java 的 UTF-16）
var bt  byte    = 255          // byte = uint8（无符号，Java byte 是有符号）
```

**零值（Go 的安全保证，声明即初始化）：**

```go
var zeroInt    int     // = 0
var zeroFloat  float64 // = 0.0
var zeroBool   bool    // = false
var zeroString string  // = ""
// 指针、slice、map、channel、interface 的零值是 nil
```

### 3.3 类型转换（必须显式，Go 无隐式转换）

```go
var i int     = 42
var f float64 = float64(i)   // 必须显式，Java 中 int→double 是自动的
var u uint    = uint(f)

// 字符串 ↔ 数字（用 strconv 包，性能优于 fmt.Sprintf）
s   := strconv.Itoa(42)      // int → string
n, err := strconv.Atoi("42") // string → int，注意有 error
if err != nil {
    fmt.Println("转换失败:", err)
}
```

---

## 四、控制流

### 4.1 if 语句

```go
// ① 条件不需要括号（与 Java 不同）
if age >= 18 {
    fmt.Println("成年")
} else if age >= 12 {
    fmt.Println("青少年")
} else {
    fmt.Println("儿童")
}

// ② if 初始化语句（Go 特色，极其常用！）
// 变量 err 的作用域仅限于 if/else 块
if user, err := findUser(id); err != nil {
    fmt.Println("找不到用户:", err)
} else {
    fmt.Println("用户名:", user.Name)
}
```

### 4.2 for 循环（Go 只有 for，没有 while）

```go
// ① 标准 for（类比 Java for）
for i := 0; i < 5; i++ {
    fmt.Println(i)
}

// ② 只有条件（类比 Java while）
n := 1
for n < 100 {
    n *= 2
}

// ③ 无限循环（类比 Java while(true)）
for {
    break // 用 break 退出
}

// ④ range 遍历 slice（类比 Java for-each）
nums := []int{1, 2, 3, 4, 5}
for index, value := range nums {
    fmt.Printf("index=%d, value=%d\n", index, value)
}
// 只要值，用 _ 忽略下标
for _, value := range nums {
    fmt.Println(value)
}

// ⑤ range 遍历 map（顺序不固定！）
scores := map[string]int{"Alice": 95, "Bob": 87}
for name, score := range scores {
    fmt.Printf("%s: %d\n", name, score)
}

// ⑥ range 遍历字符串（得到 Unicode 码点，非字节）
for i, ch := range "Hello,世界" {
    fmt.Printf("位置%d: %c\n", i, ch)
}
```

> ⚠️ **遍历 map 的顺序不固定**，每次运行可能不同，这是 Go 刻意设计的。

### 4.3 switch 语句（默认不穿透，无需 break）

```go
// ① 基本 switch（不需要 break，默认不穿透）
switch day {
case "Saturday", "Sunday": // 多个 case 合并
    fmt.Println("周末")
case "Monday":
    fmt.Println("星期一")
default:
    fmt.Println("其他工作日")
}

// ② 无条件 switch（等价于 if-else if 链，更清晰）
switch {
case score >= 90:
    fmt.Println("优秀")
case score >= 60:
    fmt.Println("及格")
default:
    fmt.Println("不及格")
}
```

---

## 五、函数

### 5.1 函数定义

```go
// ① 基本函数（返回类型在最后）
// Java: public int add(int a, int b) { return a + b; }
func add(a int, b int) int {
    return a + b
}

// ② 同类型参数可简写
func add2(a, b int) int {
    return a + b
}

// ③ 多返回值（Go 最重要的特性之一）
// Java 没有原生多返回值，需要封装 Pair 或自定义类
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("除数不能为 0")
    }
    return a / b, nil
}

// ④ 命名返回值（裸 return）
func minMax(nums []int) (min, max int) {
    min, max = nums[0], nums[0]
    for _, n := range nums[1:] {
        if n < min { min = n }
        if n > max { max = n }
    }
    return  // 自动返回命名变量 min 和 max
}
```

### 5.2 函数是一等公民

```go
// ① 函数作为参数（类比 Java Function<T,R> 函数式接口）
func applyOperation(a, b int, op func(int, int) int) int {
    return op(a, b)
}

result := applyOperation(10, 3, func(x, y int) int {
    return x + y  // 匿名函数（Lambda）
})

// ② 闭包（捕获外部变量）
multiplier := 3
triple := func(x int) int {
    return x * multiplier  // 捕获外部变量
}
fmt.Println(triple(5)) // 15
```

---

## 六、错误处理哲学

> Go 的错误处理是**一等公民**，而非异常机制。这是从 Java 迁移时最大的思维转变。

### 6.1 核心规则

```go
// ❌ Java 开发者常犯错误：用 _ 忽略 error
result, _ := divide(10, 0)  // 危险！

// ✅ 正确做法：每个 error 都必须处理
result, err := divide(10, 0)
if err != nil {
    fmt.Println("错误:", err)
    return  // 或 log.Fatal(err)
}
fmt.Println("结果:", result)
```

### 6.2 错误包装与解包（错误链）

```go
// 自定义哨兵错误（类比 Java 自定义 Exception 类型）
var ErrNotFound = errors.New("记录不存在")

// 包装错误，保留上下文（%w 是关键！）
// 类比 Java: throw new ServiceException("查询用户失败", cause)
func getUser(id int64) (*User, error) {
    user, err := db.Find(id)
    if err != nil {
        // fmt.Errorf + %w 会保留原始 error 的链条
        return nil, fmt.Errorf("getUser id=%d: %w", id, ErrNotFound)
    }
    return user, nil
}

// 解包错误（类比 Java: catch (NotFoundException e)）
err := getUser(999)
if errors.Is(err, ErrNotFound) {
    // 即使 err 被多层包装，errors.Is 也能找到
    fmt.Println("用户不存在，返回 404")
}
```

### 6.3 错误 vs 异常哲学对比

| | Java | Go |
|---|---|---|
| 机制 | 异常（Exception）抛出，可不处理 | error 返回值，必须处理 |
| 强制性 | `checked exception` 强制，`RuntimeException` 可忽略 | 编译器不强制，但 code review 规范要求处理 |
| 性能 | 抛异常有栈展开开销 | 返回 error 无额外开销 |
| 哲学 | 异常是"例外情况" | error 是"预期内的失败结果" |

---

## 七、综合练习代码

```go
package main

import (
    "errors"
    "fmt"
    "strconv"
)

// 自定义哨兵错误
var ErrNegativeNumber = errors.New("不支持负数")

// 将字符串切片转换为整数切片
func parseNumbers(strs []string) ([]int, error) {
    nums := make([]int, 0, len(strs))
    for i, s := range strs {
        n, err := strconv.Atoi(s)
        if err != nil {
            return nil, fmt.Errorf("parseNumbers: 第%d个元素'%s'不是有效整数: %w", i+1, s, err)
        }
        nums = append(nums, n)
    }
    return nums, nil
}

// 计算统计信息（最小值、最大值、平均值）
func statistics(nums []int) (min, max int, avg float64, err error) {
    if len(nums) == 0 {
        err = errors.New("输入不能为空")
        return
    }

    min, max = nums[0], nums[0]
    sum := 0

    for _, n := range nums {
        if n < 0 {
            err = fmt.Errorf("statistics: %w, 发现值: %d", ErrNegativeNumber, n)
            return
        }
        if n < min { min = n }
        if n > max { max = n }
        sum += n
    }

    avg = float64(sum) / float64(len(nums))
    return
}

func main() {
    // ---- 测试正常情况 ----
    inputs := []string{"5", "3", "8", "1", "9", "2"}

    nums, err := parseNumbers(inputs)
    if err != nil {
        fmt.Println("解析失败:", err)
        return
    }

    min, max, avg, err := statistics(nums)
    if err != nil {
        fmt.Println("统计失败:", err)
        if errors.Is(err, ErrNegativeNumber) {
            fmt.Println("原因：包含负数")
        }
        return
    }

    fmt.Printf("最小值: %d\n", min)
    fmt.Printf("最大值: %d\n", max)
    fmt.Printf("平均值: %.2f\n", avg)

    // ---- 测试错误情况 ----
    fmt.Println("\n--- 测试非法输入 ---")
    badInputs := []string{"1", "2", "abc", "4"}
    _, err = parseNumbers(badInputs)
    fmt.Println("错误信息:", err)
}
```

**期望输出：**

```
最小值: 1
最大值: 9
平均值: 4.67

--- 测试非法输入 ---
错误信息: parseNumbers: 第3个元素'abc'不是有效整数: strconv.Atoi: parsing "abc": invalid syntax
```

---

## 八、知识点检验清单

完成 Day 1-2 后，请确认自己能回答以下问题：

- [ ] `var x = 5` 和 `x := 5` 的区别？（后者只能在函数体内，且是新变量声明）
- [ ] 为什么 Go 声明了变量不使用会报编译错误？（强制代码整洁，避免无用变量）
- [ ] `for range` 遍历 map 的顺序是固定的吗？（不固定，Go 刻意随机化）
- [ ] `fmt.Errorf("msg: %w", err)` 中 `%w` 的作用是什么？（包装 error，支持 `errors.Is` 解包）
- [ ] Go 的多返回值和 Java `Optional<T>` 有什么本质区别？（多返回值在编译期强制处理，Optional 可被忽略）
- [ ] Go 的零值机制有什么好处？（声明即安全，避免 NullPointerException）
- [ ] `byte` 和 `int8` 的区别？（byte = uint8 无符号，int8 有符号，Java 的 byte 是有符号）

---

## 九、下一节预告（Day 3-4）

**主题：struct + interface + 方法**

```
- struct：Go 的"类"，如何定义与初始化
- 值接收者 vs 指针接收者：什么时候用哪个？
- interface：隐式实现的原理，比 Java 更灵活在哪里？
- struct embedding（嵌入）：组合替代继承的具体写法
- 类型断言与类型 switch
```

**预习思考：**
> 如果 Go 没有继承，`Dog` 想复用 `Animal` 的 `Eat()` 方法，应该怎么做？

---

*笔记整理日期：2026-06-23 | Phase 1 Week 1 Day 1-2*
