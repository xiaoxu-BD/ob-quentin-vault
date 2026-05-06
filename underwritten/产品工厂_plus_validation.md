# 保险承保流程设计 — 基于 NFTurbo 架构

## 一、整体流程映射

```
NFT流程                          保险承保流程
─────────────────────────────────────────────────────────
/book 预定          ──→         /quote 报价
下单校验链(Validator Chain) ──→  核保(Underwriting)
订单确认(CONFIRM)    ──→         出单(Policy Issuance)
/pay 创建支付单      ──→         /pay 收费(Premium Collection)
paySuccess回调       ──→         生效(Policy Effective)
```

## 二、状态机设计

### NFT 原始状态机

```
CREATE → CONFIRM → PAID → FINISH
```

### 保险状态机（扩展）

```
QUOTED → UNDERWRITING → CONFIRMED → PAID → EFFECTIVE → FINISH
  │          │              │          │
  ↓          ↓              ↓          ↓
CLOSED    REJECTED      CLOSED     (refund)
```

### 核心枚举

```java
// PolicyOrderState.java
public enum PolicyOrderState {
    QUOTED,         // 已报价 — 对应 NFT 的 CREATE
    UNDERWRITING,   // 核保中 — 保险特有
    CONFIRMED,      // 已出单 — 对应 NFT 的 CONFIRM
    PAID,           // 已收费 — 同 NFT
    EFFECTIVE,      // 已生效 — 保险特有（支付后可能有等待期）
    FINISH,         // 已完成
    CLOSED,         // 已关闭
    DISCARD;        // 废单
}

// PolicyOrderEvent.java
public enum PolicyOrderEvent {
    QUOTE,              // 报价
    SUBMIT_UNDERWRITE,  // 提交核保
    UNDERWRITE_PASS,    // 核保通过
    UNDERWRITE_REJECT,  // 核保拒绝
    PAY,                // 支付
    EFFECTIVE,          // 生效
    CANCEL,             // 取消
    TIME_OUT,           // 超时
    FINISH,             // 完成
    DISCARD;            // 废弃
}
```

### 状态机转换表

直接复用 `BaseStateMachine` 模式：

```java
public class PolicyOrderStateMachine extends BaseStateMachine<PolicyOrderState, PolicyOrderEvent> {

    public static final PolicyOrderStateMachine INSTANCE = new PolicyOrderStateMachine();

    {
        // 报价 → 提交核保
        putTransition(QUOTED, SUBMIT_UNDERWRITE, UNDERWRITING);

        // 核保通过 → 出单
        putTransition(UNDERWRITING, UNDERWRITE_PASS, CONFIRMED);

        // 核保拒绝 → 关闭
        putTransition(UNDERWRITING, UNDERWRITE_REJECT, CLOSED);

        // 出单 → 支付
        putTransition(CONFIRMED, PAY, PAID);

        // 支付 → 生效（即时生效产品直接走这条）
        putTransition(PAID, EFFECTIVE, EFFECTIVE);

        // 生效 → 完成（如保障期满）
        putTransition(EFFECTIVE, FINISH, FINISH);

        // 超时/取消关闭
        putTransition(QUOTED, CANCEL, CLOSED);
        putTransition(QUOTED, TIME_OUT, CLOSED);
        putTransition(UNDERWRITING, CANCEL, CLOSED);
        putTransition(CONFIRMED, TIME_OUT, CLOSED);
        putTransition(CONFIRMED, CANCEL, CLOSED);

        // 废单
        putTransition(QUOTED, DISCARD, DISCARD);
        putTransition(UNDERWRITING, DISCARD, DISCARD);

        // 幂等：已支付后再支付，状态不变
        putTransition(PAID, PAY, PAID);
    }
}
```

## 三、状态流转全景

```
                     ┌───────┐
        /quote       │       │ 有效期到/取消
   ──────────────→   │ QUOTED │ ──────────→ CLOSED
                     │       │
                     └───┬───┘
                         │ submitUnderwrite
                         ↓
                     ┌───────────────┐
                     │               │ 拒保
     异步核保通过      │  UNDERWRITING │ ──────→ CLOSED
   ──────────────→   │               │
                     └───────┬───────┘
                             │ 自动出单
                             ↓
                     ┌───────────────┐
                     │               │ 超时/取消
                     │   CONFIRMED   │ ──────→ CLOSED
                     │               │
                     └───────┬───────┘
                             │ /pay
                             ↓
                     ┌───────────┐
                     │           │
                     │   PAID    │
                     │           │
                     └─────┬─────┘
                           │ paySuccess callback
                           ↓          即时生效产品：直接生效
                     ┌───────────┐   等待期产品：定时任务推进
                     │           │
                     │ EFFECTIVE │
                     │           │
                     └─────┬─────┘
                           │ 保障期满
                           ↓
                     ┌───────────┐
                     │  FINISH   │
                     └───────────┘
```

## 四、领域实体

```java
@Setter
@Getter
public class PolicyOrder extends BaseEntity {

    public static final int DEFAULT_UNDERWRITE_TIMEOUT_HOURS = 24;
    public static final int DEFAULT_PAY_TIMEOUT_MINUTES = 30;

    private String orderId;

    // ── 投保人 ──
    private String applicantId;       // 投保人ID（= NFT 的 buyerId）
    private UserType applicantType;
    private String reverseApplicantId;

    // ── 被保人 ──
    private String insuredId;
    private String insuredName;
    private String insuredIdCard;     // 身份证号（核保关键字段）

    // ── 产品 ──
    private String productId;         // 险种ID（= NFT 的 goodsId）
    private String productName;
    private GoodsType productType;    // 可复用，或新增 INSURANCE 枚举

    // ── 金额 ──
    private BigDecimal premium;           // 保费（= NFT 的 orderAmount）
    private BigDecimal sumInsured;        // 保额（保险特有）
    private BigDecimal paidAmount;

    // ── 保单信息 ──
    private String policyNo;              // 保单号（出单后生成）
    private PolicyOrderState orderState;
    private String payChannel;
    private String payStreamId;
    private String closeType;

    // ── 时间 ──
    private Date quoteTime;               // 报价时间
    private Date underwritingTime;        // 核保时间
    private Date confirmTime;             // 出单时间
    private Date paySucceedTime;
    private Date effectiveTime;           // 生效时间
    private Date expireTime;              // 失效时间
    private Date orderClosedTime;

    // ── 核保相关 ──
    private String underwriteResult;      // PASS / REJECT / MANUAL
    private String underwriteRemark;      // 核保备注（拒保原因等）
    private Integer snapshotVersion;      // 产品快照版本（价格变更校验）

    // ── 保障期间 ──
    private Date coverageStartDate;       // 保障开始日
    private Date coverageEndDate;         // 保障结束日
}
```

## 五、完整接口流程

### Step 1: 报价 `/quote` — 对应 NFT 的 `/book`

```java
@PostMapping("/quote")
public Result<String> quote(@Valid @RequestBody QuoteParam quoteParam) {
    String userId = (String) StpUtil.getLoginId();

    // 1. 获取产品信息，计算保费
    BaseGoodsVO product = goodsFacadeService.getGoods(
        quoteParam.getProductId(), GoodsType.INSURANCE);
    if (product == null || !product.available()) {
        throw new TradeException(TradeErrorCode.GOODS_NOT_FOR_SALE);
    }

    // 2. 保费计算 — NFT 价格固定，保险保费需要动态计算
    BigDecimal premium = premiumCalculator.calculate(
        product.getPrice(),           // 基础费率
        quoteParam.getInsuredAge(),   // 被保人年龄
        quoteParam.getInsuredGender(),// 性别
        quoteParam.getSumInsured(),   // 保额
        quoteParam.getCoverageYears() // 保障年限
    );

    // 3. 生成报价单
    String orderId = DistributeID.generateWithSnowflake(
        BusinessCode.POLICY_ORDER, WorkerIdHolder.WORKER_ID, userId);

    PolicyOrder policyOrder = new PolicyOrder();
    policyOrder.setOrderId(orderId);
    policyOrder.setApplicantId(userId);
    policyOrder.setProductId(quoteParam.getProductId());
    policyOrder.setProductName(product.getGoodsName());
    policyOrder.setPremium(premium);
    policyOrder.setSumInsured(quoteParam.getSumInsured());
    policyOrder.setInsuredName(quoteParam.getInsuredName());
    policyOrder.setInsuredIdCard(quoteParam.getInsuredIdCard());
    policyOrder.setOrderState(PolicyOrderState.QUOTED);
    policyOrder.setQuoteTime(new Date());
    policyOrder.setExpireTime(DateUtils.addDays(new Date(), 7));

    policyOrderService.save(policyOrder);

    return Result.success(orderId);
}
```

### Step 2: 核保 `/submitUnderwrite` — 保险独有

```java
@PostMapping("/submitUnderwrite")
public Result<Boolean> submitUnderwrite(@Valid @RequestBody UnderwriteParam param) {
    String userId = (String) StpUtil.getLoginId();

    // 1. 查报价单
    PolicyOrder policyOrder = policyOrderService.getByOrderId(param.getOrderId());
    if (policyOrder == null) {
        throw new TradeException(TradeErrorCode.GOODS_NOT_EXIST);
    }

    // 2. 报价单校验（= NFT 的 Validator Chain）
    underwriteValidatorChain.validate(policyOrder);

    // 3. 推进状态：QUOTED → UNDERWRITING
    policyOrderService.submitUnderwrite(policyOrder);

    // 4. 异步执行核保规则引擎
    applicationContext.publishEvent(new UnderwriteEvent(policyOrder));

    return Result.success(true);
}
```

#### 核保 Validator Chain（复用 `BaseOrderCreateValidator` 模式）

```java
// ── 1. 投保人校验 ──
@Component
public class ApplicantValidator extends BasePolicyOrderValidator {
    @Override
    protected void doValidate(PolicyOrder order) {
        UserQueryResponse user = userFacadeService.query(
            new UserQueryRequest(Long.valueOf(order.getApplicantId())));
        if (!user.getSuccess() || user.getData().getState() != UserState.ACTIVE) {
            throw new OrderException(APPLICANT_INVALID);
        }
    }
}

// ── 2. 产品状态校验 ──
@Component
public class ProductValidator extends BasePolicyOrderValidator {
    @Override
    protected void doValidate(PolicyOrder order) {
        BaseGoodsVO product = goodsFacadeService.getGoods(
            order.getProductId(), GoodsType.INSURANCE);
        if (product == null || !product.available()) {
            throw new OrderException(PRODUCT_NOT_AVAILABLE);
        }
        // 价格快照校验（报价后产品费率可能调整）
        if (!product.getVersion().equals(order.getSnapshotVersion())) {
            throw new OrderException(PRODUCT_PRICE_CHANGED);
        }
    }
}

// ── 3. 报价有效期校验 ──
@Component
public class QuoteExpireValidator extends BasePolicyOrderValidator {
    @Override
    protected void doValidate(PolicyOrder order) {
        if (order.getExpireTime().before(new Date())) {
            throw new OrderException(QUOTE_EXPIRED);
        }
    }
}

// ── 4. 重复投保校验 ──
@Component
public class DuplicatePolicyValidator extends BasePolicyOrderValidator {
    @Override
    protected void doValidate(PolicyOrder order) {
        PolicyOrder existing = policyOrderMapper.selectEffectivePolicy(
            order.getInsuredIdCard(), order.getProductId());
        if (existing != null) {
            throw new OrderException(DUPLICATE_POLICY);
        }
    }
}

// ── 核保链配置 ──
@Configuration
public class UnderwriteValidatorConfig {
    @Bean
    public OrderCreateValidator underwriteValidatorChain(
            ApplicantValidator applicant,
            ProductValidator product,
            QuoteExpireValidator expire,
            DuplicatePolicyValidator duplicate) {
        applicant.setNext(product);
        product.setNext(expire);
        expire.setNext(duplicate);
        return applicant;
    }
}
```

#### 异步核保处理（核保规则引擎）

```java
@Component
public class UnderwriteEventListener {

    @Async
    @EventListener
    public void handleUnderwrite(UnderwriteEvent event) {
        PolicyOrder order = event.getPolicyOrder();

        // 核保规则 1: 年龄限制
        if (order.getInsuredAge() < 18 || order.getInsuredAge() > 65) {
            rejectOrder(order, "被保人年龄不符合投保要求");
            return;
        }

        // 核保规则 2: 保额上限
        BigDecimal maxSumInsured = productConfig.getMaxSumInsured(order.getProductId());
        if (order.getSumInsured().compareTo(maxSumInsured) > 0) {
            rejectOrder(order, "保额超过产品上限");
            return;
        }

        // 核保规则 3: 职业类别（高风险职业拒保）
        if (highRiskOccupationService.isHighRisk(order.getApplicantOccupation())) {
            rejectOrder(order, "被保人职业类别不符合要求");
            return;
        }

        // 核保规则 4: 健康告知（可对接外部风控）
        // ... 可对接第三方健康问卷或风控系统

        // 全部通过 → 核保通过
        passUnderwrite(order);
    }

    private void passUnderwrite(PolicyOrder order) {
        UnderwritePassRequest request = new UnderwritePassRequest();
        request.setOrderId(order.getOrderId());
        request.setOperateTime(new Date());
        request.setOperatorType(UserType.PLATFORM);
        policyOrderService.underwritePass(request);
    }

    private void rejectOrder(PolicyOrder order, String reason) {
        UnderwriteRejectRequest request = new UnderwriteRejectRequest();
        request.setOrderId(order.getOrderId());
        request.setRemark(reason);
        request.setOperateTime(new Date());
        request.setOperatorType(UserType.PLATFORM);
        policyOrderService.underwriteReject(request);
    }
}
```

### Step 3: 出单 — 核保通过后自动触发

```java
// 在 PolicyOrderManageService 中
public OrderResponse underwritePass(UnderwritePassRequest request) {
    // 状态推进: UNDERWRITING → CONFIRMED
    return doExecute(request, policyOrder -> {
        policyOrder.setUnderwriteResult("PASS");
        policyOrder.setUnderwriteTime(request.getOperateTime());

        // 生成保单号
        String policyNo = policyNoGenerator.generate(policyOrder);
        policyOrder.setPolicyNo(policyNo);

        // 设置保障期间
        policyOrder.setCoverageStartDate(request.getOperateTime());
        policyOrder.setCoverageEndDate(
            DateUtils.addYears(request.getOperateTime(), policyOrder.getCoverageYears()));

        // 状态转换
        PolicyOrderState newState = PolicyOrderStateMachine.INSTANCE
            .transition(policyOrder.getOrderState(), PolicyOrderEvent.UNDERWRITE_PASS);
        policyOrder.setOrderState(newState);
        policyOrder.setConfirmTime(request.getOperateTime());
    });
}
```

### Step 4: 收费 `/pay` — 几乎完全复用 NFT 支付流程

```java
@PostMapping("/pay")
public Result<PayOrderVO> pay(@Valid @RequestBody PayParam payParam) {
    String userId = (String) StpUtil.getLoginId();
    PolicyOrder policyOrder = policyOrderService.getByOrderId(payParam.getOrderId());

    if (policyOrder == null) {
        throw new TradeException(TradeErrorCode.GOODS_NOT_EXIST);
    }
    if (policyOrder.getOrderState() != PolicyOrderState.CONFIRMED) {
        throw new TradeException(TradeErrorCode.ORDER_IS_CANNOT_PAY);
    }
    if (isTimeout(policyOrder)) {
        doAsyncTimeoutOrder(policyOrder);
        throw new TradeException(TradeErrorCode.ORDER_IS_CANNOT_PAY);
    }

    // 创建支付单 — 直接复用 PayFacadeService
    PayCreateRequest payCreateRequest = new PayCreateRequest();
    payCreateRequest.setOrderAmount(policyOrder.getPremium());
    payCreateRequest.setBizNo(policyOrder.getOrderId());
    payCreateRequest.setBizType(BizOrderType.POLICY_ORDER);
    payCreateRequest.setMemo("保费：" + policyOrder.getProductName());
    payCreateRequest.setPayChannel(payParam.getPayChannel());
    payCreateRequest.setPayerId(policyOrder.getApplicantId());
    payCreateRequest.setPayerType(policyOrder.getApplicantType());

    PayCreateResponse payCreateResponse = RemoteCallWrapper.call(
        req -> payFacadeService.generatePayUrl(req), payCreateRequest, "generatePayUrl");

    if (payCreateResponse.getSuccess()) {
        PayOrderVO payOrderVO = new PayOrderVO();
        payOrderVO.setPayOrderId(payCreateResponse.getPayOrderId());
        payOrderVO.setPayUrl(payCreateResponse.getPayUrl());
        return Result.success(payOrderVO);
    }
    throw new TradeException(TradeErrorCode.PAY_CREATE_FAILED);
}
```

### Step 5: 生效 — `paySuccess` 回调

在 `PayApplicationService.paySuccess()` 中增加 `INSURANCE` 分支：

```java
@GlobalTransactional(rollbackFor = Exception.class)
public boolean paySuccess(PaySuccessEvent paySuccessEvent) {
    PayOrder payOrder = payOrderService.queryByOrderId(paySuccessEvent.getPayOrderId());
    if (payOrder.isPaid()) {
        return true;
    }

    OrderPayRequest orderPayRequest = getOrderPayRequest(paySuccessEvent, payOrder);
    OrderResponse orderResponse = orderFacadeService.paySuccess(orderPayRequest);

    // 退款逻辑（和 NFT 一致）
    if (needChargeBack(orderResponse)) {
        payOrderService.paySuccess(paySuccessEvent);
        doChargeBack(paySuccessEvent);
        return true;
    }

    // 根据业务类型分支处理
    TradeOrderVO orderVO = getTradeOrder(payOrder.getBizNo());
    switch (orderVO.getGoodsType()) {
        case COLLECTION:
            handleCollectionPaySuccess(orderVO);
            break;

        case INSURANCE:
            handleInsurancePaySuccess(orderVO);
            break;
    }

    payOrderService.paySuccess(paySuccessEvent);
    return true;
}

private void handleInsurancePaySuccess(TradeOrderVO orderVO) {
    InsurancePolicyRequest request = new InsurancePolicyRequest();
    request.setOrderId(orderVO.getOrderId());
    request.setPaySucceedTime(new Date());

    if (isImmediateEffective(orderVO.getProductId())) {
        // 即时生效：PAID → EFFECTIVE
        policyOrderFacadeService.effective(request);
    } else {
        // 有等待期：定时任务等待期后推进
        policyEffectiveScheduler.schedule(orderVO.getOrderId(),
            getWaitingPeriodDays(orderVO.getProductId()));
    }
}
```

## 六、代码复用对照表

| 组件 | NFT 原有 | 保险复用方式 |
|---|---|---|
| `BaseStateMachine` | 状态机基类 | 直接复用，新建 `PolicyOrderStateMachine` |
| `BaseOrderCreateValidator` | 校验链基类 | 直接复用，新建 `BasePolicyOrderValidator` |
| `PayFacadeService` | 支付创建 | 完全复用，新增 `BizOrderType.POLICY_ORDER` |
| `PayApplicationService` | 支付回调 | 扩展 `switch` 分支，新增 `INSURANCE` case |
| `OrderManageService.doExecute()` | 幂等+事务+流水库 | 抽取为通用基类，`PolicyOrderManageService` 复用 |
| `DistributeID` | 分布式ID | 直接复用，新增 `BusinessCode.POLICY_ORDER` |
| `RemoteCallWrapper` | 远程调用包装 | 直接复用 |
| `@DistributeLock` | 分布式锁 | 直接复用 |
| `@GlobalTransactional` | Seata分布式事务 | 直接复用 |

## 七、复杂点分析

### 1. 核保是异步+可能人工介入的

NFT 的 Validator Chain 是同步的、纯自动的。保险核保不同——自动规则引擎可能判定为"转人工"，这时需要一个中间状态 `UNDERWRITING` 等待人工操作。状态机需要支持 `UNDERWRITING → CONFIRMED`（人工通过）和 `UNDERWRITING → CLOSED`（人工拒绝），不只是自动 PASS/REJECT。需要一个后台管理界面处理待核保队列。

### 2. 生效不等于保障开始

NFT 的 `PAID → FINISH` 很直接。保险有等待期（如医疗险 30 天、重疾险 90 天）。`paySuccess` 之后订单进入 `PAID`，但 `PAID → EFFECTIVE` 的转换由定时任务控制。需要一个 `PolicyEffectiveScheduler` 定时扫描已过等待期的保单，推进状态。

### 3. 报价单有有效期且价格可变

NFT 预定没有"过期"概念（虽然订单有 30 分钟超时）。保险报价单的有效期可能长达 7 天，期间产品费率可能调整。需要 `snapshotVersion` 做价格快照校验——报价时锁定费率版本，出单时校验版本是否变更。如果变了，需要重新报价。

### 4. 保费计算 ≠ 商品定价

NFT 商品价格是固定的，从商品信息直接读取。保险保费是动态计算的——基于年龄、性别、保额、保障年限、职业等多因素。`PremiumCalculator` 需要是一个策略模式，不同险种有不同的费率表和计算公式。

### 5. 重复投保校验

NFT 同一用户同一藏品只能买一次（通过 identifier 幂等）。保险更复杂——同一被保人同一险种不能重复投保（不是同一投保人，可能是帮家人投保），校验维度从 `buyerId` 变成了 `insuredIdCard + productId`。

### 6. 逆向流程复杂度不同

NFT 退款一笔搞定。保险退保有犹豫期（通常 15 天全额退保）、非犹豫期退保（退现金价值，金额不等于已交保费）、部分退保等。退保金额的计算又是一个策略问题。支付模块的退款流程需要扩展，不能简单复用。

### 7. 订单关联方多了一层

NFT 只有 buyer + seller。保险有投保人、被保人（可能不同人）、受益人。权限校验逻辑需要扩展——投保人可以操作订单，但被保人可能只有查询权限。

## 八、总结

技术骨架（状态机、校验链、支付、幂等、分布式事务）高度可复用，约 80% 的基础设施代码不用动。核心新增工作量在三个领域特有组件：

1. **核保规则引擎** — 异步规则判定 + 人工审核队列
2. **保费计算器** — 多因素动态定价策略模式
3. **生效调度器** — 等待期管理 + 定时状态推进
