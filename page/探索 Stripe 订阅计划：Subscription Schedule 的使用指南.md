# 探索 Stripe 订阅计划：Subscription Schedule 的使用指南

在 Stripe 中，更新 Subscription 是一项常见的操作，其 API 使用起来也比较简单。然而，在某些情况下，由于账单日期或产品订阅类型的限制，我们无法立即更新 Subscription。这时，就需要借助 **Subscription Schedule** 功能，在未来某个时间点创建或更新订阅计划。

## Subscription Schedule 的强大之处

Subscription Schedule 是一个功能强大的工具，但在我个人的开发体验中，初次使用时也遇到了一些困惑。尽管 Stripe 提供了详细的 [API 文档](https://docs.stripe.com/api/subscription_schedules) 和 [用例](https://docs.stripe.com/billing/subscriptions/subscription-schedules)，但由于 API 的不断演进，一些新对象被引入，某些字段被弃用，而最新文档并未详细说明这些变化。此外，用例与 API 文档之间也存在一些细微的差异，比如 `Price` 替代了 `Plan`，或者在 Phase Items 中弃用的 `coupon` 字段。

在本文中，我将分享一个实际案例——如何在未来的某个时间点安排更新单个 Subscription Item 的 `quantity`，同时保持原有的 `coupon` 有效期。此外，我还会探讨在这个过程中遇到的挑战及解决方案。

## 场景设定

假设我们有一个 Subscription，其中包含一个 Subscription Item——SaaS Member Fee，单价为 $10，数量为 5，同时附带一个有效期三个月的免费 coupon。我们需要在 9 月 1 日将其数量更新为 10，实现的效果如下图所示：

![Subscription Schedule](https://bbtdd.com/img/913331570.webp)

**TL;DR**：如果不想看细节，可以直接跳到 [解决方案](#解决方案) 部分查看代码。

## Stripe 的处理方式

尽管有 [API 文档](https://docs.stripe.com/api/subscription_schedules) 和 [用例](https://docs.stripe.com/billing/subscriptions/subscription-schedules) 作为参考，初次对接这个 API 时还是让人感到困惑。我知道需要创建一个 Subscription Schedule，但具体参数如何传递却让人犯难。

针对创建 Schedule 的 API `POST /v1/subscription_schedules`，文档中给出的示例如下：

ruby
Stripe::SubscriptionSchedule.create({
  customer: 'cus_NcI8FsMbh0OeFs',
  start_date: 1680716828,
  end_behavior: 'release',
  phases: [
    {
      items: [
        {
          price: 'price_1Mr3YcLkdIwHu7ixYCFhXHNb',
          quantity: 1,
        },
      ],
      iterations: 12,
    },
  ],
})


然而，这个示例并不适用于我的案例，因为我需要修改一个已经存在的 Subscription。那么，我至少应该传递 Subscription ID 吧？查阅文档后，发现有一个参数 `from_subscription`，它接受一个现有的 Subscription ID。文档中写道：

> 当使用此参数时，其他参数（如 phase 值）无法设置。要创建具有其他修改的 Subscription Schedule，建议进行两次单独的 API 调用。

这意味着，如果设置了 `from_subscription`，其他参数就无法设置，而是需要进行两次 API 调用。那么，具体该怎么做呢？

### Stripe 的实现方式

我们知道，在 Stripe 中手动进行 Schedule Update 时，也是通过调用 Stripe 的 API 实现的。打开 Chrome 的开发者工具，我们可以看到 Stripe 如何处理这个操作。Stripe 进行了两次 API 调用：

**第一次 API 请求：**

json
POST /v1/subscription_schedules

Request Body:
{
  from_subscription: "sub_1PeCNNLiQSlGYPd5Wpmvb1n6"
}

Response:
{
  id: "sub_sched_1PeH6LLiQSlGYPd599aOVdoW",
  # ... 其他字段省略
}


**第二次 API 请求：**

json
POST /v1/subscription_schedules/sub_sched_1PeH6LLiQSlGYPd599aOVdoW

Request Body:
{
  proration_behavior: "none",
  phases: [
    {
      start_date: 1721378477,
      end_date: "now",
      iterations: "",
      discounts: [
        {
          coupon: "EDHtWtdk"
        }
      ],
      default_tax_rates: "",
      automatic_tax: {
        enabled: false
      },
      items: [
        {
          price: "price_1PeCLRLiQSlGYPd50cFz00Na",
          quantity: 5
        }
      ],
      collection_method: "charge_automatically"
    },
    {
      start_date: "now",
      end_date: 1725217200,
      discounts: [
        {
          coupon: "EDHtWtdk"
        }
      ],
      default_tax_rates: "",
      items: [
        {
          quantity: 5,
          tax_rates: "",
          price: "price_1PeCLRLiQSlGYPd50cFz00Na"
        }
      ]
    },
    {
      start_date: 1725217200,
      discounts: [
        {
          coupon: "EDHtWtdk"
        }
      ],
      default_tax_rates: "",
      items: [
        {
          quantity: 10,
          tax_rates: "",
          price: "price_1PeCLRLiQSlGYPd50cFz00Na"
        }
      ],
      proration_behavior: "none",
      collection_method: "charge_automatically",
      invoice_settings: {
        description: "Thank you for your business!"
      }
    }
  ],
  end_behavior: "release"
}


第一次 API 请求用于创建一个 Subscription Schedule，只传递 `from_subscription` 参数；在获取到 Schedule ID 之后，第二次 API 请求对该 Schedule 进行修改，并传入一系列参数。这涉及到几个关键概念，如 **Phase**、**Proration** 等。

### 什么是 Phase？

Phase 代表 Subscription Schedule 中的一段特定时间，在这段时间内会应用我们所指定的 Price、Charge、Tax 等参数。也就是说，我们可以指定一组 Phases，每个 Phase 都有自己的计费参数。

一个 Phase 的关键参数包括：

1. **start_date** 和 **end_date**：标识每个 Phase 的开始日期和结束日期。
2. **price** 和 **quantity**：可以在每个 Phase 中为订阅项目指定不同的价格和数量。
3. **discounts**：在该 Phase 中使用的特定折扣和优惠券。
4. **tax_rates** 和 **automatic_tax**：税率相关参数。
5. **collection_method**：计费方式。

从上述的第二个请求来看，Stripe 设定了 3 个 Phases：

1. **Phase#0**：start_date 为原先 Subscription 的起始时间，end_date 为 `now`，同时 discounts 包含了原先的 coupon 代码，items 中有一个对象，包含 price 和 quantity 参数，值与当前 Subscription 一致。这个 Phase 的意义在于立即终止当前的订阅。
2. **Phase#1**：start_date 为 `now`，end_date 为预期的时间，items 与 Phase#0 保持一致。
3. **Phase#2**：start_date 为预期的时间，end_date 为空，items[0] 中的 quantity 更新为 10。

### 什么是 Proration？

当我们对 Subscription 进行更改时，默认会自动处理按比例折算，即 **Proration**。我们可以使用 `proration_behavior` 来指定其行为，这个参数有以下三个值：

- **create_prorations**：默认值，在计费周期发生变化时创建按比例调整。
- **none**：计费周期发生变化时，不进行 Proration。
- **always_invoice**：立即创建按比例调整账单。

### 什么是 end_behavior？

`end_behavior` 指的是 Schedule 结束（即到达最后一个 Phase 的 end_date）时，相关联的 Subscription 应执行的操作。`end_behavior` 的枚举值为 `release` 或 `cancel`，它们的区别在于：

- **release**：默认值，当 Schedule 结束时，Subscription 保持那一刻的状态继续存续。
- **cancel**：当 Schedule 结束时，Subscription 直接结束（取消）。

## 解决方案

给定一个 Subscription，我们希望在某个时间点修改其 item 的 quantity。我们只需使用两个 Phases，代码如下：

ruby
schedule = Stripe::SubscriptionSchedule.create from_subscription: subscription.id

phase0 = schedule.phases[0]
phase0_item = phase0.items[0]

original_quantity = phase0_item.quantity
price_id = phase0_item.price
discounts = phase0.discounts.collect(&:to_hash)

phases = [
  # Phase 0 - 修改现有订阅阶段，结束日期为 `time`
  {
    start_date: phase0.start_date,
    end_date: time.to_i,
    proration_behavior: 'none',
    discounts: discounts,
    items: [
      {
        price: price_id,
        quantity: original_quantity,
      }
    ]
  },
  # Phase 1 - 设置一个新的订阅阶段，开始日期为 `time`，数量为 `quantity`
  {
    start_date: time.to_i,
    end_date: time.to_i + 60,
    proration_behavior: 'none',
    discounts: discounts,
    items: [
      {
        price: price_id,
        quantity: quantity,
      }
    ]
  }
]

Stripe::SubscriptionSchedule.update schedule.id, phases: phases, proration_behavior: 'none'


### 关键点说明：

1. **discounts** 参数：更新 Subscription Schedule 时，如果没有明确传递 `discounts` 参数，现有 coupon 将被取消。要保留任何当前的 discount 或 coupon，需要将它们包含在更新的 Phase 中。
2. **item 对象中的 coupon 参数**：已被弃用，取而代之的是在外层传入 discounts 数组。
3. **设置结束日期**：最好为最后一个阶段设置一个结束日期，以确保 Schedule 自动 release，避免触及只能有一个 active Schedule 的限制。

## 结语

利用 Stripe 的 Subscription Schedule 功能可以大大简化订阅管理过程。虽然初次使用时可能会遇到一些困惑，但通过仔细阅读文档和实践，我们可以充分发挥这个强大工具的潜力。希望本文对大家理解和使用 Subscription Schedule 有所帮助。

👉 [WildCard | 一分钟注册，轻松订阅海外线上服务](https://bbtdd.com/WildCard)

---