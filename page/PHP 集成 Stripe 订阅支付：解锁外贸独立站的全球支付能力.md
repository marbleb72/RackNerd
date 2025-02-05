# PHP 集成 Stripe 订阅支付：解锁外贸独立站的全球支付能力

Stripe 是全球知名的支付处理平台，支持多种支付方式和多个国家/地区的货币。通过集成 Stripe 订阅支付，独立站可以接受来自全球的订单，并实现便捷的跨境支付。本教程将详细讲解如何使用 PHP 集成 Stripe 订阅支付，帮助外贸独立站轻松实现会员订阅功能。

## Stripe 订阅 API 简介

Stripe **订阅 API** 提供了在网站上集成定期付款的简单方法。通过 Stripe 支付网关，开发者可以轻松将定期付款与计划和订阅 API 集成。Stripe 订阅不仅支持全球多种信用卡支付（如 Visa、Mastercard、American Express、Discover、JCB、Diners Club、UnionPay 等），还提供了快速有效的支付流程，用户无需离开网站即可完成支付。

在本教程中，我们将通过 PHP 实现以下功能：

- 创建 HTML 表单以选择订阅计划并提供信用卡信息。
- 使用 Stripe JS 库将 Stripe 卡元素附加到 HTML 表单。
- 使用 Stripe API 安全传输卡详细信息、验证并创建订阅。
- 使用 Stripe API 检索 PaymentIntent 和订阅信息。
- 通过 3D 安全身份验证（SCA）确认卡付款。
- 验证卡并使用 Stripe API 创建订阅计划。
- 将交易数据和订阅详细信息存储在数据库中，并显示付款状态。

## 文件结构

在开始之前，我们先看一下项目文件的目录结构：


stripe_subscription_payment_php/
├── config.php          # 存放 Stripe API 和数据库配置
├── dbConnect.php       # 用于 PHP 和 MySQL 连接数据库
├── index.php           # Stripe 订阅表格
├── payment_init.php    # 处理订阅付款
├── payment-status.php  # 付款状态
├── stripe-php/         # Stripe PHP 库
├── js/
│   └── checkout.js     # 订阅结帐处理程序脚本
└── css/
    └── style.css       # 页面样式


## Stripe 测试 API 密钥

在 Stripe 订阅支付上线之前，需要测试 API 密钥以确保订阅流程正常。以下是获取测试 API 密钥的步骤：

1. 登录 [Stripe 账户](https://dashboard.stripe.com/) 并导航至 **开发人员 » API 密钥** 页面。
2. 在 **测试数据** 块中，您将看到 API 密钥（可发布密钥和秘密密钥）列在标准密钥部分下。点击 **展示测试密钥** 按钮以显示密钥。

请收集 **可发布密钥** 和 **秘密密钥** 以供后续使用。

## 创建数据库表

我们需要在数据库中创建三个表：`plans`、`users` 和 `user_subscriptions`。以下是创建表的 SQL 语句：

sql
CREATE TABLE `user_subscriptions` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) NOT NULL DEFAULT 0 COMMENT 'users table 的外键',
  `plan_id` int(5) DEFAULT NULL COMMENT 'plans table 的外键',
  `payment_method` enum('stripe') COLLATE utf8_unicode_ci NOT NULL DEFAULT 'stripe',
  `stripe_subscription_id` varchar(50) COLLATE utf8_unicode_ci NOT NULL,
  `stripe_customer_id` varchar(50) COLLATE utf8_unicode_ci NOT NULL,
  `stripe_payment_intent_id` varchar(50) COLLATE utf8_unicode_ci NOT NULL,
  `paid_amount` float(10,2) NOT NULL,
  `paid_amount_currency` varchar(10) COLLATE utf8_unicode_ci NOT NULL,
  `plan_interval` varchar(10) COLLATE utf8_unicode_ci NOT NULL,
  `plan_interval_count` tinyint(2) NOT NULL DEFAULT 1,
  `customer_name` varchar(50) COLLATE utf8_unicode_ci DEFAULT NULL,
  `customer_email` varchar(50) COLLATE utf8_unicode_ci DEFAULT NULL,
  `created` datetime NOT NULL,
  `plan_period_start` datetime DEFAULT NULL,
  `plan_period_end` datetime DEFAULT NULL,
  `status` varchar(50) COLLATE utf8_unicode_ci NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;


接下来，我们需要在 `plans` 表中插入一些订阅计划数据：

sql
INSERT INTO `plans` (`id`, `name`, `price`, `interval`) VALUES 
(1, '每周订阅', 25.00, '周'),
(2, '每月订阅', 75.00, '月'),
(3, '年度订阅', 799.00, '年');


## Stripe API 和数据库配置 (config.php)

在 `config.php` 文件中，我们定义了 Stripe API 和数据库的配置常量：

php
<?php
// Stripe API 配置
define('STRIPE_API_KEY', 'Your_API_Secret_key');
define('STRIPE_PUBLISHABLE_KEY', 'Your_API_Publishable_key');
define('STRIPE_CURRENCY', '$');

// 数据库配置
define('DB_HOST', 'MySQL_Database_Host');
define('DB_USERNAME', 'MySQL_Database_Username');
define('DB_PASSWORD', 'MySQL_Database_Password');
define('DB_NAME', 'MySQL_Database_Name');
?>


## 数据库连接 (dbConnect.php)

`dbConnect.php` 文件用于连接数据库：

php
<?php
// 连接数据库
$db = new mysqli(DB_HOST, DB_USERNAME, DB_PASSWORD, DB_NAME);

// 连接失败时显示错误
if ($db->connect_errno) {
    printf("Connect failed: %s\n", $db->connect_error);
    exit();
}
?>


## Stripe 订阅表格 (index.php)

`index.php` 文件包含了订阅表单和处理逻辑：

php
<?php
// 包含配置文件
require_once 'config.php';
// 包含数据库连接文件
include_once 'dbConnect.php';
// 从数据库获取计划
$sqlQ = "SELECT * FROM plans";
$stmt = $db->prepare($sqlQ);
$stmt->execute();
$stmt->store_result();
?>


## 启用 Stripe 支付网关

在测试完成后，您可以启用 Stripe 支付网关。以下是启用步骤：

1. 登录 [Stripe 账户](https://dashboard.stripe.com/) 并导航至 **开发人员 » API 密钥** 页面。
2. 从 **实时数据** 部分获取 API 密钥（可发布密钥和秘密密钥）。
3. 在 `config.php` 文件中，将测试 API 密钥替换为实时 API 密钥。

php
define('STRIPE_API_KEY', 'Insert_Stripe_Live_API_Secret_Key_HERE');
define('STRIPE_PUBLISHABLE_KEY', 'Insert_Stripe_Live_API_Publishable_Key_HERE');


## 总结

Stripe 订阅支付 API 是在线接受订阅付款的便捷选择。通过 PHP 和 Stripe API，您可以轻松将定期计费功能添加到 Web 应用程序中。本教程的示例脚本还集成了 **3D 安全身份验证**（SCA）功能，确保支付流程的安全性。您可以根据业务需求进一步扩展此脚本。

👉 [WildCard | 一分钟注册，轻松订阅海外线上服务](https://bbtdd.com/WildCard)