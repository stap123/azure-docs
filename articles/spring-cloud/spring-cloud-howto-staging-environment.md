---
title: Set up a staging environment in Azure Spring Cloud | Microsoft Docs
description: Learn how to use blue-green deployment with Azure Spring Cloud
author: MikeDodaro
ms.service: spring-cloud
ms.topic: conceptual
ms.date: 01/14/2021
ms.author: brendm
ms.custom: devx-track-java, devx-track-azurecli
---

# Set up a staging environment in Azure Spring Cloud

**This article applies to:** ✔️ Java

This article explains how to set up a staging deployment using the blue-green deployment pattern in Azure Spring Cloud. Blue-green deployment is an Azure DevOps continuous delivery pattern that relies on keeping an existing (blue) version live, while a new (green) one is deployed. This article shows you how to put that staging deployment into production without changing the production deployment.

## Prerequisites

* Azure Spring Cloud instance on *Standard* **Pricing tier**.
* Azure CLI [Azure Spring Cloud extension](/cli/azure/azure-cli-extensions-overview)

This article uses an application built from the Spring Initializer. If you want to use a different application for this example, you will need to make a simple change in a public-facing portion of the application to differentiate your staging deployment from production.

>[!TIP]
> Azure Cloud Shell is a free interactive shell that you can use to run the instructions in this article.  It has common, preinstalled Azure tools, including the latest versions of Git, JDK, Maven, and the Azure CLI. If you're signed in to your Azure subscription, start your [Azure Cloud Shell](https://shell.azure.com).  To learn more, see [Overview of Azure Cloud Shell](../cloud-shell/overview.md).

To set up blue-green deployments in Azure Spring Cloud, follow the instructions in the next sections.

## Install the Azure CLI extension

Install the Azure Spring Cloud extension for the Azure CLI by using the following command:

```azurecli
az extension add --name spring-cloud
```
## Prepare app and deployments
To build the application follow these steps:
1. Generate the code for the sample app using The Spring Initializer with [this configuration](https://start.spring.io/#!type=maven-project&language=java&platformVersion=2.3.4.RELEASE&packaging=jar&jvmVersion=1.8&groupId=com.example&artifactId=hellospring&name=hellospring&description=Demo%20project%20for%20Spring%20Boot&packageName=com.example.hellospring&dependencies=web,cloud-eureka,actuator,cloud-starter-sleuth,cloud-starter-zipkin,cloud-config-client).

2. Download the code.
3. Add the following source file HelloController.java to the folder `\src\main\java\com\example\hellospring\`.
```java
package com.example.hellospring; 
import org.springframework.web.bind.annotation.RestController; 
import org.springframework.web.bind.annotation.RequestMapping; 

@RestController 

public class HelloController { 

@RequestMapping("/") 

  public String index() { 

      return "Greetings from Azure Spring Cloud!"; 
  } 

} 
```
4. Build the .jar file:
```azurecli
mvn clean packge -DskipTests
```
5. Create the app in your Azure Spring Cloud instance:
```azurecli
az spring-cloud app create -n demo -g <resourceGroup> -s <Azure Spring Cloud instance> --assign-endpoint
```
6. Deploy the app to Azure Spring Cloud:
```azurecli
az spring-cloud app deploy -n demo -g <resourceGroup> -s <Azure Spring Cloud instance> --jar-path target\hellospring-0.0.1-SNAPSHOT.jar
```
7. Modify the code for your staging deployment:
```java
package com.example.hellospring; 
import org.springframework.web.bind.annotation.RestController; 
import org.springframework.web.bind.annotation.RequestMapping; 

@RestController 

public class HelloController { 

@RequestMapping("/") 

  public String index() { 

      return "Greetings from Azure Spring Cloud! THIS IS THE GREEN DEPLOYMENT"; 
  } 

} 
```
8. Rebuild the .jar file:
```azurecli
mvn clean packge -DskipTests
```
9. Create the green deployment: 
```azurecli
az spring-cloud app deployment create -n green --app demo -g <resourceGroup> -s <Azure Spring Cloud instance> --jar-path target\hellospring-0.0.1-SNAPSHOT.jar 
```

## View apps and deployments

View deployed apps using the following procedures.

1. Go to your Azure Spring Cloud instance in the Azure portal.

1. From the left navigation pane open the "Apps" blade to view apps for your service instance.

    [ ![Apps-dashboard](media/spring-cloud-blue-green-staging/app-dashboard.png)](media/spring-cloud-blue-green-staging/app-dashboard.png)

1. You can click an app and view details.

    [ ![Apps-overview](media/spring-cloud-blue-green-staging/app-overview.png)](media/spring-cloud-blue-green-staging/app-overview.png)

1. Open **Deployments** to see all deployments of the app. The grid shows both production and staging deployments.

    [ ![App/Deployments dashboard](media/spring-cloud-blue-green-staging/deployments-dashboard.png)](media/spring-cloud-blue-green-staging/deployments-dashboard.png)

1. Click the URL to open the currently deployed application.
    ![URL deployed](media/spring-cloud-blue-green-staging/running-blue-app.png)
1. Click **Production** in the **State** column to see the default app.
    ![Default running](media/spring-cloud-blue-green-staging/running-default-app.png)
1. Click **Staging** in the **State** column to see the staging app.
    ![Staging running](media/spring-cloud-blue-green-staging/running-staging-app.png)

>[!TIP]
> * Confirm that your test endpoint ends with a slash (/) to ensure that the CSS file is loaded correctly.  
> * If your browser requires you to enter login credentials to view the page, use [URL decode](https://www.urldecoder.org/) to decode your test endpoint. URL decode returns a URL in the form "https://\<username>:\<password>@\<cluster-name>.test.azureapps.io/gateway/green".  Use this form to access your endpoint.

>[!NOTE]    
> Config server settings apply to both your staging environment and production. For example, if you set the context path (`server.servlet.context-path`) for your app gateway in config server as *somepath*, the path to your green deployment changes to "https://\<username>:\<password>@\<cluster-name>.test.azureapps.io/gateway/green/somepath/...".
 
 If you visit your public-facing app gateway at this point, you should see the old page without your new change.

## Set the green deployment as the production environment

1. After you've verified your change in your staging environment, you can push it to production. On the **Apps**/**Deployments** page, select the application currently in `Production`.

1. Click the ellipses after the **Registration status** of the green deployment and set the staging build to production. 

   [ ![Set production to staging](media/spring-cloud-blue-green-staging/set-staging-deployment.png)](media/spring-cloud-blue-green-staging/set-staging-deployment.png)

1. Now the URL of the app should display your changes.

   ![Staging now in deployment](media/spring-cloud-blue-green-staging/new-production-deployment.png)

>[!NOTE]
> After you've set the green deployment as the production environment, the previous deployment becomes the staging deployment.

## Modify the staging deployment

If you're not satisfied with your change, you can modify your application code, build a new jar package, and upload it to your green deployment by using the Azure CLI.

```azurecli
az spring-cloud app deploy  -g <resource-group-name> -s <service-instance-name> -n gateway -d green --jar-path gateway.jar
```

## Delete the staging deployment

To delete your staging deployment from the Azure port, go to your staging deployment page, and then select the **Delete** button.

Alternatively, delete your staging deployment from the Azure CLI by running the following command:

```azurecli
az spring-cloud app deployment delete -n <staging-deployment-name> -g <resource-group-name> -s <service-instance-name> --app gateway
```

## Next steps

* [CI/CD for Azure Spring Cloud](/azure/spring-cloud/spring-cloud-howto-cicd?pivots=programming-language-java)