## Lab 5:  Monitor Applications End-to-End (Optional)

Duration: 40 minutes

In this unit, you will explore live application metrics and query logs to know the health of your applications.

### Task 1: Add Instrumentation Key to Key Vault

1. The Application Insights Instrumentation Key must be provided for the non-java applications.

   > **Note:** In future iterations, the build packs for non-java applications will support Application Insights binding, and this step will be unnecessary.

2. To retrieve the Instrumentation Key for Application Insights and add it to the Key Vault, run the following command in the Git Bash window. Replace azure-spring-apps-SUFFIX with your **azure-spring-apps-<inject key="DeploymentID" enableCopy="false" />**

```shell
   export INSTRUMENTATION_KEY=$(az monitor app-insights component show --app azure-spring-apps-SUFFIX --resource-group $RESOURCE_GROUP | jq -r '.connectionString')

   az keyvault secret set --vault-name ${KEY_VAULT} \
      --name "ApplicationInsights--ConnectionString" --value ${INSTRUMENTATION_KEY}
```

   > **Note:** You can ignore any warning related to extension application insights.

  ![](Images/mjv2-45.png) 

### Task 2: Update Sampling Rate

1. To increase the sampling rate for the Application Insights binding, run the following command in the Git Bash pane:

```shell
az spring build-service builder buildpack-binding set \
    --builder-name default \
    --name default \
    --type ApplicationInsights \
    --properties sampling-rate=100 connection_string=${INSTRUMENTATION_KEY}
```

   > **Note:** The above command could take up to **15-20** minutes to complete. Please wait until it runs successfully. Meanwhile, you can learn more about sampling in application insights by clicking [here](https://learn.microsoft.com/en-us/azure/azure-monitor/app/sampling?tabs=net-core-new).

### Task 3: Reload Applications

1. Run the following command to restart applications to reload the configuration. (For the Java applications, this will allow the new sampling rate to take effect. For non-java applications, this will allow them to access the Instrumentation Key from the Key Vault.)

```shell
   az spring app restart -n ${CART_SERVICE_APP}
   az spring app restart -n ${ORDER_SERVICE_APP}
   az spring app restart -n ${IDENTITY_SERVICE_APP}
   az spring app restart -n ${CATALOG_SERVICE_APP}
   az spring app restart -n ${PAYMENT_SERVICE_APP}
```

   ![](Images/mjv2-28-new.png)
   
   
   > **Note:** The above spring apps can take up to **7** minutes to finish restarting. Also, if any app has failed to restart, please run the above command again for that app only.

### Task 4: Get the log stream for an application

1. Run the following command to get the latest 100 lines of app console logs from the Catalog Service.

```shell
az spring app logs \
    -n ${CATALOG_SERVICE_APP} \
    --lines 100
```

   ![](Images/mjv2-46.png)

   > **Note:** You can ignore any warning related to app logs or if the output is not expected. you can proceed to next task.

2. Run the following command by adding the `-f` parameter, so that you can get real-time log streaming from an app. Try log streaming for the Catalog Service.

```shell
az spring app logs \
    -n ${CATALOG_SERVICE_APP} \
    -f
```
   > Note: This command can take up to **10-15** minutes to run successfully. You don't need to wait for the  command to get executed completely; in the meantime, you can continue with the next task.
   
   ![](Images/mjv2-47.png) 

You can use `az spring app logs -h` to explore more parameters and log stream functionalities.

   > **Note:** After finishing the task, be sure not to close the Git Bash window. 

### Task 5: Start monitoring apps and dependencies - in Application Insights

  > **Note:** At this point in the workshop, only a limited number of data visualisations may be populated. (So the result and the graphs in the screenshots below may vary.)

1. Move back to the Azure portal, and in the **search resources, services and docs bar**, type **Application insight** and select it from the suggestions, as shown below: 

   ![](Images/mjv2-48.png)  

2. Under the Application Insights page, select **azure-spring-apps-<inject key="DeploymentID" enableCopy="false" />**.
  
   ![](Images/mjv2-49.png)

3. From the left panel, navigate to the `Application Map` blade under Investigate and then select the time filter as **Last 4 hours**.

   ![](Images/mjv2-50.png)
   
4. From the left panel, navigate to the `Performance` blade under Investigate and then click on **Operations**.

   ![](Images/mjv2-51.png)

5. Now, navigate to the `Performance/Dependencies` blade; you can see the performance number for dependencies:

   ![](Images/mjv2-52.png)

6. Navigate to the `Performance/Roles` blade. You can see the performance metrics for individual instances or roles:

   ![](Images/mjv2-53.png)
      
   
7. Now, from the left panel, navigate to the `Failures` blade under Investigate and then select the `Exceptions` panel. You can see a collection of exceptions:

   ![](Images/mjv2-54.png)
   
8. Now, from the left panel, navigate to the `Metrics` blade under Monitoring, where you can see metrics contributed by Spring Boot apps, Spring Cloud modules, and dependencies. The chart below shows `http_server_requests` and `Heap Memory Used`.

   ![](Images/mjv2-55.png)

   ![](Images/mjv2-56.png)

   > **Note:**  Spring Boot registers a lot of core metrics: JVM, CPU, Tomcat, Logback,...The Spring Boot auto-configuration enables the instrumentation of requests handled by Spring MVC. The REST controllers `ProductController`, and `PaymentController` have been instrumented by the `@Timed` Micrometer annotation at the class level.

   * `acme-catalog` application has the following custom metrics enabled:
   * @Timed: `store.products`

9. You can see these custom metrics in the `Metrics` blade:

   ![](Images/mjv2-57.png)
   
10. Now, from the left panel, navigate to the `Live Metrics` blade under Investigate - you can see live metrics on screen with low latencies < 1 second:

     ![](Images/mjv2-60.png)

### Task 6: Start monitoring ACME Fitness Store's logs and metrics in Azure Log Analytics

1. In the **search resources, services and docs bar**, type **Log analytics workspace** and select it from the suggestions, as shown below: 

   ![](Images/mjv2-58.png)

2. Under the Log Analytics Workspaces page, select **Log-analytics-workspace** created.
   
   ![](Images/Ex5-T6-S2.png)
   
3. On the Log Analytics page, select `Logs` blade **(1)** under General and close the default query page by clicking on `X` **(2)** in the top right corner.

   ![](Images/log-welcome.png)

4. In the **Logs** blade (1), paste the below Kusto query **(2)** and click on **Run (3)** to see the application logs:

```sql
 AppPlatformLogsforSpring 
 | where TimeGenerated > ago(24h) 
 | limit 500
 | sort by TimeGenerated
 | project TimeGenerated, AppName, Log
```

>**Note:** If you see the message "The query was stopped", then please wait for a few minutes and try again as there might be a chance that services are still being deployed.
   
   ![](Images/mjv2-61.png)

5. Click on `+` **(1)** to create the new query. Now paste the below Kusto query **(2)** and click on **Run (3)** to see `catalog-service` application logs:

```sql
 AppPlatformLogsforSpring 
 | where AppName has "catalog-service"
 | limit 500
 | sort by TimeGenerated
 | project TimeGenerated, AppName, Log
```
  ![](Images/mjv2-62-new.png)

6. Click on `+` **(1)** to create the new query. Now paste the below Kusto query **(2)** and click on **Run (3)** to see errors and exceptions thrown by each app:
  
```sql
 AppPlatformLogsforSpring 
 | where Log contains "error" or Log contains "exception"
 | extend FullAppName = strcat(ServiceName, "/", AppName)
 | summarize count_per_app = count() by FullAppName, ServiceName, AppName, _ResourceId
 | sort by count_per_app desc 
 | render piechart
```
   ![](Images/mjv2-63-new.png)

7. Click on `+` **(1)** to create the new query. Now paste the below Kusto query **(2)** and click on **Run (3)** to see all the inbound calls into Azure Spring Apps:

```sql
    AppPlatformIngressLogs
    | project TimeGenerated, RemoteAddr, Host, Request, Status, BodyBytesSent, RequestTime, ReqId, RequestHeaders
    | sort by TimeGenerated
```
   ![](Images/mjv2-64-new.png)

8. Click on `+` **(1)** to create the new query. Now paste the below Kusto query **(2)** and click on **Run (3)** to see all the logs from Spring Cloud Gateway managed by Azure Spring Apps:

```sql
    AppPlatformSystemLogs
    | where LogType contains "SpringCloudGateway"
    | project TimeGenerated,Log
```
   ![](Images/mjv2-65-new.png)

9. Click on `+` **(1)** to create the new query. Now paste the below Kusto query **(2)** and click on **Run (3)** to see all the logs from the Spring Cloud Service Registry managed by Azure Spring Apps:

```sql
    AppPlatformSystemLogs
    | where LogType contains "ServiceRegistry"
    | project TimeGenerated, Log
```
   ![](Images/mjv2-66-new.png)
   

> Now, click on **Next** in the lab guide section in the bottom right corner to jump to the next exercise instructions.
