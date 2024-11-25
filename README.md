# **Inventory Requisition Template**

### **Dashboard Link**  
[[Click here to view the dashboard](https://app.powerbi.com/groups/me/reports/384d017e-e935-44dc-9e7d-1626c1a36de1/ReportSection)](https://ceqafoods.sharepoint.com/sites/BranchRequisitionsv2/SitePages/Branch-Requisitions-V2.aspx )

---

## **Problem Statement**

The fast-food restaurant chain operates over 80 branches, requiring efficient inventory management to ensure uninterrupted supply. However, due to a limited number of delivery vehicles, fulfilling inventory requests for all branches on the same day is not feasible. The warehouses must supply inventory to branches without relying solely on their current stock levels. A predictive system is needed to estimate the expected stock levels for each branch and forecast their inventory needs. This system must factor in variables such as the day of the week, holidays, branch category, and location.

---

## **Steps Followed**

### **Step 1: Data Loading**
The following data sources were loaded into Power BI Desktop:  
Date Dim, Request Type, Inventory Item Dim, Inventory Dim, Closest Branches, Standard Quantity, Standard Requisitions, Requisition Team, Fryer Info, Overstock Adjustments, Branch MCL Adjustment, Item MCL Adjustment, Buffer Inventory, Consumption Estimates, Product Bundle, Stock Used, Sales Data, Sales, Items Requested, Previous Day's Supply, Today's Supply, Stock Transfers, Branches Closing Stocks, Holidays, Date, MCL (Maximum Consumption Limits), Branch Dim.

### **Step 2: Data Transformation in Power Query**
1. **Merging**:
   - Merged the **Holiday** table with date-related tables for holiday-specific logic.
   - Merged the **Branch MCL** table with the **Branch Dim** table to enrich branch-level details with maximum consumption limits.
   
2. **Transformations**:
   - Filtered irrelevant rows and columns for streamlined analysis.
   - Standardized data types (dates, numbers, text) for consistency.
   - Unpivoted pivoted datasets to create tabular formats for analysis.
   - Added custom columns to compute stock surplus, deficits, and replenishment forecasts.

3. **Loading**:
   - Applied queries and loaded the transformed data into Power BI tables.

### **Step 3: Semantic Model**
1. **Fact Tables**:  
   - Sales  
   - Stock Transfers  
   - Consumption Estimates  
   - MCL Adjustments  

2. **Lookup Tables**:  
   - Branch Dim  
   - Inventory Dim  
   - Date Dim  
   - Holidays  

3. **Relationships**:  
   - Established one-to-many relationships between dimensions and facts (e.g., **Date Dim** to **Sales**).  
   - Enabled bidirectional filtering for dynamic analysis.  
   - Created hierarchies (e.g., **Date Dim**: Year > Month > Day; **Branch Dim**: Region > Branch).  

---

## **DAX Measures**

### **Today's Actual E.O.D Closing Adjusted**

This measure calculates the adjusted end-of-day closing stock for today, factoring in overstock adjustments based on the warehouse type.

```DAX
Today's Actual E.O.D Closing Adjusted = 
VAR ActualEODClosing = [Today's Actual E.O.D Closing]
VAR ActualClosingAdjusted = 
    IF(
        SELECTEDVALUE('Overstock Adjustments'[Warehouse Type]) = "Source Warehouse",
        ActualEODClosing - [Overstock Adjustment],
        ActualEODClosing + [Overstock Adjustment]
    )
RETURN 
    ActualClosingAdjusted
```

**Explanation**:  
- **`ActualEODClosing`**: Refers to the original closing stock for the day.  
- **Adjustment Logic**: Subtracts overstock adjustments for "Source Warehouse" types and adds adjustments for others.  
- This ensures accurate E.O.D stock values for predictive calculations.

---

### **Tomorrow’s MCL**

This measure predicts the Maximum Consumption Limits (MCL) for the next day by accounting for branch-level and item-level adjustments and incorporating holiday-specific logic.

```DAX
Tomorrow's MCL = 
VAR TomorrowItemMCLAdjust = 
    CALCULATE(
        SUM('Item MCL Adjustment'[MCL Adjustment]),
        'Item MCL Adjustment'[Day] = "Tomorrow's Factor"
    )

VAR BranchMCLAdjust = 
    CALCULATE(
        SUM('Branch MCL Adjustment'[MCL Adjustment])
    )

VAR ActualMCL = 
    IF(
        ISBLANK(
            CALCULATE(
                MAX('MCLs'[MCL]),
                'Date Table'[Date] = SELECTEDVALUE('Date Table'[Date]) + 1
            )
        ),
        0,
        CALCULATE(
            MAX('MCLs'[MCL]), 
            'Date Table'[Date] = SELECTEDVALUE('Date Table'[Date]) + 1
        )
    )

VAR HolidayMCL = 
    CALCULATE(
        MAX('MCLs'[MCL]), 
        FILTER(
            ALL('Date Table'),
            'Date Table'[DayOfWeekName] = "Sunday"
        )
    )

VAR TomorrowHolidayCheck = 
    CALCULATE(
        MAX('Date Table'[Holiday Name]), 
        'Date Table'[Date] = SELECTEDVALUE('Date Table'[Date]) + 1
    )

VAR TomorrowMCL = 
    IF(
        TomorrowHolidayCheck = "Working Day", 
        TomorrowItemMCLAdjust * BranchMCLAdjust * ActualMCL,
        TomorrowItemMCLAdjust * BranchMCLAdjust * HolidayMCL
    )

RETURN
IF(
    ISBLANK(TomorrowMCL),
    0,
    TomorrowMCL
)
```

**Explanation**:  
- **`TomorrowItemMCLAdjust`**: Calculates item-level adjustments for tomorrow.  
- **`BranchMCLAdjust`**: Computes branch-level adjustments.  
- **`ActualMCL`**: Retrieves the base MCL for tomorrow.  
- **`HolidayMCL`**: Substitutes MCL with Sunday-specific values if tomorrow is a holiday.  
- **`TomorrowMCL`**: Combines these factors to predict tomorrow’s total MCL.  

---

### **Tomorrow’s Inventory Supply**

This measure calculates the inventory gap for tomorrow, factoring in the predicted MCL, buffer stock, and today’s adjusted E.O.D closing stock.

```DAX
Tomorrow's Inventory Supply = 
VAR InventoryDiff = [Tomorrow's MCL + Buffer] - [Today's Actual E.O.D Closing Adjusted]
RETURN 
    IF(InventoryDiff > 0, InventoryDiff, 0)
```

**Explanation**:  
- **`InventoryDiff`**: Calculates the difference between tomorrow’s total inventory needs (MCL + Buffer) and today’s adjusted closing stock.  
- **Logic**: Ensures a positive supply requirement, defaulting to 0 if no gap exists.

---

## **Outcome**

These DAX measures and transformations provide a comprehensive system for daily inventory predictions and requisition planning, ensuring accurate stock levels while considering consumption patterns, holidays, and warehouse adjustments.

---

**Semantic Model Snapshot**  
![Semantic Model](https://github.com/user-attachments/assets/1d2c59e6-6e5e-4e6d-903b-f61280eb1b88)

**Material Request Dashboard Snapshot**  
![Material Request Dashboard](https://github.com/user-attachments/assets/3b1326a3-fa08-4bfa-b809-11ac23151daa)

---
