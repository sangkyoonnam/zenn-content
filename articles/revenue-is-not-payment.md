---
title: 'お金を受け取った日が売上日ではない'
emoji: '🔧'
type: 'tech'
topics: ['settlement', 'revenuerecognition', 'saas', 'kotlin']
published: false
---

> この記事は [Namsang Labs 技術ブログ](https://blog.namsang.co/ja/posts/revenue-is-not-payment/) からの転載です。

四半期決算ミーティングだった。会計法人の担当者が画面を共有し、B社の行で止まった。「ここ、1月の売上が300万ウォンで計上されてますが、合ってますか？」合っていると答えると、担当者は首を振った。「購読の開始日が1月20日ですよね。1月に実際にサービスを提供したのは12日間ですから、その分だけが売上です。19万ウォンほどですね」

B社は1月20日に6ヶ月分の購読料300万ウォンを決済した。システムに決済通知が届き、ダッシュボードの1月売上が300万ウォン上がった。決済された金額がそのまま売上だと思っていたが、会計ではそうではなかった。購読期間は1月20日から7月19日までで、そのうち1月のサービスは12日間、その分だけが1月の売上として認識される。およそ19万ウォンだ。

残りの281万ウォンは口座に入っているが、残りの購読期間に提供すべきサービスの対価だ。これは繰延収益（deferred revenue）という負債として計上される。

## 現金主義と発生主義

売上をいつ認識するかについて、会計には2つの方式がある。現金主義と発生主義だ。現金主義はお金が入った時点で売上を認識する。直感に近く、コードに移すのも簡単だ。PG社のwebhookが届いたら`payment_completed_at`にタイムスタンプを刻み、`amount`に金額を入れる。決済イベントが起きた瞬間に売上として認識される。

```kotlin
// 現金主義：決済時点で全額を売上計上
fun onPaymentCompleted(payment: Payment) {
    revenueRepository.save(
        Revenue(
            amount = payment.amount,         // 3,000,000ウォン全額
            recognizedAt = payment.completedAt // 決済時点
        )
    )
}
```

発生主義は違う。サービスを提供した時点で売上を認識する。お金を受け取った時点ではない。非同期処理に例えるなら、リクエストを受け取った時点ではなく処理が進んだ分だけ記録する方式だ。決済はリクエストの時点、売上は進行の時点だ。

K-IFRS第1115号(韓国)もASC 606(米国)もこの原則に従う。法人であれば、売上処理に発生主義が強制される。

```kotlin
// 発生主義：サービス提供期間にわたって分割計上
fun onPaymentCompleted(payment: Payment, subscription: Subscription) {
    // 決済金額は繰延収益（負債）として計上する
    deferredRevenueRepository.save(
        DeferredRevenue(
            amount = payment.amount,         // 3,000,000ウォン
            serviceStart = subscription.startDate,
            serviceEnd = subscription.endDate
        )
    )
    // 毎月、サービス提供分だけ繰延収益 → 売上に振り替える（後段で具体化）
}
```

## 300万ウォンのうち1月分はいくらか

B社の購読条件はこうだ。

- 契約金額：3,000,000ウォン（月500,000ウォン × 6ヶ月）
- 購読開始：2026-01-20
- 購読終了：2026-07-19
- 1月のサービス日数：12日、1月20日から31日まで

購読開始日が1月1日だったなら単純だった。1月の売上は月額購読料500,000ウォンそのままだ。実際の開始日は1月20日で、1月にサービスを提供したのは12日間だけだ。1ヶ月分をまるごと認識することはできないので、12日分だけを切り出さなければならない。日割り計算（pro-rata）が必要になる地点だ。方法は2つある。

**1. 月ごとの日数で割る**：月額購読料をその月の実際の日数で割り、サービス提供日を掛ける。毎月の購読料がその月の中だけで配分される。

```
1月の売上 = 500,000 / 31 × 12 = 193,548.38...
```

**2. 契約期間で平準化する**：契約総額を全体のサービス日数で割って日単価を先に求め、それを各月のサービス日数に掛ける。契約全体を1つの履行義務（performance obligation）として見る方式だ。

```
// 全体のサービス日数：2026-01-20 ~ 2026-07-19 = 181日
1月の売上 = 3,000,000 / 181 × 12 = 198,895.02...
```

193,548ウォンと198,895ウォン。同じ契約で5,000ウォン以上の差が出る。どちらが正しいかをコードは決めない。会計ポリシーが決める。

## 何日で割るか

月額購読料が請求と請求書の基本単位となる韓国の実務では、1番の方式がより一般的だ。月ごとの日数で割ると、月によって1日単価が変わる。1月は31日で2月は28日だからだ。

```kotlin
val dailyJanuary = BigDecimal("500000")
    .divide(BigDecimal("31"), 10, RoundingMode.HALF_UP)
// 16,129.03...

val dailyFebruary = BigDecimal("500000")
    .divide(BigDecimal("28"), 10, RoundingMode.HALF_UP)
// 17,857.14...
```

同じ料金プランなのに、2月の1日単価は1,700ウォン以上高い。この差を扱うポリシーは3つだ。

- 実日数基準。その月の実際の日数である28、29、30、31で割る。最も精密で、監査実務で好まれる。月ごとに1日単価が変わる。
- 30日固定基準。すべての月を30日とみなす。金融業界でよく使う。1日単価が一定で運用が楽だ。1月は1日少なく、2月は2日多く認識する。
- 365日基準。年間購読料を365で割って日単価を求める。月額購読料という概念を使わない。うるう年なら366だ。この記事の6ヶ月契約には使わない。

```kotlin
enum class DayCountPolicy {
    ACTUAL,      // 当該月の実日数
    FIXED_30     // すべての月を30日とみなす
}

fun daysInMonth(yearMonth: YearMonth, policy: DayCountPolicy): BigDecimal =
    when (policy) {
        DayCountPolicy.ACTUAL -> BigDecimal(yearMonth.lengthOfMonth())
        DayCountPolicy.FIXED_30 -> BigDecimal("30")
    }
```

年間や複数年の購読が標準のグローバルSaaSでは、2番の方式を好む傾向がある。契約全体を1つの履行義務として見て期間にわたって平準化する形を、ASC 606とIFRS 15が自然に受け入れるからだ。契約期間全体に配分するとしても分母を何にするかは別に決めなければならず、同じ契約でも基準が変われば日単価が変わる。

- 契約日数そのまま。2026-01-20から2026-07-19まで181日。実際のサービス期間をそのまま分母に使う。
- 30日固定換算。6ヶ月 × 30 = 180日。平準化の中にも月単位の抽象を持ち込む。

```
// 契約日数そのまま
1月の売上 = 3,000,000 / 181 × 12 = 198,895.02...
```

分母を180日に変えるだけでは30日換算の方式は完成しない。1月20日から31日までを何日と見るか、月末をどう換算するかまで決めなければならず、その規則なしに実日数の181日をそのまま掛けると、6ヶ月の合計が300万ウォンを超える。だからここでは、契約日数181日を使う計算だけを完結した例として置く。同じ契約、同じ12日なのに、分母1つで1月の数字が変わるという点はそのままだ。

## 発生時点はどこで区切るか

売上の金額を決めたら、その売上をどの月に帰属させるかが残る。境界は0時という一点のように見えるが、その点をコードでどう閉じるかによって、どの月にも属さない件が生まれる。

1月31日 23:59:59に発生した売上は1月だ。1秒後の2月1日 00:00:00なら2月だ。当たり前のようでいて、コードでこの境界を捉える方法は厄介だ。JavaとKotlinの`LocalDateTime`はナノ秒、Pythonの`datetime`はマイクロ秒、JavaScriptの`Date`はミリ秒まで表現する。23:59:59の後にさらに桁があるという意味で、境界を秒単位で閉じてしまうと、その桁に置かれた件がどの月にも入らない。その桁が実際に保存されるかどうかは、言語ではなくカラムが決める。MySQLの`DATETIME`はデフォルトの精度が0なので、小数秒をデフォルトでは丸めて保存し、設定によっては切り捨てる。丸めなら23:59:59.5が翌月の0時に繰り越されることもある。`DATETIME(6)`まで指定して初めてマイクロ秒が残り、PostgreSQLの`timestamp`はデフォルトがマイクロ秒だ。

```kotlin
// 1月分の売上を集計
val start = LocalDateTime.of(2026, 1, 1, 0, 0, 0)
val end = LocalDateTime.of(2026, 1, 31, 23, 59, 59)

val januaryRecords = revenueRepository
    .findByRecognizedAtBetween(start, end)
```

23:59:59.500に発生したレコードはどこへ行くのか。23:59:59までで照会すると、それ以降のミリ秒単位のレコードが漏れる。2月1日 00:00:00から照会してもこのレコードは引っかからない。どの月にも属さない売上が生まれる。

```kotlin
// 半開区間（half-open interval）
val start = LocalDateTime.of(2026, 1, 1, 0, 0, 0)
val end = LocalDateTime.of(2026, 2, 1, 0, 0, 0)

val januaryRecords = revenueRepository
    .findByRecognizedAtGreaterThanEqualAndRecognizedAtLessThan(start, end)
// >= 1月1日 00:00:00, < 2月1日 00:00:00
```

始まりは含み、終わりは含まない。こう取れば境界で漏れも重複も起きない。会計ではこの境界処理をカットオフと呼ぶ。

タイムゾーンも一緒に見なければならない。DBがUTCを使っていても、請求書は事業所のタイムゾーン、韓国の事業者ならKST基準だ。1月31日 午後11時30分 KSTに発生したレコードはUTCでは1月31日 14:30で、どちらで見ても1月だ。2月1日 午前3時 KSTはUTCでは1月31日 18:00だ。UTCで集計するとこのレコードが1月に入り、帳簿とずれる。カットオフはKST基準で切らなければならない。

カットオフが必ずしも0時である必要はない。営業が0時を越える飲食店やコンビニのような業種では、締めの時刻を未明にずらす方が自然だ。サービスによっては、精算日の締めを未明まで遅らせるオプションを提供することもある。

## 決済テーブル1つでは足りない

分母の日数とカットオフを決めても、決済テーブルには誰がいつ、いくら払ったかしかない。この金額のうち今月認識する金額はいくらかに答えるには、別のテーブルが必要だ。

```kotlin
// 決済：キャッシュフロー
data class Payment(
    val id: Long,
    val customerId: Long,
    val amount: BigDecimal,       // 決済金額：3,000,000
    val completedAt: Instant      // 決済時点
)

// 売上認識スケジュール：発生主義基準
data class RevenueSchedule(
    val paymentId: Long,
    val yearMonth: YearMonth,        // 帰属月：2026-01
    val recognizedAmount: BigDecimal, // 認識売上：193,548
    val deferredAmount: BigDecimal,  // 繰延残高：2,806,452
    val serviceDays: Int,            // サービス日数：12
    val totalDaysInMonth: Int        // 当該月の日数：31
)
```

決済が発生したら、購読期間全体にわたって認識スケジュールを作る。B社の場合、1月から7月まで7件のレコードが作られる。月ごとの日数の方式なら、1月は193,548ウォン、2月から6月までは各500,000ウォン、7月は残りの306,452ウォンだ。月末ごとにその月の認識売上を合計すればその月の売上になり、7件の合計が契約金額と合うかを検算する。

[丸め](https://blog.namsang.co/ja/posts/rounding-is-policy/)で0.5の行方がそうだったように、決済金額のうち今月の売上がいくらかもコードではなくポリシーが決める。日割り計算の分母、売上帰属のカットオフ、その結果を収めるテーブルまで、すべての段階の決定を会計ポリシーが下した。売上認識はポリシーの累積だ。

残った問いが2つある。毎月小数点を切り捨てて6ヶ月分の認識売上の合計が300万ウォンに届かなければ、どう合わせるのか。そして3ヶ月目にB社が解約を求めたら、まだ認識していない繰延収益はどうなるのか。

## 参考資料

**会計基準（一次出典）**

- [FASB ASC 606、Revenue from Contracts with Customers](https://www.fasb.org/page/PageContent?pageId=/projects/recentlycompleted/revenue-recognition-summary.html): 米国GAAPの売上認識基準だ。
- [IFRS 15、Revenue from Contracts with Customers](https://www.ifrs.org/issued-standards/list-of-standards/ifrs-15-revenue-from-contracts-with-customers/): 国際会計基準の売上認識基準で、K-IFRS第1115号の母体だ。
- [K-IFRS第1115号、顧客との契約から生じる収益](https://www.kasb.or.kr/): 韓国採択国際会計基準の収益認識基準書だ。

**実務ガイド**

- [PwC、Revenue Recognition: An IFRS 15 Guide for Software and SaaS Sellers 2022](https://viewpoint.pwc.com/content/dam/pwc-madison/ditaroot/gx/en/pwc/industry/software/revenue-recognition-an-ifrs-15-guide-for-software-and-saas-sellers/assets/Revenue_Recognition_An_IFRS_15_Guide_for_Software_and_SaaS_sellers_2022.pdf): SaaSの売上認識にIFRS 15を適用する方法を案内している。
- [The SaaS CFO、Guide to SaaS Revenue Recognition and Deferred Revenue](https://www.thesaascfo.com/deferred-revenue-saas/): SaaSの繰延収益の構造と、日割り認識と平準化認識の違いを説明している。
- [Chargebee、Ultimate Guide to SaaS Revenue Recognition](https://www.chargebee.com/resources/guides/saas-revenue-recognition-guide/): ASC 606の5ステップの収益認識と日割り計算の例を示している。

---

*広告やCRMからAIエージェントまで、大規模なプラットフォームを作り、率いてきた経験を [Namsang Labs 技術ブログ](https://blog.namsang.co/ja/?utm_source=zenn&utm_medium=crosspost) に書いています。*
