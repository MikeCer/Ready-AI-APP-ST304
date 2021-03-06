# Rolling Updates deployment

## 1. Clean-up existing BookService deployment

1. Using the PowerShell session alredy used for _kubectl_ remove existing _bookservice_ deployments and service by executing the following commands:

    +++kubectl delete deployment bookservice-1.0 bookservice-2.0 ; kubectl delete service bookservice+++

    that will return

    ```nocopy
    deployment.extensions "bookservice-1.0" deleted
    deployment.extensions "bookservice-2.0" deleted
    service "bookservice" deleted
    ```
    _Note that if the poller.ps1 is running, you will experience Http 503 result codes_.

2. Wait few seconds and then double check the results of the delete operation by executing:

    +++kubectl get pod; kubectl get service+++

    that confirms the lack of references to the _bookservice_ pods and service

    ```nocopy
    NAME                            READY   STATUS    RESTARTS   AGE
    bookinfo-spa-57bdd84f98-92r2q   2/2     Running   0          39m
    NAME           TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)        AGE
    bookinfo-spa   LoadBalancer   10.0.75.9    104.42.174.161   80:30654/TCP   2d5h
    kubernetes     ClusterIP      10.0.0.1     <none>           443/TCP        3d2h
    ```

## 2. Deploy the BookService

1. Re-deploy the original BookService API by executing:

    +++kubectl apply -f C:\Labs\k8sconfigurations\rolling-updates\bookservice-V1-reduntant.yaml+++

    that will return an output similar to this:

    ```nocopy
    deployment.apps/bookservice created
    service/bookservice created
    ```

2. Wait few minutes and then double check the result of the apply operation by executing:

    +++kubectl get pods; kubectl get service+++

    that will clearly shows the presence of 6 replicas of the BookService. Note that we need more than one replica of the pod in order to achieve an incremental rollout.

    ![BookService 6 replicas](https://github.com/felucian/Ready-AI-APP-ST304/blob/master-private/Lab_Modules/03_RollingUpdates/imgs/mod_03_img_01.png?raw=true)

## 3. Execute the Poller script

1. Using the PowerShell session, run the _poller.ps1_ (or leverage the running one) script or use the web application to get some traffic for the BookService API by executing:

    +++C:\Labs\Lab_Modules\Tools\Poller.ps1 -PublicIP $publicIP+++

    in order to see how the BookService API correctly returns HTTP 200 for the Book ID 2 reviews

    ![BookService 6 replicas](https://github.com/felucian/Ready-AI-APP-ST304/blob/master-private/Lab_Modules/03_RollingUpdates/imgs/mod_03_img_02.png?raw=true)

## 4. Deploy BookService using Rolling Updates strategy

We are now going to deploy a new version of the book service with a rolling update strategy; the new version will return an error when retrieving book reviews for BookId = 2 and BookId = 4.

1. While the _poller.ps1_ script is running, from the PowerShell session we used to run _kubectl_ commands, run the following command in order to proceed with the deployment of the BookService API with a rolling updates strategy

   +++kubectl apply -f C:\Labs\K8sConfigurations\rolling-updates\bookservice-V2-rolling-update.yaml+++

   that returns

   ```nocopy
   deployment.apps/bookservice configured
   service/bookservice unchanged
   ```

   The rolling update strategy we are using will create two pods with the new version and terminate two pods with the old version every 40 seconds. For more details check the file _C:\Labs\K8sConfigurations/rolling-updates/bookservice-V2-rolling-update.yaml_.

2. Let's monitor the PowerShell session where _poller.ps1_ script is running, you'll start to see some failures for the  [BookId 2] HTTP call

    ![BookService RU deploy 1](https://github.com/felucian/Ready-AI-APP-ST304/blob/master-private/Lab_Modules/03_RollingUpdates/imgs/mod_03_img_03.png?raw=true)

     You can use the following command to monitor the rolling out of your deployment:

    +++kubectl rollout status deployment bookservice+++

    that allows you to easily follow the whole operation until all new replicas result updated and the old one terminated

    ![BookService RU deploy 3](https://github.com/felucian/Ready-AI-APP-ST304/blob/master-private/Lab_Modules/03_RollingUpdates/imgs/mod_03_img_08.png?raw=true)
    _During the rollout, you may receive a few HTTP 503 (Service Unavailable) codes._

    Then we can see that number of failures will gradually increase as the rolling update strategy upgrades all replicas to the new version

    ![BookService RU deploy 2](https://github.com/felucian/Ready-AI-APP-ST304/blob/master-private/Lab_Modules/03_RollingUpdates/imgs/mod_03_img_04.png?raw=true)

    As the screenshots show, you can use

    +++kubectl rollout status deployment bookservice+++

    to monitor the rolling out operation

    +++kubectl get deployments+++

    to monitor the deployment status and

    +++kubectl get pods+++

    to get details about the status of each replica pod, e.g. the age (time passed since the pod has started) and status.

3. You can use Application Insight and Log Analytics to monitor the number of failed request generated by the BookService version deployed using the rolling-updates strategy.

   Wait a couple of minutes, needed for Azure Application Insights to collect telemetry, and paste the content of the "_c:\Labs\k8configurations\rolling-updates\LogAnalyticsQuery.md_" file into Azure Log Analytics.

    +++requests | where customDimensions["VersionTag"] contains "RU-" | summarize duration = avg(duration), requestCount = count() by name, podVersion = tostring(customDimensions["VersionTag"]), resultCode | sort by name, podVersion+++

    Then hit "Run" query and you should get something similar to the following image:

    ![BookService RU deploy 2](https://github.com/felucian/Ready-AI-APP-ST304/blob/master-private/Lab_Modules/03_RollingUpdates/imgs/mod_03_img_06.png?raw=true)  
    _(Please expect few differences in number between your query results and the above image)_

    where different Http result codes will display between "**V1-RU-BookService**" and the "**V2-RU-BookService**". 

## 5. Apply the rollback

1. Start the undo deployment operation using _kubectl_ by executing:

    +++kubectl rollout undo deployment bookservice+++

    that will returns an output similar to this:

    ```nocopy
    deployment.extensions/bookservice rolled back
    ```

2. On the _poller.ps1_ script PowerShell session checks the status codes received and, in the meantime, monitor the rollback by executing:

    +++kubectl rollout status deployment bookservice+++

    As you can see on the following screenshot

    ![BookService RU Undo 1](https://github.com/felucian/Ready-AI-APP-ST304/blob/master-private/Lab_Modules/03_RollingUpdates/imgs/mod_03_img_09.png?raw=true)

    you may expect to receive a few HTTP 503 (Service Unavailable) response while the roll back is in progress, then after it completes all requests to "BookId 2" will consistently succeed.

    ![BookService RU Undo 2](https://github.com/felucian/Ready-AI-APP-ST304/blob/master-private/Lab_Modules/03_RollingUpdates/imgs/mod_03_img_10.png?raw=true)

