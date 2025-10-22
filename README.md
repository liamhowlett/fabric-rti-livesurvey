# 🧪 Fabric RTI Demo for Live Surveys

**Purpose**: A real-time interactive demo designed to kick off technical workshops with live attendee input and dynamic visualisation.

---

## 📍 What Did I Create and Why?

In my Power BI "Dashboard in a Day" workshops, I wanted a way to engage attendees from the start and tailor the session to their interests and experience. This demo collects live input via Microsoft Forms and visualises it instantly in a dashboard using Real-Time Intelligence (RTI) in Microsoft Fabric. It makes for an engaging and stimulating demo, and is especially useful for presenters to quickly understand the needs and experience levels of large audiences, helping shape the session dynamically.

https://github.com/user-attachments/assets/e99298cc-7c48-4e77-ac28-c0f7a7a1effb

*Demo: Recording of the solution in action*

---

## 🧠 Architecture and Components

1. Attendees fill out a Microsoft Form with questions about their experience and interests
2. A Logic App is triggered upon submission
3. The Logic App pushes the data into a Fabric Eventstream
4. The Eventstream processes and streams the data into a KQL database in an Eventhouse
5. The data is visualised in real time on a RTI Dashboard in Fabric

<img width="675" height="229" alt="image" src="https://github.com/user-attachments/assets/b45e47d4-50f8-4b2e-baf0-261b0725db7a" />

*Diagram: End-to-end flow of the RTI demo*

---
## 🛠️ Pre-requisites

- An Azure subscription to be able to spin up:
  - A Fabric Capacity
  - Logic Apps
- Microsoft Forms

> [!NOTE]
> You can of course replace a Logic App with Power Automate Premium, and both those technologies can accept different collection methods instead of a Microsoft Form if desired.

---

## 🛠️ How to Recreate It

### 🔹 Step 1: Create the Form
- In Microsoft Forms, create a new Form with questions that you would like to ask your attendees. For example, in my Dashboard in a Day, I have a mix of multiple choice and free text questions:
  - Overall experience of Power BI
  - How experienced are you at building reports?
  - How experienced are you at modelling data?
  - How experienced are you in Excel?
  - Do you have any specific topics of interest?
  - Do you have any existing plans for how you will use today's learnings?
  - How would you describe your role?
- Make sure the Form is set to 'Anyone Can Respond' if you want attendees from outside your org to respond

> [!TIP]
> I add the QR code for the Form on my opening slide of my presentation as well as a shortened URL so people can easily access whilst I introduce the workshop

<img width="1852" height="912" alt="image" src="https://github.com/user-attachments/assets/779983f9-599f-4951-90bf-d13c978b2e21" />


### 🔹 Step 2: Create the EventHouse and KQL Database
This is where your data is going to be streamed to and persisted

- In a Fabric Workspace, select new item and create an EventHouse
- Select the KQL Database that's been created on the left hand side and click the elipsis on 'Tables' to create a new table
- Create a column using the GUI for each question/response that will be generated from the Form. You'll need these exact column names later on.

<img width="1852" height="914" alt="image" src="https://github.com/user-attachments/assets/418213bd-b8bf-4a06-93df-46a3d2a548d6" />

### 🔹 Step 3: Set Up the Eventstream
This will be the piece that takes data from Logic Apps to the Eventhouse, but we need to publish it first so we have something to connect the Logic App to

- Create a new Eventstream in Fabric.
- Select 'Use Custom Endpoint' and give it a name
- Publish the Eventstream

<img width="1849" height="912" alt="image" src="https://github.com/user-attachments/assets/bedcb9fc-acc6-4f75-981e-0aab397e45b1" />


### 🔹 Step 4: Build the Logic App

- In the Azure Portal, create a new Logic App. Open it and click 'Edit'
  - **Trigger**: Microsoft Forms - When a new response is submitted.
    - Select the Form you created earlier.
  - **Action**: Microsoft Forms - Get response details.
    - Click in 'Response ID' and click the lightning bolt to select the 'When a new response is submitted' parameter
  - **Action**: Event Hubs - Send Event
    - For the connection, you will need to select 'Access Key' as the authentication type.
    - Go back to your Fabric Eventstream, select the custom endpoint and select SAS Key Authentication. You will need to copy 'Connection string-primary key' and 'Event hub name'
    - <img width="1843" height="909" alt="image" src="https://github.com/user-attachments/assets/ccc73428-70df-4d44-a1d2-83cb37834c55" />
    - Go back to your Logic App and paste the 'Connection string-primary key' into the 'Connection String' and then click 'Create New'
    - Under 'Event Hub Name', it may show an error. You need to dropdown, select custom value and then paste the 'Event hub name' that you copied earlier
    - Then select 'Content' from the advanced parameters dropdown
    - In Content, you are now going to define the JSON that will be sent. Create a column name for each question input and then click the lighning symbol again each time to put the related dynamic parameter. For example mine looks like this (just replace <Form Parameter> with your own dynamic parameter):
```
    {
      "PBIExperience": "<Form Parameter>",
      "ReportBuildingExperience": "<Form Parameter>",
      "ModellingExperience": "<Form Parameter>",
      "ExcelExperience": "<Form Parameter>",
      "TopicsOfInterest": "<Form Parameter>",
      "LearningPlans": "<Form Parameter>",
      "Role": "<Form Parameter>"
    }
```

  - Save the Logic App

<img width="1849" height="911" alt="image" src="https://github.com/user-attachments/assets/d6d3745a-c6c7-4b56-9ab0-0f8caed02be2" />


### 🔹 Step 5: Complete Eventstream setup

- Submit a test response into your Form. We need to send it a test so the Eventstream knows the schema it's working with
- Go to your Eventstream and observe the Data Preview. Click refresh and you should see a line of responses in your Eventstream
- Go to 'Edit' mode
- Next, click on the box that says 'Transform events or add a destination' and select 'Manage fields'. This gives us the opportunity to make sure the source columns will match the destination
- Select the pencil icon on 'Manage fields' and add the fields from 'Imported Schema' that you want to stream into your Eventhouse
> [!TIP]
> Eventstream is capable of basic transformations, aggregations and prepping your data to an extent before sending to the Eventhouse. For example, if your columns have a different name, this is where you can tidy that up
- Next, hover over 'Manage fields' and click the plus sign to add an Eventhouse destination. Fill in all the required fields and select the table you created earlier
- If the dots aren't joined up on the workflow, make sure they are and then publish the Eventstream

<img width="1849" height="913" alt="image" src="https://github.com/user-attachments/assets/fb8a26cf-556b-4e07-9dc2-5d9750e3c9d9" />


### 🔹 Step 6: Create the real time dashboard
Now data is streaming successfully into your Eventhouse, it's time to visualise it. Usually we turn to Power BI for anything to do with visuals, but Power BI is not really built for true real time, so instead we're going to use an RTI Dashboard. 
> [!NOTE]
> The visual capabilities in an RTI Dashboard are relatively basic. If you are looking to use the more advanced report functionality in Power BI, but still want a real time feel, then you could consider looking at Page Auto Refresh against DirectQuery, but note that this could potentially be an expensive and inefficient solution if designed poorly.

- Open your Eventhouse and KQL Database
- Click 'Real-Time Dashboard' in the top menu bar and give it a name
- In the top right, change to 'Editing' mode
- Your datasource should already be sorted if you created it from the context of the KQL Database, so go and click on 'New tile'
- You will now be presented with a blank query pane. This uses KQL (Kusto Query Language). If you are unfamiliar with it, notice there is a Copilot button on the right to help you create your queries. Here is a very simple example of my first query:

  ```kql
  ['WorkshopQuestionnaire-KQLTable']
  | summarize countPBIExperience=count() by PBIExperience
  | sort by countPBIExperience desc
  ```

- This code creates a column for each category with a count of how many responses for that category, and then sorts it. This is great for a column chart or similar
- Once you run the query, you can then select the visual formatting and pick how you want the data to be visualised. When you're done select 'Apply changes'
- Repeat this process for the other columns that you want to visualise in your dashboard

> [!TIP]
> You can create multiple pages on this dashboard. I have a 'Total' for all events, but have created a 'Today' page so it only shows the attendee responses for today's session. You simply need to add a Where statement to your query in each visual that filters it to today:
>   ```kql
>  ['WorkshopQuestionnaire-KQLTable']
>  | where ingestion_time() >= startofday(now())
>  | summarize countPBIExperience=count() by PBIExperience
>  | sort by countPBIExperience desc
>  ```

- To enable real time, you now need to select 'Manage' and enable 'Auto refresh' in the top nav bar. 'Continuous' will give you your real time view
- Save, switch to viewing mode and send through some more test responses from the Form

### Congratulations, you have a real time survey!

