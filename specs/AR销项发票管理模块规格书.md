# AR销项发票管理模块 — 需求分析与技术规格说明书

## AR Sales Invoice Management Module — Technical Specification

> **项目**: Dev_Dept ERP 财务管理系统  
> **模块**: AR销项发票管理 (Accounts Receivable Sales Invoice Management)  
> **版本**: v1.1
> **日期**: 2026-04-20（初版）→ 2026-05-06（v1.1 补充企业收据/商业发票类型）
> **对标系统**: 金蝶云星空 → 应收管理 → 销项发票/销售开票  
> **关联模块**: 实收收款模块规格书.md（下游收款）→ 本模块（全流程）→ 税务管理模块规格书.md（销项税申报）→ 总账凭证引擎规格书.md（凭证落地）  
> **优先级**: **P0（核心缺失）** — 连接销售业务与税务开票的关键桥梁  
> **状态**: 规格设计阶段

---

## 1. 模块定位与业务目标

### 1.1 一句话定义

**AR销项发票管理模块是ERP系统中"订单到收款(O2C)"流程的税务锚点，负责从销售业务出发，经过开票申请、金税开票/全电开票、发票交付、收入确认、销项税申报，最终生成应收凭证写入总账的全流程管理。**

### 1.2 在整体流程中的位置

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐
│ 销售报价  │─▶│ 销售订单  │─▶│ 发货出库  │─▶│ 客户要求开票       │
│ QT       │  │ SO       │  │ DN       │  │ (纸质/全电/专用)   │
└──────────┘  └──────────┘  └──────────┘  └────────┬─────────┘
                                                       │
                                                       ▼
                              ┌─────────────────────────────────────┐
                              │  🧾 AR销项发票管理 (本模块)          │
                              │  开票→交付→收入确认→凭证→税务申报    │
                              └──────────┬──────────────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
             ┌──────────┐        ┌──────────┐        ┌──────────┐
             │ 💰 实收收款 │        │ 🧾 总账凭证 │        │ 🏛️ 税务管理 │
             │ (下游收款) │        │ (凭证落地) │        │ (销项税申报) │
             └──────────┘        └──────────┘        └──────────┘
```

### 1.3 与AP进项发票的镜像关系

| 维度 | AP进项发票（采购端） | AR销项发票（销售端） |
|:---|:---|:---|
| 发票方向 | 供应商→我们（收票） | 我们→客户（开票） |
| 触发起点 | 收到供应商发票 | 销售业务需开票 |
| 匹配逻辑 | 三单匹配（PO+GR+Invoice） | 开票匹配（SO+DN+Invoice） |
| 税务处理 | 进项认证→抵扣 | 销项开票→申报 |
| 凭证方向 | 借:库存/费用 借:进项税 贷:应付 | 借:应收 贷:收入 贷:销项税 |
| 下游模块 | 实付付款 | 实收收款 |
| 凭证编码 | AP01~AP07 | AR01~AR07 |

### 1.4 核心业务目标

| 目标 | 指标 | 当前痛点 |
|:---|:---|:---|
| **开票自动化** | 全电发票自动开票率≥80% | 手工金税系统逐张开票 |
| **税控准确率** | 开票金额与SO/DN差异≤0.01% | 人工核对容易出错 |
| **收入确认及时** | 开票后≤1天完成收入确认 | 开票与入账脱节 |
| **发票交付效率** | 全电发票≤5分钟送达客户 | 纸质发票邮寄2-3天 |
| **销项税完整** | 100%销项发票纳入税务申报 | 漏报/错报风险 |
| **红字规范** | 红字发票100%走规范流程 | 随意开红字、冲销混乱 |

### 1.5 用户角色

| 角色 | 权限范围 | 典型操作 |
|:---|:---|:---|
| **销售助理** | 申请开票、查看开票进度 | 提交开票申请、查询发票状态 |
| **财务会计** | 审核开票、开具发票、入账确认 | 审核开票申请、金税开票、确认收入 |
| **财务主管** | 审批异常、红字审批、参数配置 | 审批红字发票、配置开票规则 |
| **税务专员** | 税务申报、销项税核对 | 月度销项汇总、申报表核对 |
| **系统管理员** | 金税接口配置、开票参数 | 金税参数、开票限额、税码配置 |

---

## 2. 功能需求清单

### 2.1 功能全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                    AR销项发票管理模块功能架构                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                │
│  │ 🔵 开票申请 │  │ 🟠 发票开具 │  │ 🟢 发票交付 │                │
│  │ 管理       │  │ 管理       │  │ 管理       │                │
│  ├────────────┤  ├────────────┤  ├────────────┤                │
│  │ • 自动开票 │  │ • 金税开票  │  │ • 全电交付  │                │
│  │ • 手动申请 │  │ • 全电开票  │  │ • 邮寄管理  │                │
│  │ • 合并开票 │  │ • 批量开票  │  │ • 交付确认  │                │
│  │ • 拆分开票 │  │ • 红字开票  │  │ • 交付追踪  │                │
│  └────────────┘  └────────────┘  └────────────┘                │
│                                                                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                │
│  │ 🟣 收入确认 │  │ 🔴 红字冲红 │  │ 🟤 销项税务 │                │
│  │ 处理       │  │ 管理       │  │ 管理       │                │
│  ├────────────┤  ├────────────┤  ├────────────┤                │
│  │ • 自动确认 │  │ • 红字申请  │  │ • 销项汇总  │                │
│  │ • 凭证生成 │  │ • 客户确认  │  │ • 申报对接  │                │
│  │ • 暂估调整 │  │ • 冲红凭证  │  │ • 税负分析  │                │
│  │ • 汇率处理 │  │ • 退换票    │  │ • 开票限额  │                │
│  └────────────┘  └────────────┘  └────────────┘                │
│                                                                 │
│  ┌──────────────────────────────────────────────────┐           │
│  │ ⚙️ 基础配置: 税码/开票类型/客户开票信息/商品税收编码  │           │
│  └──────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 功能需求明细

#### F1 — 开票申请管理

| 编号 | 功能 | 描述 | 优先级 |
|:---|:---|:---|:---:|
| F1-01 | 自动开票触发 | SO发货确认后自动生成开票申请（可配置触发条件） | P0 |
| F1-02 | 手动开票申请 | 销售助理手动提交开票申请（关联SO/无SO） | P0 |
| F1-03 | 合并开票 | 多个SO/DN合并为一张发票（同一客户+同一税码） | P1 |
| F1-04 | 拆分开票 | 一个SO拆分为多张发票（分批开票/多税码） | P1 |
| F1-05 | 开票金额校验 | 开票金额≤未开票金额，超额拦截 | P0 |
| F1-06 | 开票信息预填 | 自动从SO/客户主数据预填开票信息 | P0 |
| F1-07 | 开票审批流 | 金额≥¥50万或红字发票需审批 | P0 |
| F1-08 | 开票申请撤回 | 未审核前可撤回 | P1 |

#### F2 — 发票开具管理

| 编号 | 功能 | 描述 | 优先级 |
|:---|:---|:---|:---:|
| F2-01 | 金税盘开票 | 通过金税接口在税控盘中开具增值税专用/普通发票 | P0 |
| F2-02 | 全电发票开具 | 通过电子发票服务平台开具全电发票 | P0 |
| F2-03 | 纸质发票打印 | 金税发票打印+邮寄管理 | P1 |
| F2-04 | 批量开票 | 多张申请一次性开票（月度集中开票场景） | P1 |
| F2-05 | 发票作废 | 当月开具的发票可作废（跨月走红字） | P0 |
| F2-06 | 发票号码分配 | 金税盘号码段管理/全电自动分配 | P0 |
| F2-07 | 开票限额检查 | 单张/月累计开票限额校验 | P0 |
| F2-08 | 商品税收编码 | 自动匹配商品→税收分类编码 | P0 |

#### F3 — 发票交付管理

| 编号 | 功能 | 描述 | 优先级 |
|:---|:---|:---|:---:|
| F3-01 | 全电发票自动交付 | 开票后自动推送至客户邮箱/手机 | P0 |
| F3-02 | 纸质发票邮寄登记 | 快递单号、签收状态跟踪 | P1 |
| F3-03 | 交付确认 | 客户确认收到发票（系统/人工） | P1 |
| F3-04 | 交付提醒 | 超期未送达自动提醒 | P2 |
| F3-05 | 发票下载链接 | 生成PDF下载链接供客户自助下载 | P1 |

#### F4 — 收入确认处理

| 编号 | 功能 | 描述 | 优先级 |
|:---|:---|:---|:---:|
| F4-01 | 开票即确认收入 | 开票后自动生成收入确认+应收凭证 | P0 |
| F4-02 | 分期确认收入 | 按合同约定分期确认收入（多期凭证） | P1 |
| F4-03 | 外币收入折算 | 外币发票按汇率折算本位币入账 | P1 |
| F4-04 | 暂估收入调整 | 已暂估收入→实际开票后差异调整 | P2 |
| F4-05 | 自动生成凭证（触发AR01~AR07） | 与总账凭证引擎无缝衔接 | P0 |
| F4-06 | 收入确认撤回 | 入账前可撤回，入账后走红字冲销 | P0 |

#### F5 — 红字冲红管理

| 编号 | 功能 | 描述| 优先级 |
|:---|:---|:---|:---:|
| F5-01 | 红字信息表申请 | 需先申请红字信息表（购买方确认/销售方单方） | P0 |
| F5-02 | 客户确认流程 | 购买方2日内确认，超时自动确认 | P0 |
| F5-03 | 红字发票开具 | 信息表审核通过后开具红字发票 | P0 |
| F5-04 | 部分红冲 | 部分退货/折让，红冲部分金额 | P1 |
| F5-05 | 全额红冲 | 全额退货，红冲全部金额 | P0 |
| F5-06 | 冲红凭证自动生成 | AR06红字冲回凭证 | P0 |
| F5-07 | 退换票管理 | 退票→换票流程管理 | P2 |

#### F6 — 销项税务管理

| 编号 | 功能 | 描述 | 优先级 |
|:---|:---|:---|:---:|
| F6-01 | 销项税月度汇总 | 按月汇总销项税额，对接税务申报 | P0 |
| F6-02 | 销项税额核对 | 销项发票总税额 vs 申报表数据 | P0 |
| F6-03 | 开票限额预警 | 月度开票额接近限额预警 | P1 |
| F6-04 | 零税率/免税发票管理 | 出口免税/零税率发票单独统计 | P1 |
| F6-05 | 税负率分析 | 销项-进项/收入 税负率监控 | P2 |

---

## 3. 数据库设计

### 3.1 ER关系图

```
ar_invoice_apply ──1:N──▶ ar_invoice_apply_line
       │
       │ 1:1
       ▼
ar_invoice ──1:N──▶ ar_invoice_item
       │
       ├──1:1──▶ ar_invoice_deliver
       ├──1:1──▶ ar_revenue_confirm
       └──1:N──▶ ar_red_letter
```

### 3.2 DDL — 核心表

#### 3.2.1 ar_invoice_apply — 开票申请表

```sql
CREATE TABLE ar_invoice_apply (
    apply_id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    apply_no          VARCHAR(32) NOT NULL UNIQUE COMMENT '申请单号 AR-AP-YYYYMMDD-NNN',
    so_id             BIGINT COMMENT '关联销售订单ID',
    so_no             VARCHAR(32) COMMENT '关联销售订单号',
    customer_id       BIGINT NOT NULL COMMENT '客户ID',
    customer_name     VARCHAR(200) NOT NULL COMMENT '客户名称',
    taxpayer_no       VARCHAR(30) COMMENT '客户纳税人识别号',
    invoice_type      VARCHAR(20) NOT NULL COMMENT '发票类型: ENTERPRISE企业收据/SPECIAL专票/COMMON普票/FULL_ELECTRONIC_S全电专票/FULL_ELECTRONIC_C全电普票/ZERO_TAX免税',
    invoice_region    CHAR(2) DEFAULT 'CN' COMMENT '开票区域: CN国内/海外走海外模块',
    apply_amount      DECIMAL(18,2) NOT NULL COMMENT '申请开票金额(含税)',
    tax_amount        DECIMAL(18,2) NOT NULL COMMENT '申请税额',
    exclude_tax_amount DECIMAL(18,2) NOT NULL COMMENT '不含税金额',
    currency          VARCHAR(3) DEFAULT 'CNY' COMMENT '币种',
    merge_flag        TINYINT DEFAULT 0 COMMENT '合并开票标志 0否1是',
    merge_source_ids  JSON COMMENT '合并来源申请ID列表',
    status            VARCHAR(20) NOT NULL DEFAULT 'DRAFT' COMMENT '状态: DRAFT/PENDING_REVIEW/APPROVED/INVOICED/CANCELLED',
    apply_user_id     BIGINT COMMENT '申请人ID',
    apply_date        DATETIME NOT NULL COMMENT '申请日期',
    review_user_id    BIGINT COMMENT '审核人ID',
    review_date       DATETIME COMMENT '审核日期',
    remark            VARCHAR(500) COMMENT '备注',
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_so (so_id),
    INDEX idx_customer (customer_id),
    INDEX idx_status (status),
    INDEX idx_date (apply_date)
) COMMENT 'AR开票申请表';
```

#### 3.2.2 ar_invoice_apply_line — 开票申请明细

```sql
CREATE TABLE ar_invoice_apply_line (
    line_id           BIGINT PRIMARY KEY AUTO_INCREMENT,
    apply_id          BIGINT NOT NULL COMMENT '申请ID',
    dn_id             BIGINT COMMENT '发货单ID',
    dn_no             VARCHAR(32) COMMENT '发货单号',
    product_id        BIGINT NOT NULL COMMENT '商品ID',
    product_name      VARCHAR(200) NOT NULL COMMENT '商品名称',
    tax_category_code VARCHAR(30) NOT NULL COMMENT '税收分类编码',
    unit_price        DECIMAL(18,4) NOT NULL COMMENT '单价(不含税)',
    quantity          DECIMAL(18,4) NOT NULL COMMENT '数量',
    tax_rate          DECIMAL(6,4) NOT NULL COMMENT '税率',
    line_amount       DECIMAL(18,2) NOT NULL COMMENT '行金额(不含税)',
    line_tax          DECIMAL(18,2) NOT NULL COMMENT '行税额',
    line_total        DECIMAL(18,2) NOT NULL COMMENT '行价税合计',
    discount_rate     DECIMAL(6,4) DEFAULT 0 COMMENT '折扣率',
    remark            VARCHAR(200) COMMENT '行备注',
    INDEX idx_apply (apply_id),
    FOREIGN KEY (apply_id) REFERENCES ar_invoice_apply(apply_id)
) COMMENT 'AR开票申请明细表';
```

#### 3.2.3 ar_invoice — 销项发票表

```sql
CREATE TABLE ar_invoice (
    invoice_id        BIGINT PRIMARY KEY AUTO_INCREMENT,
    apply_id          BIGINT COMMENT '关联开票申请ID',
    invoice_no        VARCHAR(30) COMMENT '发票号码(开票后回填)',
    invoice_code      VARCHAR(20) COMMENT '发票代码(纸质发票)',
    invoice_type      VARCHAR(20) NOT NULL COMMENT '发票类型: ENTERPRISE企业收据/SPECIAL专票/COMMON普票/FULL_ELECTRONIC_S全电专票/FULL_ELECTRONIC_C全电普票/ZERO_TAX免税',
    invoice_category  VARCHAR(20) NOT NULL DEFAULT 'BLUE' COMMENT '发票种类: BLUE蓝字/RED红字',
    red_letter_id     BIGINT COMMENT '关联红字信息表ID(仅税控发票)',
    original_invoice_id BIGINT COMMENT '原蓝字发票ID(红字关联)',
    converted_from_id BIGINT COMMENT '从企业收据转入的原始ID(转正式开票时)',
    customer_id       BIGINT NOT NULL COMMENT '客户ID',
    customer_name     VARCHAR(200) NOT NULL COMMENT '客户名称',
    taxpayer_no       VARCHAR(30) NOT NULL COMMENT '纳税人识别号',
    invoice_amount    DECIMAL(18,2) NOT NULL COMMENT '开票金额(不含税)',
    tax_amount        DECIMAL(18,2) NOT NULL COMMENT '税额',
    total_amount      DECIMAL(18,2) NOT NULL COMMENT '价税合计',
    currency          VARCHAR(3) DEFAULT 'CNY' COMMENT '币种',
    tax_rate          DECIMAL(6,4) NOT NULL COMMENT '主税率',
    invoice_date      DATE COMMENT '开票日期',
    invoice_method    VARCHAR(20) COMMENT '开票方式: GOLDEN_TAX金税盘/ELECTRONIC全电',
    status            VARCHAR(20) NOT NULL DEFAULT 'DRAFT' COMMENT '状态: DRAFT/INVOICED/DELIVERED/REVENUE_CONFIRMED/VOIDED/RED_REVERSED/CONVERTED(企业收据已转正式)',
    void_flag         TINYINT DEFAULT 0 COMMENT '作废标志 0否1是',
    void_date         DATE COMMENT '作废日期',
    period            VARCHAR(7) COMMENT '归属期间 YYYY-MM',
    revenue_confirmed TINYINT DEFAULT 0 COMMENT '收入确认标志 0未确认1已确认',
    voucher_no        VARCHAR(30) COMMENT '凭证号',
    voucher_id        BIGINT COMMENT '凭证ID',
    remark            VARCHAR(500) COMMENT '备注',
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE INDEX idx_invoice_no (invoice_no),
    INDEX idx_customer (customer_id),
    INDEX idx_status (status),
    INDEX idx_date (invoice_date),
    INDEX idx_period (period),
    FOREIGN KEY (apply_id) REFERENCES ar_invoice_apply(apply_id)
) COMMENT 'AR销项发票表';
```

#### 3.2.4 ar_invoice_item — 销项发票明细

```sql
CREATE TABLE ar_invoice_item (
    item_id           BIGINT PRIMARY KEY AUTO_INCREMENT,
    invoice_id        BIGINT NOT NULL COMMENT '发票ID',
    product_id        BIGINT NOT NULL COMMENT '商品ID',
    product_name      VARCHAR(200) NOT NULL COMMENT '商品名称',
    specification     VARCHAR(200) COMMENT '规格型号',
    unit              VARCHAR(20) COMMENT '计量单位',
    tax_category_code VARCHAR(30) NOT NULL COMMENT '税收分类编码',
    quantity          DECIMAL(18,4) NOT NULL COMMENT '数量',
    unit_price        DECIMAL(18,4) NOT NULL COMMENT '单价(不含税)',
    discount_amount   DECIMAL(18,2) DEFAULT 0 COMMENT '折扣金额',
    line_amount       DECIMAL(18,2) NOT NULL COMMENT '金额(不含税)',
    tax_rate          DECIMAL(6,4) NOT NULL COMMENT '税率',
    tax_amount        DECIMAL(18,2) NOT NULL COMMENT '税额',
    total_amount      DECIMAL(18,2) NOT NULL COMMENT '价税合计',
    zero_tax_flag     TINYINT DEFAULT 0 COMMENT '零税率标志 0否1是',
    zero_tax_type     VARCHAR(10) COMMENT '零税率类型: EXEMPT免税/ZERO零税',
    INDEX idx_invoice (invoice_id),
    FOREIGN KEY (invoice_id) REFERENCES ar_invoice(invoice_id)
) COMMENT 'AR销项发票明细表';
```

#### 3.2.5 ar_invoice_deliver — 发票交付表

```sql
CREATE TABLE ar_invoice_deliver (
    deliver_id        BIGINT PRIMARY KEY AUTO_INCREMENT,
    invoice_id        BIGINT NOT NULL COMMENT '发票ID',
    deliver_method    VARCHAR(20) NOT NULL COMMENT '交付方式: EMAIL电子/DIRECT推送/COURIER快递/HAND手递',
    recipient_email   VARCHAR(200) COMMENT '接收邮箱(电子交付)',
    recipient_phone   VARCHAR(20) COMMENT '接收手机号',
    courier_company   VARCHAR(100) COMMENT '快递公司',
    tracking_no       VARCHAR(50) COMMENT '快递单号',
    deliver_date      DATETIME COMMENT '交付日期',
    deliver_status    VARCHAR(20) NOT NULL DEFAULT 'PENDING' COMMENT '交付状态: PENDING/SENT/RECEIVED/FAILED',
    confirm_date      DATETIME COMMENT '客户确认日期',
    pdf_url           VARCHAR(500) COMMENT 'PDF下载链接',
    retry_count       INT DEFAULT 0 COMMENT '重试次数',
    remark            VARCHAR(500) COMMENT '备注',
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_invoice (invoice_id),
    FOREIGN KEY (invoice_id) REFERENCES ar_invoice(invoice_id)
) COMMENT 'AR发票交付表';
```

#### 3.2.6 ar_revenue_confirm — 收入确认表

```sql
CREATE TABLE ar_revenue_confirm (
    confirm_id        BIGINT PRIMARY KEY AUTO_INCREMENT,
    invoice_id        BIGINT NOT NULL COMMENT '发票ID',
    confirm_type      VARCHAR(20) NOT NULL DEFAULT 'FULL' COMMENT '确认类型: FULL全额/INSTALLMENT分期/ADJUST调整',
    period            VARCHAR(7) NOT NULL COMMENT '确认期间 YYYY-MM',
    revenue_amount    DECIMAL(18,2) NOT NULL COMMENT '确认收入金额(不含税)',
    tax_amount        DECIMAL(18,2) NOT NULL COMMENT '确认税额',
    receivable_amount DECIMAL(18,2) NOT NULL COMMENT '确认应收金额(含税)',
    currency          VARCHAR(3) DEFAULT 'CNY' COMMENT '币种',
    exchange_rate     DECIMAL(12,6) COMMENT '汇率(外币时)',
    base_amount       DECIMAL(18,2) COMMENT '本位币金额(外币折算后)',
    voucher_no        VARCHAR(30) COMMENT '凭证号',
    voucher_id        BIGINT COMMENT '凭证ID',
    voucher_template  VARCHAR(20) NOT NULL COMMENT '凭证模板: AR01~AR07',
    confirm_date      DATE NOT NULL COMMENT '确认日期',
    confirm_user_id   BIGINT COMMENT '确认人ID',
    status            VARCHAR(20) NOT NULL DEFAULT 'DRAFT' COMMENT '状态: DRAFT/CONFIRMED/REVERSED',
    reverse_reason    VARCHAR(200) COMMENT '冲回原因',
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_invoice (invoice_id),
    INDEX idx_period (period),
    FOREIGN KEY (invoice_id) REFERENCES ar_invoice(invoice_id)
) COMMENT 'AR收入确认表';
```

#### 3.2.7 ar_red_letter — 红字信息表

```sql
CREATE TABLE ar_red_letter (
    red_id            BIGINT PRIMARY KEY AUTO_INCREMENT,
    red_no            VARCHAR(32) NOT NULL UNIQUE COMMENT '红字信息表编号',
    original_invoice_id BIGINT NOT NULL COMMENT '原蓝字发票ID',
    original_invoice_no VARCHAR(30) NOT NULL COMMENT '原发票号码',
    apply_type        VARCHAR(20) NOT NULL COMMENT '申请类型: SELLER_SINGLE销售方单方/BUYER_CONFIRM购买方确认',
    red_reason        VARCHAR(200) NOT NULL COMMENT '红冲原因',
    red_amount        DECIMAL(18,2) NOT NULL COMMENT '红冲金额(不含税)',
    red_tax           DECIMAL(18,2) NOT NULL COMMENT '红冲税额',
    red_total         DECIMAL(18,2) NOT NULL COMMENT '红冲价税合计',
    is_partial        TINYINT DEFAULT 0 COMMENT '部分红冲 0否1是',
    buyer_confirm_status VARCHAR(20) COMMENT '购买方确认状态: PENDING/CONFIRMED/REJECTED/TIMEOUT',
    buyer_confirm_date DATETIME COMMENT '购买方确认日期',
    buyer_timeout_deadline DATETIME COMMENT '确认截止时间(申请后48小时)',
    status            VARCHAR(20) NOT NULL DEFAULT 'DRAFT' COMMENT '状态: DRAFT/PENDING_CONFIRM/APPROVED/INVOICED/CANCELLED',
    red_invoice_id    BIGINT COMMENT '开具的红字发票ID',
    apply_user_id     BIGINT COMMENT '申请人ID',
    apply_date        DATETIME COMMENT '申请日期',
    review_user_id    BIGINT COMMENT '审核人ID',
    review_date       DATETIME COMMENT '审核日期',
    remark            VARCHAR(500) COMMENT '备注',
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_original (original_invoice_id),
    INDEX idx_status (status),
    FOREIGN KEY (original_invoice_id) REFERENCES ar_invoice(invoice_id)
) COMMENT 'AR红字信息表';
```

---

## 4. 业务规则与凭证模板

### 4.1 开票类型与税率规则

> **v1.1更新**：新增"企业收据/商业发票"(ENTERPRISE)类型，支持企业自行出具的收款凭证，不走税控系统。企业收据与税控发票统一在同模块管理，通过`invoice_type`字段区分。

| 发票类型 | 枚举值 | 适用场景 | 税率 | 开票方式 | 交付方式 | 纳入税务申报 |
|:---|:---|:---|:---|:---|:---|:---|
| **企业收据/商业发票** | `ENTERPRISE` | 客户不需增值税发票、内部结算、对账凭证 | 不涉及 | 系统自动生成编号+PDF | 即时推送/打印 | ❌ 否 |
| 增值税专用发票 | `SPECIAL` | B2B一般纳税人客户 | 13%/9%/6% | 金税盘/全电 | 邮寄/电子 | ✅ 是 |
| 增值税普通发票 | `COMMON` | B2B小规模/B2C | 13%/9%/6% | 金税盘/全电 | 邮寄/电子 | ✅ 是 |
| 全电发票(专用) | `FULL_ELECTRONIC_S` | B2B一般纳税人 | 13%/9%/6% | 电子税务局 | 即时推送 | ✅ 是 |
| 全电发票(普通) | `FULL_ELECTRONIC_C` | B2B小规模/B2C | 13%/9%/6% | 电子税务局 | 即时推送 | ✅ 是 |
| 免税发票 | `ZERO_TAX` | 出口/免税业务 | 0% | 全电/金税 | 邮寄/电子 | ✅ 是(零税) |

#### 4.1.1 企业收据(ENTERPRISE)详细说明

企业收据是企业在不需要开具国家增值税发票时，自行出具的**收款凭证/结算凭证**，在金蝶云星空中对应"应收单"概念。

| 维度 | 企业收据 ENTERPRISE | 税控发票 SPECIAL/COMMON/... |
|:---|:---|:---|
| 开票方式 | 系统自动生成编号+PDF，无需税控接口 | 调用金税盘/电子税务局接口 |
| 必填字段 | 客户+金额+摘要即可 | 还需纳税人识别号、商品税收编码、税率 |
| 审批流 | 金额≥阈值才需审批 | 所有税控发票都需财务审核 |
| 凭证模板 | AR08（企业收据入账） | AR01~AR07 |
| 税务申报 | ❌ 不纳入销项税申报 | ✅ 纳入 |
| 发票号码 | 系统自动编号（RCP-YYYYMM-NNN） | 税控系统分配号码 |
| 转税控发票 | ✅ 支持"转正式开票" | — |
| 红字冲红 | 直接冲销（无需红字信息表） | 走红字信息表流程 |
| 核销收款 | ✅ 与税控发票一视同仁 | ✅ |

### 4.2 开票金额校验规则

| 校验项 | 规则 | 异常处理 |
|:---|:---|:---|
| 累计开票≤合同金额 | 累计已开票金额(含企业收据+税控发票) + 本次申请 ≤ SO总金额 | 超额拦截，需特殊审批 |
| **企业收据+税控互斥** | **同一SO+DN，企业收据金额 + 税控发票金额 ≤ 合同总金额，防止重复开票** | **超额拦截+提示已有企业收据** |
| **企业收据→转正式** | **转正式开票后，原企业收据金额从累计中移除，税控发票金额加入** | **原子操作，不允许部分转换** |
| 单张开票限额 | 不超过金税盘授权单张限额（仅税控发票校验） | 超额需拆分开票 |
| 月度开票限额 | 月累计 ≤ 月度授权限额（仅税控发票统计） | 接近限额预警(80%/90%/95%) |
| 税额校验 | 金额 × 税率 = 税额（尾差≤¥0.01）（仅税控发票校验） | 尾差自动调整到最后一行 |
| 退货红冲≤原票 | 红冲金额 ≤ 原发票金额 | 超额拦截 |

### 4.3 红字发票业务规则

| 场景 | 申请方 | 流程 | 时效 |
|:---|:---|:---|:---|
| **企业收据冲销** | 销售方单方 | **直接冲销原企业收据，无需红字信息表，无需购买方确认** | 即时 |
| 专用发票-购买方已认证 | 购买方申请 | 购买方填红字信息表→销售方确认→开红字 | 48小时确认 |
| 专用发票-购买方未认证 | 销售方申请 | 销售方填红字信息表→自动通过→开红字 | 即时 |
| 普通发票 | 销售方单方 | 销售方直接开红字（无需购买方确认） | 即时 |
| 全电发票 | 同专用发票规则 | 电子税务局线上流程 | 48小时确认 |
| 当月发票作废 | — | 未交付且当月可作废（不需红字） | 即时 |

### 4.4 凭证模板矩阵

本模块新增 **8套凭证模板**，编码前缀 `AR`（v1.1新增AR08）：

| 编码 | 凭证名称 | 借方 | 贷方 | 说明 |
|:---|:---|:---|:---|:---|
| **AR01** | 销项发票入账（可抵扣-专票） | 应收账款-XX客户 | 主营业务收入 + 应交税费-销项税额 | 专用发票标准入账 |
| **AR02** | 销项发票入账（不可抵扣-普票） | 应收账款-XX客户 | 主营业务收入 + 应交税费-销项税额 | 普票入账（同AR01，区分统计） |
| **AR03** | 预开票确认收入 | 应收账款-XX客户 | 预收账款-XX客户 + 应交税费-销项税额 | 先开票后发货 |
| **AR04** | 外币发票折算 | 应收账款(外币) | 主营业务收入 + 应交税费-销项税额 + 汇兑损益 | 外币业务 |
| **AR05** | 零税率/免税发票 | 应收账款-XX客户 | 主营业务收入 | 出口/免税无销项税 |
| **AR06** | 红字冲回 | 主营业务收入 + 应交税费-销项税额 | 应收账款-XX客户 | 红字全额/部分冲回 |
| **AR07** | 收入调整/折让 | 主营业务收入(红字) / 销售折让 | 应收账款-XX客户(红字) | 销售折让单独入账 |
| **AR08** | **企业收据入账** | 应收账款-XX客户 | 主营业务收入 | 企业自行出具收据，不涉及销项税 |

#### AR01 详细分录（标准销项入账）

```
借: 1122 应收账款-XX客户              113,000.00
  贷: 6001 主营业务收入                 100,000.00
  贷: 2221 应交税费-应交增值税-销项税额   13,000.00
```

#### AR03 详细分录（预开票确认收入）

```
借: 1122 应收账款-XX客户              113,000.00
  贷: 2203 预收账款-XX客户             100,000.00
  贷: 2221 应交税费-应交增值税-销项税额   13,000.00

(发货确认后追加)
借: 2203 预收账款-XX客户             100,000.00
  贷: 6001 主营业务收入                100,000.00
```

#### AR06 详细分录（红字冲回）

```
借: 6001 主营业务收入                  100,000.00  (红字)
借: 2221 应交税费-应交增值税-销项税额    13,000.00  (红字)
  贷: 1122 应收账款-XX客户             113,000.00  (红字)
```

### 4.5 凭证模板JSON定义

```json
{
  "AR.INVOICE.SPECIAL": "AR01",
  "AR.INVOICE.COMMON": "AR02",
  "AR.INVOICE.PRE_INVOICE": "AR03",
  "AR.INVOICE.FOREIGN": "AR04",
  "AR.INVOICE.ZERO_TAX": "AR05",
  "AR.INVOICE.RED_REVERSAL": "AR06",
  "AR.INVOICE.ADJUSTMENT": "AR07",
  "AR.INVOICE.ENTERPRISE": "AR08"
}
```

### 4.6 凭证触发事件映射

| 触发时机 | 事件编码 | 凭证模板 | 说明 |
|:---|:---|:---|:---|
| 专票开票确认 | AR.INVOICE.SPECIAL | AR01 | 销售专票入账 |
| 普票开票确认 | AR.INVOICE.COMMON | AR02 | 销售普票入账 |
| 预开票确认 | AR.INVOICE.PRE_INVOICE | AR03 | 先票后货 |
| 外币发票确认 | AR.INVOICE.FOREIGN | AR04 | 外币折算 |
| 零税率/免税确认 | AR.INVOICE.ZERO_TAX | AR05 | 出口免税 |
| 红字冲回确认 | AR.INVOICE.RED_REVERSAL | AR06 | 红字冲销 |
| 销售折让确认 | AR.INVOICE.ADJUSTMENT | AR07 | 折让调整 |
| **企业收据确认** | **AR.INVOICE.ENTERPRISE** | **AR08** | **企业收据入账（不涉及销项税）** |

---

## 5. API设计

### 5.1 RESTful API清单

| 方法 | 路径 | 说明 | 权限 |
|:---|:---|:---|:---|
| POST | `/api/ar/invoice-apply` | 创建开票申请 | 销售助理+ |
| GET | `/api/ar/invoice-apply/{id}` | 查询开票申请详情 | 全部只读 |
| GET | `/api/ar/invoice-apply` | 开票申请列表（分页+筛选） | 全部只读 |
| PUT | `/api/ar/invoice-apply/{id}` | 修改开票申请（DRAFT状态） | 销售助理+ |
| POST | `/api/ar/invoice-apply/{id}/submit` | 提交审核 | 销售助理+ |
| POST | `/api/ar/invoice-apply/{id}/approve` | 审核通过 | 财务会计+ |
| POST | `/api/ar/invoice-apply/{id}/reject` | 审核驳回 | 财务会计+ |
| POST | `/api/ar/invoice-apply/{id}/cancel` | 撤回申请 | 销售助理(本人) |
| POST | `/api/ar/invoice` | 创建发票（开票） | 财务会计+ |
| GET | `/api/ar/invoice/{id}` | 查询发票详情 | 全部只读 |
| GET | `/api/ar/invoice` | 发票列表（分页+筛选） | 全部只读 |
| POST | `/api/ar/invoice/{id}/void` | 发票作废（当月） | 财务会计+ |
| POST | `/api/ar/invoice/{id}/deliver` | 发票交付 | 财务会计+ |
| POST | `/api/ar/invoice/{id}/confirm-revenue` | 收入确认 | 财务会计+ |
| POST | `/api/ar/invoice/{id}/reverse-revenue` | 收入确认撤回 | 财务主管+ |
| POST | `/api/ar/invoice/{id}/convert-official` | 企业收据转正式开票 | 财务会计+ |
| POST | `/api/ar/red-letter` | 申请红字信息表 | 财务会计+ |
| GET | `/api/ar/red-letter/{id}` | 红字信息表详情 | 全部只读 |
| POST | `/api/ar/red-letter/{id}/buyer-confirm` | 购买方确认 | 外部接口 |
| POST | `/api/ar/red-letter/{id}/issue` | 开具红字发票 | 财务会计+ |
| GET | `/api/ar/tax-summary` | 销项税月度汇总 | 税务专员+ |
| GET | `/api/ar/invoice-statistics` | 开票统计报表 | 全部只读 |

---

## 6. 金税接口对接

### 6.1 接口架构

```
┌───────────────┐     ┌───────────────────┐     ┌──────────────┐
│  AR销项发票模块  │────▶│  金税适配层(Adapter) │────▶│  金税盘/全电平台  │
│  (业务逻辑)    │◀────│  统一接口屏蔽差异    │◀────│  (税控设备)    │
└───────────────┘     └───────────────────┘     └──────────────┘
                            │
                     ┌──────┼──────┐
                     ▼      ▼      ▼
              ┌─────────┐ ┌─────────┐ ┌─────────┐
              │ 百旺金税 │ │ 航天金税 │ │ 全电发票 │
              │ (百旺)  │ │ (航信)  │ │ (电子税局) │
              └─────────┘ └─────────┘ └─────────┘
```

### 6.2 核心接口方法

| 方法 | 说明 | 金税盘 | 全电 |
|:---|:---|:---|:---|
| `IssueInvoice` | 开具发票 | ✅ | ✅ |
| `VoidInvoice` | 作废发票 | ✅ | ✅ |
| `QueryInvoice` | 查询发票状态 | ✅ | ✅ |
| `ApplyRedLetter` | 申请红字信息表 | ✅ | ✅ |
| `IssueRedInvoice` | 开具红字发票 | ✅ | ✅ |
| `GetInvoicePdf` | 获取发票PDF | ✅ | ✅ |
| `GetTaxCategory` | 获取税收分类编码 | ✅ | ✅ |

### 6.3 开票流程（金税盘）

```
开票申请审核通过 → 构造金税报文 → 调用IssueInvoice → 解析返回结果
                                                    ↓
                                            成功 → 回填发票号码/代码
                                            失败 → 记录错误 → 重试/人工处理
```

---

## 7. 异常处理与风控

### 7.1 异常场景

| 异常类型 | 触发条件 | 处理方式 | 严重度 |
|:---|:---|:---|:---:|
| 金税盘离线 | 开票时金税接口不可达 | 自动重试3次→降级为"待开票"队列 | 🔴 高 |
| 开票金额超额 | 申请金额 > SO未开票余额 | 拦截+审批 | 🟡 中 |
| 发票号码跳号 | 金税盘号码不连续 | 记录跳号原因→税局报备 | 🟡 中 |
| 月度限额告警 | 开票额达到限额80% | 预警通知税务专员 | 🟠 预警 |
| 红字确认超时 | 购买方48小时未确认 | 自动确认(可配置) | 🟡 中 |
| 重复开票 | 同一SO重复申请开票 | 拦截+提醒已有申请 | 🔴 高 |
| 客户信息缺失 | 纳税人识别号/地址为空 | 阻止开票+提醒补全 | 🟡 中 |

### 7.2 审批规则

| 场景 | 条件 | 审批层级 |
|:---|:---|:---|
| 大额开票 | 单张≥¥50万 | 财务主管 |
| 超额开票 | 超SO金额开票 | 财务经理+销售总监 |
| 红字发票 | 任何红字申请 | 财务主管 |
| 发票作废 | 当月发票作废 | 财务会计 |
| 跨月红冲 | 跨月/跨年红字 | 财务经理 |
| 外币开票 | 非CNY币种 | 财务主管 |

---

## 8. 与其他模块的集成

### 8.1 集成矩阵

| 模块 | 集成点 | 数据流向 | 方式 |
|:---|:---|:---|:---|
| **实收收款** | 开票→应收→收款核销 | AR发票 → 收款单核销 | 事件驱动 |
| **总账凭证引擎** | 收入确认→凭证 | AR01~AR07 → GL | 事件触发 |
| **税务管理** | 销项税汇总→申报 | 月度汇总 → 申报表 | 定时任务 |
| **发票查验** | 发票号码校验 | 查重/验真 | 同步调用 |
| **海外发票管理** | 海外客户发票 | invoice_region=海外 → 海外模块 | 条件分流 |
| **预算管理** | 开票金额 vs 预算 | 预算校验 | 同步调用 |

### 8.2 凭证总览更新

本模块完成后，系统总凭证模板从 **37套** 增至 **45套**（v1.1 AR从7→8套）：

| 模块 | 编码 | 数量 |
|:---|:---|:---:|
| 实收收款 | S01~S06 | 6 |
| 实付付款 | P01~P08 | 8 |
| 票据往来 | B01~B08 | 8 |
| 应付票据 | PA01~PA07 | 7 |
| AP进项发票 | AP01~AP07 | 7 |
| **AR销项发票** | **AR01~AR08** | **8** |
| 海外发票 | OP01/OS01/FX01 | 3 |
| 总账 | GL01~GL03 | 3 |
| 供应商扣款 | CT01~CT08 | 8 |
| **合计** | | **50** |

---

## 9. 开发路线图

| 阶段 | 内容 | 工期 | 交付物 |
|:---|:---|:---:|:---|
| **Phase 1** | 开票申请+基础配置 | 2天 | 申请CRUD+审批流+税码/开票类型配置 |
| **Phase 2** | 金税/全电开票 | 3天 | 金税适配层+开票接口+号码管理+批量开票 |
| **Phase 3** | 发票交付+收入确认 | 2天 | 交付管理+收入确认+AR01~AR05凭证 |
| **Phase 4** | 红字冲红 | 1.5天 | 红字申请+购买方确认+AR06冲回凭证 |
| **Phase 5** | 销项税务+统计 | 1天 | 月度汇总+申报对接+开票统计 |
| **Phase 6** | 集成测试+优化 | 0.5天 | 与实收/总账/税务联调 |
| **合计** | | **10天** | |

---

## 📝 附录

### A. 状态机

**开票申请状态流:**
```
DRAFT → PENDING_REVIEW → APPROVED → INVOICED
  │           │              │
  └── CANCELLED ←────────────┘
```

**销项发票状态流:**
```
DRAFT → INVOICED → DELIVERED → REVENUE_CONFIRMED
  │         │
  │         ├── VOIDED (当月作废)
  │         └── RED_REVERSED (跨月红冲)
  └── CANCELLED
```

**企业收据状态流(ENTERPRISE):**
```
DRAFT → INVOICED(自动生成PDF) → DELIVERED → REVENUE_CONFIRMED
  │         │
  │         ├── CONVERTED (已转正式税控发票，原收据标记)
  │         └── RED_REVERSED (直接冲销，无需红字信息表)
  └── CANCELLED
```

**企业收据→转正式开票流程:**
```
企业收据(REVENUE_CONFIRMED)
       │
       ▼  操作: "转正式开票"
生成新开票申请(自动关联原SO/DN，金额=原收据金额)
       │
       ▼  审核通过 → 金税/全电开票
生成税控发票 → 原企业收据状态标记为 CONVERTED
```

**红字信息表状态流:**
```
DRAFT → PENDING_CONFIRM → APPROVED → INVOICED
  │           │
  └── CANCELLED ←──────────┘
```

### B. 发票号码规则

| 发票类型 | 号码格式 | 示例 | 说明 |
|:---|:---|:---|:---|
| **企业收据** | **RCP-YYYYMM-NNN** | **RCP-202605-001** | **系统自动分配** |
| 增值税专用发票 | 8位数字 | 01234567 | 金税盘分配 |
| 增值税普通发票 | 8位数字 | 87654321 | 金税盘分配 |
| 全电发票(专票) | 20位 | 0_20260420_12345678 | 电子税局分配 |
| 全电发票(普票) | 20位 | 1_20260420_12345678 | 电子税局分配 |
