# NashAzureBootcamp2019

# PREREQUISITS

- Microsoft Azure subscription (non-Microsoft subscription)
- Local machine or a virtual machine configured with ( **complete the day before the lab!** ):
  - Visual Studio Community 2017 or greater, **version 15.4** or later
    - [https://www.visualstudio.com/vs/](https://www.visualstudio.com/vs/)
  - Azure development workload for Visual Studio 2017
    - [https://docs.microsoft.com/azure/azure-functions/functions-develop-vs#prerequisites](https://docs.microsoft.com/azure/azure-functions/functions-develop-vs#prerequisites)
  - .NET Framework 4.7 runtime (or higher)
    - [https://www.microsoft.com/net/download/windows](https://www.microsoft.com/net/download/windows)
- Office 365 account. If required, you can sign up for an Office 365 trial at:
  - [https://portal.office.com/Signup/MainSignup15.aspx?Dap=False&amp;QuoteId=79a957e9-ad59-4d82-b787-a46955934171&amp;ali=1](https://portal.office.com/Signup/MainSignup15.aspx?Dap=False&amp;QuoteId=79a957e9-ad59-4d82-b787-a46955934171&amp;ali=1)
- GitHub account. You can create a free account at [https://github.com](https://github.com/).



## **Solution architecture**

Below is a diagram of the solution architecture you will build in this lab. Please study this carefully, so you understand the whole of the solution as you are working on the various components.

![The Solution diagram is described in the text following this diagram.](images/image2.png 'Solution diagram')

The solution begins with vehicle photos being uploaded to an Azure Storage blobs container, as they are captured. A blob storage trigger fires on each image upload, executing the photo processing **Azure Function** endpoint (on the side of the diagram), which in turn sends the photo to the **Cognitive Services Computer Vision API OCR** service to extract the license plate data. If processing was successful and the license plate number was returned, the function submits a new Event Grid event, along with the data, to an Event Grid topic with an event type called &quot;savePlateData&quot;. However, if the processing was unsuccessful, the function submits an Event Grid event to the topic with an event type called &quot;queuePlateForManualCheckup&quot;. Two separate functions are configured to trigger when new events are added to the Event Grid topic, each filtering on a specific event type, both saving the relevant data to the appropriate **Azure Cosmos DB** collection for the outcome, using the Cosmos DB output binding. A **Logic App** that runs on a 15-minute interval executes an Azure Function via its HTTP trigger, which is responsible for obtaining new license plate data from Cosmos DB and exporting it to a new CSV file saved to Blob storage. If no new license plate records are found to export, the Logic App sends an email notification to the Customer Service department via their Office 365 subscription. **Application Insights** is used to monitor all of the Azure Functions in real-time as data is being processed through the serverless architecture. This real-time monitoring allows you to observe dynamic scaling first-hand and configure alerts when certain events take place.



# Challenge 1 : Create Resources

You must provision a few resources in Azure before you start developing the solution. Ensure all resources use the same resource group for easier cleanup.  Put resources in the same region as the resource group.  Remember that some resources need to have unique names.

In this challenge, you will provision a blob storage account using the Hot tier, and create two containers within to store uploaded photos and exported CSV files. You will then provision two Function Apps instances, one you will deploy from Visual Studio, and the other you will manage using the Azure portal. Next, you will create a new Event Grid topic. After that, you will create an Azure Cosmos DB account with two collections. Finally, you will provision a new Cognitive Services Computer Vision API service for applying object character recognition (OCR) on the license plates.

_HINT : Record names and keys_

1. Create a resource group
2. Create a storage account (refer to this one as INIT)
  * In the blobs, create a container &quot;images&quot;
  * In the blobs, create a container &quot;export&quot;
3. Create a function app (put &quot;App&quot; in the name)
  * For your tollbooth app, consumption plan, .NET runtime stack
  * Create new storage and disable application insights
4. Create a function app (put &quot;Events&quot; in the name)
  * For your tollbooth events, consumption plan, Javascript runtime stack
  * Create new storage and disable application insights
5. Create an Event Grid Topic (leave schema as Event Grid Schema)
6. Create an Azure Cosmos DB account
  * API : Core (SQL)
  * Disable Geo-redundency and multi-region writes
  * Create a collection
    * Database ID &quot;LicensePlates&quot;
    * Leave **Provision database throughput** unchecked.
    * Collection ID &quot;Processed&quot;
    * Partition key **: &quot;**** /licensePlateText&quot;**
    * 5000 throughput
  * Create a collection
    * Database ID &quot;LicensePlates&quot;
    * Leave **Provision database throughput** unchecked.
    * Collection ID &quot;NeedsManualReview&quot;
    * Partition key **: &quot;**** /fileName&quot;**
    * 5000 throughput
  * _Hint : copy the_ _read-write keys_ _for URI and Primary Key_
7. Create a Computer Vision API service (S1 pricing tier)



# Challenge 2 : Configuration

Use Visual Studio 2017 and its integrated Azure Functions tooling to develop and debug the functions locally, and then publish them to Azure. The starter project solution, TollBooths, contains most of the code needed. You will add in the missing code before deploying to Azure.

1. Add the application settings in the **first** function app (with name containing &quot;App&quot;) you created as follows:

| **Application Key** | **Value** |
| --- | --- |
| computerVisionApiUrl | Computer Vision API endpoint you copied earlier. Append **vision/v2.0/ocr** to the end. Example: [https://westus.api.cognitive.microsoft.com/vision/v2.0/ocr](https://westus.api.cognitive.microsoft.com/vision/v2.0/ocr) |
| computerVisionApiKey | Computer Vision API key |
| eventGridTopicEndpoint | Event Grid Topic endpoint |
| eventGridTopicKey | Event Grid Topic access key |
| cosmosDBEndPointUrl | Cosmos DB URI |
| cosmosDBAuthorizationKey | Cosmos DB Primary Key |
| cosmosDBDatabaseId | Cosmos DB database id (LicensePlates) |
| cosmosDBCollectionId | Cosmos DB processed collection id (Processed) |
| exportCsvContainerName | Blob storage CSV export container name (export) |
| blobStorageConnection | Blob storage connection string |

2. Open the Tollbooth solution.

_There is a completed version and one with TODOs. If you are running out of time, open  the solution in the &quot;Completed&quot; folder and skip to Challenge 3_

3. Open the task list
4. Open ProcessImage.cs. Notice that the Run method is decorated with the FunctionName attribute, which sets the name of the Azure Function to &quot;ProcessImage&quot;. This is triggered by HTTP requests sent to it from the Event Grid service. You tell Event Grid that you want to get these notifications at your function&#39;s URL by creating an event subscription, which you will do in a later task, in which you subscribe to blob-created events. The function&#39;s trigger watches for new blobs being added to the images container of the storage account that was created in Exercise 1. The data passed to the function from the Event Grid notification includes the URL of the blob. That URL is in turn passed to the input binding to obtain the uploaded image from Blob storage.
5. The following code represents the completed task in ProcessImage.cs:

// \*\*TODO 1: Set the licensePlateText value by awaiting a new FindLicensePlateText.GetLicensePlate method.\*\*

licensePlateText =awaitnewFindLicensePlateText(log, \_client).GetLicensePlate(licensePlateImage);

6. Open FindLicensePlateText.cs. This class is responsible for contacting the Computer Vision API to find and extract the license plate text from the photo, using OCR. Notice that this class also shows how you can implement a resilience pattern using Polly, an open source .NET library that helps you handle transient errors. This is useful for ensuring that you do not overload downstream services, in this case, the Computer Vision API. This will be demonstrated later on when visualizing the Function&#39;s scalability.
7. The following code represents the completed task in FindLicensePlateText.cs:

// TODO 2: Populate the below two variables with the correct AppSettings properties.

var uriBase = Environment.GetEnvironmentVariable(&quot;computerVisionApiUrl&quot;);

var apiKey = Environment.GetEnvironmentVariable(&quot;computerVisionApiKey&quot;);

8. Open SendToEventGrid.cs. This class is responsible for sending an Event to the Event Grid topic, including the event type and license plate data. Event listeners will use the event type to filter and act on the events they need to process. Make note of the event types defined here (the first parameter passed into the Send method), as they will be used later on when creating new functions in the second Function App you provisioned earlier.
9. The following code represents the completed tasks in SendToEventGrid.cs:

// TODO 3: Modify send method to include the proper eventType name value for saving plate data.

awaitSend(&quot;savePlateData&quot;, &quot;TollBooth/CustomerService&quot;, data);

// TODO 4: Modify send method to include the proper eventType name value for queuing plate for manual review.

awaitSend(&quot;queuePlateForManualCheckup&quot;, &quot;TollBooth/CustomerService&quot;, data);

# Challenge 3 : Deployment

1. Deploy the Tollbooth project to the &quot;App&quot; function app you created earlier.  Do not check the box to create a zip file

**Make sure the publish is successful before moving to the next step**

2. In the portal, add the event grid subscription to the &quot;Process Image&quot; function
  * Event Schema: Event Grid Schema.
  * Topic Type : Storage Accounts.
  * Resource : your recently created Event Grid.
  * _Uncheck_ Subscribe to all event types, then select Blob Created from the event types dropdown list.
  * Leave Web Hook as the Endpoint Type.

# Challenge 4 : Create Functions in the Portal

Create two new Azure Functions written in Node.js, using the Azure portal. These will be triggered by Event Grid and output to Azure Cosmos DB to save the results of license plate processing done by the ProcessImage function.

1. Navigate to the function app &quot;Events&quot;
2. Create a function that is triggered by event grid (install extensions if prompted)
  * Name : SavePlateData
3. Replace the code with the following:

module.exports=function (context, eventGridEvent) {

    context.log(typeof eventGridEvent);

    context.log(eventGridEvent);

    context.bindings.outputDocument= {

        fileName :eventGridEvent.data[&quot;fileName&quot;],

        licensePlateText :eventGridEvent.data[&quot;licensePlateText&quot;],

        timeStamp :eventGridEvent.data[&quot;timeStamp&quot;],

        exported :false

    }

    context.done();

};

4. Add an event grid subscription
  * Name should contain &quot;SAVE&quot;
  * Event Schema: Event Grid Schema.
  * Topic Type: Event Grid Topics.
  * Resource: your recently created Event Grid.
  * Ensure that Subscribe to all event types _is checked_. You will enter a custom event type later.
  * Leave Web Hook as the Endpoint Type.
5. Add a Cosmos DB Output to the function (install extensions if needed)
  * Select the Cosmos DB account created earlier
  * Database Name : LicensePlates
  * Collection Name : Processed
6. Create another function that is triggered by event grid
  * Name : QueuePlateForManualCheckup
7. Replace the code with the following:

module.exports=asyncfunction (context, eventGridEvent) {

    context.log(typeof eventGridEvent);

    context.log(eventGridEvent);

    context.bindings.outputDocument= {

        fileName :eventGridEvent.data[&quot;fileName&quot;],

        licensePlateText :&quot;&quot;,

        timeStamp :eventGridEvent.data[&quot;timeStamp&quot;],

        resolved :false

    }

    context.done();

};

8. Add an event grid subscription
  * Name should contain &quot;QUEUE&quot;
  * Event Schema: Event Grid Schema.
  * Topic Type: Event Grid Topics.
  * Resource: your recently created Event Grid.
  * Ensure that Subscribe to all event types _is checked_. You will enter a custom event type later.
  * Leave Web Hook as the Endpoint Type.
9. Add a Cosmos DB Output to the QueuePlateForManualCheckup function
  * Select the Cosmos DB account created earlier
  * Database Name : LicensePlates
  * Collection Name : NeedsManualReview
10. Navigate to your event grid topic
11. Modify the &quot;SAVE&quot; subscription so it no longer subscribes to all events and add an event type named &quot;savePlateData&quot;.  _Note – If you changed this in the solution, you will have to make sure the name matches what was in the solution_
12. Modify the &quot;QUEUE&quot; subscription so it no longer subscribes to all events and add an event type named &quot; **queuePlateForManualCheckup&quot;**.  _Note – If you changed this in the solution, you will have to make sure the name matches what was in the solution_

# Challenge 5 : Monitoring

Application Insights can be integrated with Azure Function Apps to provide robust monitoring for your functions. In this exercise, you will provision a new Application Insights account and configure your Function Apps to send telemetry to it.

1. Create an Application Insights resource
  * Name : Similar to TollboothMonitor
  * Application Type: ASP.NET web application
  * _Hint : Copy the instrumentation key_
2. Add application insights to your &quot;App&quot; function app and &quot;Events&quot; function app
  * Name: APPINSIGHTS\_INSTRUMENTATIONKEY
3. Open the Live Metrics Stream for the app insights in the &quot;App&quot; function app (may take a few minutes for App Insights to appear)
4. Go back to the solution in Visual Studio.  Expand the UploadImages project and open the App.config.  Update the &quot;blobStorageConnection&quot; to the connection string for your blob storage account
5. Start a new instance of the UploadImages project.  Press 1 in the console.
6. Go back to the portal to view the telemetry.  You should start seeing new telemetry arrive, showing the number of servers online, the incoming request rate, CPU process amount, etc. You can select some of the sample telemetry in the list to the side to view output data.
7. Leave the Live Metrics Stream window open once again, and close the console window for the image upload. Debug the UploadImages project again, then enter **2** and press **ENTER**. This will upload 1,000 new photos.
8. Switch back to the Live Metrics Stream window and observe the activity as the photos are uploaded. It is possible that the process will run so efficiently that no more than two servers will be allocated at a time. You should also notice things such as a steady cadence for the Request Rate monitor, the Request Duration hovering below ~500ms second, and the Process CPU percentage roughly matching the Request Rate.
9. Close the console window when done.

# Optional Challenge A : Scale the Cognitive Service

In this challenge, you will change the Computer Vision API to the Free tier. This will limit the number of requests to the OCR service to 10 per minute. Once changed, run the UploadImages console app to upload 1,000 images again. The resiliency policy programmed into the FindLicensePlateText.MakeOCRRequest method of the ProcessImage function will begin exponentially backing off requests to the Computer Vision API, allowing it to recover and lift the rate limit. This intentional delay will greatly increase the function&#39;s response time, thus causing the Consumption plan&#39;s dynamic scaling to kick in, allocating several more servers. You will watch all of this happen in real time using the Live Metrics Stream view.

1. Change the Computer Vision service to the Free tier
2. Start a new instance of the UploadImages project.  Press 2 in the console.
3. Observe the Live Metrics Stream.  After running for a couple of minutes, you should start to notice a few things. The Request Duration will start to increase over time. As this happens, you should notice more servers being brought online. Each time a server is brought online, you should see a message in the Sample Telemetry stating that it is &quot;Generating 2 job function(s)&quot;, followed by a Starting Host message. You should also see messages logged by the resilience policy that the Computer Vision API server is throttling the requests. This is known by the response codes sent back from the service (429). A sample message is &quot;Computer Vision API server is throttling our requests. Automatically delaying for 32000ms&quot;.
4. Return the Computer Vision service to S1 Standard

# Optional Challenge B : Data in Cosmos DB

In this challenge, you will use the Azure Cosmos DB Data Explorer in the portal to view saved license plate data.

1. In your cosmos DB account, open the Data Explorer
2. Examine the contents of the Processed and NeedsManualReview
3. Create a new SQL Query of processed documents where exported = false



# Challenge 6 : Data export workflow

In this exercise, you create a new Logic App for your data export workflow. This Logic App will execute periodically and call your ExportLicensePlates function, then conditionally send an email if there were no records to export.

1. Create a logic app
  * Name : Similar to TollboothLogic
  * Make sure Log Analytics is Off
  * Trigger should be Recurrence, 15 minutes
2. Add an action to call your &quot;App&quot; function app function name ExportLicensePlates
3. Add a condition control
  * Value : Status Code parameter
  * Operator : is equal to
  * Second value : 200
4. Ignore the True condition
5. In the False condition, send an O365 email
  * To : your email
  * Subject : enter something meaningful
  * Message Body : enter something here and include the status code value
6. Save and Run
