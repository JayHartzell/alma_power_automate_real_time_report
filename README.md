# Real-Time Reporting in Alma: Two Methods

## Request 

A department within our institution that loan materials for both in-library and multi-day use requested a report that contains a list of any outstanding in-library use loans (short loans). 

The report should arrive in a shared inbox 15 minutes before closing, and list any outstanding loans from the day. 

Staff would use this report as part of their closing procedures to check-in any materials that were missed throughout the day.

### Problem

Generating a scheduled report natively in Alma requires the use of a report in Oracle Analytics, scheduled as an Analytics Object. But due to the timing of the ETL (Extract, Transform, Load) of Alma data into Analytics, producing same-day reports becomes challenging, if not impossible.

Example:

Sample Analytics data availability:

Data available as of: 11/14/2024 906 PM,PST

- The time that the load of the data to Alma Analytics was completed

Data updated as of: 11/14/2024 405 PM,PST

- The time the data was extracted from Alma.


The service desk closing time: 

- 7pm M - TH  
- 4pm F

Ideal report scheduling:

- 6:45 pm M - TH  
- 3:45 pm F

### Analysis

This is essentially impossible out-of-the box using analytics. 

You could produce a report that contains loan information from that day, but it would be constrained in the following ways:

- Only include loans up to around 4 pm  
- Only be able to be produced after 9 pm

To get the loan information from 4pm - 7 pm you have to wait until the subsequent day’s ETL. So loan data from 4pm - 7pm wouldn’t be available until the subsequent evening at 9 pm, more than 24 hours after the loan was due.

## API Solution

### Keys required:

Configuration and Administration - read-only  
Bibs - read-only

### Platform:

Power Automate

#### Summary

Create a logical set that captures the current loans for the location you want to generate a report for. Schedule an API call to retrieve set members. Use the returned link to make an API call to retrieve loan information. Use the returned loan information to populate a report and schedule it to be sent at the appropriate times.

### Process:

#### Create a logical set

In Alma, create an advanced query that produces the number of in-house items on loan at the desired service desk. Whether this is even possible and how you do it will depend on your system. We use item policies as exceptions to a default fulfillment rule, so any in-library use item at this service desk will have an empty item policy field. 

I start with a simple query capturing the desired location, a process type of loan and an empty item policy:

Set name ocn_stemlc_library_use_on_loan   
Set type Logical   
where (Current location equals ((Oceanside Library : OCN STEMLC)) AND Process type equals "Loan" AND Item policy is empty)

#### Retrieve set members

Make a GET API call to the Retrieve Set Members endpoint:

``` python:  
https://api-na.hosted.exlibrisgroup.com/almaws/v1/conf/sets/{set_id}/members?format=json&apiKey={conf_key}&limit=100  
```

You’ll be returned an array of member objects, each containing an id, description, and link.:

``` json:  
{  
  "member": [  
    {  
      "id": "2331678820005274",  
      "description": "1000501025",  
      "link": "https://api-na.hosted.exlibrisgroup.com/almaws/v1/bibs/991000010309705274/holdings/2231678880005274/items/2331678820005274"  
    },  
    {  
      "id": "2331743040005274",  
      "description": "1000501871",  
      "link": "https://api-na.hosted.exlibrisgroup.com/almaws/v1/bibs/991000029039705274/holdings/2282733230005274/items/2331743040005274"  
    },  
    {  
      "id": "2331752900005274",  
      "description": "1000499824",  
      "link": "https://api-na.hosted.exlibrisgroup.com/almaws/v1/bibs/991000016329705274/holdings/2282733020005274/items/2331752900005274"  
    }  
  ],  
  "total_record_count": 3  
}  
```

#### Get item information

We’re going to take the link value returned for each set member and make a GET API call to the bibs /loans endpoint. You do so by taking the value and appending the rest of the required url:

```/loans?&format=json&apiKey={bib_key}```

You’ll be returned an item_loan object, which we’ll use to build and populate the report.

#### Compile the item information

You’ll want to create an array to hold the data returned by each bib API call. 
You’ll append each loan object as it is processed. In Power Automate I do this in a compose statement, 
while also adjusting the datetime from UTC to PST.

``` json:
{
    "title": "@body('Parse_JSON_1')?['title']",
    "author": "@body('Parse_JSON_1')?['author']",
    "barcode": "@body('Parse_JSON_1')?['item_barcode']",
    "loan_date": "@formatDateTime(convertTimeZone(body('Parse_JSON_1')?['loan_date'], 'UTC', 'Pacific Standard Time'), 'MM/dd/yyyy HH:mm')",
    "due_date": "@formatDateTime(convertTimeZone(body('Parse_JSON_1')?['due_date'], 'UTC', 'Pacific Standard Time'), 'MM/dd/yyyy HH:mm')",
    "borrower_id": "@body('Parse_JSON_1')?['user_id']",
    "loan_ticks": "@substring(string(ticks(body('Parse_JSON_1')?['loan_date'])), 3, 4)"
}
```

You’ll be left with something like this:
``` json:
[  
    {  
        "title": "IPhone charger",  
        "author": null,  
        "barcode": "1000499840",  
        "loan_date": "11/14/2024 13:30",  
        "due_date": "11/14/2024 15:30",  
        "borrower_id": "XXXXXXX",  
        "loan_ticks": "XXXX"  
    },  
    {  
        "title": "TI-84 Calculator Charger",  
        "author": null,  
        "barcode": "1000501871",  
        "loan_date": "11/14/2024 11:33",  
        "due_date": "11/14/2024 13:33",  
        "borrower_id": "XXXXXXX",  
        "loan_ticks": "XXXX"  
    },  
    {  
        "title": "Lecture - tutorials for introductory astronomy",  
        "author": "Prather, Slater, Adams et al.",  
        "barcode": "1000501025",  
        "loan_date": "11/14/2024 12:29",  
        "due_date": "11/14/2024 14:29",  
        "borrower_id": "XXXXXXX",  
        "loan_ticks": "XXXX"  
    }  
]
```
In Power Automate we’re going to create an HTML table from this array. This is pretty much done automatically with the Create HTML Table action, but we’ll need to apply some styling to the output so it doesn’t look like hot garbage. We do this in a compose action:
``` HTML:
<!DOCTYPE html>  
<html>  
<head>  
   <style>  
 table {  
           width: 100%;  
           border-collapse: collapse;  
           font-family: Arial, sans-serif;  
       }  
       th, td {  
           border: 1px solid #444444;  
           text-align: left;  
           padding: 12px;  
       }  
       th {  
           background-color: #4A536B;  
           color: white;  
       }  
       tr:nth-child(even) {  
           background-color: #f2f2f2;  
       }  
       tr:nth-child(odd) {  
           background-color: #ffffff;  
       }  
       tr:hover {  
           background-color: #ddd;  
       }  
   </style>  
</head>  
<h2 style="text-align: center;"> Sample Report Title</h2>  
<body>  
{HTML Table body output}  
</body>  
</html>
```
After that, we schedule the email and put the above output in the body of the email.

## Webhook Approach

### Keys required:

None

### Platform:

Power Automate

### Summary

We set up a loan webhook and log the short loan transactions in a Sharepoint list. New loans are logged, and returns are deleted. Each day we send a scheduled email containing anything remaining in the list.

### Process

#### Set up a flow to accept webhook responses

Create a new Instant Cloud Flow, with Request - When an HTTP Request is Received

You can’t do anything with it yet, but you have to have an action along with the trigger. So, just do a Compose action underneath and put the output Body of the HTTP Request. Save the Flow. Edit the flow, go to the Request trigger and copy the URL.

#### Set up a webhook integration profile

Add an integration profile:  
- type webhook  
- for the webhook url use the url produced in the previous step   
- add a secret to it   
- JSON format  
- subscribe to Loans  
- Activate  
- Save  
   
Go perform a loan.

#### Setting up the Webhook

Back in Power Automate you should have a completed run showing the loan information for the loan you just performed. 

Go into the completed run, find the output of the Response object and copy it. Edit the flow, select the trigger, and use the copied payload to generate the request schema. This allows you to more easily access the values within the JSON payload in subsequent steps.

Add a switch condition.

- Switch on triggerBody()?['event']?['value']  
- Add two cases: LOAN_CREATED, LOAN_RETURNED  
- Save

Within the LOAN_CREATED case, add another switch condition.

Switch on triggerBody()?['item_loan']?['circ_desk']?['value']

- Add a case for the circ desk you want to track.  
- Save

#### Create a Sharepoint List for daily tracking

Create a simple Sharepoint list to track daily short loans.

Columns:

| column_name | column_type |
| :---- | :---- |
| loan_id | text |
| barcode | number |
| loan_date | date and time |
| due_date | date and time |
| title | text |
| author | text |

#### Logging and Reconciling

You’ll log each loan’s data as a new list item in the Sharepoint list, then delete upon Return. Navigate back into your webhook flow.

Within the Case for the circulation desk you want to log, add a condition. 

On the left side, use: 
```
formatDateTime(  
convertTimeZone(  
triggerBody()?['item_loan']?['loan_date'],  
'UTC',  
'Pacific Standard Time'  
),  
'MM/dd/yyyy'  
)
```
 is equal to 

right side:
```
formatDateTime(  
convertTimeZone(  
triggerBody()?['item_loan']?['due_date'],  
'UTC',  
'Pacific Standard Time'  
),  
'MM/dd/yyyy'  
)
```
In the True branch, we’re going to use the Add Item Sharepoint connector. I adjust the loan_date and due_date time into something more readable.


``` json
{
    "loan_id": "@triggerBody()?['item_loan']?['loan_id']",
    "barcode": "@triggerBody()?['item_loan']?['item_barcode']",
    "loan_date": "@formatDateTime(triggerBody()?['item_loan']?['loan_date'], 'MM/dd/yyyy HH:mm')",
    "due_date": "@formatDateTime(triggerBody()?['item_loan']?['due_date'], 'MM/dd/yyyy HH:mm')",
    "title": "@triggerBody()?['item_loan']?['title']",
    "author": "@triggerBody()?['item_loan']?['author']"
}
```

   
Next, add a Terminate type: succeeded Step

#### Processing Returns

Navigate back over to the LOAN_RETURNED case. Copy or create another switch, again on ```triggerBody()?['item_loan']?['circ_desk']?['value']```

Add another case for the same circulation desk. Under that case, add the Get Items Sharepoint step. Point it towards the list you created, and use the Filter Query to filter on the loan id column: ```loan_id eq '@{triggerBody()?['item_loan']?['loan_id']}'```

Now add a Delete Item Sharepoint step. Point it towards your list. In the ‘Id’ field, enter this function to use the sharepoint list ID of the first returned result: ```first(body('Get_items')?['value'])?['ID']```

Add a Terminate Step type: succeeded. Within that step, under the Configure Run After settings, select both ‘Succeeded’ and ‘Failed’ for the Delete Items step. 

Save.

### Compile the report

Create a scheduled Flow, set to run when you want the report emailed.

Add a Get Items Sharepoint step. Adjust the Top Count if you anticipate needing more than 500 (- 5000) rows. Power Automate will squawk that you don’t have a filter, but for this use case it’s probably not a big deal.

Use the Select step. Use the output from the Get Items step as the input array. I add title, author, barcode, loan_date, due_date. Apply the following time format to the loan_date and due_date using expressions:
``` json:
"loan date": "@convertTimeZone(item()?['loan_date'], 'UTC', 'Pacific Standard Time', 'MM/dd/yyyy HH:mm')",  
"due date": "@convertTimeZone(item()?['due_date'], 'UTC', 'Pacific Standard Time', 'MM/dd/yyyy HH:mm')"
```
Create an HTML table from the output of the Select action. Follow it up with a Compose that contains some styling:
``` html:  
<!DOCTYPE html>  
<html>  
<head>  
   <style>  
 table {  
           width: 100%;  
           border-collapse: collapse;  
           font-family: Arial, sans-serif;  
       }  
       th, td {  
           border: 1px solid #444444;  
           text-align: left;  
           padding: 12px;  
       }  
       th {  
           background-color: #4A536B;  
           color: white;  
       }  
       tr:nth-child(even) {  
           background-color: #f2f2f2;  
       }  
       tr:nth-child(odd) {  
           background-color: #ffffff;  
       }  
       tr:hover {  
           background-color: #ddd;  
       }  
   </style>  
</head>  
<h2 style="text-align: center;"> Sample Report Title</h2>  
<body>  
{HTML Table body output}  
</body>  
</html>

Use the output of the Compose step as the body of your email.   
Save.  
```
