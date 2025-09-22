# Azure AI Foundry: Performance Testing File Uploads

This repo contains a Jupyter notebook to performance-test the file upload capabilities of **Azure AI Foundry** (specifically targeting the ["30 requests per second" limit](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/quotas-limits?tabs=REST#quotas-and-limits-reference)). It provides a practical solution for measuring and understanding the difference between request submission rates and end-to-end processing throughput.

***

## 📑 Table of Contents:
- [Part 1: Configuring the Environment](#part-1-configuring-the-environment)
- [Part 2: Methodology - Understanding Performance Metrics](#part-2-methodology---understanding-performance-metrics)
- [Part 3: Code Walkthrough](#part-3-code-walkthrough)
- [Part 4: Results and Comprehensive Analysis](#part-4-results-and-comprehensive-analysis)
- [Appendix: Housekeeping](#appendix-housekeeping)

## Part 1: Configuring the Environment
To run the provided Jupyter notebook, you will need to set up your Azure AI environment and install the required Python packages.

### 1.1 Pre-requisites
You need an **Azure AI Foundry** resource and project. The notebook is designed to work with the Azure AI Foundry API and its vector store capabilities.

### 1.2 Authentication
The demo uses **Microsoft Entra ID** authentication via `DefaultAzureCredential` from the `azure.identity` package. To enable this, ensure you are authenticated in your environment, for example by using the Azure CLI (`az login`), setting environment variables for a service principal or using a managed identity.

### 1.3 Environment Variables
Configure the following environment variables with your Azure AI Foundry resource details. This is the most secure way to handle credentials, as it avoids hardcoding them into the notebook.

| Environment Variable             | Description                                                     |
| :------------------------------- | :-------------------------------------------------------------- |
| `AZURE_OPENAI_API_BASE`          | The endpoint URL for your Azure OpenAI resource.                |
| `AZURE_OPENAI_API_VERSION`       | The API version you are targeting (e.g., `2024-05-01-preview`). |
| `ZURE_OPENAI_API_DEPLOY`         | The deployment name of your GPT model (e.g., `gpt-4.1`)         |

### 1.4 Required Libraries
Install the necessary Python packages using pip command.

```bash
pip install -r requirements.txt
```

***

## Part 2: Methodology - Understanding Performance Metrics
The primary goal of this notebook is to clarify a common point of confusion: the difference between the API's rate limit and the total processing time a user experiences.

### 2.1 Request Submission Rate (The 30 RPS Limit)
This metric measures how fast your client can **send** requests to the Azure AI Foundry endpoint. Think of it like stuffing letters into a mailbox—the 30 RPS limit governs how fast you can put letters *in*. This is the rate that is subject to API throttling (HTTP 429 errors). Our test measures this by calculating the time elapsed between the first request being sent and the last request being sent.

`Submission Rate = Total Files / (Time of Last Request - Time of First Request)`

### 2.2 Completion Rate (End-to-End Throughput)
This metric measures how fast the entire process **finishes** for a batch of files. It includes network latency, server-side processing (scanning, indexing the file) and the response time. This is like waiting for a delivery confirmation for every letter you mailed. This rate is always lower than the submission rate because server-side work takes time.

`Completion Rate = Total Successful Files / (Time of Last Completion - Time of First Request)`

***

## Part 3: Code Walkthrough
The notebook is structured to first establish a baseline and then run high-concurrency tests to measure the two key performance rates.

### 3.1 Helper Function - `run_individual_sequential_test()`
This function provides a baseline performance measurement. It processes 10 files one by one, in a single thread. This demonstrates the performance without any parallelism and serves as a point of comparison.

``` Python
def run_individual_sequential_test(file_paths, vector_store_id):
    print("\n" + "="*70)
    print(f"Running Individual Sequential Test: {len(file_paths)} files")
    print("="*70)
    
    start_time = time.time()
    successful_uploads = 0
    
    for i, file_path in enumerate(file_paths):
        try:
            # Step 1: Upload from local disk
            with open(file_path, 'rb') as f:
                uploaded_file = client.files.create(file=f, purpose="assistants")
            
            # Step 2: Add to vector store
            client.vector_stores.files.create(
                vector_store_id=vector_store_id,
                file_id=uploaded_file.id
            )
            successful_uploads += 1
            print(f"  Processed file {i+1}/{len(file_paths)}")
        except Exception as e:
            print(f"  Failed to process file {i+1}: {e}")
            
    total_duration = time.time() - start_time
    completion_rate = successful_uploads / total_duration if total_duration > 0 else 0
    
    print("\n--- Individual Test Results ---")
    return {
        "Test Scenario": "1. Individual Sequential",
        "Files": len(file_paths),
        "Workers": 1,
        "Successful": successful_uploads,
        "Submission Rate (RPS)": f"{completion_rate:.2f}",
        "Completion Rate (files/sec)": f"{completion_rate:.2f}",
        "Total Time (s)": f"{total_duration:.2f}",
        "Rate Limit Errors": 0
    }
```

### 3.2 Helper Function - `run_unified_concurrent_test()`
This is the core of the performance test. The function is designed to be a realistic and accurate simulation of a high-load scenario. For each file, every worker thread performs the complete, unified workflow:
1.  **Uploads the file** from the local disk to Azure AI Foundry's staging area (`client.files.create`).
2.  **Adds the uploaded file** to the target vector store (`client.vector_stores.files.create`).

``` Python
def run_unified_concurrent_test(file_paths, vector_store_id, num_workers):
    print("\n" + "="*70)
    print(f"Running Unified Concurrent Test: {len(file_paths)} files with {num_workers} workers")
    print(f"Target Vector Store: {vector_store_id}")
    print("="*70)
    
    results = []

    # The task for each thread. This represents one complete, realistic operation.
    def upload_and_add_worker(file_path):
        request_start_time = time.time()
        try:
            # Step 1: Upload the file from the local machine.
            with open(file_path, 'rb') as f:
                uploaded_file = client.files.create(file=f, purpose="assistants")
            
            # Step 2: Add the now-uploaded file to the vector store.
            client.vector_stores.files.create(
                vector_store_id=vector_store_id,
                file_id=uploaded_file.id
            )
            return {'success': True, 'request_time': request_start_time, 'completion_time': time.time(), 'error': None}
        except Exception as e:
            return {'success': False, 'request_time': request_start_time, 'completion_time': time.time(), 'error': str(e)}

    # Use ThreadPoolExecutor to run the complete workflow concurrently
    with ThreadPoolExecutor(max_workers=num_workers) as executor:
        futures = [executor.submit(upload_and_add_worker, fp) for fp in file_paths]
        for i, future in enumerate(futures):
            results.append(future.result())
            if (i + 1) % 20 == 0:
                print(f"  ... {i+1}/{len(file_paths)} tasks completed.")

    # --- Analyze the results ---
    request_times = [r['request_time'] for r in results]
    completion_times = [r['completion_time'] for r in results]
    successful_results = [r for r in results if r['success']]
    
    submission_window = max(request_times) - min(request_times)
    submission_rate = len(file_paths) / submission_window if submission_window > 0 else float('inf')
    
    total_duration = max(completion_times) - min(request_times)
    completion_rate = len(successful_results) / total_duration if total_duration > 0 else 0
    
    rate_limit_errors = sum(1 for r in results if not r['success'] and '429' in r['error'])

    return {
        "Test Scenario": f"2. Concurrent ({num_workers} Workers)",
        "Files": len(file_paths),
        "Workers": num_workers,
        "Successful": len(successful_results),
        "Submission Rate (RPS)": f"{submission_rate:.2f}",
        "Completion Rate (files/sec)": f"{completion_rate:.2f}",
        "Total Time (s)": f"{total_duration:.2f}",
        "Rate Limit Errors": rate_limit_errors
    }
```

By running this unified task concurrently with a configurable number of workers (e.g., 20, 50, 100), it accurately measures both the **Request Submission Rate** and the final **Completion Rate**.

``` Python
scenarios_to_run = [20, 50, 100]
all_concurrent_results = []

for worker_count in scenarios_to_run:
    # **# BOLD: Call the test function, passing the main_vector_store_id.**
    test_result = run_unified_concurrent_test(
        file_paths=test_files,
        vector_store_id=vector_store_id,
        num_workers=worker_count
    )
    all_concurrent_results.append(test_result)
```

***

## Part 4: Results and Comprehensive Analysis
The final cell of the notebook collects the data from all test runs and presents it in a clear summary table, followed by an explanation for end-users.

### 4.1 Sample Results Table
The output is a pandas DataFrame that allows for easy comparison of the different test scenarios.

``` JSON
==============================================================================================================
PERFORMANCE ANALYSIS OF UNIFIED UPLOAD WORKFLOW
==============================================================================================================
              Test Scenario  Files  Workers  Successful Submission Rate (RPS) Completion Rate (files/sec) Total Time (s)  Rate Limit Errors
   1. Individual Sequential     10        1          10                  0.56                        0.56          17.70                  0
 2. Concurrent (20 Workers)    200       20         200                  8.78                        7.88          25.37                  0
 2. Concurrent (50 Workers)    200       50         200                 15.42                       10.49          19.06                  0
2. Concurrent (100 Workers)    200      100         199                 19.47                        9.44          21.08                  1
==============================================================================================================
```

### 4.2 Key Findings
1.  **Submission Rate Scales but Reaches a Practical Limit**: As the number of concurrent workers increases from 20 to 100, the **Submission Rate** scales accordingly from **8.78 RPS** up to **19.47 RPS**. While this did not reach the documented 30 RPS, it proves that parallelism dramatically increases the request rate. The ceiling below 30 RPS suggests that the true performance limit is a combination of client-side resources (network/CPU), network latency, and server-side backpressure.

2.  **Completion Rate Plateaus and Shows Diminishing Returns**: The end-to-end **Completion Rate** peaked at **10.49 files/sec** with 50 workers. Increasing the workers to 100 did not improve this rate; it actually decreased slightly to **9.44 files/sec**. This is a classic sign of a bottleneck on the server-side processing queue. Adding more concurrent requests from the client doesn't help if the server can only process files at a finite speed.

3.  **Rate-Limiting Error Confirms the Ceiling**: The test with 100 workers triggered **one rate-limit error**. This is a critical finding, as it provides concrete evidence that we successfully found the performance ceiling for this specific environment and workload. It confirms that at around 19-20 requests per second, we are beginning to saturate the service's capacity to complete new requests.

***

## Appendix: Housekeeping
The final cell in the notebook is dedicated to cleaning up all resources created during the tests. It will:
-   Delete the local directory of dummy files.
-   Delete all files uploaded to the Azure AI Foundry's storage account.
-   Delete the vector store created for the tests.

``` Python
def cleanup():
    print("\n" + "="*50)
    print("CLEANUP")
    print("="*50)
    
    # Clean up local files
    if 'temp_dir' in globals() and temp_dir and os.path.exists(temp_dir):
        shutil.rmtree(temp_dir)
        print(f"Deleted test files in the local directory")
    
    # Clean up the vector store and any remaining files.
    print("Deleting main vector store and all uploaded files from Azure OpenAI...")
    
    all_uploaded_files = client.files.list()
    file_ids_to_delete = [f.id for f in all_uploaded_files]
    
    deleted_count = 0
    print(f"Found {len(file_ids_to_delete)} total files in the account to delete...")
    for file_id in file_ids_to_delete:
        try:
            client.files.delete(file_id)
            deleted_count += 1
        except Exception:
            # Ignore errors for files that might already be deleted
            pass
    print(f"Successfully deleted {deleted_count}/{len(file_ids_to_delete)} files.")
    
    # Delete the main vector store
    try:
        client.vector_stores.delete(vector_store_id=vector_store_id)
        print(f"Deleted vector store: {vector_store_id}")
    except Exception as e:
        print(f"Failed to delete vector store {vector_store_id}: {e}")
```
