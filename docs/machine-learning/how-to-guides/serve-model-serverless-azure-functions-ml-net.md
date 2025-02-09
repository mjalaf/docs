---
title: Deploy a model to Azure Functions
description: Serve ML.NET sentiment analysis machine learning model for prediction over the internet using Azure Functions
ms.date: 12/16/2022
author: luisquintanilla
ms.author: luquinta
ms.custom: mvc, how-to
ms.topic: how-to
#Customer intent: As a developer, I want to use my ML.NET Machine Learning model to make predictions through the internet using Azure Functions
---

# Deploy a model to Azure Functions

Learn how to deploy a pretrained ML.NET machine learning model for predictions over HTTP through an Azure Functions serverless environment.

## Prerequisites

- [Visual Studio 2022](https://visualstudio.microsoft.com/downloads/) with the **.NET desktop development** and **Azure development** workloads installed. The .NET SDK is automatically installed when you select this workload.
- [Azure Functions Tools](/azure/azure-functions/functions-develop-vs#check-your-tools-version)
- PowerShell
- Pre-trained model. Download this [pretrained sentiment analysis machine learning model](https://github.com/dotnet/samples/blob/main/machine-learning/models/sentimentanalysis/sentiment_model.zip) or use the [ML.NET Sentiment Analysis tutorial](../tutorials/sentiment-analysis.md) to build your own model.

## Azure Functions sample overview

This sample is a **C# HTTP Trigger Azure Functions application** that uses a pretrained binary classification model to categorize the sentiment of text as positive or negative. Azure Functions provides an easy way to run small pieces of code at scale on a managed serverless environment in the cloud. The code for this sample can be found on the [dotnet/machinelearning-samples](https://github.com/dotnet/machinelearning-samples/tree/main/samples/csharp/end-to-end-apps/ScalableMLModelOnAzureFunction) repository on GitHub.

## Create Azure Functions project

1. In Visual Studio 2022 open the **Create a new project** dialog.
1. In the "Create a new project" dialog, select the **Azure Functions** project template.
1. In the **Name** text box, type "SentimentAnalysisFunctionsApp" and select **Next**.
1. In the "Additional information dialog", leave all the defaults as is and select **Create**.
1. Install the **Microsoft.ML NuGet Package**

    1. In Solution Explorer, right-click on your project and select **Manage NuGet Packages**.
    1. Choose "nuget.org" as the Package source.
    1. Select the "Browse" tab.
    1. Search for **Microsoft.ML**.
    1. Select that package in the list, and select **Install**.
    1. Select the **OK** button on the **Preview Changes** dialog
    1. Select the **I Accept** button on the **License Acceptance** dialog if you agree with the license terms for the packages listed.

    Follow the same steps to install the **Microsoft.Extensions.ML**, **Microsoft.Extensions.DependencyInjection**, and **Microsoft.Azure.Functions.Extensions** NuGet packages.

## Add pretrained model to project

1. Create a directory named *MLModels* in your project to save your prebuild model:
    In Solution Explorer, right-click on your project and select **Add > New Folder**. Type "MLModels" and select <kbd>Enter</kbd>.
1. Copy your prebuilt model to the *MLModels* folder.
1. In Solution Explorer, right-click your prebuilt model file and select **Properties**. Under **Advanced**, change the value of **Copy to Output Directory** to **Copy if newer**.

## Create Azure Function to analyze sentiment

Create a class to predict sentiment. Add a new class to your project:

1. In **Solution Explorer**, right-click the project, and then select **Add** > **New Azure Function**.

1. In the **Add New Item** dialog box, select **Azure Function** and change the **Name** field to *AnalyzeSentiment.cs*. Then, select **Add**.

1. In the **New Azure Function** dialog box, select **Http Trigger** and choose **Anonymous** from the Authorization level dropdown. Then, select **OK**.

    The *AnalyzeSentiment.cs* file opens in the code editor. Add the following `using` directive to the top of *AnalyzeSentiment.cs*:

    [!code-csharp [AnalyzeUsings](~/machinelearning-samples/samples/csharp/end-to-end-apps/ScalableMLModelOnAzureFunction/SentimentAnalysisFunctionsApp/AnalyzeSentiment.cs#L1-L11)]

    By default, the `AnalyzeSentiment` class is `static`. Make sure to remove the `static` keyword from the class definition.

    ```csharp
    public class AnalyzeSentiment
    {

    }
    ```

## Create data models

You need to create some classes for your input data and predictions. Add a new class to your project:

1. Create a directory named *DataModels* in your project to save your data models:
    In Solution Explorer, right-click on your project and select **Add > New Folder**. Type "DataModels" and select <kbd>Enter</kbd>.
2. In Solution Explorer, right-click the *DataModels* directory, and then select **Add > Class**.
3. In the **Add New Item** dialog box, select **Class** and change the **Name** field to *SentimentData.cs*. Then, select **Add**.

    The *SentimentData.cs* file opens in the code editor. Add the following `using` directive to the top of *SentimentData.cs*:

    [!code-csharp [SentimentDataUsings](~/machinelearning-samples/samples/csharp/end-to-end-apps/ScalableMLModelOnAzureFunction/SentimentAnalysisFunctionsApp/DataModels/SentimentData.cs#L1)]

    Remove the existing class definition and add the following code to the *SentimentData.cs* file:

    [!code-csharp [SentimentData](~/machinelearning-samples/samples/csharp/end-to-end-apps/ScalableMLModelOnAzureFunction/SentimentAnalysisFunctionsApp/DataModels/SentimentData.cs#L5-L13)]

4. In Solution Explorer, right-click the *DataModels* directory, and then select **Add > Class**.
5. In the **Add New Item** dialog box, select **Class** and change the **Name** field to *SentimentPrediction.cs*. Then, select **Add**. The *SentimentPrediction.cs* file opens in the code editor. Add the following `using` directive to the top of *SentimentPrediction.cs*:

    [!code-csharp [SentimentPredictionUsings](~/machinelearning-samples/samples/csharp/end-to-end-apps/ScalableMLModelOnAzureFunction/SentimentAnalysisFunctionsApp/DataModels/SentimentPrediction.cs#L1)]

    Remove the existing class definition and add the following code to the *SentimentPrediction.cs* file:

    [!code-csharp [SentimentPrediction](~/machinelearning-samples/samples/csharp/end-to-end-apps/ScalableMLModelOnAzureFunction/SentimentAnalysisFunctionsApp/DataModels/SentimentPrediction.cs#L5-L14)]

    `SentimentPrediction` inherits from `SentimentData` which provides access to the original data in the `SentimentText` property as well as the output generated by the model.

## Register PredictionEnginePool service

To make a single prediction, you have to create a [`PredictionEngine`](xref:Microsoft.ML.PredictionEngine%602). [`PredictionEngine`](xref:Microsoft.ML.PredictionEngine%602) is not thread-safe. Additionally, you have to create an instance of it everywhere it is needed within your application. As your application grows, this process can become unmanageable. For improved performance and thread safety, use a combination of dependency injection and the `PredictionEnginePool` service, which creates an [`ObjectPool`](xref:Microsoft.Extensions.ObjectPool.ObjectPool%601) of [`PredictionEngine`](xref:Microsoft.ML.PredictionEngine%602) objects for use throughout your application.

The following link provides more information if you want to learn more about [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection).

1. In **Solution Explorer**, right-click the project, and then select **Add** > **Class**.
1. In the **Add New Item** dialog box, select **Class** and change the **Name** field to *Startup.cs*. Then, select **Add**.
1. Add the following `using` directives to the top of *Startup.cs*:

    [!code-csharp [StartupUsings](~/machinelearning-samples/samples/csharp/end-to-end-apps/ScalableMLModelOnAzureFunction/SentimentAnalysisFunctionsApp/Startup.cs#L1-L7)]

1. Remove the existing code below the `using` directives and add the following code:

    ```csharp
    [assembly: FunctionsStartup(typeof(Startup))]
    namespace SentimentAnalysisFunctionsApp
    {
        public class Startup : FunctionsStartup
        {

        }
    }
    ```

1. Define variables to store the environment the app is running in and the file path where the model is located inside the `Startup` class

    [!code-csharp [DefineStartupVars](~/machinelearning-samples/samples/csharp/end-to-end-apps/ScalableMLModelOnAzureFunction/SentimentAnalysisFunctionsApp/Startup.cs#L14-L15)]

1. Below that, create a constructor to set the values of the `_environment` and `_modelPath` variables. When the application is running locally, the default environment is *Development*.

    [!code-csharp [StartupCtor](~/machinelearning-samples/samples/csharp/end-to-end-apps/ScalableMLModelOnAzureFunction/SentimentAnalysisFunctionsApp/Startup.cs#L17-L30)]

1. Then, add a new method called `Configure` to register the `PredictionEnginePool` service below the constructor.

    [!code-csharp [ConfigureServices](~/machinelearning-samples/samples/csharp/end-to-end-apps/ScalableMLModelOnAzureFunction/SentimentAnalysisFunctionsApp/Startup.cs#L32-L36)]

At a high level, this code initializes the objects and services automatically for later use when requested by the application instead of having to manually do it.

Machine learning models are not static. As new training data becomes available, the model is retrained and redeployed. One way to get the latest version of the model into your application is to restart or redeploy your application. However, this introduces application downtime. The `PredictionEnginePool` service provides a mechanism to reload an updated model without restarting or redeploying your application.

Set the `watchForChanges` parameter to `true`, and the `PredictionEnginePool` starts a [`FileSystemWatcher`](xref:System.IO.FileSystemWatcher) that listens to the file system change notifications and raises events when there is a change to the file. This prompts the `PredictionEnginePool` to automatically reload the model.

The model is identified by the `modelName` parameter so that more than one model per application can be reloaded upon change.

> [!TIP]
> Alternatively, you can use the `FromUri` method when working with models stored remotely. Rather than watching for file changed events, `FromUri` polls the remote location for changes. The polling interval defaults to 5 minutes. You can increase or decrease the polling interval based on your application's requirements. In the code sample below, the `PredictionEnginePool` polls the model stored at the specified URI every minute.
>
>```csharp
>builder.Services.AddPredictionEnginePool<SentimentData, SentimentPrediction>()
>   .FromUri(
>       modelName: "SentimentAnalysisModel",
>       uri:"https://github.com/dotnet/samples/raw/main/machine-learning/models/sentimentanalysis/sentiment_model.zip",
>       period: TimeSpan.FromMinutes(1));
>```

## Load the model into the function

Insert the following code inside the *AnalyzeSentiment* class:

[!code-csharp [AnalyzeCtor](~/machinelearning-samples/samples/csharp/end-to-end-apps/ScalableMLModelOnAzureFunction/SentimentAnalysisFunctionsApp/AnalyzeSentiment.cs#L20-L23)]

This code assigns the `PredictionEnginePool` by passing it to the function's constructor which you get via dependency injection.

## Use the model to make predictions

Replace the existing implementation of *Run* method in *AnalyzeSentiment* class with the following code:

[!code-csharp [AnalyzeRunMethod](~/machinelearning-samples/samples/csharp/end-to-end-apps/ScalableMLModelOnAzureFunction/SentimentAnalysisFunctionsApp/AnalyzeSentiment.cs#L26-L44)]

When the `Run` method executes, the incoming data from the HTTP request is deserialized and used as input for the `PredictionEnginePool`. The `Predict` method is then called to make predictions using the `SentimentAnalysisModel` registered in the `Startup` class and returns the results back to the user if successful.

## Test locally

Now that everything is set up, it's time to test the application:

1. Run the application
1. Open PowerShell and enter the code into the prompt where PORT is the port your application is running on. Typically the port is 7071.

    ```powershell
    Invoke-RestMethod "http://localhost:<PORT>/api/AnalyzeSentiment" -Method Post -Body (@{SentimentText="This is a very bad steak"} | ConvertTo-Json) -ContentType "application/json"
    ```

    If successful, the output should look similar to the text below:

    ```powershell
    Negative
    ```

Congratulations! You have successfully served your model to make predictions over the internet using an Azure Function.

## Next Steps

- [Deploy to Azure](/azure/azure-functions/functions-develop-vs#publish-to-azure)
