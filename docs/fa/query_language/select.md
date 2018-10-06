<div dir="rtl">

# سینتکس query های SELECT

دستور `SELECT` برای بازیابی اطلاعات مورد استفاده قرار می گیرد.

</div>

```sql
SELECT [DISTINCT] expr_list
    [FROM [db.]table | (subquery) | table_function] [FINAL]
    [SAMPLE sample_coeff]
    [ARRAY JOIN ...]
    [GLOBAL] ANY|ALL INNER|LEFT JOIN (subquery)|table USING columns_list
    [PREWHERE expr]
    [WHERE expr]
    [GROUP BY expr_list] [WITH TOTALS]
    [HAVING expr]
    [ORDER BY expr_list]
    [LIMIT [n, ]m]
    [UNION ALL ...]
    [INTO OUTFILE filename]
    [FORMAT format]
    [LIMIT n BY columns]
```

<div dir="rtl">

تمامی موارد بالا اختیاری هستند، به جز ستون هایی که بعد از دستور SELECT مشخص می شوند. موارد زیر تقریبا به همان ترتیب query execution conveyor شرح داده می شوند.

اگر query بدون `DISTINCT`، `GROUP BY` و `ORDER BY` و همچنین بدون `IN` و `JOIN` و subquery باشد، query کاملا به صورت stream پردازش شده و مقدار O(1) از رم را مصرف می کند. در غیر این صورت، query ممکن است در صورت محدود نشدنِ موارد: `max_memory_usage`, `max_rows_to_group_by`, `max_rows_to_sort`, `max_rows_in_distinct`, `max_bytes_in_distinct`, `max_rows_in_set`, `max_bytes_in_set`, `max_rows_in_join`, `max_bytes_in_join`, `max_bytes_before_external_sort`, `max_bytes_before_external_group_by` مقدار زیادی از RAM را مصرف کند. برای اطلاعات بیشتر بخش "تنظیمات" مشاهده کنید. امکان استفاده از external sorting ( ذخیره جدول موقت روی دیسک) و external aggregation وجود دارد.

### دستور FROM

اگر دستور FROM حذف شود، داده ها از جدول `system.one` خوانده می شوند. جدول 'system.one' دارای دقیقا یک سطر است. (این جدول دقیقا همان اهدافی که جدول DUAL در دیگر DBMS ها دارد را دنبال می کند)

دستور FROM، جدولی که قصد خواندن آن را داریم و یا یک subquery و یا یک table function را مشخص می کند؛ ممکن است ARRAY JOIN و JOIN هم شامل شود.

به جای جدول، ممکن است، Select subquery در براکت تعیین شود. در این مورد، پردازش subquery به صورت یک query بیرونی (external) پردازش خواهد شد. در تضاد با SQL استاندارد، نیاز به مشخص نمودن synonym بعد از یک subquery وجود ندارد. برای سازگاری، امکان استفاده از 'AS name' بعد از subquery وجود دارد، اما نام مشخص شده در هر جایی استفاده نی شود.

table function می تواند به جای جدول مشخص شود. برای اطلاعات بیشتر بخش "Table functions" را مشاهده کنید.

برای اجرای یک query، تمام ستون های لیست شده در query از جدول مناسب extract می شوند. هر ستونی که برای query بیرونی (external) مورد نیاز نیست، از subquery های حذف می شود. اگر یک query ستون هاش رو مشخص نکند (برای مثال SELECT count() FROM t)، برای محاسبه تعداد سطر ها، یک ستون از جدول extract می شود (کوچکترین ستون ترجیح داده می شود).

FINAL modifier فقط در هنگام SELECT از جدول CollapsingMergeTree می تواند استفاده شود. وقتی شما FINAL رو مشخص می کنید، دیتا به صورت کامل "collapsed" انتخاب می شوند. در نظر داشته باشید که استفاده از FINAL، علاوه بر ستون های مشخص شده در SELECT، منجر به انتخاب ستون های مربوط به Primary Key خواهد شد. علاوه بر این، query در یک stream اجرا خواهد شد و داده ها در طول اجرای query با هم merge می شوند. این به این معنی است که در هنگام استفاده از FINAL، ممکن است query شما کندتر اجرا شود. در بیشتر موارد، شما باید از دستور FINAL دوری کنید. برای اطلاعات بیشتر بخش "موتور CollapsingMergeTree" را ببینید.

### دستور SAMPLE

دستور SAMPLE اجازه ی پردازش query به صورت تقریبی را می دهد. پردازش تقریبی query فقط در موتور MergeTree* پشتیبانی می شود، و تنها اگر sampling expression در هنگام ساخت جدول مشخص شده باشد (بخش "موتور MergeTree" را ببینید).

دستور `SAMPLE` فرمت `SAMPLE k` دارد،  جایی که `k` یک عدد بین 0 تا 1 می باشد، یا `SAMPLE n` جایی که 'n' یک عدد integer به اندازه کافی بزرگ می باشد.

در مورد اول، query بر روی 'k' درصد از داده ها اجرا می شود. برای مثال `SAMPLE 0.1` روی 10 درصد از داده ها اجرا می شود. در مورد دوم، query دقیقا بر روی 'n' سطر و نه بیشتر اجرا می شود. برای مثال `SAMPLE 10000000` نهایتا بر روی 10,000,000 سطر اجرا می شود.

مثال:

</div>

```sql
SELECT
    Title,
    count() * 10 AS PageViews
FROM hits_distributed
SAMPLE 0.1
WHERE
    CounterID = 34
    AND toDate(EventDate) >= toDate('2013-01-29')
    AND toDate(EventDate) <= toDate('2013-02-04')
    AND NOT DontCountHits
    AND NOT Refresh
    AND Title != ''
GROUP BY Title
ORDER BY PageViews DESC LIMIT 1000
```
<div dir="rtl">


در این مثال، query بر روی 0.1 (10 درصد) از داده ها اجرا می شود. مقادیر توابع aggregate به صورت اتوماتیک اصلاح نمی شوند، پس برای دستیابی به یک نتیجه ی تقریبی، مقدار 'count()' به صورت دستی 10 برابر می شود.

هنگام استفاده از چیزی شبیه به `SAMPLE 10000000`، هیچ اطلاعاتی در مورد درصد نسبی از داده های پردازش شده یا توابع aggregate که باید به چند برابر شوند وجود ندارد، بنابراین همیشه این روش نوشتن مناسب نیست. 

یک نمونه با ضریب نسبی "consistent" است: اگر ما نگاهی به تمام داده های احتمالی را که می توان در جدول قرار داد بیندازیم،  (هنگام استفاده از یک فرمول نمونه گیری مشخص در طی ایجاد جدول)  با ضریب مشابه همیشه زیرمجموعه ای از داده های ممکن را انتخاب می کند. به عبارت دیگر یک نمونه از جداول مختلف در سرور های مختلف و در زمان های مختلف به همان شیوه ساخته می شود.

For example, a sample of user IDs takes rows with the same subset of all the possible user IDs from different tables. This allows using the sample in subqueries in the IN clause, as well as for manually correlating results of different queries with samples.

### دستور ARRAY JOIN

اجازه ی اجرای JOIN با یک آرایه یا با یک nested data structure را می دهد. هدف این دستور شبیه به تابع 'arrayJoin' می باشد، اما قابلیت آن گسترده تر است.

`ARRAY JOIN` اساسا همان `INNER JOIN` با استفاده از آرایه است. مثال:

</div>

```text
:) CREATE TABLE arrays_test (s String, arr Array(UInt8)) ENGINE = Memory

CREATE TABLE arrays_test
(
    s String,
    arr Array(UInt8)
) ENGINE = Memory

Ok.

0 rows in set. Elapsed: 0.001 sec.

:) INSERT INTO arrays_test VALUES ('Hello', [1,2]), ('World', [3,4,5]), ('Goodbye', [])

INSERT INTO arrays_test VALUES

Ok.

3 rows in set. Elapsed: 0.001 sec.

:) SELECT * FROM arrays_test

SELECT *
FROM arrays_test

┌─s───────┬─arr─────┐
│ Hello   │ [1,2]   │
│ World   │ [3,4,5] │
│ Goodbye │ []      │
└─────────┴─────────┘

3 rows in set. Elapsed: 0.001 sec.

:) SELECT s, arr FROM arrays_test ARRAY JOIN arr

SELECT s, arr
FROM arrays_test
ARRAY JOIN arr

┌─s─────┬─arr─┐
│ Hello │   1 │
│ Hello │   2 │
│ World │   3 │
│ World │   4 │
│ World │   5 │
└───────┴─────┘

5 rows in set. Elapsed: 0.001 sec.
```

<div dir="rtl">

نام alias می تواند برای یک آرایه در یک ARRAY JOIN استفاده شود. در این مورد، آیتم های آرایه می تواند از طریق alias قابل دسترسی باشند، اما خود آرایه با استفاده از نام اصلیش. مثال: 

</div>

```text
:) SELECT s, arr, a FROM arrays_test ARRAY JOIN arr AS a

SELECT s, arr, a
FROM arrays_test
ARRAY JOIN arr AS a

┌─s─────┬─arr─────┬─a─┐
│ Hello │ [1,2]   │ 1 │
│ Hello │ [1,2]   │ 2 │
│ World │ [3,4,5] │ 3 │
│ World │ [3,4,5] │ 4 │
│ World │ [3,4,5] │ 5 │
└───────┴─────────┴───┘

5 rows in set. Elapsed: 0.001 sec.
```

<div dir="rtl">

چندین آرایه با سایز یکسان می توانند به صورت comma-separated در ARRAY JOIN قرار بگیرند. در این مورد، JOIN به طور همزمان با آنها انجام می شود  (the direct sum, not the direct product). مثال:

</div>

```text
:) SELECT s, arr, a, num, mapped FROM arrays_test ARRAY JOIN arr AS a, arrayEnumerate(arr) AS num, arrayMap(x -> x + 1, arr) AS mapped

SELECT s, arr, a, num, mapped
FROM arrays_test
ARRAY JOIN arr AS a, arrayEnumerate(arr) AS num, arrayMap(lambda(tuple(x), plus(x, 1)), arr) AS mapped

┌─s─────┬─arr─────┬─a─┬─num─┬─mapped─┐
│ Hello │ [1,2]   │ 1 │   1 │      2 │
│ Hello │ [1,2]   │ 2 │   2 │      3 │
│ World │ [3,4,5] │ 3 │   1 │      4 │
│ World │ [3,4,5] │ 4 │   2 │      5 │
│ World │ [3,4,5] │ 5 │   3 │      6 │
└───────┴─────────┴───┴─────┴────────┘

5 rows in set. Elapsed: 0.002 sec.

:) SELECT s, arr, a, num, arrayEnumerate(arr) FROM arrays_test ARRAY JOIN arr AS a, arrayEnumerate(arr) AS num

SELECT s, arr, a, num, arrayEnumerate(arr)
FROM arrays_test
ARRAY JOIN arr AS a, arrayEnumerate(arr) AS num

┌─s─────┬─arr─────┬─a─┬─num─┬─arrayEnumerate(arr)─┐
│ Hello │ [1,2]   │ 1 │   1 │ [1,2]               │
│ Hello │ [1,2]   │ 2 │   2 │ [1,2]               │
│ World │ [3,4,5] │ 3 │   1 │ [1,2,3]             │
│ World │ [3,4,5] │ 4 │   2 │ [1,2,3]             │
│ World │ [3,4,5] │ 5 │   3 │ [1,2,3]             │
└───────┴─────────┴───┴─────┴─────────────────────┘

5 rows in set. Elapsed: 0.002 sec.
```

<div dir="rtl">

همچنین دستور ARRAY JOIN در nested data structure کار می کند. مثال:

</div>

```text
:) CREATE TABLE nested_test (s String, nest Nested(x UInt8, y UInt32)) ENGINE = Memory

CREATE TABLE nested_test
(
    s String,
    nest Nested(
    x UInt8,
    y UInt32)
) ENGINE = Memory

Ok.

0 rows in set. Elapsed: 0.006 sec.

:) INSERT INTO nested_test VALUES ('Hello', [1,2], [10,20]), ('World', [3,4,5], [30,40,50]), ('Goodbye', [], [])

INSERT INTO nested_test VALUES

Ok.

3 rows in set. Elapsed: 0.001 sec.

:) SELECT * FROM nested_test

SELECT *
FROM nested_test

┌─s───────┬─nest.x──┬─nest.y─────┐
│ Hello   │ [1,2]   │ [10,20]    │
│ World   │ [3,4,5] │ [30,40,50] │
│ Goodbye │ []      │ []         │
└─────────┴─────────┴────────────┘

3 rows in set. Elapsed: 0.001 sec.

:) SELECT s, nest.x, nest.y FROM nested_test ARRAY JOIN nest

SELECT s, `nest.x`, `nest.y`
FROM nested_test
ARRAY JOIN nest

┌─s─────┬─nest.x─┬─nest.y─┐
│ Hello │      1 │     10 │
│ Hello │      2 │     20 │
│ World │      3 │     30 │
│ World │      4 │     40 │
│ World │      5 │     50 │
└───────┴────────┴────────┘

5 rows in set. Elapsed: 0.001 sec.
```

<div dir="rtl">

زمانی که در ARRAY JOIN برای nested data structure اسم می گذاریم، معنای همان ARRAY JOIN با تمام element های آرایه که از آن تشکیل شده است می دهد. مثال:

</div>

```text
:) SELECT s, nest.x, nest.y FROM nested_test ARRAY JOIN nest.x, nest.y

SELECT s, `nest.x`, `nest.y`
FROM nested_test
ARRAY JOIN `nest.x`, `nest.y`

┌─s─────┬─nest.x─┬─nest.y─┐
│ Hello │      1 │     10 │
│ Hello │      2 │     20 │
│ World │      3 │     30 │
│ World │      4 │     40 │
│ World │      5 │     50 │
└───────┴────────┴────────┘

5 rows in set. Elapsed: 0.001 sec.
```

<div dir="rtl">

این تنوع همجنین احساس می شود:

</div>

```text
:) SELECT s, nest.x, nest.y FROM nested_test ARRAY JOIN nest.x

SELECT s, `nest.x`, `nest.y`
FROM nested_test
ARRAY JOIN `nest.x`

┌─s─────┬─nest.x─┬─nest.y─────┐
│ Hello │      1 │ [10,20]    │
│ Hello │      2 │ [10,20]    │
│ World │      3 │ [30,40,50] │
│ World │      4 │ [30,40,50] │
│ World │      5 │ [30,40,50] │
└───────┴────────┴────────────┘

5 rows in set. Elapsed: 0.001 sec.
```

<div dir="rtl">

برای انتخاب نتایج JOIN و یا source array از اسم alias می توان در nested data structure استفاده کرد.

</div>

```text
:) SELECT s, n.x, n.y, nest.x, nest.y FROM nested_test ARRAY JOIN nest AS n

SELECT s, `n.x`, `n.y`, `nest.x`, `nest.y`
FROM nested_test
ARRAY JOIN nest AS n

┌─s─────┬─n.x─┬─n.y─┬─nest.x──┬─nest.y─────┐
│ Hello │   1 │  10 │ [1,2]   │ [10,20]    │
│ Hello │   2 │  20 │ [1,2]   │ [10,20]    │
│ World │   3 │  30 │ [3,4,5] │ [30,40,50] │
│ World │   4 │  40 │ [3,4,5] │ [30,40,50] │
│ World │   5 │  50 │ [3,4,5] │ [30,40,50] │
└───────┴─────┴─────┴─────────┴────────────┘

5 rows in set. Elapsed: 0.001 sec.
```

<div dir="rtl">

مثالی از استفاده از تابع arrayEnumerate:

</div>

```text
:) SELECT s, n.x, n.y, nest.x, nest.y, num FROM nested_test ARRAY JOIN nest AS n, arrayEnumerate(nest.x) AS num

SELECT s, `n.x`, `n.y`, `nest.x`, `nest.y`, num
FROM nested_test
ARRAY JOIN nest AS n, arrayEnumerate(`nest.x`) AS num

┌─s─────┬─n.x─┬─n.y─┬─nest.x──┬─nest.y─────┬─num─┐
│ Hello │   1 │  10 │ [1,2]   │ [10,20]    │   1 │
│ Hello │   2 │  20 │ [1,2]   │ [10,20]    │   2 │
│ World │   3 │  30 │ [3,4,5] │ [30,40,50] │   1 │
│ World │   4 │  40 │ [3,4,5] │ [30,40,50] │   2 │
│ World │   5 │  50 │ [3,4,5] │ [30,40,50] │   3 │
└───────┴─────┴─────┴─────────┴────────────┴─────┘

5 rows in set. Elapsed: 0.002 sec.
```

<div dir="rtl">

query فقط می تواند یک ARRAY JOIN مشخص کند.

تبدیل مربوطه می تواند قبل از دستورات WHERE/PREWHERE (اگر نتیجه ی آن در این دستور نیاز باشد)، یا بعد از تکیل WHERE/PREWHERE انجام شود (برای کاهش حجم محاسبات).

### JOIN clause

JOIN عادی، که به ARRAY JOIN که در بالا توضیح داده شده است مرتبط نیست.

</div>

```sql
[GLOBAL] ANY|ALL INNER|LEFT [OUTER] JOIN (subquery)|table USING columns_list
```

<div dir="rtl">

در ابتدای پردازش query، subquery که بعد از join مشخص شده اجرا می شود و نتایج آن در memory قرار می گیرد. سپس از جدول سمت "چپ" که در FROM مشخص شده، در هنگام خواندن سطرهای آن، به ازای هر یک از سطرهای خوانده شده از جدول سمت چپ، سطرهایی که از نتایج subquery جدول راست به دست آمده اند و شرطهای مشخص شده بعد از USING را دارند انتخاب می شوند و در خروجی نمایش داده می شوند.

نام یک جدول به جای یک subquery در join می تواند قرار گیرد. این کار به استثنا موارد خاص که جدول از موتور Join استفاده می کند، معادل `SELECT * FROM table` می باشد.

همه ی ستون هایی که در هنگام JOIN نیاز نیستند از subquery حذف می شوند.

چندین نوع از JOIN های وجود دارد:

`INNER` یا `LEFT`: اگر از INNER استفاده شود، نتیجه ی query فقط شامل سطرهایی می شود که سطر مطابق در جدول راست دارند.
اگر از LEFT استفاده شود، هر سطری از جدول چپ که با جدول راستی مطابقت ندارد در خروجی حاضر می شود اما با مقدار پیش فرض zeros یا empty پر می شود. به جای LEFT ممکن است LEFT OUTER نوشته شود که کلمه ی OUTER تاثیری در خروجی ندارد.

`ANY` یا `ALL`: اگر از `ANY` استفاده شود و جدول راست چندین سطر یکسان، مطابق با جدول چپ داشته باشد، فقط اولین سطر مطابق join می شود. اگر از `ALL` استفاده شود و جدول سمت راست چندین ستون یکسان، مطابق با جدول چپ داشته باشد، همه ی آنها در خروجی نمایش داده می شوند.

استفاده از ALL مطابق با JOIN عادی در SQL استاندارد می باشد. استفاده از ANY بهینه است. اگر جدول سمت راست فقط یک سطر مطابق داشته باشد، خروجی ANY و ALL یکسان خواهد بود. شما باید ANY یا ALL را انتخاب کنید (هیچکدام به صورت پیش فرض انتخاب نمی شوند).

`GLOBAL` distribution:

در هنگام استفاده از JOIN عادی، query به سرور های remote ارسال می شود. subquery ها به ترتیب در هر یک از آنها برای ساخت جدول راست اجرا می شوند، و join با این جدول به دست آمده انجام می شود. به باید دیگر جدول راست در هر یک از سرور ها به صورت جداگانه شکل می گیرد.

هنگام استفاده از `GLOBAL ... JOIN`، اولین سرور، subquery رو برای محاسبه جدول راست اجرا می کند. این جدول موقت به هر یک از سرور های remote پاس داده می شود، و query ها روی همان جدول موقت با داده هایی که منتقل شده اند اجرا می شود.

در هنگام استفاده از GLOBAL JOIN ها دقت کنید. برای اطلاعات بیشتر، بخش "Distributed subqueries" را ببینید.

هر ترکیبی از JOIN ها ممکن است. برای مثال: `GLOBAL ANY LEFT OUTER JOIN`.

در هنگام اجرای یک JOIN، هیچ بهینه سازی در ارتباط با ترتیب اجرا به سایر مراحل وجود ندارد. join (جستجو در جدول سمت راست) قبل از WHERE و قبل از aggregation اجرا می شود. توصیه می کنیم در هنگام استفاده از join از subquery استفاده کنید.

مثال:

</div>

```sql
SELECT
    CounterID,
    hits,
    visits
FROM
(
    SELECT
        CounterID,
        count() AS hits
    FROM test.hits
    GROUP BY CounterID
) ANY LEFT JOIN
(
    SELECT
        CounterID,
        sum(Sign) AS visits
    FROM test.visits
    GROUP BY CounterID
) USING CounterID
ORDER BY hits DESC
LIMIT 10
```

```text
┌─CounterID─┬───hits─┬─visits─┐
│   1143050 │ 523264 │  13665 │
│    731962 │ 475698 │ 102716 │
│    722545 │ 337212 │ 108187 │
│    722889 │ 252197 │  10547 │
│   2237260 │ 196036 │   9522 │
│  23057320 │ 147211 │   7689 │
│    722818 │  90109 │  17847 │
│     48221 │  85379 │   4652 │
│  19762435 │  77807 │   7026 │
│    722884 │  77492 │  11056 │
└───────────┴────────┴────────┘
```

<div dir="rtl">

subquery ها به شما اجازه ی تنظیم اسامی یا استفاده از آنها برای refrence دادن به ستون مشخص دیگر را نمی دهند. ستون های مشخص شده در USING باید در هر دو subquery یکسان باشند، و دیگر ستون ها باید نام متفاوتی داشته باشند. شما می توانید از اسم alias برای تغییر اسم ستون های subquery ها استفاده کنید (مثلا استفاده از اسم 'hits' و 'visits').

دستور USING یک یا چند ستون را برای join مشخص می کند، که شرط برابری ستون های را برقرار می کند. این لیست از ستون ها بدون براکت ست می شوند. دیگر شرطهای پیچیده ی join پشتیبانی نمی شوند.

جدول راست (نتیجه ی subquery) در RAM نگه داری می شود. اگر memory به اندازه کافی موجود نباشد، شما نمی توان JOIN را اجرا کنید.

یک JOIN فقط می توانید در یک query مشخص شود. برای اجرای چندین JOIN، باید آنها رو در subquery ها قرار بدید.

هر بار که یک query با JOIN یکسان اجرا می شود، subquery مجددا اجرا می شود - نتایج JOIN کش نمی شوند. برای جلوگیری از آن از موتور جدول ویژه ی 'Join' استفاده کنید، که آرایه ی آماده برای JOIN را همیشه در RAM دارد. برای اطلاعات بیشتر بخش "موتور های جدول، موتور Join" را ببینید.

در بعضی موارد، استفاده از IN به جای JOIN کارآمدتر است. در میان انواع مختلف JOIN ها، کارآمدترین آن ANY LEFT JOIN و ANY INNER JOIN می باشد. کمترین کارآمد ALL LEFT JOIN و ALL INNER JOIN می باشد.

اگر شما نیاز به JOIN برای join زدن با جداول dimension دارید (این جداول نسبتا کوچک هستند و شامل پروپرتی های dimension، مثل نام برای کمپین های تبلیغاتی می باشند)، JOIN ممکن است به دلیل سینتکس bulky و این واقعیت که جدول راست برای هر query قابل دسترسی است، خیلی راحت نباشد. برای بعضی موارد، باید از ویژگی "external dictionaries" به جای JOIN استفاده کنید. برای اطلاعات بیشتر بخش "External dictionaries" را ببینید.

با تنظیم join_use_nulls، رفتار JOIN تحت تاثیر قرار می گیرد. در صورت تنظیم `join_use_nulls=1`، دستور JOIN مشابه SQL استاندارد کار خواهد کرد.

اگر کلید های JOIN فیلد های Nullable باشند، سطرهایی که حداقل یکی از کلید های آن دارای مقدار Null باشند، join نمیشوند.

### دستور WHERE

اگر دستور WHERE استفاده شود، باید حاوی شرطی با نوع Uint8 باشد. معمولا این شرط از نوع مقایسه و اپراتورهای منطقی می باشد. از شرط برای فیلتر کردن دیتا قبل از انتقال آنها استفاده می شود.

اگر ایندکس توسط موتور جدول پشتیبانی شود، شرط توانایی استفاده از ایندکس را دارا خواهد بود.

### دستور PREWHERE

این دستور هم معنی دستور WHERE می باشد. تفاوت آنها فقط در این است که کدام یک از دیتا باید از جدول خوانده شود. در هنگام استفاده از PREWHERE، در ابتدا ستون هایی که برای اجرای PREWHERE واجب هستند خوانده می شوند. سپس بقیه ستون هایی که برای اجرای query نیاز هستند خوانده می شوند. البته به شرطی این ستون ها خوانده می شوند که شرط PREWHERE را دارا باشند.

اگر شرط فیلتر کردنی وجود دارد که مناسب برای index نیست و توسط ستون های کمی در query استفاده می شود اما یک فیلتر قوی روی داده ها اعمال می کند، در این شرایط استفاده از PREWHERE مناسب است. این باعث کاهش حجم خواندن داده ها می شود.

برای مثال، برای query هایی که تعداد زیادی ستون extract می کنن اما فقط روی چند ستون عملیات فیلترینگ دارند، نوشتن PREWHERE مناسب است.

PREWHERE فقط در جداول با موتور `MergeTree*` پشتیبانی می شود.

یک query می تواند همزمان هم PREWHEER و هم WHERE داشته باشد اما ابتدا PREWHERE اجرا می شود.

به یاد داشته باشید که استفاده از PREWHERE در ستون هایی که دارای ایندکس هستند چندان پراهمیت نیست، چون در هنگام استفاده از ایندکس، فقط بلوک داده هایی که مطابق با ایندکس هستند خوانده می شوند.

اگر 'optimize_move_to_prewhere' برابر با 1 باشد و از PREWHERE استفاده نشود، سیستم به صورت اتوماتیک بخشی از دستورات WHERE رو به PREWHERE منتقل می کند.

### GROUP BY clause

This is one of the most important parts of a column-oriented DBMS.

If there is a GROUP BY clause, it must contain a list of expressions. Each expression will be referred to here as a "key".
All the expressions in the SELECT, HAVING, and ORDER BY clauses must be calculated from keys or from aggregate functions. In other words, each column selected from the table must be used either in keys or inside aggregate functions.

If a query contains only table columns inside aggregate functions, the GROUP BY clause can be omitted, and aggregation by an empty set of keys is assumed.

Example:

</div>

```sql
SELECT
    count(),
    median(FetchTiming > 60 ? 60 : FetchTiming),
    count() - sum(Refresh)
FROM hits
```

However, in contrast to standard SQL, if the table doesn't have any rows (either there aren't any at all, or there aren't any after using WHERE to filter), an empty result is returned, and not the result from one of the rows containing the initial values of aggregate functions.

As opposed to MySQL (and conforming to standard SQL), you can't get some value of some column that is not in a key or aggregate function (except constant expressions). To work around this, you can use the 'any' aggregate function (get the first encountered value) or 'min/max'.

Example:

```sql
SELECT
    domainWithoutWWW(URL) AS domain,
    count(),
    any(Title) AS title -- getting the first occurred page header for each domain.
FROM hits
GROUP BY domain
```

For every different key value encountered, GROUP BY calculates a set of aggregate function values.

GROUP BY is not supported for array columns.

A constant can't be specified as arguments for aggregate functions. Example: sum(1). Instead of this, you can get rid of the constant. Example: `count()`.

#### WITH TOTALS modifier

If the WITH TOTALS modifier is specified, another row will be calculated. This row will have key columns containing default values (zeros or empty lines), and columns of aggregate functions with the values calculated across all the rows (the "total" values).

This extra row is output in JSON\*, TabSeparated\*, and Pretty\* formats, separately from the other rows. In the other formats, this row is not output.

In JSON\* formats, this row is output as a separate 'totals' field. In TabSeparated\* formats, the row comes after the main result, preceded by an empty row (after the other data). In Pretty\* formats, the row is output as a separate table after the main result.

`WITH TOTALS` can be run in different ways when HAVING is present. The behavior depends on the 'totals_mode' setting.
By default, `totals_mode = 'before_having'`. In this case, 'totals' is calculated across all rows, including the ones that don't pass through HAVING and 'max_rows_to_group_by'.

The other alternatives include only the rows that pass through HAVING in 'totals', and behave differently with the setting `max_rows_to_group_by` and `group_by_overflow_mode = 'any'`.

`after_having_exclusive` – Don't include rows that didn't pass through `max_rows_to_group_by`. In other words, 'totals' will have less than or the same number of rows as it would if `max_rows_to_group_by` were omitted.

`after_having_inclusive` – Include all the rows that didn't pass through 'max_rows_to_group_by' in 'totals'. In other words, 'totals' will have more than or the same number of rows as it would if `max_rows_to_group_by` were omitted.

`after_having_auto` – Count the number of rows that passed through HAVING. If it is more than a certain amount (by default, 50%), include all the rows that didn't pass through 'max_rows_to_group_by' in 'totals'. Otherwise, do not include them.

`totals_auto_threshold` – By default, 0.5. The coefficient for `after_having_auto`.

If `max_rows_to_group_by` and `group_by_overflow_mode = 'any'` are not used, all variations of `after_having` are the same, and you can use any of them (for example, `after_having_auto`).

You can use WITH TOTALS in subqueries, including subqueries in the JOIN clause (in this case, the respective total values are combined).

#### GROUP BY in external memory

You can enable dumping temporary data to the disk to restrict memory usage during GROUP BY.
The `max_bytes_before_external_group_by` setting determines the threshold RAM consumption for dumping GROUP BY temporary data to the file system. If set to 0 (the default), it is disabled.

When using `max_bytes_before_external_group_by`, we recommend that you set max_memory_usage about twice as high. This is necessary because there are two stages to aggregation: reading the date and forming intermediate data (1) and merging the intermediate data (2). Dumping data to the file system can only occur during stage 1. If the temporary data wasn't dumped, then stage 2 might require up to the same amount of memory as in stage 1.

For example, if `max_memory_usage` was set to 10000000000 and you want to use external aggregation, it makes sense to set `max_bytes_before_external_group_by` to 10000000000, and max_memory_usage to 20000000000. When external aggregation is triggered (if there was at least one dump of temporary data), maximum consumption of RAM is only slightly more than ` max_bytes_before_external_group_by`.

With distributed query processing, external aggregation is performed on remote servers. In order for the requestor server to use only a small amount of RAM, set ` distributed_aggregation_memory_efficient`  to 1.

When merging data flushed to the disk, as well as when merging results from remote servers when the ` distributed_aggregation_memory_efficient` setting is enabled, consumes up to 1/256 \* the number of threads from the total amount of RAM.

When external aggregation is enabled, if there was less than ` max_bytes_before_external_group_by`  of data (i.e. data was not flushed), the query runs just as fast as without external aggregation. If any temporary data was flushed, the run time will be several times longer (approximately three times).

If you have an ORDER BY with a small LIMIT after GROUP BY, then the ORDER BY CLAUSE will not use significant amounts of RAM.
But if the ORDER BY doesn't have LIMIT, don't forget to enable external sorting (`max_bytes_before_external_sort`).

### LIMIT N BY clause

LIMIT N BY COLUMNS selects the top N rows for each group of COLUMNS. LIMIT N BY is not related to LIMIT; they can both be used in the same query. The key for LIMIT N BY can contain any number of columns or expressions.

Example:

```sql
SELECT
    domainWithoutWWW(URL) AS domain,
    domainWithoutWWW(REFERRER_URL) AS referrer,
    device_type,
    count() cnt
FROM hits
GROUP BY domain, referrer, device_type
ORDER BY cnt DESC
LIMIT 5 BY domain, device_type
LIMIT 100
```

The query will select the top 5 referrers for each `domain, device_type` pair, but not more than 100 rows (`LIMIT n BY + LIMIT`).

### HAVING clause

Allows filtering the result received after GROUP BY, similar to the WHERE clause.
WHERE and HAVING differ in that WHERE is performed before aggregation (GROUP BY), while HAVING is performed after it.
If aggregation is not performed, HAVING can't be used.

<a name="query_language-queries-order_by"></a>

### ORDER BY clause

The ORDER BY clause contains a list of expressions, which can each be assigned DESC or ASC (the sorting direction). If the direction is not specified, ASC is assumed. ASC is sorted in ascending order, and DESC in descending order. The sorting direction applies to a single expression, not to the entire list. Example: `ORDER BY Visits DESC, SearchPhrase`

For sorting by String values, you can specify collation (comparison). Example: `ORDER BY SearchPhrase COLLATE 'tr'` - for sorting by keyword in ascending order, using the Turkish alphabet, case insensitive, assuming that strings are UTF-8 encoded. COLLATE can be specified or not for each expression in ORDER BY independently. If ASC or DESC is specified, COLLATE is specified after it. When using COLLATE, sorting is always case-insensitive.

We only recommend using COLLATE for final sorting of a small number of rows, since sorting with COLLATE is less efficient than normal sorting by bytes.

Rows that have identical values for the list of sorting expressions are output in an arbitrary order, which can also be nondeterministic (different each time).
If the ORDER BY clause is omitted, the order of the rows is also undefined, and may be nondeterministic as well.

When floating point numbers are sorted, NaNs are separate from the other values. Regardless of the sorting order, NaNs come at the end. In other words, for ascending sorting they are placed as if they are larger than all the other numbers, while for descending sorting they are placed as if they are smaller than the rest.

Less RAM is used if a small enough LIMIT is specified in addition to ORDER BY. Otherwise, the amount of memory spent is proportional to the volume of data for sorting. For distributed query processing, if GROUP BY is omitted, sorting is partially done on remote servers, and the results are merged on the requestor server. This means that for distributed sorting, the volume of data to sort can be greater than the amount of memory on a single server.

If there is not enough RAM, it is possible to perform sorting in external memory (creating temporary files on a disk). Use the setting `max_bytes_before_external_sort` for this purpose. If it is set to 0 (the default), external sorting is disabled. If it is enabled, when the volume of data to sort reaches the specified number of bytes, the collected data is sorted and dumped into a temporary file. After all data is read, all the sorted files are merged and the results are output. Files are written to the /var/lib/clickhouse/tmp/ directory in the config (by default, but you can use the 'tmp_path' parameter to change this setting).

Running a query may use more memory than 'max_bytes_before_external_sort'. For this reason, this setting must have a value significantly smaller than 'max_memory_usage'. As an example, if your server has 128 GB of RAM and you need to run a single query, set 'max_memory_usage' to 100 GB, and 'max_bytes_before_external_sort' to 80 GB.

External sorting works much less effectively than sorting in RAM.

### SELECT clause

The expressions specified in the SELECT clause are analyzed after the calculations for all the clauses listed above are completed.
More specifically, expressions are analyzed that are above the aggregate functions, if there are any aggregate functions.
The aggregate functions and everything below them are calculated during aggregation (GROUP BY).
These expressions work as if they are applied to separate rows in the result.

### DISTINCT clause

If DISTINCT is specified, only a single row will remain out of all the sets of fully matching rows in the result.
The result will be the same as if GROUP BY were specified across all the fields specified in SELECT without aggregate functions. But there are several differences from GROUP BY:

- DISTINCT can be applied together with GROUP BY.
- When ORDER BY is omitted and LIMIT is defined, the query stops running immediately after the required number of different rows has been read.
- Data blocks are output as they are processed, without waiting for the entire query to finish running.

DISTINCT is not supported if SELECT has at least one array column.

### LIMIT clause

LIMIT m allows you to select the first 'm' rows from the result.
LIMIT n, m allows you to select the first 'm' rows from the result after skipping the first 'n' rows.

'n' and 'm' must be non-negative integers.

If there isn't an ORDER BY clause that explicitly sorts results, the result may be arbitrary and nondeterministic.

### UNION ALL clause

You can use UNION ALL to combine any number of queries. Example:

```sql
SELECT CounterID, 1 AS table, toInt64(count()) AS c
    FROM test.hits
    GROUP BY CounterID

UNION ALL

SELECT CounterID, 2 AS table, sum(Sign) AS c
    FROM test.visits
    GROUP BY CounterID
    HAVING c > 0
```

Only UNION ALL is supported. The regular UNION (UNION DISTINCT) is not supported. If you need UNION DISTINCT, you can write SELECT DISTINCT from a subquery containing UNION ALL.

Queries that are parts of UNION ALL can be run simultaneously, and their results can be mixed together.

The structure of results (the number and type of columns) must match for the queries. But the column names can differ. In this case, the column names for the final result will be taken from the first query.

Queries that are parts of UNION ALL can't be enclosed in brackets. ORDER BY and LIMIT are applied to separate queries, not to the final result. If you need to apply a conversion to the final result, you can put all the queries with UNION ALL in a subquery in the FROM clause.

### INTO OUTFILE clause

Add the `INTO OUTFILE filename` clause (where filename is a string literal) to redirect query output to the specified file.
In contrast to MySQL, the file is created on the client side. The query will fail if a file with the same filename already exists.
This functionality is available in the command-line client and clickhouse-local (a query sent via HTTP interface will fail).

The default output format is TabSeparated (the same as in the command-line client batch mode).

### FORMAT clause

Specify 'FORMAT format' to get data in any specified format.
You can use this for convenience, or for creating dumps.
For more information, see the section "Formats".
If the FORMAT clause is omitted, the default format is used, which depends on both the settings and the interface used for accessing the DB. For the HTTP interface and the command-line client in batch mode, the default format is TabSeparated. For the command-line client in interactive mode, the default format is PrettyCompact (it has attractive and compact tables).

When using the command-line client, data is passed to the client in an internal efficient format. The client independently interprets the FORMAT clause of the query and formats the data itself (thus relieving the network and the server from the load).

### IN operators

The `IN`, `NOT IN`, `GLOBAL IN`, and `GLOBAL NOT IN` operators are covered separately, since their functionality is quite rich.

The left side of the operator is either a single column or a tuple.

Examples:

```sql
SELECT UserID IN (123, 456) FROM ...
SELECT (CounterID, UserID) IN ((34, 123), (101500, 456)) FROM ...
```

If the left side is a single column that is in the index, and the right side is a set of constants, the system uses the index for processing the query.

Don't list too many values explicitly (i.e. millions). If a data set is large, put it in a temporary table (for example, see the section "External data for query processing"), then use a subquery.

The right side of the operator can be a set of constant expressions, a set of tuples with constant expressions (shown in the examples above), or the name of a database table or SELECT subquery in brackets.

If the right side of the operator is the name of a table (for example, `UserID IN users`), this is equivalent to the subquery `UserID IN (SELECT * FROM users)`. Use this when working with external data that is sent along with the query. For example, the query can be sent together with a set of user IDs loaded to the 'users' temporary table, which should be filtered.

If the right side of the operator is a table name that has the Set engine (a prepared data set that is always in RAM), the data set will not be created over again for each query.

The subquery may specify more than one column for filtering tuples.
Example:

```sql
SELECT (CounterID, UserID) IN (SELECT CounterID, UserID FROM ...) FROM ...
```

The columns to the left and right of the IN operator should have the same type.

The IN operator and subquery may occur in any part of the query, including in aggregate functions and lambda functions.
Example:

```sql
SELECT
    EventDate,
    avg(UserID IN
    (
        SELECT UserID
        FROM test.hits
        WHERE EventDate = toDate('2014-03-17')
    )) AS ratio
FROM test.hits
GROUP BY EventDate
ORDER BY EventDate ASC
```

```text
┌──EventDate─┬────ratio─┐
│ 2014-03-17 │        1 │
│ 2014-03-18 │ 0.807696 │
│ 2014-03-19 │ 0.755406 │
│ 2014-03-20 │ 0.723218 │
│ 2014-03-21 │ 0.697021 │
│ 2014-03-22 │ 0.647851 │
│ 2014-03-23 │ 0.648416 │
└────────────┴──────────┘
```

For each day after March 17th, count the percentage of pageviews made by users who visited the site on March 17th.
A subquery in the IN clause is always run just one time on a single server. There are no dependent subqueries.

<a name="queries-distributed-subrequests"></a>

#### Distributed subqueries

There are two options for IN-s with subqueries (similar to JOINs): normal `IN`  / ` OIN`  and `IN GLOBAL`  / `GLOBAL JOIN`. They differ in how they are run for distributed query processing.

!!! attention
    Remember that the algorithms described below may work differently depending on the [settings](../operations/settings/settings.md#settings-distributed_product_mode) `distributed_product_mode` setting.

When using the regular IN, the query is sent to remote servers, and each of them runs the subqueries in the `IN` or `JOIN` clause.

When using `GLOBAL IN`  / `GLOBAL JOINs`, first all the subqueries are run for `GLOBAL IN`  / `GLOBAL JOINs`, and the results are collected in temporary tables. Then the temporary tables are sent to each remote server, where the queries are run using this temporary data.

For a non-distributed query, use the regular `IN` / `JOIN`.

Be careful when using subqueries in the  `IN` / `JOIN` clauses for distributed query processing.

Let's look at some examples. Assume that each server in the cluster has a normal **local_table**. Each server also has a **distributed_table** table with the **Distributed** type, which looks at all the servers in the cluster.

For a query to the **distributed_table**, the query will be sent to all the remote servers and run on them using the **local_table**.

For example, the query

```sql
SELECT uniq(UserID) FROM distributed_table
```

will be sent to all remote servers as

```sql
SELECT uniq(UserID) FROM local_table
```

and run on each of them in parallel, until it reaches the stage where intermediate results can be combined. Then the intermediate results will be returned to the requestor server and merged on it, and the final result will be sent to the client.

Now let's examine a query with IN:

```sql
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM local_table WHERE CounterID = 34)
```

- Calculation of the intersection of audiences of two sites.

This query will be sent to all remote servers as

```sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM local_table WHERE CounterID = 34)
```

In other words, the data set in the IN clause will be collected on each server independently, only across the data that is stored locally on each of the servers.

This will work correctly and optimally if you are prepared for this case and have spread data across the cluster servers such that the data for a single UserID resides entirely on a single server. In this case, all the necessary data will be available locally on each server. Otherwise, the result will be inaccurate. We refer to this variation of the query as "local IN".

To correct how the query works when data is spread randomly across the cluster servers, you could specify **distributed_table** inside a subquery. The query would look like this:

```sql
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```

This query will be sent to all remote servers as

```sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```

The subquery will begin running on each remote server. Since the subquery uses a distributed table, the subquery that is on each remote server will be resent to every remote server as

```sql
SELECT UserID FROM local_table WHERE CounterID = 34
```

For example, if you have a cluster of 100 servers, executing the entire query will require 10,000 elementary requests, which is generally considered unacceptable.

In such cases, you should always use GLOBAL IN instead of IN. Let's look at how it works for the query

```sql
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID GLOBAL IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```

The requestor server will run the subquery

```sql
SELECT UserID FROM distributed_table WHERE CounterID = 34
```

and the result will be put in a temporary table in RAM. Then the request will be sent to each remote server as

```sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID GLOBAL IN _data1
```

and the temporary table `_data1` will be sent to every remote server with the query (the name of the temporary table is implementation-defined).

This is more optimal than using the normal IN. However, keep the following points in mind:

1. When creating a temporary table, data is not made unique. To reduce the volume of data transmitted over the network, specify DISTINCT in the subquery. (You don't need to do this for a normal IN.)
2. The temporary table will be sent to all the remote servers. Transmission does not account for network topology. For example, if 10 remote servers reside in a datacenter that is very remote in relation to the requestor server, the data will be sent 10 times over the channel to the remote datacenter. Try to avoid large data sets when using GLOBAL IN.
3. When transmitting data to remote servers, restrictions on network bandwidth are not configurable. You might overload the network.
4. Try to distribute data across servers so that you don't need to use GLOBAL IN on a regular basis.
5. If you need to use GLOBAL IN often, plan the location of the ClickHouse cluster so that a single group of replicas resides in no more than one data center with a fast network between them, so that a query can be processed entirely within a single data center.

It also makes sense to specify a local table in the `GLOBAL IN` clause, in case this local table is only available on the requestor server and you want to use data from it on remote servers.

### Extreme values

In addition to results, you can also get minimum and maximum values for the results columns. To do this, set the **extremes** setting to 1. Minimums and maximums are calculated for numeric types, dates, and dates with times. For other columns, the default values are output.

An extra two rows are calculated – the minimums and maximums, respectively. These extra two rows are output in JSON\*, TabSeparated\*, and Pretty\* formats, separate from the other rows. They are not output for other formats.

In JSON\* formats, the extreme values are output in a separate 'extremes' field. In TabSeparated\* formats, the row comes after the main result, and after 'totals' if present. It is preceded by an empty row (after the other data). In Pretty\* formats, the row is output as a separate table after the main result, and after 'totals' if present.

Extreme values are calculated for rows that have passed through LIMIT. However, when using 'LIMIT offset, size', the rows before 'offset' are included in 'extremes'. In stream requests, the result may also include a small number of rows that passed through LIMIT.

### Notes

The `GROUP BY` and `ORDER BY` clauses do not support positional arguments. This contradicts MySQL, but conforms to standard SQL.
For example, `GROUP BY 1, 2` will be interpreted as grouping by constants (i.e. aggregation of all rows into one).

You can use synonyms (`AS` aliases) in any part of a query.

You can put an asterisk in any part of a query instead of an expression. When the query is analyzed, the asterisk is expanded to a list of all table columns (excluding the `MATERIALIZED` and `ALIAS` columns). There are only a few cases when using an asterisk is justified:

- When creating a table dump.
- For tables containing just a few columns, such as system tables.
- For getting information about what columns are in a table. In this case, set `LIMIT 1`. But it is better to use the `DESC TABLE` query.
- When there is strong filtration on a small number of columns using `PREWHERE`.
- In subqueries (since columns that aren't needed for the external query are excluded from subqueries).

In all other cases, we don't recommend using the asterisk, since it only gives you the drawbacks of a columnar DBMS instead of the advantages. In other words using the asterisk is not recommended.

</div>