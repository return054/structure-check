# Structure Check

**别再相信 AI 说「做完了」。** Structure Check 把「功能是否实现」从模型的自我报告，变成**退出码裁决的机械核对**。

AI 编码代理最常见的失败不是写不出代码，而是**虚报完成**——声称实现了某个函数，结果函数是空壳；声称接好了模块，结果导出了却没人调用；声称完成了需求，结果要求的符号根本不存在。靠提示词要求模型「自觉验证」必然出现验证真空。

Structure Check 的思路很简单：让 agent 从任务要求生成一份**要求清单**，工具逐条做机械结构核对，**退出码是唯一裁决**——模型只许逐字转述输出块，无权改判。

```
=== STRUCTURE CHECK ===
checklist: checklist.json
requirements: 4
task: 购物车模块
summary: passed 3, failed 1
[1/4] PASS exists Cart @ src/cart.js —— 已找到定义（class）
[2/4] PASS exported Cart @ src/cart.js —— 已导出
[3/4] FAIL called addItem @ src/index.js —— 未发现调用点（仅声明/导出不算调用）
[4/4] PASS notStub computeTotal @ src/cart.js —— 函数体非空（87 字符）
result: FAIL
=== END STRUCTURE CHECK ===
```

## 特性

- **零依赖**：单个 Node 脚本，Node.js ≥ 18 即可运行
- **退出码裁决**：0 全过 / 1 有未实现 / 2 清单非法——结果可被脚本、CI、上层 agent 无歧义消费
- **注释免疫**：核对前先剥离注释，注释里写的代码不算数
- **四类检查**覆盖「虚报完成」的主要形态（见下表）
- **JS/TS 全支持**；Python 支持 `exists` / `notStub`

## 检查类型

| type | PASS 条件 | 典型 FAIL 场景 |
|---|---|---|
| `exists` | 符号在本文件有定义（function/class/const/interface/type/enum/方法简写/def） | 要求的函数/类根本没写 |
| `exported` | 符号已导出（export / export default / module.exports / exports.x） | 定义了但没接上模块出口 |
| `called` | 符号在本文件存在调用点（排除定义行；`new X()` 算调用） | 导出了但没有任何地方使用（假接线） |
| `notStub` | 函数体非空壳 | 空体、裸 `return`、TODO/未实现抛错、Python 的 `pass`/`...` 冒充实现 |

## 快速开始

```bash
node structure-check.mjs checklist.json
echo $LASTEXITCODE   # 0=全过 1=有未实现 2=清单非法
```

清单格式：

```json
{
  "task": "实现购物车模块",
  "requirements": [
    { "desc": "Cart 类已定义",        "type": "exists",   "file": "src/cart.js",  "symbol": "Cart" },
    { "desc": "Cart 已导出",          "type": "exported", "file": "src/cart.js",  "symbol": "Cart" },
    { "desc": "addItem 在入口被调用",  "type": "called",   "file": "src/index.js", "symbol": "addItem" },
    { "desc": "computeTotal 非空壳",   "type": "notStub",  "file": "src/cart.js",  "symbol": "computeTotal" }
  ]
}
```

`file` 相对执行时的工作目录解析；文件不存在按该条 FAIL，不豁免。

## 实战记录

在真实项目上验证过：4 项已交付内容全部核实 PASS；故意混入两个未实现项——`applyCoupon` 在模块内无任何调用点（假接线）、不存在的 `ghostFunction`——均被精确抓到并 exit 1。夹具覆盖 22 个场景（四种类型 × PASS/FAIL、方法简写、箭头函数、Python 空壳、文件缺失、8 种坏清单边界）。

## 守卫家族

| 工具 | 管什么 |
|---|---|
| ai-code-verification-guard | 改完后的行为回归 |
| tdd-guard | 开发顺序（红绿重构循环） |
| **structure-check** | **结构存在性：要求的东西在不在、接没接上** |

三者互补，可同时启用。

## License

[MIT](LICENSE)
