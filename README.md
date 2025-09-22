# Azure AI Foundry: Performance Testing File Uploads to Vector Stores

This repository contains a Jupyter Notebook designed to performance-test the file upload capabilities of **Azure OpenAI**, specifically targeting the "30 requests per second" limit for vector stores. It provides a practical solution for measuring and understanding the difference between request submission rates and end-to-end processing throughput.

> [!IMPORTANT]
> The key takeaway from this analysis is that the **30 requests per second (RPS) limit applies to *request submission*, not the total end-to-end processing time.** This notebook demonstrates that a client can easily submit requests far faster than 30 RPS, while the final completion rate is dependent on server-side processing.

***

## 📑 Table of Contents:
- [Part 1: Configuring the Environment](#part-1-configuring-the-environment)
- [Part 2: Methodology - Understanding Performance Metrics](#part-2-methodology---understanding-performance-metrics)
- [Part 3: Code Walkthrough](#part-3-code-walkthrough)
- [Part 4: Results and Comprehensive Analysis](#part-4-results-and-comprehensive-analysis)
- [Appendix: Housekeeping](#appendix-housekeeping)

## Part 1: Configuring the Environment
To run the provided Jupyter notebook, you will need to set up your Azure OpenAI environment and install the required Python packages.

### 1.1 Prerequisites
You need an **Azure OpenAI** resource and project. The notebook is designed to work with the Assistants API and its vector store capabilities.

### 1.2 Authentication
The demo uses **Microsoft Entra ID** authentication via `DefaultAzureCredential` from the `azure.identity` package. To enable this, ensure you are authenticated in your environment, for example by using the Azure CLI (`az login`), setting environment variables for a service principal, or using a managed identity.

### 1.3 Environment Variables
Configure the following environment variables with your Azure OpenAI resource details. This is the most secure way to handle credentials, as it avoids hardcoding them into the notebook.

| Environment Variable             | Description                                               |
| :------------------------------- | :-------------------------------------------------------- |
| `AZURE_OPENAI_API_BASE`          | The endpoint URL for your Azure OpenAI resource.          |
| `AZURE_OPENAI_API_VERSION`       | The API version you are targeting (e.g., `2024-05-01-preview`). |

### 1.4 Required Libraries
Install the necessary Python packages using pip. It is recommended to use a `requirements.txt` file.

```bash
# requirements.txt
openai
azure-identity
pandas
```

Install with:
```bash
pip install -r requirements.txt
```

***

## Part 2: Methodology - Understanding Performance Metrics
The primary goal of this notebook is to clarify a common point of confusion: the difference between the API's rate limit and the total processing time a user experiences.

### 2.1 Request Submission Rate (The 30 RPS Limit)
This metric measures how fast your client can **send** requests to the Azure OpenAI endpoint. Think of it like stuffing letters into a mailbox—the 30 RPS limit governs how fast you can put letters *in*. This is the rate that is subject to API throttling (HTTP 429 errors). Our test measures this by calculating the time elapsed between the first request being sent and the last request being sent.

`Submission Rate = Total Files / (Time of Last Request - Time of First Request)`

### 2.2 Completion Rate (End-to-End Throughput)
This metric measures how fast the entire process **finishes** for a batch of files. It includes network latency, server-side processing (scanning, indexing the file), and the response time. This is like waiting for a delivery confirmation for every letter you mailed. This rate is always lower than the submission rate because server-side work takes time.

`Completion Rate = Total Successful Files / (Time of Last Completion - Time of First Request)`

This notebook proves that while the **Completion Rate** may be around 10-15 files/second, the **Submission Rate** can easily exceed the 30 RPS limit.

***

## Part 3: Code Walkthrough
The notebook is structured to first establish a baseline and then run high-concurrency tests to measure the two key performance rates.

### 3.1 `run_individual_sequential_test()`
This function provides a baseline performance measurement. It processes 10 files one by one, in a single thread. This demonstrates the performance without any parallelism and serves as a point of comparison.

### 3.2 `run_unified_concurrent_test()`
This is the core of the performance test. The function is designed to be a realistic and accurate simulation of a high-load scenario. For each file, every worker thread performs the complete, unified workflow:
1.  **Uploads the file** from the local disk to Azure OpenAI's staging area (`client.files.create`).
2.  **Adds the uploaded file** to the target vector store (`client.vector_stores.files.create`).

By running this unified task concurrently with a configurable number of workers (e.g., 20, 50, 100), it accurately measures both the **Request Submission Rate** and the final **Completion Rate**.

***

## Part 4: Results and Comprehensive Analysis
The final cell of the notebook collects the data from all test runs and presents it in a clear summary table, followed by an explanation for end-users.

### 4.1 Sample Results Table
The output is a pandas DataFrame that allows for easy comparison of the different test scenarios.

```
==============================================================================================================
COMPREHENSIVE PERFORMANCE ANALYSIS OF UNIFIED UPLOAD WORKFLOW
==============================================================================================================
            Test Scenario  Files  Workers  Successful Submission Rate (RPS) Completion Rate (files/sec) Total Time (s) Rate Limit Errors
 1. Individual Sequential     10        1          10                  0.55                        0.55          18.12                 0
 2. Concurrent (20 Workers)    210       20         210                 26.55                       13.91          15.09                 0
 3. Concurrent (50 Workers)    210       50         210                 58.11                       14.21          14.78                 0
4. Concurrent (100 Workers)    210      100         210                 98.59                       14.43          14.55                 0
==============================================================================================================
```

### 4.2 Key Findings
1.  **Submission Rate Scales with Workers**: As the number of concurrent workers increases from 20 to 100, the **Submission Rate** skyrockets from ~26 RPS to ~98 RPS. This proves that the client is fully capable of exceeding the 30 RPS limit.
2.  **Completion Rate Remains Stable**: The **Completion Rate** remains relatively constant at around 14 files/second, regardless of the number of workers. This demonstrates that the true bottleneck is not the client or the network, but the finite speed at which the Azure OpenAI service can process and index the files on the backend.
3.  **No Rate-Limiting Encountered**: Even when submitting at nearly 100 RPS, no HTTP 429 errors were encountered in this test run, indicating the service has a robust capacity for ingesting requests.
