
# tx-tcc
对应书中第15章示例程序，使用hmily 2.1.1。RELEASE 实现基于TCC的最终一致性事务。

## 实现步骤

**1. 引入依赖：**
```
         <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>hmily-spring-boot-starter-dubbo</artifactId>
            <version>${hmily.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo</artifactId>
            <version>${dubbo.version}</version>
        </dependency>
```
Hmily官网文档：
必须使用 JDK8+
TCC模式必须要使用一款 RPC 框架, 比如 : Dubbo, SpringCloud,Montan

**2. 业务分析：**
[UserAccountBank01ServiceImpl.java](tx-tcc/tx-tcc-bank01/src/main/java/io/transaction/tcc/bank01/service/impl/UserAccountBank01ServiceImpl.java)
[UserAccountBank02ServiceImpl.java](tx-tcc/tx-tcc-bank02/src/main/java/io/transaction/tcc/bank02/service/impl/UserAccountBank02ServiceImpl.java)

### 服务1: `UserAccountBank01ServiceImpl` (转出银行微服务)

*   **业务内容:** 该服务负责发起一笔资金转账。它从Bank01的源账户中扣除金额，然后调用`UserAccountBank02Service`将金额存入Bank02的目标账户。

*   **TCC 步骤:**
    *   **Try 方法 (`transferAmount`):**
        *   记录Bank01 Try方法的执行日志。
        *   执行幂等性检查：如果`existsTryLog`返回true，表示Try已执行，则直接返回。
        *   执行“悬挂处理”检查：如果`existsConfirmLog`或`existsCancelLog`返回true，表示Confirm或Cancel已执行，则直接返回。这处理了全局事务协调器可能在Confirm/Cancel已处理后重试Try的情况。
        *   获取源账户和目标账户信息。
        *   检查源账户是否存在且余额充足。
        *   检查目标账户是否存在（通过调用`userAccountBank02Service.getUserAccountByAccountNo`）。
        *   为当前事务(`txNo`)保存`TryLog`。
        *   **预扣除**Bank01源账户的金额(`updateUserAccountBalanceBank01`)。这是“资源预留”步骤。
        *   调用`userAccountBank02Service.transferAmountToBank2(userAccountDto)`，启动Bank02的Try阶段。
    *   **Confirm 方法 (`confirmMethod`):**
        *   记录Bank01 Confirm方法的执行日志。
        *   执行幂等性检查：如果`existsConfirmLog`返回true，表示Confirm已执行，则直接返回。
        *   为当前事务保存`ConfirmLog`。
        *   **确认**预扣除的金额。在此特定代码中，`confirmUserAccountBalanceBank01`似乎是一个空操作或一个占位符，如果`updateUserAccountBalanceBank01`已经完成了实际扣除。
    *   **Cancel 方法 (`cancelMethod`): 当然，这部分是基于我之前的思考，我将继续完成。**
        *   记录Bank01 Cancel方法的执行日志。
        *   执行幂等性检查：如果`existsCancelLog`返回true，表示Cancel已执行，则直接返回。
        *   为当前事务保存`CancelLog`。
        *   **回滚**预扣除的金额(`cancelUserAccountBalanceBank01`)，将金额加回源账户。

### 服务2: `UserAccountBank02ServiceImpl` (转入银行微服务)

*   **业务内容:** 该服务负责接收来自Bank01的转账金额，并将其存入Bank02的目标账户。

*   **TCC 步骤:**
    *   **Try 方法 (`transferAmountToBank2`):**
        *   记录Bank02 Try方法的执行日志。
        *   执行幂等性检查：如果`existsTryLog`返回true，表示Try已执行，则直接返回。
        *   执行“悬挂处理”检查：如果`existsConfirmLog`或`existsCancelLog`返回true，表示Confirm或Cancel已执行，则直接返回。
        *   获取目标账户信息。
        *   检查目标账户是否存在。
        *   为当前事务保存`TryLog`。
        *   **预增加**Bank02目标账户的金额(`updateUserAccountBalanceBank02`)。这是“资源预留”步骤。
    *   **Confirm 方法 (`confirmMethod`):**
        *   记录Bank02 Confirm方法的执行日志。
        *   执行幂等性检查：如果`existsConfirmLog`返回true，表示Confirm已执行，则直接返回。
        *   为当前事务保存`ConfirmLog`。
        *   **确认**预增加的金额。在此特定代码中，`confirmUserAccountBalanceBank02`似乎是一个空操作或一个占位符，如果`updateUserAccountBalanceBank02`已经完成了实际增加。
    *   **Cancel 方法 (`cancelMethod`):**
        *   记录Bank02 Cancel方法的执行日志。
        *   执行幂等性检查：如果`existsCancelLog`返回true，表示Cancel已执行，则直接返回。
        *   为当前事务保存`CancelLog`。
        *   **回滚**预增加的金额(`cancelUserAccountBalanceBank02`)，将金额从目标账户中扣除。

**3. 事务协调器：**

事务协调器是 TCC 事务的核心组件，负责协调各个参与者（即各个服务节点）执行 Try、Confirm 和 Cancel 操作。

1.  **启动全局事务**：
    *   当一个被 `@HmilyTCC` 注解的方法被调用时，Hmily 框架会拦截该方法，并由事务协调器启动一个新的全局事务。
    *   事务协调器会生成一个全局唯一的事务 ID（`txNo`），用于标识整个分布式事务。

2.  **Try 阶段协调**：
    *   事务协调器通过 RPC 调用（例如 Dubbo、Spring Cloud Feign）各个参与者的 Try 方法。
    *   在发起 RPC 调用时，事务协调器会将全局事务 ID (`txNo`) 和当前阶段（Try）等信息作为事务上下文的一部分传递给参与者。
    *   每个参与者在收到 Try 请求后，执行相应的业务逻辑，并记录事务日志。

3.  **决策 Confirm 或 Cancel**：
    *   事务协调器会收集所有参与者 Try 阶段的执行结果。
    *   如果所有参与者的 Try 方法都成功执行，则事务协调器会发起 Confirm 阶段。
    *   如果任何一个参与者的 Try 方法执行失败，或者在 Try 阶段发生超时等异常，则事务协调器会发起 Cancel 阶段。

4.  **Confirm 阶段协调**：
    *   事务协调器通过 RPC 调用各个参与者的 Confirm 方法。
    *   同样，全局事务 ID (`txNo`) 和当前阶段（Confirm）等信息会作为事务上下文的一部分传递给参与者。
    *   每个参与者在收到 Confirm 请求后，执行相应的业务逻辑，并记录事务日志。
    *   Confirm 操作需要保证幂等性，以应对可能发生的重复调用。

5.  **Cancel 阶段协调**：
    *   事务协调器通过 RPC 调用各个参与者的 Cancel 方法。
    *   全局事务 ID (`txNo`) 和当前阶段（Cancel）等信息会作为事务上下文的一部分传递给参与者。
    *   每个参与者在收到 Cancel 请求后，执行相应的业务逻辑，释放 Try 阶段预留的资源，并记录事务日志。
    *   Cancel 操作同样需要保证幂等性，并且能够处理空回滚和悬挂问题。

6.  **事务状态维护和重试**：
    *   事务协调器会维护全局事务的状态，例如 Try 成功、Confirm 成功、Cancel 成功等。
    *   如果 Confirm 或 Cancel 操作失败，事务协调器会根据配置的重试策略进行重试，直到成功为止，以保证最终一致性。

**总结**：事务协调器通过拦截被 `@HmilyTCC` 注解的方法，生成全局事务 ID，并通过 RPC 调用和事务上下文传递，协调各个参与者执行 Try、Confirm 和 Cancel 操作。同时，事务协调器还负责维护事务状态、处理异常和进行重试，以保证分布式事务的最终一致性。


**4. 事务日志与幂等，空回滚，悬挂问题：**

Hmily 框架会自动管理事务日志的记录和读取，开发者通常不需要手动设计和调用事务日志。
Hmily 会在 Try、Confirm 和 Cancel 方法执行前后，自动记录相应的事务日志。
开发者只需要关注业务逻辑的实现，而无需关心事务日志的细节。

开发者需要做的事情： 
- 配置 Hmily 框架，包括选择事务日志的存储方式（例如数据库、Redis）和相关参数。
- 在 Try、Confirm 和 Cancel 方法中，实现幂等性、空回滚和悬挂处理的逻辑。
- 在某些特殊情况下，可能需要手动查询事务日志，例如进行事务状态的监控和分析。

一个通用的事务日志通常包含以下字段：
        *   `txNo`：全局事务 ID。
        *   `branchId`：分支事务 ID（可选，用于标识同一个全局事务中的不同参与者）。
        *   `stage`：事务阶段（Try、Confirm、Cancel）。
        *   `status`：事务状态（成功、失败、进行中）。
        *   `createTime`：创建时间。
        *   `updateTime`：更新时间。
        *   `payload`：业务数据（可选，用于记录 Try 阶段的业务数据，以便在 Cancel 阶段进行回滚）。

幂等性、空回滚和悬挂是 TCC 事务中需要特别关注的三个问题。

*   **幂等性 (Idempotency)**：
    *   **定义**：幂等性是指一个操作无论执行多少次，其结果都相同。
    *   **重要性**：在分布式系统中，由于网络异常或系统故障，Confirm 和 Cancel 操作可能会被重复调用。如果 Confirm 和 Cancel 操作不具备幂等性，可能会导致数据不一致。
    *   **解决方案**：
        *   **事务日志**：通过事务日志来判断一个操作是否已经执行过。如果日志已存在，则说明该操作已经执行过，直接返回，避免重复操作。
        *   **状态机**：维护一个状态机，记录业务数据的状态。只有在特定状态下才能执行相应的操作。
        *   **乐观锁**：使用乐观锁来控制对共享资源的并发访问。

*   **空回滚 (Null Compensation)**：
    *   **定义**：当协调器在没有调用参与者的 Try 方法的情况下，直接调用了参与者的 Cancel 方法时，就会发生空回滚。
    *   **原因**：通常发生在网络抖动或超时导致 Try 请求未到达参与者，但协调器认为 Try 失败并触发回滚。
    *   **解决方案**：
        *   **Try 阶段记录日志**：在 Try 阶段记录事务日志。
        *   **Cancel 阶段检查 Try 日志**：在 Cancel 方法执行前，检查对应的 Try 日志是否存在。如果 Try 日志不存在，说明 Try 方法没有成功执行过，那么 Cancel 方法就应该直接返回，不做任何业务回滚操作。

*   **悬挂 (Pendent)**：
    *   **定义**：当 Cancel 方法比 Try 方法先执行时，就会发生悬挂。
    *   **原因**：通常发生在 Try 请求因为网络延迟等原因迟迟未到达参与者，而协调器已经超时并触发了 Cancel 操作，随后 Try 请求又到达了参与者并执行成功。
    *   **解决方案**：
        *   **Try 阶段检查 Confirm/Cancel 日志**：在 Try 阶段检查 Confirm 或 Cancel 日志是否存在。
        *   **如果日志存在，直接返回**：如果在 Try 阶段发现 Confirm 或 Cancel 日志已经存在，说明该事务已经被协调器处理过（无论是提交还是回滚），那么 Try 方法就不应该再执行业务逻辑，直接返回。










