# lookup-and-row-based-evaluation
Lookup and Row-Based Evaluation

## Row Lookup KPI Demo
This DAX measure provides a personalized performance summary for each customer using advanced row-context operations like row lookup, aggregation, ranking, and date-based filtering. It combines customer interaction, store visit history, and product preferences to generate insights in real-time.

The measure is designed to be used in a matrix or table visualization where the customer_id is on rows. It surfaces key customer KPIs including their most visited store, first call date, last month’s satisfaction score, and top product purchased.

https://www.tiktok.com/@catalystbytes/video/7502487689837563154

```dax
Row Lookup KPI Demo = 
VAR SelectedCustomer = SELECTEDVALUE(customer_dim[customer_id])

/* ----------------------------------------
   1. LOOKUP: Most Frequent Store Name
   ----------------------------------------
   Pattern    : GROUPBY + COUNT + TOPN
   Purpose    : Find the store where the customer shopped most often.
   Use Case   : Identify primary customer location (e.g., for targeting).
   ---------------------------------------- */
VAR MostVisitedStoreID =
    CALCULATE(
        MAXX(
            TOPN(
                1,
                SUMMARIZE(
                    FILTER(sales_fact, sales_fact[customer_id] = SelectedCustomer),
                    sales_fact[store_id],
                    "VisitCount", COUNTROWS(sales_fact)
                ),
                [VisitCount], DESC
            ),
            sales_fact[store_id]
        )
    )

VAR MostVisitedStoreName =
    LOOKUPVALUE(
        store_dim[store_name],
        store_dim[store_id], MostVisitedStoreID
    )

/* ----------------------------------------
   2. FIRST VALUE: Get First Call Date
   ---------------------------------------- */
VAR FirstCallDate =
    CALCULATE(
        MIN(call_fact[call_date]),
        FILTER(call_fact, call_fact[customer_id] = SelectedCustomer)
    )

/* ----------------------------------------
   3. ROW NAVIGATION: Previous Month Satisfaction Score
   ---------------------------------------- */
VAR SalesCurrentMonth = MAX(call_fact[call_date])
VAR SalesPreviousMonth = EDATE(SalesCurrentMonth, -1)

VAR PrevMonthScore =
    CALCULATE(
        AVERAGE(call_fact[satisfaction_score]),
        FILTER(
            call_fact,
            call_fact[customer_id] = SelectedCustomer &&
            call_fact[call_date] >= SalesPreviousMonth &&
            call_fact[call_date] < SalesCurrentMonth
        )
    )

/* ----------------------------------------
   4. RANK / INDEX: Top Selling Product by Units
   ---------------------------------------- */
VAR TopProduct =
    CALCULATE(
        MAXX(
            TOPN(
                1,
                SUMMARIZE(
                    sales_fact,
                    product_dim[product_name],
                    "TotalUnits", SUM(sales_fact[units_sold])
                ),
                [TotalUnits], DESC
            ),
            product_dim[product_name]
        )
    )

/* ----------------------------------------
   RETURN: Display KPI Summary (Line by Line)
   ---------------------------------------- */
RETURN
    "Most Visited Store: " & MostVisitedStoreName & UNICHAR(10) &
    "First Call: " & FORMAT(FirstCallDate, "yyyy-mm-dd") & UNICHAR(10) &
    "Prev Score: " & COALESCE(PrevMonthScore, "N/A") & UNICHAR(10) &
    "Top Product: " & TopProduct


```
