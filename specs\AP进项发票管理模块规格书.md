# AP进项发票管理模块 — 需求分析与技术规格说明书

## AP Purchase Invoice Management Module — Technical Specification

> **项目**: Dev_Dept ERP 财务管理系统  
> **模块**: AP进项发票管理 (Accounts Payable Purchase Invoice Management)  
> **版本**: v1.0  
> **日期**: 2026-04-20  
> **对标系统**: 金蝶云星空 → 应付管理 → 进项发票/采购发票  
> **关联模块**: 发票查验模块规格书.md（上游验真）→ 本模块（全流程）→ 实付付款模块规格书.md（下游付款）→ 总账凭证引擎规格书.md（凭证落地）  
> **优先级**: **P0（核心缺失）** — 连接发票查验与应付付款的关键桥梁  
> **状态**: 规格设计阶段  

---

## 1. 模块定位与业务目标

### 1.1 一句话定义

**AP进项发票管理模块是ERP系统中"采购到付款(P2P)"流程的凭证锚点，负责从供应商发票录入/查验开始，经过三单匹配、入账确认、税务认证、进项抵扣，最终生成应付凭证写入总账的全流程管理。**

### 1.2 在整体流程中的位置

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐
│ 采购申请  │─▶│ 采购订单  │─▶│ 收货入库  │─▶│ 供应商发票到达     │
│ PR       │  │ PO       │  │ GR       │  │ (纸质/电子/全电)   │
└──────────┘  └──────────┘  └──────────┘  └────────┬─────────┘
                                                       │
                                                       ▼
                                            ┌──────────────────┐
                                            │  📥 发票查验      │ ← 发票查验模块(已建)
                                            │  验真+状态检测     │
                                            └────────┬─────────┘
                                                     │ 通过
                                                     ▼
                               ┌─────────────────────────────────────────┐
                               │  📋 AP进项发票管理（本模块）              │
                               │                                         │
                               │  F1 发票录入    →  多渠道采集+标准化       │
                               │  F2 三单匹配    →  PO+GR+Invoice自动校验   │
                               │  F3 入账确认    →  应付挂账+税额处理        │
                               │  F4 认证抵扣    →  勾选确认+进项税抵扣      │
                               │  F5 红字处理    →  红字信息表+冲回          │
                               │  F6 台账报表    →  进项台账+抵扣统计        │
                               └─────────────────┬───────────────────────┘
                                                 │
                      ┌──────────────────────────┼──────────────────────┐
                      ▼                          ▼                      ▼
              ┌──────────────┐         ┌──────────────┐       ┌──────────────┐
              │ 🧾 总账凭证    │         │ 💳 实付付款    │       │ 🏛️ 税务管理    │
              │ AP01~AP07    │         │ P01~P08      │       │ 进项抵扣汇总  │
              └──────────────┘         └──────────────┘       └──────────────┘
```

### 1.3 与已有模块的衔接关系

| 上游模块 | 衔接点 | 本模块消费的数据 |
|:---|:---|:---|
| **发票查验模块** | 查验通过后的发票数据 | 验真结果、票面信息、查验状态 |
| **采购订单(PO)** | 采购入库时关联 | 采购订单号、供应商、物料明细、含税价 |
| **收货入库(GR)** | 三单匹配的关键锚点 | 入库单号、实收数量、入库日期 |
| **供应商主数据** | 供应商信息校验 | 供应商税号、银行账户、开票信息 |

| 下游模块 | 衔接点 | 本模块产出的数据 |
|:---|:---|:---|
| **实付付款模块** | 应付确认 → 付款申请 | 应付单号、金额、供应商、到期日 |
| **总账凭证引擎** | 入账 → 自动凭证 | AP01~AP07 凭证模板触发事件 |
| **税务管理模块** | 进项抵扣 → 增值税计算 | 认证状态、可抵扣税额、抵扣期间 |
| **海外发票管理** | 海外进项走人工审核 | invoice_region=海外时走海外流程 |

### 1.4 核心业务目标

| 目标 | 指标 | 当前痛点 |
|:---|:---|:---|
| **三单匹配率** | 自动匹配率≥85%，异常率≤5% | 手工核对PO/GR/发票，耗时长易出错 |
| **入账及时性** | 发票到达到应付确认≤3个工作日 | 月末集中入账，应付余额平时不准 |
| **认证完整性** | 可抵扣发票100%完成勾选认证 | 漏认证导致多交税，无法追溯 |
| **进项抵扣准确** | 抵扣税额与税局申报100%一致 | 手工统计进项，与申报表经常对不上 |
| **重复发票拦截** | 100%拦截重复报销/重复入账 | 同一张发票被不同人提交多次 |
| **凭证自动化** | 入账确认后自动生成凭证 | 手工做分录，进项税/应付/采购容易搞错 |

### 1.5 用户角色

| 角色 | 权限范围 | 典型操作 |
|:---|:---|:---|
| **应付会计** | 创建/编辑进项发票、三单匹配、入账 | 录入发票、匹配校验、确认入账 |
| **税务会计** | 认证勾选、抵扣确认、进项转出 | 批量勾选认证、处理不可抵扣项 |
| **财务主管** | 审核异常匹配、红字审批 | 审批红字信息表、异常三单匹配 |
| **采购人员** | 查看发票匹配状态、提交发票 | 上传供应商发票、查看匹配结果 |
| **出纳** | 查看待付款发票 | 查看应付到期日、准备付款 |
| **财务总监** | 进项统计报表 | 抵扣分析、税负监控 |

---

## 2. 功能全景图

### 2.1 六大功能域

```
AP进项发票管理
├── F1 进项发票录入
│   ├── F1.1 手工录入发票
│   ├── F1.2 OCR智能识别（腾讯云VatInvoiceOCR）
│   ├── F1.3 批量导入（Excel/图片包）
│   ├── F1.4 电子发票/XML自动采集
│   ├── F1.5 查验联动（自动触发发票查验模块）
│   ├── F1.6 重复发票检测
│   └── F1.7 发票信息补录与修正
│
├── F2 三单匹配
│   ├── F2.1 自动匹配引擎（PO+GR+Invoice）
│   ├── F2.2 匹配规则配置（容差/必填/阈值）
│   ├── F2.3 匹配结果审批（差异超阈值需审批）
│   ├── F2.4 部分匹配处理（分批到货/分票场景）
│   ├── F2.5 无PO处理（费用类/零星采购）
│   └── F2.6 匹配异常处理（价格差异/数量差异/税码差异）
│
├── F3 入账确认
│   ├── F3.1 应付挂账（确认应付金额）
│   ├── F3.2 税额分离（进项税额/不含税金额自动拆分）
│   ├── F3.3 多税码行处理（一张发票多税率明细）
│   ├── F3.4 费用分摊（按部门/项目/成本中心分摊）
│   ├── F3.5 自动生成凭证（触发AP01~AP07）
│   └── F3.6 入账撤回（审核前可撤回）
│
├── F4 认证抵扣
│   ├── F4.1 勾选认证（单张/批量勾选确认）
│   ├── F4.2 认证期间管理（增值税申报期关联）
│   ├── F4.3 进项转出（非应税项目/集体福利/个人消费等）
│   ├── F4.4 不可抵扣处理（专用于免税项目/非正常损失）
│   ├── F4.5 加计抵减（先进制造业/生活服务业10%/5%）
│   └── F4.6 认证撤销（申报前可撤回勾选）
│
├── F5 红字处理
│   ├── F5.1 红字信息表申请
│   ├── F5.2 供应商红字发票确认
│   ├── F5.3 红字冲回凭证
│   ├── F5.4 部分红冲处理
│   └── F5.5 红字与原蓝字关联追踪
│
└── F6 台账报表
    ├── F6.1 进项发票台账（全量+多维筛选）
    ├── F6.2 认证抵扣统计表（按期间/税码/供应商）
    ├── F6.3 三单匹配差异报告
    ├── F6.4 进项税额明细表
    ├── F6.5 应付账龄分析（按供应商/到期日）
    └── F6.6 异常发票监控看板
```

### 2.2 功能清单（按优先级）

| 编号 | 功能 | 域 | 优先级 | 说明 |
|:---|:---|:---:|:---:|:---|
| F1.1 | 手工录入发票 | F1 | **P0** | 基础录入 |
| F1.2 | OCR智能识别 | F1 | **P0** | 效率提升 |
| F1.5 | 查验联动 | F1 | **P0** | 与查验模块自动衔接 |
| F1.6 | 重复发票检测 | F1 | **P0** | 合规底线 |
| F2.1 | 自动匹配引擎 | F2 | **P0** | 核心引擎 |
| F2.2 | 匹配规则配置 | F2 | **P0** | 匹配规则 |
| F2.6 | 匹配异常处理 | F2 | **P0** | 异常必须处理 |
| F3.1 | 应付挂账 | F3 | **P0** | 核心入账 |
| F3.2 | 税额分离 | F3 | **P0** | 进项税必须准确 |
| F3.5 | 自动生成凭证 | F3 | **P0** | 凭证自动化 |
| F4.1 | 勾选认证 | F4 | **P0** | 税务合规 |
| F4.2 | 认证期间管理 | F4 | **P0** | 申报期绑定 |
| F4.3 | 进项转出 | F4 | **P0** | 合规必须 |
| F5.1 | 红字信息表申请 | F5 | **P0** | 红字起点 |
| F5.3 | 红字冲回凭证 | F5 | **P0** | 凭证冲回 |
| F1.3 | 批量导入 | F1 | P1 | 批量效率 |
| F1.4 | 电子发票自动采集 | F1 | P1 | 全电票普及后必需 |
| F1.7 | 信息补录修正 | F1 | P1 | 补充修正 |
| F2.3 | 匹配结果审批 | F2 | P1 | 差异审批 |
| F2.4 | 部分匹配处理 | F2 | P1 | 分批场景 |
| F2.5 | 无PO处理 | F2 | P1 | 费用类发票 |
| F3.3 | 多税码行处理 | F3 | P1 | 多税率明细 |
| F3.4 | 费用分摊 | F3 | P1 | 多维分摊 |
| F3.6 | 入账撤回 | F3 | P1 | 灵活处理 |
| F4.4 | 不可抵扣处理 | F4 | P1 | 不抵扣场景 |
| F4.5 | 加计抵减 | F4 | P2 | 特定行业 |
| F4.6 | 认证撤销 | F4 | P1 | 申报前撤回 |
| F5.2 | 供应商红字确认 | F5 | P1 | 双方确认 |
| F5.4 | 部分红冲 | F5 | P2 | 部分退回 |
| F5.5 | 红蓝关联追踪 | F5 | P1 | 审计追溯 |
| F6.1 | 进项发票台账 | F6 | **P0** | 全量台账 |
| F6.2 | 认证抵扣统计 | F6 | **P0** | 税务统计 |
| F6.3 | 三单匹配差异报告 | F6 | P1 | 异常分析 |
| F6.4 | 进项税额明细表 | F6 | P1 | 税额明细 |
| F6.5 | 应付账龄分析 | F6 | P1 | 资金管理 |
| F6.6 | 异常发票监控看板 | F6 | P2 | 实时监控 |

---

## 3. 数据模型设计

### 3.1 核心ER关系

```
┌──────────────────┐          ┌──────────────────┐
│  ap_invoice       │          │  ap_invoice_item │
│  ★进项发票主表     │ 1─────N  │  ★发票明细行     │
│                  │          │                  │
│  API202604000001 │          │  行1:物料A ¥10k  │
│  供应商:XX公司    │          │  行2:物料B ¥20k  │
├──────────────────┤          ├──────────────────┤
│  ap_match_result │          │  ap_cert_record  │
│  三单匹配结果     │ 1─────1  │  认证抵扣记录     │
│                  │          │                  │
│  PO:匹配 ✅      │          │  2026-04勾选     │
│  GR:差异 ⚠️      │          │  抵扣税额¥2.5k   │
├──────────────────┤          ├──────────────────┤
│  ap_red_letter   │          │  ap_tax_split    │
│  红字信息表       │          │  税额拆分明细     │
│                  │          │                  │
│  红冲原票关联     │          │  进项税/销项税    │
└──────────────────┘          └──────────────────┘
```

### 3.2 表结构DDL

#### 表1: ap_invoice（进项发票主表）

```sql
CREATE TABLE ap_invoice (
    id                BIGINT PRIMARY KEY AUTO_INCREMENT,
    invoice_no        VARCHAR(20)    NOT NULL        COMMENT '发票号码(8位或20位全电)',
    invoice_code      VARCHAR(20)    DEFAULT NULL    COMMENT '发票代码(全电票为空)',
    invoice_date      DATE           NOT NULL        COMMENT '开票日期',
    invoice_kind      VARCHAR(4)     NOT NULL        COMMENT '票种代码:01专票/04普票/08电子专/10电子普等',
    
    -- 供应商信息
    supplier_id       BIGINT         NOT NULL        COMMENT '供应商ID(关联supplier主表)',
    supplier_name     VARCHAR(200)   NOT NULL        COMMENT '供应商名称(冗余)',
    supplier_tax_code VARCHAR(30)    NOT NULL        COMMENT '供应商税号',
    
    -- 购方信息(即本公司)
    buyer_name        VARCHAR(200)   DEFAULT NULL    COMMENT '购方名称',
    buyer_tax_code    VARCHAR(30)    DEFAULT NULL    COMMENT '购方税号',
    
    -- 金额信息
    amount_with_tax   DECIMAL(14,2)  NOT NULL        COMMENT '价税合计',
    amount_no_tax     DECIMAL(14,2)  NOT NULL        COMMENT '不含税金额',
    tax_amount        DECIMAL(14,2)  NOT NULL        COMMENT '税额合计',
    currency          VARCHAR(6)     NOT NULL DEFAULT 'CNY' COMMENT '币种',
    
    -- 发票来源
    source_type       VARCHAR(20)    NOT NULL DEFAULT 'manual' 
        COMMENT '来源: manual手工/ocr识别/batch批量/xml电子/xml_auto自动采集',
    source_ref_id     BIGINT         DEFAULT NULL    COMMENT '关联来源ID(查验记录ID等)',
    
    -- 查验联动
    verify_log_id     BIGINT         DEFAULT NULL    COMMENT '关联发票查验记录ID',
    verify_status     TINYINT        NOT NULL DEFAULT 0 
        COMMENT '查验状态: 0未查/1通过/2异常/3拒绝',
    
    -- 三单匹配
    match_status      TINYINT        NOT NULL DEFAULT 0 
        COMMENT '匹配状态: 0未匹配/1自动匹配/2人工确认/3差异待审/4豁免匹配',
    match_result_id   BIGINT         DEFAULT NULL    COMMENT '关联匹配结果ID',
    
    -- 入账状态
    accounting_status TINYINT        NOT NULL DEFAULT 0 
        COMMENT '入账状态: 0待入账/1已入账/2已撤回/3已红冲',
    accounting_date   DATE           DEFAULT NULL    COMMENT '入账日期',
    voucher_id        BIGINT         DEFAULT NULL    COMMENT '关联凭证ID',
    
    -- 认证状态
    cert_status       TINYINT        NOT NULL DEFAULT 0 
        COMMENT '认证状态: 0未认证/1已勾选/2已抵扣/3进项转出/4不可抵扣',
    cert_period       VARCHAR(20)    DEFAULT NULL    COMMENT '认证所属期间(如2026-04)',
    
    -- 红字关联
    is_red_letter     TINYINT        NOT NULL DEFAULT 0 COMMENT '是否红字发票: 0否 1是',
    original_invoice_id BIGINT       DEFAULT NULL    COMMENT '原蓝字发票ID(红字时关联)',
    red_letter_id     BIGINT         DEFAULT NULL    COMMENT '关联红字信息表ID',
    
    -- 发票类型标记
    invoice_category  VARCHAR(20)    NOT NULL DEFAULT 'purchase' 
        COMMENT '发票类别: purchase采购/expense费用/asset资产/import进口',
    has_po            TINYINT        NOT NULL DEFAULT 1 COMMENT '是否有关联PO: 0无 1有',
    
    -- 附件
    attachment_count  TINYINT        DEFAULT 0       COMMENT '附件数量',
    
    -- 备注
    remark            VARCHAR(500)   DEFAULT NULL    COMMENT '备注',
    
    -- 审计字段
    created_by        BIGINT         NOT NULL        COMMENT '创建人',
    created_at        DATETIME       DEFAULT CURRENT_TIMESTAMP,
    updated_by        BIGINT         DEFAULT NULL,
    updated_at        DATETIME       DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- 索引
    UNIQUE KEY uk_invoice (invoice_code, invoice_no),
    INDEX idx_supplier (supplier_id),
    INDEX idx_verify_status (verify_status),
    INDEX idx_match_status (match_status),
    INDEX idx_accounting_status (accounting_status),
    INDEX idx_cert_status (cert_status),
    INDEX idx_invoice_date (invoice_date),
    INDEX idx_invoice_kind (invoice_kind),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 
COMMENT='AP进项发票主表 - 每张发票一条记录';
```

#### 表2: ap_invoice_item（发票明细行）

```sql
CREATE TABLE ap_invoice_item (
    id                BIGINT PRIMARY KEY AUTO_INCREMENT,
    invoice_id        BIGINT         NOT NULL        COMMENT '关联进项发票ID',
    line_no           TINYINT        NOT NULL        COMMENT '行号(从1开始)',
    
    -- 商品/服务信息
    item_name         VARCHAR(200)   NOT NULL        COMMENT '商品/服务名称',
    item_spec         VARCHAR(100)   DEFAULT NULL    COMMENT '规格型号',
    unit              VARCHAR(20)    DEFAULT NULL    COMMENT '计量单位',
    quantity          DECIMAL(14,4)  NOT NULL DEFAULT 1 COMMENT '数量',
    unit_price        DECIMAL(14,4)  NOT NULL        COMMENT '单价(不含税)',
    
    -- 金额
    amount_no_tax     DECIMAL(14,2)  NOT NULL        COMMENT '不含税金额',
    tax_rate          DECIMAL(5,2)   NOT NULL        COMMENT '税率(%)',
    tax_amount        DECIMAL(14,2)  NOT NULL        COMMENT '税额',
    amount_with_tax   DECIMAL(14,2)  NOT NULL        COMMENT '价税合计',
    
    -- 匹配关联
    po_line_id        BIGINT         DEFAULT NULL    COMMENT '关联采购订单行ID',
    gr_line_id        BIGINT         DEFAULT NULL    COMMENT '关联入库单行ID',
    
    -- 会计科目(费用分摊)
    account_code      VARCHAR(20)    DEFAULT NULL    COMMENT '费用/采购科目代码',
    cost_center       VARCHAR(50)    DEFAULT NULL    COMMENT '成本中心',
    department_id     BIGINT         DEFAULT NULL    COMMENT '部门ID',
    project_id        BIGINT         DEFAULT NULL    COMMENT '项目ID',
    
    sort_order        INT            DEFAULT 0       COMMENT '排序号',
    
    created_at        DATETIME       DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME       DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_invoice_id (invoice_id),
    INDEX idx_po_line (po_line_id),
    INDEX idx_gr_line (gr_line_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 
COMMENT='AP进项发票明细行 - 对应发票上的商品行';
```

#### 表3: ap_match_result（三单匹配结果表）

```sql
CREATE TABLE ap_match_result (
    id                BIGINT PRIMARY KEY AUTO_INCREMENT,
    invoice_id        BIGINT         NOT NULL        COMMENT '关联进项发票ID',
    
    -- 匹配方式
    match_type        VARCHAR(20)    NOT NULL 
        COMMENT '匹配方式: auto自动/manual人工/exempt豁免/no_po无PO',
    
    -- PO关联
    po_id             BIGINT         DEFAULT NULL    COMMENT '关联采购订单ID',
    po_no             VARCHAR(30)    DEFAULT NULL    COMMENT '采购订单号(冗余)',
    
    -- GR关联
    gr_id             BIGINT         DEFAULT NULL    COMMENT '关联入库单ID',
    gr_no             VARCHAR(30)    DEFAULT NULL    COMMENT '入库单号(冗余)',
    
    -- 匹配结果
    match_status      TINYINT        NOT NULL DEFAULT 0
        COMMENT '匹配结果: 0待匹配/1完全匹配/2价格差异/3数量差异/4税码差异/5多差异',
    
    -- 差异详情
    price_diff        DECIMAL(14,2)  DEFAULT NULL    COMMENT '价格差异额',
    price_diff_pct    DECIMAL(5,2)   DEFAULT NULL    COMMENT '价格差异率(%)',
    qty_diff          DECIMAL(14,4)  DEFAULT NULL    COMMENT '数量差异',
    tax_diff          DECIMAL(14,2)  DEFAULT NULL    COMMENT '税额差异',
    
    -- 容差检查
    tolerance_check   VARCHAR(10)    NOT NULL DEFAULT 'PASS' 
        COMMENT '容差检查: PASS通过/WARN警告/FAIL超限',
    
    -- 审批
    approval_status   TINYINT        NOT NULL DEFAULT 0 
        COMMENT '审批状态: 0无需审批/1待审批/2已批准/3已拒绝',
    approved_by       BIGINT         DEFAULT NULL    COMMENT '审批人',
    approved_at       DATETIME       DEFAULT NULL    COMMENT '审批时间',
    approval_note     VARCHAR(500)   DEFAULT NULL    COMMENT '审批备注',
    
    -- 匹配行明细(JSON)
    match_detail      JSON           DEFAULT NULL    COMMENT '匹配行明细(逐行对比结果)',
    
    remark            VARCHAR(500)   DEFAULT NULL,
    created_by        BIGINT         NOT NULL,
    created_at        DATETIME       DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME       DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_invoice_id (invoice_id),
    INDEX idx_po_id (po_id),
    INDEX idx_match_status (match_status),
    INDEX idx_approval (approval_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 
COMMENT='三单匹配结果表 - PO+GR+Invoice三单校验';
```

#### 表4: ap_cert_record（认证抵扣记录表）

```sql
CREATE TABLE ap_cert_record (
    id                BIGINT PRIMARY KEY AUTO_INCREMENT,
    invoice_id        BIGINT         NOT NULL        COMMENT '关联进项发票ID',
    
    -- 认证信息
    cert_type         VARCHAR(20)    NOT NULL DEFAULT 'deduct' 
        COMMENT '类型: deduct可抵扣/transfer_out进项转出/no_deduct不可抵扣/surcharge加计抵减',
    cert_period       VARCHAR(20)    NOT NULL        COMMENT '认证所属期间(如2026-04)',
    
    -- 金额
    tax_amount        DECIMAL(14,2)  NOT NULL        COMMENT '认证/转出/加计税额',
    surcharge_rate    DECIMAL(5,2)   DEFAULT NULL    COMMENT '加计抵减率(%)',
    surcharge_amount  DECIMAL(14,2)  DEFAULT NULL    COMMENT '加计抵减额',
    
    -- 进项转出原因
    transfer_reason   VARCHAR(30)    DEFAULT NULL 
        COMMENT '转出原因: non_taxable非应税/collective集体福利/personal个人消费/abnormal_loss非正常损失/export免税出口/other其他',
    
    -- 凭证关联
    voucher_id        BIGINT         DEFAULT NULL    COMMENT '关联凭证ID(进项转出时生成)',
    
    -- 状态
    status            TINYINT        NOT NULL DEFAULT 1 
        COMMENT '状态: 0已撤销/1有效',
    revoked_at        DATETIME       DEFAULT NULL    COMMENT '撤销时间',
    
    remark            VARCHAR(500)   DEFAULT NULL,
    created_by        BIGINT         NOT NULL,
    created_at        DATETIME       DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_invoice_id (invoice_id),
    INDEX idx_cert_period (cert_period),
    INDEX idx_cert_type (cert_type),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 
COMMENT='进项发票认证抵扣记录 - 勾选认证/进项转出/加计抵减';
```

#### 表5: ap_red_letter（红字信息表）

```sql
CREATE TABLE ap_red_letter (
    id                BIGINT PRIMARY KEY AUTO_INCREMENT,
    red_letter_no     VARCHAR(30)    NOT NULL UNIQUE COMMENT '红字信息表编号',
    
    -- 原发票关联
    original_invoice_id BIGINT       NOT NULL        COMMENT '原蓝字发票ID',
    original_invoice_no VARCHAR(20)  NOT NULL        COMMENT '原发票号码(冗余)',
    original_invoice_code VARCHAR(20) DEFAULT NULL   COMMENT '原发票代码',
    
    -- 红字信息
    red_reason        VARCHAR(30)    NOT NULL 
        COMMENT '红字原因: return退货/discount折让/quality质量问题/wrong_info开票有误/other其他',
    red_amount_no_tax DECIMAL(14,2)  NOT NULL        COMMENT '红冲不含税金额',
    red_tax_amount    DECIMAL(14,2)  NOT NULL        COMMENT '红冲税额',
    red_amount_total  DECIMAL(14,2)  NOT NULL        COMMENT '红冲价税合计',
    is_partial        TINYINT        NOT NULL DEFAULT 0 COMMENT '是否部分红冲: 0全部 1部分',
    
    -- 审批流程
    status            TINYINT        NOT NULL DEFAULT 0 
        COMMENT '状态: 0草稿/1待供应商确认/2已确认/3已开红票/4已作废',
    supplier_confirmed TINYINT       NOT NULL DEFAULT 0 COMMENT '供应商确认: 0否 1是',
    confirmed_at      DATETIME       DEFAULT NULL    COMMENT '确认时间',
    
    -- 红字发票回填
    red_invoice_id    BIGINT         DEFAULT NULL    COMMENT '收到的红字发票ID(回填)',
    red_invoice_no    VARCHAR(20)    DEFAULT NULL    COMMENT '红字发票号码(回填)',
    
    -- 凭证关联
    voucher_id        BIGINT         DEFAULT NULL    COMMENT '冲回凭证ID',
    
    remark            VARCHAR(500)   DEFAULT NULL,
    created_by        BIGINT         NOT NULL,
    created_at        DATETIME       DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME       DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_original (original_invoice_id),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 
COMMENT='红字信息表 - 进项红字发票申请与追踪';
```

#### 表6: ap_invoice_attachment（发票附件表）

```sql
CREATE TABLE ap_invoice_attachment (
    id                BIGINT PRIMARY KEY AUTO_INCREMENT,
    invoice_id        BIGINT         NOT NULL        COMMENT '关联发票ID',
    file_name         VARCHAR(200)   NOT NULL        COMMENT '文件名',
    file_path         VARCHAR(500)   NOT NULL        COMMENT '存储路径',
    file_size         INT            NOT NULL        COMMENT '文件大小(bytes)',
    file_type         VARCHAR(20)    NOT NULL 
        COMMENT '文件类型: image图片/pdf/ocr_xml/excel/other',
    uploaded_by       BIGINT         NOT NULL        COMMENT '上传人',
    uploaded_at       DATETIME       DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_invoice_id (invoice_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 
COMMENT='进项发票附件表';
```

---

## 4. 业务规则与凭证模板

### 4.1 进项发票全生命周期状态机

```
                              ┌──────────────────────────────────────┐
                              │                                      │
  [草稿] ──提交──▶ [待查验] ──查验通过──▶ [查验通过] ──匹配──▶ [已匹配] ──入账──▶ [已入账]
    │                 │                                          │            │
    │              查验异常                                       │         入账撤回
    │                 │                                          │            │
    ▼                 ▼                                          ▼            ▼
  [删除]           [查验异常]                                [差异待审]   [已撤回]
                     │                                          │
                  人工审核                                   审批通过
                     │                                          │
                     ▼                                          ▼
               [退回修改]                                  [已匹配]
               [强制入账]
```

```
                                        ┌──勾选认证──▶ [已认证] ──申报抵扣──▶ [已抵扣]
                                        │
  [已入账] ─────────────────────────────┤
                                        ├──进项转出──▶ [已转出] ──生成凭证
                                        │
                                        └──不可抵扣──▶ [不可抵扣]
```

### 4.2 三单匹配规则

#### 匹配优先级

```
优先级1: PO号 + 行号精确匹配 → 自动关联
优先级2: 供应商 + 物料编码 + 数量+金额模糊匹配 → 推荐候选
优先级3: 纯手工选择PO/GR行
```

#### 容差规则

| 匹配维度 | 容差范围 | 超限处理 |
|:---|:---|:---|
| **数量差异** | ≤ 3% 或 ≤ 最小包装量 | 自动通过 |
| **单价差异** | ≤ 2% | 自动通过 |
| **税额差异** | ≤ ¥1.00（四舍五入误差） | 自动通过 |
| **合计金额差异** | ≤ 0% | 必须完全一致 |
| 数量差异 > 3% | — | ⚠️ 需人工确认 |
| 单价差异 > 2% | — | ⚠️ 需主管审批 |
| 合计金额不符 | — | ❌ 拒绝入账 |

#### 特殊场景匹配规则

| 场景 | 匹配方式 | 说明 |
|:---|:---|:---|
| 分批到货，一张发票 | 一票对多GR | 发票金额 = ΣGR金额 |
| 一次到货，多张发票 | 多票对一GR | Σ发票金额 = GR金额 |
| 费用类发票（无PO/GR） | 豁免匹配 | 走费用审批流 |
| 零星采购（<¥500） | 简化匹配 | 仅校验供应商+金额 |
| 进口发票（海关缴款书） | 特殊匹配 | 需关联报关单 |

### 4.3 认证抵扣规则

| 场景 | 可否抵扣 | 系统处理 |
|:---|:---:|:---|
| 专票 - 应税项目 | ✅ | 自动标记可抵扣 |
| 专票 - 非应税项目 | ❌ | 标记进项转出 |
| 专票 - 集体福利 | ❌ | 标记进项转出 |
| 专票 - 个人消费 | ❌ | 标记进项转出 |
| 专票 - 非正常损失 | ❌ | 标记进项转出 |
| 普票 - 通行费(电子) | ✅ | 按规定计算抵扣 |
| 普票 - 其他 | ❌ | 自动标记不可抵扣 |
| 农产品收购票 | ✅ | 计算9%或10%抵扣 |
| 海关缴款书 | ✅ | 需单独采集录入 |
| 货运专票 | ✅ | 自动标记可抵扣 |

### 4.4 凭证模板矩阵

本模块新增 **7套凭证模板**，编码前缀 `AP`：

| 编码 | 场景 | 借方 | 贷方 | 凭证字 |
|:---|:---|:---|:---|:---:|
| **AP01** | 采购进项发票入账（可抵扣） | 原材料/库存商品 + 应交税费-进项税额 | 应付账款-XX供应商 | 转 |
| **AP02** | 采购进项发票入账（不可抵扣/普票） | 原材料/库存商品（含税） | 应付账款-XX供应商 | 转 |
| **AP03** | 费用类发票入账 | 管理费用/销售费用+应交税费-进项税额 | 应付账款-XX供应商 | 转 |
| **AP04** | 进项转出 | 原材料/管理费用等 | 应交税费-进项税额转出 | 转 |
| **AP05** | 红字冲回（全部红冲） | 应付账款-XX供应商 | 原材料/库存商品+应交税费-进项税额 | 转 |
| **AP06** | 加计抵减 | 应交税费-未交增值税 | 其他收益/营业外收入 | 转 |
| **AP07** | 进口关税/海关缴款书 | 原材料/库存商品+应交税费-进项税额 | 应付账款-XX供应商+应交税费-关税 | 转 |

#### AP01 详细分录

```
触发事件: AP.INVOICE.ACCOUNTING (采购进项-可抵扣)
触发时机: 进项发票入账确认时自动生成

借: 原材料-XX物料          amount_no_tax    (按明细行分摊)
    应交税费-应交增值税-进项税额  tax_amount
贷: 应付账款-XX供应商       amount_with_tax  (价税合计)

辅助核算:
- 原材料:    科目=物料, 供应商=XX
- 进项税额:  科目=税码(13%/9%/6%)
- 应付账款:  科目=供应商, 部门=采购部

示例:
借: 原材料-电子元器件      10,000.00
    应交税费-进项税(13%)    1,300.00
贷: 应付账款-深圳XX有限公司  11,300.00
```

#### AP03 详细分录（费用类）

```
触发事件: AP.INVOICE.EXPENSE (费用发票入账)
触发时机: 费用类进项发票入账确认时自动生成

借: 管理费用-办公费        amount_no_tax
    应交税费-进项税额       tax_amount
贷: 应付账款-XX供应商      amount_with_tax

辅助核算:
- 管理费用:  科目=部门+项目
- 应付账款:  科目=供应商
```

#### AP04 详细分录（进项转出）

```
触发事件: AP.INVOICE.TRANSFER_OUT (进项转出)
触发时机: 认证抵扣记录标记为进项转出时自动生成

借: 管理费用-福利费等       tax_amount  (转出税额计入费用)
贷: 应交税费-进项税额转出   tax_amount

辅助核算:
- 管理费用:  科目=部门+转出原因
- 进项税额转出: 科目=税码
```

#### AP05 详细分录（红字冲回）

```
触发事件: AP.INVOICE.RED_REVERSAL (红字冲回)
触发时机: 红字发票确认后自动冲回原蓝字凭证

借: 应付账款-XX供应商      amount_with_tax  (红字/负数)
贷: 原材料-XX物料          amount_no_tax    (红字/负数)
    应交税费-进项税额       tax_amount       (红字/负数)

辅助核算: 同原蓝字凭证方向
```

### 4.5 凭证模板JSON定义

```json
{
  "AP.INVOICE.ACCOUNTING": "AP01",
  "AP.INVOICE.NO_DEDUCT": "AP02",
  "AP.INVOICE.EXPENSE": "AP03",
  "AP.INVOICE.TRANSFER_OUT": "AP04",
  "AP.INVOICE.RED_REVERSAL": "AP05",
  "AP.INVOICE.SURCHARGE": "AP06",
  "AP.INVOICE.IMPORT": "AP07"
}
```

> ⚠️ 以上7个事件需同步注册到总账凭证引擎的 `VOUCHER_TEMPLATE_MAP` 中，引擎总模板数从31套增至 **38套**。

---

## 5. 核心业务流程

### 5.1 进项发票主流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       AP进项发票管理 — 主流程                             │
└─────────────────────────────────────────────────────────────────────────┘

 ①发票到达          ②录入/采集           ③查验联动          ④三单匹配
 ┌────────┐       ┌──────────┐       ┌──────────┐       ┌──────────┐
 │供应商   │──────▶│ 手工录入  │──────▶│ 自动触发  │──────▶│ PO+GR   │
 │开具发票 │       │ OCR识别  │       │ 发票查验  │       │ +Invoice │
 │纸质/电子│       │ 批量导入  │       │ (已有模块)│       │ 自动匹配 │
 │全电/XML │       │ 自动采集  │       │          │       │          │
 └────────┘       └──────────┘       └──────────┘       └──────────┘
                                                              │
                                            ┌─────────────────┼────────────────┐
                                            │ 完全匹配        │ 差异            │ 无PO
                                            ▼                 ▼                 ▼
                                       ⑤入账确认        ⑤'差异审批        ⑤''费用审批
                                       ┌──────────┐    ┌──────────┐     ┌──────────┐
                                       │ 应付挂账  │    │ 主管审批  │     │ 费用审批  │
                                       │ 税额分离  │    │ 差异说明  │     │ 部门确认  │
                                       │ 凭证生成  │    │ 审批通过  │     │ 凭证生成  │
                                       │ AP01/AP02│    │ →入账    │     │ AP03     │
                                       └────┬─────┘    └──────────┘     └──────────┘
                                            │
                                            ▼
                                       ⑥认证抵扣
                                       ┌──────────┐
                                       │ 勾选认证  │ ← 批量勾选/单张勾选
                                       │ 关联期间  │ ← 绑定增值税申报期
                                       │ 进项转出  │ ← 非应税/福利/损失
                                       │ 加计抵减  │ ← 特定行业
                                       └────┬─────┘
                                            │
                                    ┌───────┼───────┐
                                    ▼               ▼
                               ⑦实付付款        ⑧红字处理
                               ┌──────────┐    ┌──────────┐
                               │ 下游模块  │    │ 红字申请  │
                               │ 付款申请  │    │ 供应商确认│
                               │ P01~P08  │    │ 冲回凭证  │
                               └──────────┘    │ AP05     │
                                               └──────────┘
```

### 5.2 三单匹配详细流程

```
                    输入: 进项发票(已查验通过)
                              │
                              ▼
                    ┌──────────────────┐
                    │ ① 发票类别判断    │
                    │ 采购类/费用类/资产 │
                    └───────┬──────────┘
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
          采购类发票     费用类发票      资产类发票
              │             │              │
              ▼             ▼              ▼
     ┌──────────────┐  ┌──────────┐  ┌──────────────┐
     │ ② PO匹配     │  │ 豁免三单  │  │ 关联资产卡片  │
     │ 按发票号/供应商│  │ 匹配     │  │ 走资产入账   │
     │ /物料智能匹配 │  │          │  │              │
     └───────┬──────┘  └────┬─────┘  └──────┬───────┘
             │              │               │
             ▼              ▼               ▼
     ┌──────────────┐  费用审批流       资产审批流
     │ ③ GR匹配     │
     │ 匹配到PO后   │
     │ 查找对应GR   │
     └───────┬──────┘
             │
             ▼
     ┌──────────────┐     ┌──────────────────┐
     │ ④ 差异检查    │────▶│ 数量差异 ≤3%?    │
     │ 逐行对比      │     │ 单价差异 ≤2%?    │
     │ 发票 vs PO+GR │     │ 税额差异 ≤¥1?   │
     └───────┬──────┘     │ 合计金额一致?     │
             │            └──────────────────┘
             │                    │
      ┌──────┼────────────────────┼──────────────┐
      │      │ 全部通过            │ 部分超限       │ 合计不符
      ▼      ▼                    ▼               ▼
  完全匹配  自动通过           差异待审          拒绝入账
  match_    tolerance=         需主管审批        返回修改
  status=1  PASS              approval=1        
```

### 5.3 认证抵扣流程

```
┌───────────────────────────────────────────────────────────────────┐
│                    进项发票认证抵扣流程                              │
└───────────────────────────────────────────────────────────────────┘

  入账确认后                    认证期间内                     申报抵扣
  ┌───────────┐              ┌───────────────┐            ┌───────────┐
  │ 已入账     │──批量勾选──▶│ 勾选认证       │──申报期──▶│ 增值税申报 │
  │ 发票列表   │              │ 认证期间绑定   │            │ 进项抵扣   │
  └───────────┘              └───────┬───────┘            └───────────┘
                                     │
                          ┌──────────┼──────────┐
                          ▼          ▼          ▼
                      可抵扣     不可抵扣    进项转出
                      cert_type  cert_type  cert_type
                      =deduct   =no_deduct =transfer_out
                          │          │          │
                          ▼          ▼          ▼
                     直接抵扣    计入成本    生成AP04
                     更新余额    不生成凭证   凭证转出
```

---

## 6. 接口设计（RESTful API）

### 6.1 进项发票CRUD

```
POST   /api/v1/ap-invoice                     # 创建进项发票
GET    /api/v1/ap-invoice/{id}                 # 查询发票详情（含明细行）
PUT    /api/v1/ap-invoice/{id}                 # 更新发票（仅草稿态）
DELETE /api/v1/ap-invoice/{id}                 # 删除发票（仅草稿态）
GET    /api/v1/ap-invoice                      # 分页查询发票列表
```

### 6.2 查验联动

```
POST   /api/v1/ap-invoice/{id}/verify          # 触发发票查验
GET    /api/v1/ap-invoice/{id}/verify-result    # 获取查验结果
```

### 6.3 三单匹配

```
POST   /api/v1/ap-invoice/{id}/match           # 执行三单匹配
GET    /api/v1/ap-invoice/{id}/match-result     # 获取匹配结果
PUT    /api/v1/ap-invoice/{id}/match-approve    # 差异审批
POST   /api/v1/ap-invoice/{id}/match-exempt     # 豁免匹配(费用类)
```

### 6.4 入账确认

```
POST   /api/v1/ap-invoice/{id}/accounting       # 入账确认(触发凭证)
POST   /api/v1/ap-invoice/{id}/accounting-revoke # 撤回入账
```

### 6.5 认证抵扣

```
POST   /api/v1/ap-invoice/batch-certify         # 批量勾选认证
POST   /api/v1/ap-invoice/{id}/certify           # 单张认证
POST   /api/v1/ap-invoice/{id}/transfer-out      # 进项转出
POST   /api/v1/ap-invoice/{id}/no-deduct         # 标记不可抵扣
POST   /api/v1/ap-invoice/{id}/certify-revoke    # 撤销认证
```

### 6.6 红字处理

```
POST   /api/v1/ap-red-letter                    # 申请红字信息表
GET    /api/v1/ap-red-letter/{id}               # 查询红字信息表
PUT    /api/v1/ap-red-letter/{id}/confirm        # 供应商确认
PUT    /api/v1/ap-red-letter/{id}/red-invoice    # 回填红字发票号
```

### 6.7 台账与报表

```
GET    /api/v1/ap-invoice/ledger                # 进项发票台账
GET    /api/v1/ap-invoice/cert-stats            # 认证抵扣统计
GET    /api/v1/ap-invoice/match-diff-report     # 三单匹配差异报告
GET    /api/v1/ap-invoice/tax-detail            # 进项税额明细
GET    /api/v1/ap-invoice/aging                 # 应付账龄分析
GET    /api/v1/ap-invoice/monitor               # 异常发票监控看板
```

### 6.8 批量操作

```
POST   /api/v1/ap-invoice/batch-import          # 批量导入
POST   /api/v1/ap-invoice/batch-verify          # 批量查验
POST   /api/v1/ap-invoice/batch-match           # 批量匹配
POST   /api/v1/ap-invoice/batch-accounting       # 批量入账
```

---

## 7. 与现有模块的集成规范

### 7.1 与发票查验模块集成

```
本模块调用查验模块的接口:
POST /api/v1/invoice/verify

触发时机: 
- F1.1 手工录入后自动触发
- F1.2 OCR识别后自动触发
- F1.3 批量导入后批量触发

数据流转:
ap_invoice.verify_log_id → invoice_verification_log.id
查验结果自动回写:
  verify_status ← 查验模块的verify_status
  verify_log_id ← 查验记录ID

查验不通过的发票:
  verify_status=2或3 → 禁止进入三单匹配环节
  → 走异常池处理流程(复用查验模块的异常池)
```

### 7.2 与实付付款模块集成

```
本模块产出 → 实付付款模块消费:

入账确认后(ap_invoice.accounting_status=1):
  产生应付记录 → 可创建付款申请
  关联字段: ap_invoice.id → payment的source_ref_id
  
数据接口:
GET /api/v1/ap-invoice/{id}/payable-detail
  返回: 应付金额、供应商、到期日、已付金额、未付余额
  
实付付款核销后回调:
PUT /api/v1/ap-invoice/{id}/payment-status
  更新: 已付金额、付款状态
```

### 7.3 与总账凭证引擎集成

```
本模块触发凭证事件:

| 触发时机 | 事件编码 | 凭证模板 | 说明 |
|:---|:---|:---|:---|
| 入账确认(可抵扣) | AP.INVOICE.ACCOUNTING | AP01 | 采购+进项税 |
| 入账确认(不可抵扣) | AP.INVOICE.NO_DEDUCT | AP02 | 含税入成本 |
| 费用发票入账 | AP.INVOICE.EXPENSE | AP03 | 费用+进项税 |
| 进项转出 | AP.INVOICE.TRANSFER_OUT | AP04 | 税额转出 |
| 红字冲回 | AP.INVOICE.RED_REVERSAL | AP05 | 冲回原凭证 |
| 加计抵减 | AP.INVOICE.SURCHARGE | AP06 | 加计抵减额 |
| 进口发票入账 | AP.INVOICE.IMPORT | AP07 | 含关税 |

调用方式:
POST /api/v1/gl/voucher/auto-generate
  body: {
    "event_key": "AP.INVOICE.ACCOUNTING",
    "source_module": "AP",
    "source_bill_id": 10086,
    "source_bill_no": "API202604000001",
    "params": {
      "supplier_id": 100,
      "amount_no_tax": 10000.00,
      "tax_amount": 1300.00,
      "amount_with_tax": 11300.00,
      "account_code": "1403-01",
      "tax_code": "2221-01-01"
    }
  }
```

### 7.4 与税务管理模块集成

```
本模块产出 → 税务管理模块消费:

认证抵扣记录(ap_cert_record) → 增值税计算
  - cert_type=deduct → 可抵扣进项税额汇总
  - cert_type=transfer_out → 进项转出额汇总
  - cert_type=surcharge → 加计抵减额汇总

数据接口:
GET /api/v1/ap-invoice/cert-summary?period=2026-04
  返回: {
    "period": "2026-04",
    "total_deduct": 125000.00,
    "total_transfer_out": 5000.00,
    "total_surcharge": 12000.00,
    "invoice_count": 156,
    "cert_count": 148
  }
```

### 7.5 与海外发票管理模块集成

```
当 ap_invoice.invoice_region ≠ 'CN' 或为空时:
  → 走海外发票管理模块流程
  → 不走本模块的三单匹配和认证抵扣流程
  → 海外模块处理完后回写本模块状态

判断逻辑(在发票录入时):
  IF supplier.country_code IN ('VN','TH','PH','SA'):
      invoice_region = supplier.country_code
      → 路由到海外发票模块
  ELSE:
      invoice_region = 'CN'
      → 走本模块标准流程
```

---

## 8. 重复发票检测机制

### 8.1 检测规则

```
检测时机: 发票录入/导入时实时检测
检测维度: 发票代码 + 发票号码 + 发票金额(可选)

精确匹配:
  同一 invoice_code + invoice_no 已存在于系统中 → 拦截

模糊匹配(预警):
  同一供应商 + 相近金额(±5%) + 7天内 → 预警提示

跨模块去重:
  同时检查 invoice_verification_log 表
  避免查验环节和入账环节重复录入
```

### 8.2 检测结果处理

| 检测结果 | 系统行为 | 用户操作 |
|:---|:---|:---|
| 精确重复 | ❌ 拦截，禁止录入 | 查看已有发票详情 |
| 同供应商近金额 | ⚠️ 预警，可继续 | 确认非重复后继续 |
| 无重复 | ✅ 通过 | 正常流程 |

---

## 9. 前端交互原型

### 9.1 进项发票列表页

```
╔═══════════════════════════════════════════════════════════════════════╗
║  📥 AP进项发票管理                                                    ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  ┌─ 筛选条件 ───────────────────────────────────────────────────┐   ║
║  │ 发票号码[________] 供应商[▼全部▼] 票种[▼全部▼]               │   ║
║  │ 查验状态[▼全部▼]   匹配状态[▼全部▼] 入账状态[▼全部▼]          │   ║
║  │ 认证状态[▼全部▼]   期间[2026-04]~[2026-04]                   │   ║
║  │                                              [🔍查询] [↻重置] │   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                       ║
║  [+新建发票] [📷OCR录入] [📥批量导入] [✅批量勾选] [📤导出Excel]     ║
║                                                                       ║
║  ┌─ 发票列表 ───────────────────────────────────────────────────┐   ║
║  │ ☐ │发票号码    │供应商      │票种│价税合计│查验│匹配│入账│认证│   ║
║  │───┼───────────┼──────────┼───┼──────┼───┼───┼───┼───│   ║
║  │ ☐ │04138864   │深圳XX有限公司│专票│11300 │✅ │✅ │✅ │🔒 │   ║
║  │ ☐ │23456789   │广州YY科技   │普票│5,680 │✅ │⚠️ │⏳ │—  │   ║
║  │ ☐ │98765432   │北京ZZ贸易   │电专│45,000│⏳ │—  │—  │—  │   ║
║  │ ☐ │...         │...         │... │...   │...│...│...│...│   ║
║  └──────────────────────────────────────────────────────────────┘   ║
║                                                                       ║
║  📊 汇总: 总发票 256张 | 待查验 12 | 待匹配 8 | 待入账 15 | 可抵扣 ¥1.2M  ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### 9.2 三单匹配对比页

```
╔═══════════════════════════════════════════════════════════════════════╗
║  🔗 三单匹配 — 发票 04138864                                         ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  ┌─ 匹配结果: ✅ 完全匹配 ─────────────────────────────────────┐    ║
║  │                                                              │    ║
║  │  维度        │ 采购订单(PO)    │ 入库单(GR)    │ 发票(Invoice) │    ║
║  │ ────────────┼───────────────┼──────────────┼────────────── │    ║
║  │ 单号         │ PO2026040089   │ GR2026040156  │ 04138864      │    ║
║  │ 供应商       │ 深圳XX有限公司  │ 深圳XX有限公司 │ 深圳XX有限公司 │    ║
║  │ ────────────┼───────────────┼──────────────┼────────────── │    ║
║  │ 物料A 数量   │ 100            │ 100           │ 100           │    ║
║  │ 物料A 单价   │ 100.00         │ —             │ 100.00        │    ║
║  │ 物料A 金额   │ 10,000.00      │ —             │ 10,000.00     │    ║
║  │ ────────────┼───────────────┼──────────────┼────────────── │    ║
║  │ 税率         │ 13%            │ —             │ 13%           │    ║
║  │ 税额         │ 1,300.00       │ —             │ 1,300.00      │    ║
║  │ 价税合计     │ 11,300.00      │ —             │ 11,300.00     │    ║
║  │                                                              │    ║
║  │  差异: 数量 0% ✅  单价 0% ✅  税额 ¥0 ✅  合计一致 ✅        │    ║
║  └──────────────────────────────────────────────────────────────┘    ║
║                                                                       ║
║                      [✔ 确认入账]  [📋 查看明细]  [↩ 返回列表]        ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### 9.3 批量勾选认证页

```
╔═══════════════════════════════════════════════════════════════════════╗
║  ✅ 进项发票勾选认证 — 2026年04月                                     ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  认证期间: [2026-04]  申报截止: 2026-05-15                            ║
║                                                                       ║
║  ┌─ 可勾选发票 ────────────────────────────────────────────────┐    ║
║  │ ☐ │发票号码    │供应商        │票种│可抵扣税额 │入账日期      │    ║
║  │───┼───────────┼────────────┼───┼──────────┼────────────│    ║
║  │ ☑ │04138864   │深圳XX有限公司 │专票│ ¥1,300.00│ 2026-04-12  │    ║
║  │ ☑ │23456789   │广州YY科技    │专票│ ¥680.00 │ 2026-04-15  │    ║
║  │ ☐ │34567890   │上海WW实业    │电专│ ¥5,200.00│ 2026-04-18  │    ║
║  │ ☐ │...         │...          │... │...       │...          │    ║
║  └──────────────────────────────────────────────────────────────┘    ║
║                                                                       ║
║  ┌─ 汇总 ──────────────────────────────────────────────────────┐    ║
║  │ 已勾选: 2张  │ 可抵扣税额合计: ¥1,980.00                     │    ║
║  │ 期间累计已抵扣: ¥125,000.00  │ 本批抵扣后合计: ¥126,980.00   │    ║
║  └──────────────────────────────────────────────────────────────┘    ║
║                                                                       ║
║              [✅ 确认勾选]  [📤 导出认证清单]  [↩ 返回]                ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

## 10. 安全与风控

### 10.1 数据安全

| 措施 | 说明 |
|:---|:---|
| 发票影像加密存储 | 上传的发票图片/PDF加密存储，访问需鉴权 |
| 供应商税号脱敏 | 列表展示时中间4位脱敏 |
| 操作日志 | 所有关键操作(录入/入账/认证/红冲)留审计日志 |
| 金额防篡改 | 入账后金额字段只读，修改需走冲回流程 |

### 10.2 风控规则

| 规则 | 触发条件 | 处理 |
|:---|:---|:---|
| 重复发票拦截 | 同发票代码+号码 | 禁止录入 |
| 超期发票预警 | 开票日期超过90天未认证 | 预警提醒 |
| 大额发票审批 | 单张发票金额 > ¥100,000 | 需主管审批 |
| 税码异常 | 发票税率与PO约定税率不一致 | 拦截入账 |
| 供应商黑名单 | 供应商被标记为异常 | 拦截入账+通知 |

---

## 11. 实施路线图

| 阶段 | 内容 | 工期 | 交付物 | 依赖 |
|:---|:---|:---:|:---|:---|
| **Phase 1** | 发票录入+查验联动 | 2天 | 手工录入、OCR录入、查验自动触发、重复检测 |
| **Phase 2** | 三单匹配引擎 | 3天 | 自动匹配、容差规则、差异审批、费用豁免 |
| **Phase 3** | 入账确认+凭证 | 2天 | 应付挂账、税额分离、AP01~AP03凭证、入账撤回 |
| **Phase 4** | 认证抵扣 | 2天 | 勾选认证、期间绑定、进项转出AP04、加计抵减AP06 |
| **Phase 5** | 红字处理 | 1天 | 红字信息表、供应商确认、冲回凭证AP05 |
| **Phase 6** | 台账报表+集成 | 2天 | 进项台账、抵扣统计、与付款/税务模块联调 |

**总计**: 约 **12个工作日** 可完成MVP版本（Phase 1-4为核心）

---

## 附录A: 与金蝶云星空功能对标

| 金蝶云星空功能 | 本系统实现 | 说明 |
|:---|:---|:---|
| 采购发票录入 | ✅ F1 全渠道录入 | 手工/OCR/批量/自动采集 |
| 采购发票-发票查验 | ✅ F1.5 查验联动 | 复用已有查验模块 |
| 采购发票-三单匹配 | ✅ F2 自动匹配 | PO+GR+Invoice自动校验 |
| 采购发票-入账 | ✅ F3 入账确认 | 应付挂账+税额分离+凭证 |
| 进项发票-勾选认证 | ✅ F4.1 勾选认证 | 批量勾选+期间绑定 |
| 进项发票-进项转出 | ✅ F4.3 进项转出 | 多种转出原因+凭证 |
| 进项发票-加计抵减 | ✅ F4.5 加计抵减 | 10%/5%加计抵减 |
| 红字发票管理 | ✅ F5 红字处理 | 红字信息表+冲回 |
| 采购发票台账 | ✅ F6.1 发票台账 | 全量台账+多维筛选 |
| 进项税额统计 | ✅ F6.2 抵扣统计 | 按期间/税码/供应商 |

## 附录B: 术语表

| 术语 | 英文 | 说明 |
|:---|:---|:---|
| 进项发票 | Purchase Invoice / Input VAT Invoice | 供应商开给本公司的增值税发票 |
| 三单匹配 | Three-way Match | 采购订单(PO)+入库单(GR)+发票(Invoice)交叉校验 |
| 勾选认证 | Certification Selection | 在增值税发票综合服务平台勾选确认抵扣 |
| 进项转出 | Input VAT Transfer Out | 不可抵扣的进项税额从抵扣中转出 |
| 红字信息表 | Red-letter Invoice Notice | 申请开具红字发票前需填写的信息表 |
| 加计抵减 | Additional Deduction | 特定行业可额外抵减进项税额的10%或5% |
| 全电发票 | Fully Electronic Invoice | 全面数字化的电子发票，无纸质发票代码 |
| 容差 | Tolerance | 三单匹配允许的合理差异范围 |
| P2P | Procure-to-Pay | 采购到付款全流程 |
| 留抵税额 | Uncredited Input VAT | 当期未抵扣完可结转下期的进项税额 |

---

> **文档维护**: Dev_Dept 团队  
> **最后更新**: 2026-04-20  
> **下次评审**: AP进项发票+AR销项发票联调后更新集成细节
