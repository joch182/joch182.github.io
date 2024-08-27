---
layout: post
title: Calculate the cost for each Glue job run using a python script
date: 2024-08-27 10:00:00 -0500
description: Use a python script to obtain the dpu usage hence the cost for each Glue job run.
img: posts_imgs/glue-job-cost-script-calculation/glue_job.jpg
tags: [python, AWS, glue, cost optimization, job run]
---

If you have used Glue service, you know that the cost of the Glue jobs is based on the DPUs and execution time of the job run. If you do not set Tags for each job then it is difficult to know exactly how much each job is costing you. For this cases, I have developed a Python script that can help you collect the information required to calculate the cost for each job run.

To start, import the libraries required and innitialize the boto3 client for each service. In this case, we will be runing the script in a Python Shell Glue job. The S3 client is required to save the reuslt as a CSV file.

```python
import boto3
from datetime import datetime, timedelta, timezone
import pandas as pd
glue_client = boto3.client('glue')
s3_client = boto3.client('s3')  
```

Then using the Glue client, we need to collect the jobs using the ["list_jobs"](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/glue/client/list_jobs.html) Glue API. All the jobs will be saved in a python list. This API returns a certain limit of jobs so we need to query multiple times using the "NextToken" value return in each call which indicates that there are still jobs pending to collect. When "NextToken" is not return, it means that there are no Job pendings to retrieve.

```python
# Collect Job names
NT = ""
jobs = []
while True:
  response = glue_client.list_jobs(NextToken=NT)
  if 'JobNames' in response and len(response['JobNames']) > 0:
    jobs = jobs + response['JobNames']
    if "NextToken" in response:
      NT = response["NextToken"]
      continue
  break
```

Now that we have a python list with all the job names, we need to retrieve all the job runs for each job name. In this case we will collect all the job runs for the last 90 days using the Glue API ["get_job_runs"](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/glue/client/get_job_runs.html). Same as before, this API, does not return all the information at once, so we need to call it multiple times as long the response includes "NextToken" which means there is still pending job runs to retrieve.

```python
# Collect job runs for each job name
job_runs = []
NT = ""
now = datetime.now(timezone.utc)
past_date_before_90days = now - timedelta(days = 90)
for job in jobs:
  old_job_run = False
  while True:
    response = glue_client.get_job_runs(JobName=job, NextToken=NT, MaxResults=200)
    print(f"response: {response}")
    if 'JobRuns' in response and len(response['JobRuns']) > 0:
      for job_run in response['JobRuns']:
        if job_run['StartedOn'] < past_date_before_90days:
          old_job_run = True
          break
        else:
          job_runs.append(job_run)
    if old_job_run or "NextToken" not in response:
      break
```

Finally, once the data is already collected, we create a python Dataframe using Pandas and save the result as CSV file into S3 using the Boto3 API ["upload_file"](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3/client/upload_file.html). We need to have a Bucket created and set the Prefix (or route) to save the file in S3.

```python
# Save data as DF
df = pd.DataFrame(job_runs) 
# Save Results to S3
df.to_csv("/tmp/output.csv", index=False)
s3_client.upload_file('/tmp/output.csv', 'BUCKET_NAME', 'PREFIX/output.csv')
```

Once we have this information in a spreadsheet, we can use the [pricing](https://aws.amazon.com/glue/pricing/) guidelines from Glue to calculate the cost of each job run based on the DPU, time consumption, job type, executtion class etc.