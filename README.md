# Nathan's Workflow

An automated n8n workflow designed to fetch order data from a company warehouse API, segment records by status, upsert active processing orders into a local data table, and broadcast a weekly financial summary of completed orders directly to Discord.

---

## Architecture & Logic Flow

```text
Manual Trigger ──┐
                 ├─► GetDataFromWarehouse ──► Check Order Status
Schedule Trigger ┘                                   │
                           ┌─────────────────────────┴─────────────────────────┐
                   status == "processing"                                  otherwise
                           │                                                   │
                  SetProcessingData                                  CalculateBookedOrders
                           │                                                   │
                   Order Data Table (Upsert)                                 Discord
```

### Execution Steps

1. **Triggers**
   - **Schedule Trigger:** Automatically runs every Monday at 09:00 UTC.
   - **Manual Trigger:** Enables instant, on-demand execution for testing and debugging.

2. **GetDataFromWarehouse (`HTTP Request`)**
   - Retrieves live order records via an authenticated endpoint (`https://learn.app.n8n.cloud/webhook/courses/n8n-quickstart/company-warehouse`).
   - Uses HTTP Header Authentication and passes the required `x-assessment-id` header.

3. **Check Order Status (`IF`)**
   - Evaluates incoming records where `orderStatus == "processing"`:
     - **Processing Branch (True):**
       - **SetProcessingData (`Edit Fields`):** Filters and standardizes fields to retain only `orderID` and `employeeName`.
       - **Order Data Table (`Data Table Upsert`):** Writes or updates rows in the internal `orders` table using `orderID` as the unique match key.
     - **Booked Orders Branch (False):**
       - **CalculateBookedOrders (`Code`):** Aggregates non-processing orders, computing the total count (`totalBooked`) and gross revenue (`bookedSum`).
       - **Discord (`Discord Webhook`):** Dispatches a structured notification reporting weekly performance:  
         > *"This week we've [X] booked orders with a total value of $[Y]"* (including the assessment ID).

---

## Prerequisites & Requirements

| Component | Type | Description |
| :--- | :--- | :--- |
| **Header Auth Credential** | n8n Credential | Configured for `GetDataFromWarehouse` to authenticate API calls. |
| **Discord Webhook** | Webhook URL | Destination channel webhook used by the `Discord` node. |
| **`orders` Data Table** | n8n Internal Table | Schema definition: `orderID` (Number), `customerID` (Number), `employeeName` (String), `orderPrice` (Number), `orderStatus` (String). |

---

## Deployment & Setup

1. **Import:** Load the workflow JSON file directly into your n8n workspace.
2. **Configure Credentials:** Link your HTTP Header Auth credentials to the `GetDataFromWarehouse` node.
3. **Set Webhook Destination:** Provide your target Discord channel webhook URL inside the `Discord` node parameters.
4. **Link Data Source:** Verify that the `orders` data table exists with matching schema columns, then select it in the `Order Data Table` node.
5. **Validate & Activate:** Execute a test run using the `Manual Trigger` to confirm data parsing and notifications, then activate the workflow to enable the recurring Monday schedule.

---

## Conclusion

This workflow delivers a clean, event-driven pattern for decoupling order processing from business intelligence reporting. By automating data ingestion, table synchronization, and cross-platform notifications, it eliminates manual reconciliation overhead and provides teams with automated, real-time operational visibility every week.
