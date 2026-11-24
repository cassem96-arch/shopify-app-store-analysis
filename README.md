# Analyst Memo — Shopify App Store Insights

**Dashboard:** Shopify App Store Analysis
**Reporting Period:** Latest available data (reviews posted July 11, 2018 to December 30, 2024)

## Key Insight

Review activity on the Shopify App Store is growing fast, but merchant satisfaction is flat and developers rarely reply.

- The marketplace has **500 apps** across 14 categories. After cleaning, there are **7,980 reviews** with an **average rating of 4.19** out of 5.
- Reviews have roughly doubled every year: 556 in 2021, 1,118 in 2022, 2,151 in 2023 and **3,804 in 2024 (+76.8% YoY)**. 2024 alone holds 47.7% of all reviews. October 2024 was the busiest month, with 387 reviews.
- The average rating has barely moved as volume grew (4.17 in 2021, 4.19 in 2023, 4.20 in 2024). 80.5% of reviews are 4 or 5 stars.
- Developers replied to only **24.8%** of reviews. The reply rate is almost the same for 1–2 star reviews (25.3%) as for 4–5 star reviews (24.9%), so unhappy merchants get no more attention than happy ones.
- Volume is concentrated in a few categories. **Seo (1,145 reviews)** and **Reviews And Ratings (1,071)** lead, each with about 24 reviews per app. **Productivity (4.39)** and **Sales And Conversion (4.35)** have the highest ratings. **Loyalty And Rewards** has the lowest rating (4.01) and the highest share of 1–2 star reviews (6.9%, against 3.2% overall).

## Business Impact

- Review volume shows how many merchants are using apps, and it grew 76.8% last year. Merchants are installing and rating more apps every year, which makes the marketplace more valuable to developers.
- About three out of four reviews get no reply, and negative reviews are no more likely to get one than positive reviews. Every unanswered 1–2 star review is a public complaint that other merchants see when they decide what to install, and a missed chance to win that merchant back.
- Loyalty And Rewards and Subscriptions (4.05) are the weakest categories for satisfaction. These apps sit close to a merchant's revenue and repeat customers, so dissatisfaction there costs merchants the most.
- Free-plan apps (39.6% of apps) get a similar number of reviews per app as paid apps (15.1 vs 16.5) and a similar rating (4.20 vs 4.19). A free plan alone doesn't make an app succeed.

## Recommendation

1. **Push developers to reply, starting with negative reviews.** Add a "reply rate" and "unanswered 1–2 star reviews" panel to the developer dashboard, and consider a reply-rate badge or a ranking boost for developers who answer within a set time. Target: lift Developer Reply % from 24.8% to at least 40% on 1–2 star reviews.
2. **Run a quality review of Loyalty And Rewards and Subscriptions.** Look at the 1–2 star reviews in these categories to find the common problems, then share the findings with those developers and tighten listing quality checks.
3. **Promote the categories merchants are happiest with.** Feature highly rated Productivity and Sales And Conversion apps in discovery placements. Use the Seo and Reviews And Ratings categories, which have the most reviews, to find where new apps are still needed.
4. **Track the trend monthly.** Use the Trend Analysis page (YoY %, MTD, YTD) to check that growth continues and that the average rating holds as volume rises.

---

## Repository contents

```
shopify-app-store-analysis/
├── README.md                 this memo and the report documentation
├── report.pbix               Power BI report (saved from Power BI Desktop)
├── data/
│   ├── apps.csv              raw app data (500 rows)
│   └── reviews.csv           raw review data (8,240 rows)
└── screenshots/
    ├── overview_page.png
    ├── trend_analysis_page.png
    └── model_view.png
```

## Data model

A star schema built in Import mode:

```
        dim_date
            ↓
apps  →  reviews
```

| Table | Type | Grain | Key |
|---|---|---|---|
| `reviews` | Fact | One row per review | `review_id` |
| `apps` | Dimension | One row per app | `app_id` |
| `dim_date` | Date dimension (DAX calculated table, **marked as date table**) | One row per day | `Date` |

Relationships (both one-to-many, single-direction filtering):

- `apps[app_id]` (1) → `reviews[app_id]` (*)
- `dim_date[Date]` (1) → `reviews[posted_at]` (*)

`dim_date` is created with DAX and covers full calendar years from the first to the last review date (January 1, 2018 to December 31, 2024), which time-intelligence functions need:

```DAX
dim_date =
ADDCOLUMNS (
    CALENDAR (
        DATE ( YEAR ( MIN ( reviews[posted_at] ) ), 1, 1 ),
        DATE ( YEAR ( MAX ( reviews[posted_at] ) ), 12, 31 )
    ),
    "Year", YEAR ( [Date] ),
    "Quarter", "Q" & QUARTER ( [Date] ),
    "Month Number", MONTH ( [Date] ),
    "Month Name", FORMAT ( [Date], "MMM" ),
    "Year-Month", FORMAT ( [Date], "YYYY-MM" ),
    "Month Start", DATE ( YEAR ( [Date] ), MONTH ( [Date] ), 1 )
)
```

`Month Name` is sorted by `Month Number`. Auto date/time is turned off, so all date logic goes through `dim_date`.

## Data cleaning (Power Query)

The applied steps, in order:

**apps**

| Problem | Step | Result |
|---|---|---|
| Extra spaces in `app_name`, `developer` | `Text.Trim` | Applied. The provided file had no leading or trailing spaces, so 0 values changed. |
| Missing developer names (empty string) | Replace `""` → null → `"Unknown Developer"` | 25 apps |
| Mixed capitalization in `category_name` | Capitalize Each Word (`Text.Proper`) | 14 consistent categories (e.g. `REVIEWS AND RATINGS` → `Reviews And Ratings`, `SEO` → `Seo`) |
| Data types | `app_id` whole number, `launch_date` date, `monthly_price_usd` decimal, text for the rest | 198 free-plan apps have no price (null) |

**reviews**

| Problem | Step | Rows |
|---|---|---|
| Raw import | | 8,240 |
| Duplicate review records | Remove duplicates on `review_id` (keeps the first occurrence) | −240 → 8,000 |
| Invalid ratings (0 and 6) | Keep rows where 1 ≤ `rating` ≤ 5 | −20 (10 zeros, 10 sixes) → **7,980** |
| Mixed date formats in `posted_at` | Change type **using locale English (United States)** | 6,387 `YYYY-MM-DD` + 1,593 `MM/DD/YYYY` values, all parsed |
| `has_developer_reply` is Yes/No | Replace Yes → 1, No → 0, then whole number | 1,979 replies |
| Data types | `review_id`, `app_id`, `rating`, `helpful_count` whole number; `posted_at` date | |

Every review links to an app (0 orphan `app_id` values). 480 of the 500 apps have at least one review.

## DAX measures

| Measure | DAX | Value (all data) |
|---|---|---|
| Total Apps | `COUNTROWS ( apps )` | 500 |
| Total Reviews | `COUNTROWS ( reviews )` | 7,980 |
| Average Rating | `AVERAGE ( reviews[rating] )` | 4.19 |
| Developer Reply % | `DIVIDE ( SUM ( reviews[has_developer_reply] ), [Total Reviews] )` | 24.8% |
| Low Rating % | `DIVIDE ( CALCULATE ( [Total Reviews], reviews[rating] <= 2 ), [Total Reviews] )` | 3.2% |
| Free Plan Apps % | `DIVIDE ( CALCULATE ( COUNTROWS ( apps ), apps[has_free_plan] = "Yes" ), COUNTROWS ( apps ) )` | 39.6% |
| Avg Monthly Price (Paid) | `AVERAGE ( apps[monthly_price_usd] )` | $32.37 |
| Reviews PY | `CALCULATE ( [Total Reviews], SAMEPERIODLASTYEAR ( dim_date[Date] ) )` | 2,151 (Year = 2024) |
| YoY % | `DIVIDE ( [Total Reviews] - [Reviews PY], [Reviews PY] )` | +76.8% (Year = 2024) |
| Reviews Prev Month | `CALCULATE ( [Total Reviews], DATEADD ( dim_date[Date], -1, MONTH ) )` | 357 (Dec 2024) |
| MoM % | `DIVIDE ( [Total Reviews] - [Reviews Prev Month], [Reviews Prev Month] )` | +2.2% (Dec 2024) |
| Reviews YTD | `TOTALYTD ( [Total Reviews], dim_date[Date] )` | 3,804 (2024) |
| Reviews MTD | `TOTALMTD ( [Total Reviews], dim_date[Date] )` | 365 (Dec 2024) |

`Total Apps` counts the `apps` table, so the Category and Has Free Plan slicers filter it but the Year slicer does not. Year filters reviews only.

## Report pages

**Overview** (F-pattern layout: title and page navigator across the top, KPI row, then slicers, then detail)

- KPI cards: Total Apps, Total Reviews, Average Rating, Developer Reply %
- Slicers: Category, Year, Has Free Plan
- Line chart: Total Reviews by `dim_date[Year-Month]`
- Horizontal bar chart: Total Reviews by category
- Table: top apps by Total Reviews with Average Rating and Developer Reply %
- Page navigator to Trend Analysis

**Trend Analysis** (Year slicer, single-select, set to 2024)

- KPI cards: Total Reviews, Reviews PY, YoY %, Reviews YTD, Reviews MTD
- Line chart: Total Reviews vs Reviews PY by month
- Column chart: Reviews YTD running total by month
- Table by month: Total Reviews, Reviews Prev Month, MoM %, Reviews PY, YoY %, Reviews YTD
- Page navigator back to Overview

In 2024, monthly reviews were above the same month of 2023 every month, by between +42.7% (August) and +122.0% (January).

## Maintaining and extending the report

- **Refreshing with new data:** replace `data/apps.csv` and `data/reviews.csv` with files that have the same columns, then click **Refresh**. `dim_date` extends automatically to cover the new review dates. If the files move, update the file paths in Power Query (Transform data → Data source settings).
- **New dirty values:** the cleaning steps are in Power Query (Transform data). New invalid ratings, duplicate IDs, blank developers and Yes/No values are handled automatically. A new date format would need its own parsing step before *Changed Type with Locale*.
- **New measures:** build them on the base measures (`[Total Reviews]`, `[Average Rating]`). For time comparisons, always use `dim_date[Date]` rather than `reviews[posted_at]`, so the date table marking works.
- **Ideas for extending it:** a developer page (reply rate and rating by developer), rating by launch-year cohort using `apps[launch_date]`, and a Helpful-weighted rating using `helpful_count`.
