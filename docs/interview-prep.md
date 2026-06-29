# 💬 Interview Preparation Q&A

This document compiles **40 interview questions and answers** modeled after technical and architectural evaluations for Senior Power BI Developers, Data Analysts, and Solution Architects. The answers draw directly on the design patterns and features of the **Tesla Employee Insights Dashboard**.

---

## 📂 Table of Contents
1. [Power BI Core (Q1–Q5)](#1-power-bi-core)
2. [DAX Reference & Logic (Q6–Q10)](#2-dax-reference--logic)
3. [Data Modeling Architecture (Q11–Q14)](#3-data-modeling-architecture)
4. [Power Query (M) ETL (Q15–Q18)](#4-power-query-m-etl)
5. [Relationship Rules & Filters (Q19–Q21)](#5-relationship-rules--filters)
6. [Dynamic Filtering Mechanics (Q22–Q26)](#6-dynamic-filtering-mechanics)
7. [Evaluation Context Depth (Q27–Q29)](#7-evaluation-context-depth)
8. [Filter Context Dynamics (Q30–Q32)](#8-filter-context-dynamics)
9. [Performance Tuning & Optimization (Q33–Q35)](#9-performance-tuning--optimization)
10. [Dashboard Design & Visual UX (Q36–Q38)](#10-dashboard-design--visual-ux)
11. [Business & HR Analytics Value (Q39–Q40)](#11-business--hr-analytics-value)

---

## 1. Power BI Core

### Q1: What is the primary function of a tooltip page in Power BI, and how is it used differently in this dashboard compared to standard reports?
**Answer:** A tooltip page is designed to render secondary visual details when a user hovers over a data point on a main chart. In this dashboard, we repurpose a Tooltip page (`page 566b5382aa4b291687e4`) as a collapsible **Slicer Panel**. By attaching this page to a "Select Slicers" action button via bookmark navigation, we isolate the filter controls off-screen, maximizing canvas space for data visualization while preserving selection persistence through sync groups.

### Q2: What is a syncGroup in Power BI, and why is it essential for the Tesla dashboard's custom filter architecture?
**Answer:** A syncGroup enables slicers across different report pages to synchronize their selection states. In this dashboard, the slicer visual `f6e1c1f5d72196d58ca4` on the main page and `8c3ef5410d5442222e96` on the tooltip slicer panel share the syncGroup `"slicer"`. Without this syncGroup, a selection made in the slicer panel (e.g., filtering for a specific department) would reset or disconnect when the user closes the panel and returns to the main page.

### Q3: Explain what `drillFilterOtherVisuals: true` accomplishes in a visual's metadata, and how it impacts user experience.
**Answer:** When `drillFilterOtherVisuals` is set to `true`, it tells the Power BI query engine to propagate selection filters from that visual to all other charts on the page. In the Tesla dashboard, every chart visual has this enabled, creating an interconnected cross-filtering experience. For instance, clicking "Yes" on the OverTime donut chart automatically restricts the filter context to overtime-active employees across the attrition pie chart, salary bar chart, and hiring trend line chart.

### Q4: How is dynamic button formatting achieved in Power BI, and how does this dashboard implement it?
**Answer:** Dynamic formatting is achieved by binding a visual property (like font color or outline color) to a DAX measure instead of a static hex value. In this dashboard, the Action Button inside `Group 3` binds its Font Color and Outline properties to the `button text color` measure. This measure dynamically returns `#5D14C3` (dark purple) when filters are active and `#AC8AE0` (light purple) when no filters are applied, providing instant visual feedback.

### Q5: What is the difference between an Action Button and a Bookmark Navigation button in Power BI?
**Answer:** A Bookmark Navigation button is a specialized type of button that navigates between saved states of a report's layout, visibility, and filter properties. An Action Button is a general-purpose element that can be programmed to run custom actions, such as navigating pages, opening URLs, or executing drill-through actions. In the Tesla dashboard, the filter panel uses bookmark triggers, while the filter counter uses an action button displaying the dynamic `Slicer count` measure.

---

## 2. DAX Reference & Logic

### Q6: Why is `CALCULATE` often called the most important function in DAX, and how does it manipulate filter context?
**Answer:** `CALCULATE` is the only function in DAX that can modify the filter context of a calculation. It does this by evaluating its filter arguments, applying them to the data model (either adding new filters, overriding existing ones via `KEEPFILTERS` or `USERELATIONSHIP`, or removing filters via `ALL` or `REMOVEFILTERS`), and then executing the main expression under this newly modified context.

### Q7: Compare Row Context and Filter Context in DAX. Give an example of each from the Tesla dashboard.
**Answer:** 
*   **Row Context** is the concept of a "current row" during iteration. It exists when creating a calculated column or inside an iterator function (like `SUMX`, `MAXX`, or `FILTER`). Example: When the clustered bar chart renders the `Salary Range` bins, it evaluates the visual's internal row context line-by-line.
*   **Filter Context** is the set of all active filters applied to the data model. Example: If the user selects "California" in the State slicer and "Yes" in the Attrition pie chart, the filter context is `Employee[State] = "California" && Employee[Attrition] = "Yes"`. All measures are evaluated under this global context.

### Q8: What does the `SWITCH` function do in DAX, and when is it preferred over nested `IF` statements?
**Answer:** `SWITCH` evaluates an expression against a list of values and returns one of multiple possible result expressions. It is preferred over nested `IF` statements because it is easier to read, write, and maintain, and the DAX engine can optimize its evaluation paths.
*   *Nested IF*: `IF(a=1, "x", IF(a=2, "y", "z"))`
*   *SWITCH*: `SWITCH(a, 1, "x", 2, "y", "z")` or `SWITCH(TRUE(), a=1, "x", a=2, "y", "z")`.

### Q9: Explain the function of `SELECTEDVALUE` in DAX. What value does it return if multiple selections are active?
**Answer:** `SELECTEDVALUE` returns the value of a column if only one distinct value is visible in the current filter context. If no values or multiple values are selected, it returns a default blank (or an optional alternative value specified as the second parameter). In our `Slicer_Visibility_Control` measure, `SELECTEDVALUE('Slicer_tabel'[slicer])` is used to capture the single active category row of the disconnected table.

### Q10: How would you write a DAX measure to count the active selections on a column while ignoring a default "All" state?
**Answer:** You can use `ISFILTERED` combined with a count of selected values. For example:
```dax
ActiveDepartmentCount = IF(ISFILTERED('Employee'[Department]), COUNTROWS(VALUES('Employee'[Department])), 0)
```
In the Tesla dashboard, the `Slicer count` measure uses a series of `ISFILTERED` checks on the 7 slicer columns to count how many distinct categories have active selections.

---

## 3. Data Modeling Architecture

### Q11: What is a Star Schema, and why is it considered the best practice for Power BI data modeling?
**Answer:** A Star Schema is a data modeling pattern where a central **Fact Table** (containing quantitative measurements/records) is surrounded by **Dimension Tables** (containing descriptive lookup attributes) connected via relationships. It is the best practice for Power BI because the VertiPaq database engine is designed to optimize single-direction relationship traversal from dimension tables to fact tables, maximizing query speed and compression ratios.

### Q12: How does a "Disconnected Table" pattern work, and what role does it play in the Tesla dashboard?
**Answer:** A disconnected table is a table in the data model that has no physical relationships (active or inactive) to any other tables. It does not propagate filters through the model by default. Instead, its contents are referenced in DAX measures using functions like `SELECTEDVALUE` or `TREATAS`. In the Tesla dashboard, the `Slicer_tabel` is disconnected and serves as a UI registry to dynamically control slicer panel visibility and styling based on user interaction.

### Q13: Identify the Fact and Dimension tables in the Tesla Employee Insights project.
**Answer:**
*   **Fact Table**: `Employee` (contains raw employee IDs, salaries, hiring dates, and transactional attributes).
*   **Dimension Tables**: `Months` (date calendar mapping), `EducationLevel`, `PerformanceRating`, `RatingLevel`, and `SatisfiedLevel` (descriptive lookups).
*   **Disconnected Control Tables**: `Slicer_tabel`, `slicer img`, and `selection`.

### Q14: What is granularity in data modeling, and how does it affect report performance?
**Answer:** Granularity refers to the level of detail or depth represented by a single row in a table. In our `Employee` table, the granularity is at the individual employee level (one row per employee). Higher granularity (more detail, like hourly time logs) increases row count and data density, which increases memory consumption and visual rendering times. Designing lookup dimensions helps keep fact tables thin and fast.

---

## 4. Power Query (M) ETL

### Q15: What is the M language in Power BI, and how does it differ from DAX?
**Answer:** M (M Formula Language) is the functional query language used inside Power Query to perform **ETL (Extract, Transform, Load)** operations during data ingestion. DAX is a formula language used inside the data model to perform **analytical calculations** and aggregations on top of loaded data. M is evaluated at data refresh time, while DAX is evaluated dynamically at query/render time.

### Q16: How would you implement a "Salary Range" conditional column in Power Query?
**Answer:** In Power Query, you can create a custom column using conditional logical branches. For example:
```powerquery
Table.AddColumn(#"PriorStep", "Salary Range", each 
    if [Salary] >= 70000 then "70k+" 
    else if [Salary] >= 60000 then "60k-70k"
    else if [Salary] >= 50000 then "50k-60k"
    else if [Salary] >= 40000 then "40k-50k"
    else "30k-40k", type text)
```
In the Tesla dashboard, this column was created in Power Query to segment employee pay into the standard visual bands.

### Q17: What is Query Folding in Power Query, and why is it important for database sources?
**Answer:** Query Folding is the process where Power Query translates M transformation steps into a single SQL query (or native query) and pushes it back to the source database server for execution. This is critical because database servers are optimized for data processing, reducing the amount of data transferred over the network and speeding up the refresh pipeline.

### Q18: How would you clean up duplicate employee records in Power Query while maintaining the latest record?
**Answer:** 
1.  Sort the table by the employee record identifier and the timestamp/date in descending order (`Table.Sort`).
2.  Select the employee ID column, right-click, and select "Remove Duplicates" (`Table.Distinct`).
Power Query preserves the first row it encounters, which represents the latest record because of the preceding sort step.

---

## 5. Relationship Rules & Filters

### Q19: Explain the difference between Active and Inactive relationships in Power BI.
**Answer:** An **Active** relationship is the default path through which filter context propagates between two tables. An **Inactive** relationship exists in the model but does not filter data unless explicitly activated inside a DAX measure using the `USERELATIONSHIP` function. Power BI allows only one active relationship between any two tables at a time to prevent filter path ambiguity.

### Q20: What is relationship cardinality, and how does it affect data integrity?
**Answer:** Cardinality defines the relationship type between tables based on the uniqueness of key columns:
*   **One-to-Many (`1:*`)**: Standard lookup design (e.g., one Calendar row maps to many Employee hire dates).
*   **Many-to-Many (`*:*`)**: Used when multiple rows in both tables map to each other. Many-to-many relationships can lead to ambiguous filter paths and unexpected aggregations, requiring careful design and often a bridge table.

### Q21: Why is Single-Direction filtering preferred over Bi-Directional filtering?
**Answer:** Single-direction filtering restricts context propagation from the "One" side of a relationship to the "Many" side. Bi-directional filtering allows filters to flow both ways. While bi-directional relationships seem convenient, they significantly degrade performance and can create recursive filter loops, resulting in incorrect data calculations.

---

## 6. Dynamic Filtering Mechanics

### Q22: Explain the end-to-end mechanism of how selecting "Department" in the Advanced Slicer reveals the Department dropdown slicer.
**Answer:** 
1.  The user clicks the "Department" row in the Advanced Slicer (sourced from `Slicer_tabel[slicer]`).
2.  This selection updates the active row value of the `selection` tracker table.
3.  The measure `Slicer_Visibility_Control` is evaluated for the Department dropdown slicer.
4.  Because the selected row matches the visual's target parameter, the measure returns `1`.
5.  Since the Department dropdown has a visual-level filter set to `Slicer_Visibility_Control = 1`, it renders on the canvas. All other dropdowns return `0` and are hidden.

### Q23: Why do we place slicers on a Tooltip page instead of using a bookmark overlay on the main page?
**Answer:** Using a Tooltip page isolates the slicer panel rendering scope. If we used bookmarks with hide/show toggles on the main page, we would have to manage overlapping visuals, layout conflicts, and bookmark actions for every possible panel state. The tooltip approach encapsulates all filtering components on a single canvas, keeping the main page definition clean and lightweight.

### Q24: What happens to active filter selections when the Slicer Panel tooltip is closed by the user?
**Answer:** Because the slicers on the tooltip page are configured with **syncGroup "slicer"**, their selection states are preserved in memory and synchronized with the hidden sync engine on the main page. When the tooltip page closes, the filter context remains active on the model, and all charts continue to render filtered data.

### Q25: How does the Active Filter Counter change its outline and shadow colors dynamically?
**Answer:** The outline and shadow properties of the indicator shapes in `Group 3` are bound to the `button text color` measure. This measure checks the `Slicer count` value. If the count is greater than 0, it returns `#5D14C3` (dark purple), causing the borders and shadows to change color and highlight the active filter state.

### Q26: What is a disadvantage of using custom dynamic filtering systems, and how did you address it?
**Answer:** The primary disadvantage is the potential processing overhead from executing multiple DAX conditional formatting rules simultaneously. We addressed this by keeping the control tables small (under 10 rows) and using metadata-only DAX functions like `ISFILTERED` instead of computing tables at runtime, keeping performance fast.

---

## 7. Evaluation Context Depth

### Q27: Define "Context Transition" in DAX. How is it triggered, and what is its impact?
**Answer:** Context Transition is the process where a Row Context is transformed into a Filter Context. It is triggered when a measure is reference-called inside a row iteration (e.g., inside `SUMX`, `FILTER`, or a calculated column), or when using an explicit `CALCULATE` wrapper. This transition forces the engine to evaluate the calculation under a filter context defined by all columns of the current iteration row.

### Q28: How does context transition affect the evaluation of the `Slicer_Visibility_Control` measure?
**Answer:** When the Advanced Slicer renders its categories, it iterates over the `Slicer_tabel` rows. This iteration creates a row context. By referencing the `selection` state table inside `Slicer_Visibility_Control`, context transition is triggered, transforming the current row's category name into a filter context to evaluate whether the category should be shown (1) or hidden (0).

### Q29: What is the difference between `ALL` and `ALLEXCEPT` in DAX context manipulation?
**Answer:** 
*   `ALL` clears all filters applied to a table or column, regardless of their source.
*   `ALLEXCEPT` clears all filters on a table *except* those applied to specific columns named in the function parameters.
In HR dashboards, `ALL` is commonly used to calculate company-wide baseline averages (e.g., overall attrition rate) to compare against department-specific metrics.

---

## 8. Filter Context Dynamics

### Q30: If a user filters by "California" using the State slicer, what is the exact filter context applied to the "Monthly Employee Hiring Trend" line chart?
**Answer:** The filter context applied to the line chart consists of:
1.  `Employee[State] = "California"` (propagated from the State slicer via syncGroup).
2.  The specific month coordinate on the line chart's X-axis (e.g., `Employee[HireDate].[Month] = "March"`).
Under this combined context, the visual evaluates the measure `Count of EmployeeID`.

### Q31: How can you prevent a visual from responding to specific page slicers in Power BI?
**Answer:** You can use the **Edit Interactions** feature on the Format tab of the ribbon. By selecting a slicer and clicking the "None" icon on target visuals, you can prevent those visuals from filtering when that slicer is changed. Alternatively, you can use DAX measures wrapped in `REMOVEFILTERS` or `ALL`.

### Q32: What is the difference between `KEEPFILTERS` and standard filter arguments inside `CALCULATE`?
**Answer:** Standard filter arguments inside `CALCULATE` replace any existing filters on those columns in the filter context. Wrapping the filter argument in `KEEPFILTERS` tells the engine to intersect the new filter with the existing filters instead of overriding them, preserving both filter constraints.

---

## 9. Performance Tuning & Optimization

### Q33: How does the VertiPaq engine compress data, and how can you design a model to maximize this compression?
**Answer:** The VertiPaq engine uses a columnar database design that compresses data using value encoding, dictionary encoding, and run-length encoding (RLE). To maximize compression:
*   Avoid importing high-cardinality columns (e.g., detailed description text, raw timestamps).
*   Split datetime columns into separate date and time fields.
*   Convert floating-point columns to integers where possible.
*   Keep fact tables narrow by offloading descriptive fields to dimension tables.

### Q34: What is the Performance Analyzer in Power BI Desktop, and how did you use it?
**Answer:** The Performance Analyzer is a built-in diagnostic tool that records the time (in milliseconds) required to update or render each visual on a page. It splits visual load time into three components: **DAX Query**, **Visual Display**, and **Other** (system overhead). We used it to profile the rendering time of the clustered bar charts under dynamic filtering, verifying that the dynamic filter system responds in under 3 seconds.

### Q35: Why did we split the Salary Distribution into two separate clustered bar charts instead of using one visual?
**Answer:** Using a single bar chart would require plotting all salary ranges (30k to 70k+) on a single visual. Because the 70k+ executive range has a much smaller count compared to the entry-level ranges, a single visual would make the 70k+ bar appear visually insignificant. Splitting the visual into two charts (one for standard ranges with a gradient fill, and one for the 70k+ band) creates clear visual emphasis and optimizes rendering paths for each group.

---

## 10. Dashboard Design & Visual UX

### Q36: Describe the color strategy used in the Tesla Employee Insights dashboard and why it is effective.
**Answer:** The dashboard uses a tailored purple color palette: primary deep purple (`#5E14C6`), secondary light purple (`#AC8AE0`), and dynamic highlights (`#5D14C3`). This color strategy is effective because:
1.  It avoids default primary colors, creating a premium look.
2.  It uses color semantic consistency: in the attrition pie chart and overtime donut chart, deep purple represents the primary category (`No` attrition, `Yes` overtime), while light purple represents the secondary category, making the dashboard easier to interpret at a glance.

### Q37: How do visual borders and glows improve user experience on interactive reports?
**Answer:** Visual borders and glows serve as visual cues that indicate interactivity. By applying subtle purple shadows and glows to interactive shapes (like the "Select Slicers" button and "Active Filters" counter), we signal to users that these elements are clickable, improving the dashboard's usability.

### Q38: What are the layout dimensions of the main page, and why is "Fit to Page" display mode chosen?
**Answer:** The main page layout uses standard `1280 × 720 px` dimensions (16:9 aspect ratio). "Fit to Page" display mode is chosen to ensure the entire dashboard scales dynamically to fit the user's screen without requiring horizontal or vertical scrolling, preserving the dashboard's clean layout.

---

## 11. Business & HR Analytics Value

### Q39: What is employee attrition churn, and how does this dashboard help HR calculate the financial impact of employee departures?
**Answer:** Attrition churn is the rate at which employees leave an organization and are replaced by new hires. Departures are costly due to recruitment expenses and lost productivity. By cross-filtering the attrition pie chart with the salary bar chart, HR can identify if departures are concentrated in high-salary roles (greater financial impact) or entry-level positions, helping estimate recruitment budgets.

### Q40: How does correlation analysis between overtime and salary distributions help optimize manufacturing shifts?
**Answer:** In manufacturing organizations like Tesla, heavy overtime is common. By cross-filtering overtime ("Yes") with salary ranges, HR can identify if overtime is concentrated among lower-paid assembly technicians or higher-paid supervisors. If overtime is concentrated in lower bands, it suggests hiring more shift workers is more cost-effective than paying overtime premiums, optimizing shift planning.
