---
{"dg-publish":true,"permalink":"/methodology-and-theory/the-shape-of-history/","title":"The Shape of History: Why We Use the Entity-Attribute-Value (EAV) Model","tags":["methodology","databases","data-modeling","digital-humanities","graph-theory"]}
---


# The Shape of History: Why We Use the Entity-Attribute-Value (EAV) Model

When historians and archaeologists first begin digitizing records, we naturally default to the "wide" format. We open a spreadsheet, put a person's name in the first column, and then add a new column for every single fact we want to track: *Year of Birth, Ship Arrived On, Acres Farmed, Lashes Received, Number of Sheep.*

For a small, uniform dataset, this works perfectly. But historical data is rarely uniform. It is incredibly sparse and highly varied. 

If we attempt to build a comprehensive relational database using this "wide" spreadsheet approach for early Australian Colonial records, we immediately hit a structural wall. We end up with a database table containing hundreds of columns, most of which are completely empty for any given individual. A free settler will have a `NULL` blank under "Lashes Received," and a convict will have a `NULL` blank under "Acres Farmed." 

To build a database that can handle the unpredictable complexity of the past, we must change the shape of our data. We move from a "Wide" model to a "Long" model, specifically using the **Entity-Attribute-Value (EAV)** format.

### What is the EAV Model?

The EAV model is a data structure designed specifically for highly sparse, heterogeneous data. Instead of adding a new column for every new type of fact, the EAV model restricts the database to just a few core columns. 

It breaks every historical record down into three fundamental components:
1. **The Entity:** *Who* or *what* are we talking about? (e.g., A unique ID for a specific convict or settler).
2. **The Attribute (The Fact Name):** *What* are we measuring or observing? (e.g., "Livestock_Sheep" or "Lashes_Received").
3. **The Value:** *What* is the actual measurement? (e.g., "50" or "Conditional Pardon").



### Visualizing the Difference

To understand why this is so powerful, look at how the exact same historical data is structured in both models.

**The "Wide" Table (The Spreadsheet Problem):**
If we find a new document that lists "Goats," we have to alter the entire database architecture to add a "Goats" column, leaving it blank for everyone else.

| Entity_ID | Name | Acres_Farmed | Livestock_Sheep | Lashes_Received | Pardon_Type |
| :--- | :--- | :--- | :--- | :--- | :--- |
| UID-001 | John Doe | 15 | 30 | *NULL* | *NULL* |
| UID-002 | Jane Smith | *NULL* | *NULL* | 50 | Conditional |

**The EAV Table (The Historical Solution):**
In the EAV model, the data is melted down into a narrow, highly stacked list of facts. If we discover "Goats," we simply add a new row. The database architecture never has to change.

| Entity_ID | Attribute (Fact_Name) | Value | Source_Document |
| :--- | :--- | :--- | :--- |
| UID-001 | Acres_Farmed | 15 | 1805 Land Muster |
| UID-001 | Livestock_Sheep | 30 | 1805 Land Muster |
| UID-002 | Lashes_Received | 50 | 1791 Punishment Ledger |
| UID-002 | Pardon_Type | Conditional | Old System Pardons |

### The Interface Advantage: Building "Smart Search"

Beyond solving storage problems, EAV data provides a massive advantage when building user interfaces for other researchers. 

If you build a "wide" database, querying it requires researchers to understand complex SQL joins or boolean logic. But an EAV database natively translates into highly intuitive "Smart Search" tools. Because all attributes exist in a single column, your database software (whether it is FileMaker, a custom web app, or a simple dashboard) can automatically populate a dropdown menu with every available fact. 

A non-technical researcher doesn't need to write code; they simply use the UI dropdowns to state: 
* *Find all [Entities] where the [Attribute] is "Livestock_Sheep" and the [Value] is "> 50".*

### EAV in the Wild

This is not just a theoretical computer science concept; it is actively relied upon in digital history. For example, projects such as the **Historical Data Grinder** (https://hdgrinder.ro/), a database designed to aggregate highly disparate nineteenth-century Transylvanian records, use an EAV architecture. Similarly, many digital prosopography projects rely on EAV to manage the hundreds of varying attributes tied to historical individuals. They use this architecture because it is the only way to make the information from a source fit the database, rather than constantly expanding the database in an attempt to cover every new quirk of a historical source.

### The Bigger Picture: EAV as the Bridge to Graph

As powerful as the EAV model is, it is rarely the final destination for historical analysis. It is the ideal **storage and extraction layer**. It allows our **[[The Digital Toolkit/Data Pipelines\|Data Pipelines]]** to reliably capture the isolated facts of the colonial archive.

However, historical research is ultimately about relationships, not just isolated facts. Who worked with whom? Which magistrate sentenced which convict? To truly map the human reality of the past, these isolated EAV facts must eventually be transformed into nodes and edges. 

The EAV model is the necessary translation layer that cleans the data so we can eventually build a **Graph Database**—but that is a methodology we will explore in a future post.

---
### Cite this post

If you found this methodology useful for your own work, you can cite it here:

> McLean, Mark. "The Shape of History: Why We Use the Entity-Attribute-Value (EAV) Model." *The Digital Stockade*. Published 2026-03-28. https://thedigitalstockade.github.io/methodology-and-theory/the-shape-of-history