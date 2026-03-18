## Dataset Overview
- **Source:** [DataCo SMART SUPPLY CHAIN FOR BIG DATA ANALYSIS](https://www.kaggle.com/datasets/shashwatwork/dataco-smart-supply-chain-for-big-data-analysis) (via Kaggle)
- **Domain:** E-commerce / Supply Chain Management
- **Scale:** 180,519 rows × 53 Columns
- **Key Metrics Evaluated:** On-Time In-Full (OTIF) rate, Order Status lifecycle, Late Delivery Risk, and Revenue Leakage.

<p align="center">
  <img src="https://github.com/user-attachments/assets/1c018db1-8e3c-4076-888c-8c3ed5e87cbf" width="49%" alt="SLA Control Tower" />
  <img src="https://github.com/user-attachments/assets/0a4dee24-f0c4-4827-9b4c-80fb254c5c38" width="49%" alt="Commercial Integrity" />
</p>

## Deep Dive: Critical Thinking & Problem Solving Log

A dashboard is only as reliable as the logical foundation beneath it. Throughout this project, my core philosophy was to **never blindly trust default BI behaviors**. I deliberately challenged the tool's mechanics and applied rigorous supply chain operations logic to ensure the data accurately reflected physical realities. 

Here is a detailed breakdown of my analytical approach, from backend data engineering to frontend visual diagnostics.

---

### Part 1: Python Data Engineering (The Backend Architecture)

**The Trap: The Denormalized "Frankenstein" File**
The raw dataset was a massive flat file containing 180,519 records across 53 columns. Inexperienced analysts often load this directly into a BI tool, resulting in sluggish performance, data redundancy, and nightmare maintenance.

**My Approach: Architecting a Star Schema via Pandas**
Instead of brute-forcing the visualization, I utilized Python (`pandas`) to deconstruct the flat file into a highly optimized Star Schema. 

* **Action:** Extracted entities into 3 Dimension tables (`Dim_Customer`, `Dim_Product`, `Dim_Geography`), leaving only measurable metrics and foreign keys in a streamlined 13-column `Fact_Orders` table.
* **The "Aha!" Moment (Surrogate Key Engineering):** I noticed the geographical data was granular but lacked a unique identifier, risking severe many-to-many relationship errors in Power BI. To prevent this, I engineered a composite Surrogate Key (`Geo_Route_Id` = `Order City` + `Order Country`), generating 3,665 unique shipping routes.
* **Quality Assurance (Data Sanity):** Before exporting to CSV, I refused to trust the join logic blindly. I wrote a custom Python function (`check_orphans`) to validate referential integrity, mathematically proving there were exactly 0 orphaned records and 0 logical shipping errors (negative shipping days). 

<img width="676" height="574" alt="image" src="https://github.com/user-attachments/assets/e055f67d-db6f-4f9a-80a1-6d323797b9bc" />

<img width="853" height="336" alt="image" src="https://github.com/user-attachments/assets/796a608c-f0a5-4ceb-8f30-82a370d26b7d" />

---

### Part 2: Power BI Modeling & Visual Diagnostics (The Frontend)

BI tools are powerful, but their default settings often defy business logic. My goal was to force the tool to tell the truth about the supply chain.

#### 1. The "Circular Dependency" Funnel Trap
* **The Problem:** When visualizing order leakage, the Funnel Chart defaulted to sorting `Order Status` by volume. This placed `COMPLETE` at the top and `PROCESSING` in the middle—a chronological impossibility in real-world fulfillment.
* **The Critical Thought:** A funnel is entirely useless if it doesn't represent the true directional flow of an order lifecycle (from creation to delivery or cancellation).
* **The Technical Fix:** I designed a custom lifecycle sequence (1: PENDING to 9: SUSPECTED_FRAUD). However, attempting to sort the text column by this new numeric column via a DAX Calculated Column threw a fatal **"Circular Dependency"** error due to infinite evaluation loops in the semantic model. 
* **The Engineering Solution:** Instead of compromising the UI, I shifted the logic upstream. I utilized Power Query (M-Code) to hardcode the `Status_Sort_Order` during the ETL phase, completely bypassing the DAX evaluation loop.

<img width="1005" height="391" alt="image" src="https://github.com/user-attachments/assets/d25e8ff9-f278-47aa-a0ba-0a463e85c2b4" />


* **The Impact:** The funnel now flowed correctly. This chronological accuracy instantly exposed a critical ~4% pipeline leakage (Fraud & Cancellations) sitting at the bottom of the funnel.

<img width="1276" height="717" alt="image" src="https://github.com/user-attachments/assets/15305db3-95fd-43b3-89bd-2252fa5964e6" />


#### 2. The Heatmap "Small Sample Size" Illusion
* **The Problem:** When building the OTIF Risk Matrix (crossing `Category Name` with `Shipping Mode`), several intersections flashed bright red, indicating a 0% OTIF rate.
* **The Critical Thought:** A 0% OTIF rate is alarming, but if that category only shipped 2 units all year, it is a statistical anomaly, not a systemic supply chain failure. Reacting to this would waste C-level resources.
* **The Technical Fix:** I layered the User Experience (UX). I kept the matrix visually clean—showing only pure OTIF percentages with Red/Green conditional formatting. However, I engineered the Tooltips to secretly carry the `[Total_Shipment_Volume]` and `[Sum of Sales]` metrics.
* **The Impact:** Stakeholders can spot a "red zone" instantly, but upon hovering, they immediately see the financial weight of that failure, ensuring operational intervention is prioritized by actual revenue impact.

<img width="515" height="192" alt="image" src="https://github.com/user-attachments/assets/a5f1d2c6-8757-4066-9d07-d182598f7a06" />


#### 3. Isolating the Bottleneck (Decomposition Logic)
* **The Problem:** A system-wide OTIF rate of 43% is too broad to act upon. 
* **The Critical Thought:** Systemic failures are rarely truly global; they are usually localized bottlenecks masquerading as global issues due to aggregated data.
* **The Technical Fix:** I deployed an AI-driven Decomposition Tree, anchoring it with a strict `Late_delivery_risk = 1` filter to isolate only the failures. I then systematically drilled down: from 98,977 total late orders -> Shipping Mode -> Region -> Product.
* **The Impact:** The tree mathematically proved the failure wasn't global. Over 41% of the delays were isolated entirely within the **Standard Class** routing, heavily anchored in the **Central America** region. This pivoted the hypothetical business strategy from "our whole system is broken" to an actionable directive: "audit Central American Standard Class freight vendors."

<img width="1029" height="738" alt="image" src="https://github.com/user-attachments/assets/ccba1068-d1b5-4dec-a348-7d3490f22e3a" />

## Strategic Recommendations (Action Plan)

Finding the root cause is only half the job. Based on the dashboard data, here is my practical action plan to fix the delivery delays and stop the money leaks:

### 1. Fixing the Late Deliveries (Central America Bottleneck)
*The Data: The Decomposition Tree shows "Standard Class" drives the highest volume of late deliveries (over 41%). However, cross-referencing with the Heatmap Matrix reveals that simply upgrading orders to "Second Class" is not a silver bullet, as it also frequently underperforms (red zones).*

**My Recommendations:**
* **Action 1 - Question the Whole Regional Network:** The Heatmap shows it's not just "Standard" shipping that fails; "Second Class" is also struggling in Central America. The whole region is a bottleneck. We need an urgent meeting with our shipping partners there to ask directly: "Are you short on trucks, or are packages stuck at customs?" and demand a real fix.
* **Action 2 - Smart Upgrades (Don't Waste Money):** We shouldn't blindly pay more to upgrade an order to "Second Class" just because it's at risk. If "Second Class" is also failing, we just lose money and still deliver late. Instead, we should tell our system to look at the Heatmap: ONLY upgrade a shipping method if the new, faster option is actually showing a "Green" (good) success rate for that specific item. 

### 2. Stopping the Money Leaks (Fraud and Cancellations)
*The Data: Looking at the bottom of the Funnel Chart, we lost about 4,000 orders to "Suspected Fraud" and another 4,000 because "Customers Canceled".*

**My Recommendations:**
* **Action 1 - Block Fake Orders at the Door:** We are wasting time processing fake orders only to cancel them later. We need to work with the IT and Payment teams to add a simple security step at checkout, like requiring an OTP (One-Time Password) for big orders or suspicious accounts. Block the bad data before it enters our system.
* **Action 2 - Pack Faster to Stop Cancellations:** Customers usually cancel when they get tired of waiting. If an order is stuck in "Pending" or "Processing" for days, they will leave. The fix is straightforward: we need to push the Warehouse team to pick and pack orders much faster. A slow warehouse directly loses us real money.
