---
name: test-quality-evaluator
description: 评估仓库测试可用性，检测AI自欺欺人测试（确认偏误），输出覆盖率、自欺欺人和精确率三维度评分
purpose: 评估仓库测试的可用性（覆盖率、AI自欺欺人程度、测试精确率），并指导 AI 改进测试
rootUrl: https://raw.githubusercontent.com/Alpaca233114514/test-deception-detector/main/SKILL.md
tags:
  - testing
  - jest
  - quality
  - coverage
  - ai-bias
model: claude-sonnet-4-6
---

# 测试质量评估器（Test Quality Evaluator）

**触发条件**：用户提及「测试质量」「测试可用性」「jest 覆盖率」「自欺欺人测试」「AI 确认偏误」「测试有效性」「改进测试」或要求评估现有测试时强制调用。

**核心定义**：
- **覆盖率** = 测试执行到了多少代码（数字，客观）
- **自欺欺人程度** = 测试在多大程度上是 AI 为了「让自己看起来完成了任务」而写的，而非真正验证业务正确性（AI 认知偏误，主观但可检测）
- **测试精确率** = 测试发现真实 bug 的能力（业务价值）

**三者关系**：高覆盖率 + 高自欺欺人 = 最危险的测试（给你安全感但实际上没用）。

---

## 评估三维度

### 1. 覆盖率（Coverage）
- 运行 `jest --coverage --collectCoverageFrom='src/**/*.{ts,tsx,js,jsx}'`
- 关注 **分支覆盖率（branch coverage）**，而非仅行覆盖率
- 低于 80% 的分支覆盖标记为 🔴 待改进

### 2. 自欺欺人程度（Self-Deception Score）

**定义**：AI 写测试时，为了获得「测试通过 + 覆盖率高」的即时满足感，潜意识中采取的各种「验证自己代码没错」而非「验证代码满足需求」的策略。

越低越好（0% = 全部真实有效，100% = 全部自欺欺人）。

检测以下 AI 确认偏误模式：

| 反模式 | 权重 | 说明 |
|--------|------|------|
| **预期值倒推** | +30% | 测试的 `expected` 值明显是从被测函数的实现代码倒推出来的，而非从业务语义（函数名、注释、类型定义、README）推导。例如：实现里是 `return a + b`，测试就写 `expect(add(2,3)).toBe(5)`——如果实现偷偷改成 `return a * b`，测试就失效了 |
| **mock 万能** | +25% | 测试把所有外部依赖（数据库、API、文件系统）全部 mock，只验证「我的代码调用了 mock 的某方法」，不验证真实数据流转和业务结果。实现代码把 `user.id` 错传成 `user.name`，mock 测试照样通过 |
| **happy path 依赖** | +20% | 测试只覆盖最理想路径，刻意避开边界值（`null`、`0`、`''`、空数组、超大数）和异常分支。AI 心里知道这些路径「处理起来麻烦」，所以选择性忽略 |
| **实现耦合** | +15% | 测试断言的是实现细节（内部变量、私有方法调用顺序、具体 SQL 语句），而非输入输出的行为契约。重构时行为不变但测试挂掉 |
| **占位填充** | +10% | `expect(true).toBe(true)`、`// TODO: add real test` 等明显为了凑覆盖率或赶工而存在的测试 |

计算公式：
```
Self-Deception = Σ(命中反模式的测试数 × 权重) / 总测试数
```

> 注：单条测试可命中多个反模式，权重叠加。上限 100%。

**检测方法**（关键）：
- **预期值倒推**：读取被测函数的 `return` 语句 / 赋值语句，对比测试中的 `expected` 是否与之完全对应。若测试中的 expected 能从实现代码直接「抄」到，而非从函数名/注释/类型推断，则命中。
- **mock 万能**：统计测试中 `jest.mock`、`jest.spyOn`、`mockResolvedValue` 的数量，若一个测试文件内所有测试的断言都是 `toHaveBeenCalled` 系列，且没有验证最终返回值/副作用，则命中。
- **happy path 依赖**：检查测试输入是否全是「正常值」。若一个函数从未被 `null`、`undefined`、`0`、`''`、`[]` 等边界值调用过，标记为 happy path。
- **实现耦合**：检查断言是否涉及内部变量（如 `_internalState`、`_cache`）、私有方法（`obj._privateMethod()`）、或具体字符串匹配（如断言错误消息的具体文本而非错误类型）。
- **占位填充**：正则匹配 `expect\(true\)`、`expect\(1\)`、`// TODO`、`skip\(`、`xit\(`、`xtest\(`。

### 3. 测试精确率（Test Precision）

评估测试「真正发现 bug 的能力」。不是「测试了多少代码」，而是「如果代码里有 bug，测试有多大可能抓住它」。

检查维度：
- **边界值覆盖**：是否有 `null`、`undefined`、`0`、`''`、`[]`、`Number.MAX_SAFE_INTEGER`、负数等边界输入
- **异常路径覆盖**：是否有对 `throw`、`reject`、`catch`、`timeout` 分支的断言
- **行为契约验证**：测试是否只关心「给定输入 X，输出是否为 Y」，而不关心内部怎么实现
- **去 Mock 比例**：核心业务逻辑测试中，不依赖 mock 的断言占比。注意：不是「不用 mock」，而是「mock 了之后还验证了真实业务结果」

精确率公式：
```
Precision = (边界值测试数 × 25% + 异常路径测试数 × 25% + 行为契约测试数 × 30% + (1 - 纯mock占比) × 20%) 
            / max(1, 总测试数) 
            × 100
```
结果映射到 0-100 分。

---

## 执行流程

### Step 1：项目检测
- 检查 `jest.config.*`、`package.json` 中是否存在 jest 配置
- 确认测试文件符合命名规范：`*.test.{ts,tsx,js,jsx}` 或 `*.spec.{ts,tsx,js,jsx}`
- 若未找到 jest，告知用户本 skill 当前仅支持 Jest 项目

### Step 2：运行覆盖率
```bash
npx jest --coverage --coverageReporters=text-summary --collectCoverageFrom='src/**/*.{ts,tsx,js,jsx}'
```
- 收集 `Statements | Branches | Functions | Lines` 四列数据

### Step 3：扫描 AI 自欺欺人反模式
遍历所有测试文件，逐条检查每个 `it()` / `test()` 块：
1. 读取对应被测源代码
2. 对比测试中的 `expected` 是否能从实现代码中「抄」到 → 识别**预期值倒推**
3. 统计 mock 使用比例与断言类型 → 识别**mock 万能**
4. 检查输入值是否全是正常值 → 识别 **happy path 依赖**
5. 检查断言是否涉及内部变量/私有方法 → 识别**实现耦合**
6. 正则匹配占位符 → 识别**占位填充**

### Step 4：计算精确率
- 统计边界值、异常路径、行为契约、纯 mock 断言占比
- 按公式计算 Precision Score

### Step 5：生成评分报告

输出格式示例：
```
📊 测试质量评估报告

覆盖率：
  Statements: 85%  ✅
  Branches:   62%  🔴
  Functions:  90%  ✅
  Lines:      83%  ✅

自欺欺人程度：55% （越高越差，AI 确认偏误严重）
  🔴 [P0] 预期值倒推: 4 处
     src/calculator.test.ts:12  expected=15 与实现 return a*b 直接对应
     src/calculator.test.ts:18  expected=0   与实现 return 0 直接对应
     风险：若实现逻辑被悄悄改错，测试不会发现

  🟡 [P1] mock 万能: 3 处
     src/api.test.ts:22, 35, 48
     只验证 fetchMock 被调用，未验证返回数据是否被正确转换为用户对象

  🟡 [P1] happy path 依赖: 5 个函数从未被边界值测试
     src/validateEmail.ts, src/parseDate.ts, src/calculateDiscount.ts

  🟢 [P2] 实现耦合: 0 处 ✅
  🟢 [P2] 占位填充: 0 处 ✅

测试精确率：45/100
  - 边界值覆盖: 2/12 个函数
  - 异常路径:   1/12 个函数
  - 行为契约:   5/12 个函数
  - 纯 Mock:    40%

⚠️ 关键问题：
1. calculator.test.ts 的预期值明显是从实现代码抄的。正确做法：根据函数名和
   业务语义（如 calculateTotal(items) 应该「计算商品总价」）来设计测试用例，
   然后看实现是否满足，而不是看了实现再写 expected。
2. api.test.ts 全部使用 mock，但并未验证「API 返回的原始 JSON 被正确映射为
   业务对象」。应该在 mock 之后再加一步：验证映射后的对象字段是否正确。
```

### Step 6：给出改进指令

针对每个发现的问题，输出**可直接执行的修改指令**：

```
🔧 改进建议（按优先级排序）：

[P0] 消除预期值倒推 — src/calculator.test.ts
  当前问题：
    test('multiplies two numbers', () => {
      expect(multiply(3, 5)).toBe(15);  // 15 是从实现 return a*b 抄的
    });
  正确做法：
    - 不要看 multiply 的实现代码
    - 根据函数名 multiply 和参数 (3, 5)，业务上「3 乘以 5」应该是 15
    - 用另一个独立算出的值验证：expect(multiply(4, 7)).toBe(28)
    - 如果实现偷偷改成 return a+b，这个测试就会失败，从而发现 bug

[P0] 补全 mock 后的业务验证 — src/api.test.ts:22
  当前：
    expect(fetchMock).toHaveBeenCalledWith('/api/users');
  缺少：
    const result = await getUsers();
    expect(result[0]).toEqual({ id: 1, name: 'Alice', email: 'alice@example.com' });
    // 验证：原始 JSON 被正确映射为 User 对象，字段名没有拼写错误

[P1] 增加边界值测试 — src/validateEmail.test.ts
  当前只测了：validateEmail('test@example.com') → true
  需要补充：
    - validateEmail('') → false
    - validateEmail(null) → false
    - validateEmail('not-an-email') → false
    - validateEmail('a@b.c') → true  // 最短合法形式
```

---

## 核心原则（防自欺欺人）

**写测试时，AI 必须遵守以下心智模型：**

1. **先写预期，后看实现**
   - 读函数名、参数类型、JSDoc 注释、README 描述
   - 根据这些「接口契约」推断输入 X 应该得到输出 Y
   - 写进测试的 expected
   - **最后**才去看实现代码，运行测试验证
   - 如果实现和预期不符，**修改实现**，而不是修改预期来匹配实现

2. **Mock 是手段，不是目的**
   - Mock 掉外部依赖后，必须再验证一层：mock 返回的数据经过被测函数处理后，最终结果是否正确
   - 不能只停在了「mock 被调用了」

3. **Happy path 只是起点**
   - 每个被测函数至少要有 1 个 happy path + 1 个边界值 + 1 个异常路径
   - 边界值选择：类型的最小值、最大值、零值、空值、非法值

4. **测试是契约，不是实现说明书**
   - 断言只关心公开接口的输入输出
   - 不关心内部变量、不关心调用顺序、不关心用了哪个私有方法
   - 重构时应该能「随意改动实现，只要行为不变，测试就不挂」

---

## 禁止事项

- **禁止**看了实现代码后把 expected 改成和实现一样——这是自欺欺人最直接的表现
- **禁止**仅因覆盖率不足就要求「加测试凑数字」——必须先分析缺失覆盖的是否是关键逻辑
- **禁止**在报告中删除或修改被测源代码以「提升覆盖率」——只改测试
- **禁止**用「这个函数太简单了不需要边界测试」来为自己开脱——简单函数的边界 bug 同样致命

---

## Usage Examples

### Example 1

**User:** 帮我看看这个仓库的测试写得怎么样

**AI:** 我来运行测试质量评估。重点检查 AI 自欺欺人程度——即测试是不是为了「看起来通过」而根据实现代码倒推预期值写的，还是真正从业务需求出发验证行为。同时输出覆盖率和精确率评分。

### Example 2

**User:** 为什么覆盖率 90% 但还出 bug？

**AI:** 高覆盖率不代表测试有效，很可能是 AI 自欺欺人测试——测试的预期值是从实现代码抄的，mock 只验证了调用没验证结果，或者只测了 happy path。我会运行评估器量化这个问题，并定位具体的自欺欺人测试。

### Example 3

**User:** 把这些测试改得更可靠一点

**AI:** 先执行质量评估，识别哪些测试是「根据实现倒推预期」的自欺欺人测试，哪些缺少边界值和异常路径，然后按优先级给出可执行的修改方案。
