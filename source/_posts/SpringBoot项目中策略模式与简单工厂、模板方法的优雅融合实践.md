---
title: SpringBoot项目中策略模式与简单工厂、模板方法的优雅融合实践
tags:
  - Java
  - SpringBoot
  - Shiro
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261140143.png'
toc: true
abbrlink: af5fe467
date: 2022-01-08 17:25:08
---

在面向对象设计中，策略模式是一种常用且强大的设计模式，帮助我们将算法的实现与使用它的客户端解耦。当业务场景中存在多个可替换的行为实现，并且这些行为经常变动或者扩展时，策略模式能带来极大的灵活性。同时，结合简单工厂模式可以简化策略对象的创建过程，模板方法模式则为算法骨架提供统一定义，鼓励子类实现具体细节。

本文围绕一个真实的“数据导出”需求展开，介绍如何将策略模式、简单工厂模式和模板方法模式结合应用于 Spring Boot 项目中，形成优雅且易维护的解决方案。

<!-- more -->

> 源码地址：[gitee（springboot-example）](https://gitee.com/liboshuai01/springboot-example.git) 的 `strategy` 分支

## 需求背景

假设我们正在开发一个支持多家银行数据导入导出的应用模块。不同银行对数据处理逻辑有不同的业务要求，且业务需求可能随着时间推移不断新增银行或调整导入导出的细节。为应对以上变化，设计一个灵活且具备良好扩展性的解决方案变得尤为重要。

主要挑战包括：

- 根据银行编码获取对应策略，实现定制化处理；
- 保证新增银行时无需大幅修改已有代码，实现开闭原则；
- 提供基础默认逻辑，防止处理器未实现时程序崩溃。

## 核心设计思路

- **策略模式**：通过定义统一的抽象策略接口（或抽象类），所有银行的具体逻辑都作为具体策略实现，实现行为的动态切换。
- **简单工厂模式**：创建一个策略持有者（BankStrategyHolder），根据银行编码动态选择对应策略实例，通过注解读取元数据，自动导入策略映射。
- **模板方法模式**：抽象类 `BankStrategy` 提供基础方法签名和默认实现，子类覆盖需要的部分，实现定制逻辑，同时不影响整体框架结构。

这种设计既满足灵活扩展的需求，又保证代码结构清晰，维护简便。

## 关键代码解析

### 常量定义

首先定义了银行编码与默认标识常量，方便后续统一调用和维护。

```java
public class Constants {
    /** 默认策略标识 */
    public static final String DEFAULT_STRATEGY_NAME = "default";
}

public class BankConstants {
    public static final String FIRST_BANK = "0001";
    public static final String SECOND_BANK = "0002";
    public static final String THIRD_BANK = "0003";
    public static final String FOURTH_BANK = "0004";
}
```

### 自定义注解

使用注解 `@StrategyIdentifier` 标记策略实现对应的银行编码，便于工厂自动扫描和映射。

```java
@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface StrategyIdentifier {
    String[] value();
}
```

### 抽象策略模板

利用抽象类 `BankStrategy` 充当模板方法模式角色，定义导入导出默认行为和骨架。默认实现为抛出不支持异常，提醒子类需自行实现。

```java
public abstract class BankStrategy {

    /**
     * 导入数据，默认不支持。
     */
    public void importData() {
        throw new UnsupportedOperationException("此银行不支持导入数据功能！");
    }

    /**
     * 导出数据，默认不支持。
     */
    public void exportData() {
        throw new UnsupportedOperationException("此银行不支持导出数据功能！");
    }
}
```

### 具体策略实现

各个银行通过继承 `BankStrategy`，结合 `@StrategyIdentifier` 注解，完成不同银行的导入导出逻辑。

```java
@Slf4j
@Component
@StrategyIdentifier({Constants.DEFAULT_STRATEGY_NAME})
public class DefaultBankStrategy extends BankStrategy {
    @Override
    public void importData() {
        log.info("默认银行数据导入逻辑");
    }

    @Override
    public void exportData() {
        log.info("默认银行数据导出逻辑");
    }
}

@Slf4j
@Component
@StrategyIdentifier({BankConstants.FIRST_BANK})
public class FirstBankStrategy extends BankStrategy{
    @Override
    public void importData() {
        log.info("第一银行数据导入逻辑");
    }

    @Override
    public void exportData() {
        log.info("第一银行数据导出逻辑");
    }
}

@Slf4j
@Component
@StrategyIdentifier({BankConstants.SECOND_BANK})
public class SecondBankStrategy extends BankStrategy{
    @Override
    public void importData() {
        log.info("第二银行数据导入逻辑");
    }

    @Override
    public void exportData() {
        log.info("第二银行数据导出逻辑");
    }
}

@Slf4j
@Component
@StrategyIdentifier({BankConstants.THIRD_BANK, BankConstants.FOURTH_BANK})
public class ThirdBankStrategy extends BankStrategy{
    // 不重写 importData，默认不支持
    @Override
    public void exportData() {
        log.info("第三、四银行数据导出逻辑");
    }
}
```

### 策略持有者（简单工厂）

`BankStrategyHolder` 扮演策略工厂角色，Spring 自动注入所有策略实现后，通过注解读取银行编码，实现银行编号与策略实例的映射管理。

```java
@Component
public class BankStrategyHolder {

    private final Map<String, BankStrategy> strategyMap = new HashMap<>();

    @Autowired
    public BankStrategyHolder(List<BankStrategy> bankStrategyList) {
        bankStrategyList.forEach(strategy -> {
            StrategyIdentifier annotation = strategy.getClass().getAnnotation(StrategyIdentifier.class);
            if (annotation != null) {
                for (String bankCode : annotation.value()) {
                    strategyMap.put(bankCode, strategy);
                }
            }
        });
    }

    /**
     * 根据银行编码获取对应策略，若不存在则返回默认策略。
     */
    public BankStrategy getByBankCode(String bankCode) {
        return strategyMap.getOrDefault(bankCode, strategyMap.get(Constants.DEFAULT_STRATEGY_NAME));
    }
}
```

### 外部调用控制器

`StrategyController` 负责处理内外部请求，通过银行编码获取对应策略，并调用导入导出方法。

```java
@Slf4j
@RestController
@Tag(name = "策略模式示例")
public class StrategyController {

    @Resource
    private BankStrategyHolder bankStrategyHolder;

    @GetMapping("/importData")
    @Operation(summary = "导入数据")
    public void importData(@RequestParam String bankCode) {
        log.info("调用 [导入数据] - 银行编码: {}", bankCode);
        bankStrategyHolder.getByBankCode(bankCode).importData();
    }

    @GetMapping("/exportData")
    @Operation(summary = "导出数据")
    public void exportData(@RequestParam String bankCode) {
        log.info("调用 [导出数据] - 银行编码: {}", bankCode);
        bankStrategyHolder.getByBankCode(bankCode).exportData();
    }
}
```

## 效果验证

向接口 `/importData` 传递不同银行代码，观察日志输出：

- 传入 `0001`：执行第一银行导入逻辑
- 传入 `0002`：执行第二银行导入逻辑
- 传入 `0003` 或 `0004`：抛出异常，提示导入功能不支持
- 传入其他（如 `0005`）：执行默认导入逻辑

同理，访问 `/exportData` 接口：

- `0001`、`0002` 分别输出各自的导出逻辑
- `0003`、`0004` 共用第三银行策略的导出逻辑
- 未匹配银行代码执行默认导出逻辑

该设计保证了行为的分离与可扩展性，新增银行只需新增对应的策略实现并添加注解，无需更改已有核心代码。

## 思考与总结

通过该案例实现，我们体验了设计模式协作带来的强大优势：

- **策略模式** 提供了灵活替换算法的能力，隔离了银行导入导出差异；
- **简单工厂** 自动管理策略类的创建和注入，简化客户端代码，减轻了耦合；
- **模板方法模式** 保障公共方法逻辑的一致性，实现默认行为，增强可靠性。

这种设计不仅符合面向对象设计原则，更加突出了代码的可维护性和拓展性。未来随着业务需求增长，新增银行业务逻辑只需编写新的策略实现，系统依然保持干净整洁。

最后，推荐读者尝试利用 Spring 的依赖注入以及自定义注解，进一步实现自动扫描和策略注册机制，打造属于自己项目的灵活策略框架。

## 参考链接

- [策略模式 – [百度百科]](https://baike.baidu.com/item/策略模式)
- [模板方法模式 – [Wikipedia]](https://en.wikipedia.org/wiki/Template_method_pattern)
- [简单工厂模式 – [菜鸟教程]](https://www.runoob.com/design-pattern/simple-factory-pattern.html)
- [SpringBoot项目源码 - strategy分支](https://gitee.com/liboshuai01/springboot-example.git)

---

希望本文对您理解并实践设计模式在Spring项目中的结合有所帮助，欢迎大家留言讨论。