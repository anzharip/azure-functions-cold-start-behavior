# azure-functions-cold-start-behavior

In January 2021, I had a project that utilizes the [Azure Functions HttpTrigger Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=javascript). During my time implementing the project, I came across [Azure Functions' cold-start behavior](https://azure.microsoft.com/en-us/blog/understanding-serverless-cold-start/). I had a goal to prevent the cold-start state to improve the service response time, while also benefitting from Azure Functions' scaling and pricing structure. 

Several notes I have made when testing the behavior: 
* So far I have not found the official documentation that states the idle duration that will trigger the cold-start state. However, I found [this](https://mikhail.io/serverless/coldstarts/azure/) and [this](https://itnext.io/how-to-tackle-the-cold-start-problem-of-azure-function-serverless-app-e90030cdb0c7) unofficial blog posts stating that the idle duration to trigger a cold-start state is around 20 minutes. 
* There is no way to configure the cold-start behavior for the Functions. I have not found any configuration reference on the documentations. 
* It seems the cold-start state is isolated per Functions, not per Azure Functions App. I found this when I tried to create a TimerTrigger in an attempt to prevent the cold-start state on the HttpTrigger endpoint. When I tried to set a TimerTrigger Functions to run for every 10 minutes within the same Azure Functions App with an HttpTrigger, the HttpTrigger Functions still experience a cold-start state (indicated by consistent >4 seconds response time in the App Insight page, instead of just <1 second response time for warmed-up state): 
![Screen Shot 2021-01-08 at 08 49 52](https://user-images.githubusercontent.com/10259593/104141584-1eda2e80-53ea-11eb-84dc-b53602b5eae3.png)

* I tried to call the HttpTrigger functions through a cron job, and the average response time improved to around 1 - 2 seconds, even though there is occasional >4 seconds response time. I explored this further through App Insight, and it seems Azure is moving around my Functions' host for around every one hour, indicated by multiple hostname in my Azure Functions App log: 
![Screen Shot 2021-01-07 at 07 39 08](https://user-images.githubusercontent.com/10259593/104141625-52b55400-53ea-11eb-918f-481f701e6a51.png)
![Screen Shot 2021-01-08 at 08 56 30](https://user-images.githubusercontent.com/10259593/104141551-eb979f80-53e9-11eb-840b-78ec39d3e40b.png)

So far, the conclusion that I made: 
* Using only TimerTrigger does not improve the response time.
* Using TimerTrigger could improve the response time, as long the TimerTrigger is calling the HttpTrigger API.
* Manually triggering the HttpTrigger API frequently (e.g. every 10 minutes) does not completely remove the cold-start state, as Azure Functions forcibly moves the Functions across multiple Host periodically. 
* Occasional response time spike (sometimes over 10 seconds) cannot be mitigated by manually triggering the Functions on an interval.
* The only solution is to [upgrade to the Premium Plan and utilize the pre-warmed instance feature](https://docs.microsoft.com/en-us/azure/azure-functions/functions-premium-plan?tabs=portal#eliminate-cold-starts). 