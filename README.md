# **Salesforce Meeting Notes – Implementation Documentation**

## **1. Business Requirement**

Sales reps need to:

1. Enter **meeting details** directly from an Account.
2. Capture **detailed notes** (with screenshots).
3. Record **meeting date**.
4. Select **internal attendees** (Salesforce Users).
5. Select **external attendees** (Contacts from the same Account).
6. **Send meeting notes** as an email to any address.
7. Have an **Account-level field** showing the count of meetings created in the current year.

---

## **2. Solution Approach**

We followed a **low-code / no-code first** philosophy, using Salesforce declarative tools wherever possible, with minimal custom code only where absolutely necessary.

---

## **3. Technical Implementation**

### **A. Custom Objects**

1. **Meeting\_\_c** (Master-Detail with Account)

   * Fields:

     * `Summary__c` – Text (Meeting summary)
     * `Detailed_Notes__c` – Rich Text Area (Detailed meeting notes with images)
     * `Meeting_Date__c` – Date
     * `Is_Meeting_This_Year__c` – Checkbox (TRUE if record created this year)
     * `Account__c` – Master-Detail to Account

2. **Meeting\_Attendee\_\_c** (Master-Detail with Meeting)

   * Fields:

     * `Attendee_Type__c` – Picklist (Internal / External)
     * `User__c` – Lookup to User (for internal attendees)
     * `Contact__c` – Lookup to Contact (for external attendees)
     * `Meeting__c` – Master-Detail to Meeting

---

### **B. Form for Salesperson**

* **Screen Flow** (`Meeting_Notes_Form_Screen_Flow`)

  * Launched from Quick Action `Create_Meeting_Notes` on Account.
  * Captures:

    1. Summary (Text)
    2. Detailed Notes (via custom LWC – see below)
    3. Meeting Date
    4. Multi-select for Internal Attendees (User multi-select → create Meeting\_Attendee\_\_c per User)
    5. Multi-select for External Attendees (Contact multi-select → create Meeting\_Attendee\_\_c per Contact)
    6. Email address to send notes
  * After save:

    * Creates Meeting record
    * Creates related Meeting\_Attendee\_\_c records
    * Sends an email with notes

---

### **C. Rich Text Area Capture**

* **Challenge:** Standard Screen Flow doesn’t support object Rich Text Area fields as editable inputs.
* **Solution:** Built lightweight LWC `screenFlowRichText`:

  * Allows formatted text and images/screenshots.
  * Outputs to Flow variable, stored in `Detailed_Notes__c`.

---

### **D. Tracking Meetings Created This Year**

* Requirement: “Count meeting notes entered this year” → must be dynamic with year change.
* **Why not formula?**

  * Formula like `YEAR(DATEVALUE(CreatedDate)) = YEAR(TODAY())` would work logically but formula fields are **not indexed** and can’t be used in Roll-Up Summary filters.
* **Solution:**

  * Stored checkbox `Is_Meeting_This_Year__c`:

    * Default TRUE for new meetings created in the current year.
    * Set via Flow on creation.
    * Reset annually by Scheduled Flow.

---

### **E. Scheduled Flow**

* Flow: `Update_Last_Year_Meeting_Records_Scheduled_Flow`
* Runs **daily**, but only updates records if `Account.Is_First_Day_Of_Year__c = TRUE` (formula: `IF(TODAY() = DATE(YEAR(TODAY()),01,01),TRUE,FALSE)`).
* Finds all `Meeting__c` where:

  * `Is_Meeting_This_Year__c = TRUE`
  * `CreatedDate` not in current year
* Sets `Is_Meeting_This_Year__c` to FALSE.

---

### **F. Roll-Up Summary on Account**

* Field: `Meeting_Notes_This_Year__c`
* Type: Count of `Meeting__c` child records
* Filter: `Is_Meeting_This_Year__c = TRUE`
* Updates dynamically after Scheduled Flow runs on Jan 1.

---

## **4. Why This Approach**

1. **Low-code first** – Screen Flows, Roll-Ups, and Scheduled Flows avoid Apex triggers or batches.
2. **Custom LWC only for RTA** – Only custom code is where Salesforce declarative tools fall short (rich text with images in Screen Flow).
3. **Performance & scalability** – Roll-up summary field uses indexed checkbox filter for efficiency.
4. **Automatic year reset** – No manual intervention needed to keep counts accurate year-to-year.

---

## **5. Data Flow Diagram**

**Account → Quick Action → Screen Flow (+ LWC) → Create Meeting\_\_c → Create Meeting\_Attendee\_\_c records → Send Email → Roll-Up Summary updates → Scheduled Flow resets annually**
