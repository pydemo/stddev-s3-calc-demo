# Scenario

Impression data files in a private S3 bucket. Each file is comma delimited and compressed using the GZIP. The files sizes usually range between 3GB and 12GB and contain data for the previous day.

      The impressions data must be joined with campaign metadata joined based on the “campaign_id”, partitioned by AdDate.
      Campaign metadata can change over time, and the existing impressions files must be updated data as campaign metadata is updated.
      Create dashboard that displays the metrics listed below:
      The status of each load
      File counts per file
      Standard deviation of the “viewing_percentage”

### Prereqs:
1.	We will need valid job queue and compute environment for AWS Batch.
2.	Also we need working Docker environment to complete this task (Amazon Linux +Docker).
3.	AWS CLI
### Build steps:
1.	Build Docker images for partition, join, and update processes.
2.	Create an Amazon ECR or Docker Hub repo for the images.
3.	Push the built image to ECR or DH.
4.	Create job script and upload it to S3.
5.	Create IAM role to be used by jobs to access S3.
6.	Create a job definition that uses the built images.
7.	Submit and run a job that executes the job script from S3.

## Workflow

1.	Partition large impressions file by date (list partition) and campaign id (hash partition). Upload results to S3 as FACT_PARTITONED.
2.	Join FACT_PARTITONED file (from step 1) with smaller campaign dimension file. Upload results to S3 as FACT_JOINED.
3.	Update FACT_JOINED file (from step 2) with data from changed campaign dimension. Upload results to S3 as FACT_UPDATED.

### Run_job.sh
```shell
#!/bin/bash
date
echo "Args: $@"
env
echo "Partition Fact."
echo "jobId: $AWS_BATCH_JOB_ID"
echo "jobQueue: $AWS_BATCH_JQ_NAME"
echo "computeEnvironment: $AWS_BATCH_CE_NAME"

python3 partition.py bucket_name/fact_file.gz tables/FACT_PARTITIONED

python3 join.py bucket_name/campaign_meta_file.gz tables/ FACT_PARTITIONED tables/FACT_JOINED

python3 update.py bucket_name/updated_campaign_meta_file.gz tables/FACT_JOINED tables/FACT_UPDATED

date
echo "Done.
```

### Python scripts:
      partition.py – partitions impressions files by date (list) and campaign_id (hash) 
      join.py – joins impressions partitioned file with campaign metadata file by date and campaign_id.
      update.py – updates impressions partitioned files with data from updated metadata file by date and campaign_id.
      count.py - total line count across all files in S3 bucket. 
      mean.py - viewing_percentage mean across all files in S3 bucket. 
      stddev.py - standard deviation for viewing_percentage across all files in S3 bucket.

## Metrics.
### You can fetch row counts from:
1.	CloudWatch logs (Write logs from your Python scripts into CloudWatch log group. Then use CW Logs Insights query language to create dashboard)
2.	S3 file metadata (assumed you updated file metadata with row counts from your python script at data load time)
3.	S3 bucket tags (assumed you created those bucket tags with row counts from your python script at data load time)
4.	Brute–force recount all rows in a bucket FACT_UPDATED using Python.
### Job status:
1.	CloudWatch logs (Write job leg status from your Python scripts into CloudWatch log group. Then use CW Logs Insights query language to create dashboard)

### Standard deviation of the column “viewing_percentage” over the past 200 days.
1.	FACT_UPDATED
```
   [‘01/01/2019’, '50000', 'campaign_00', 'video_0', '0.5']
   [‘02/02/2020’, '50018', 'campaign_08', 'video_0', '0.33']
   [‘03/03/2020’, '50020', 'campaign_00', 'video_0', '0.25']
   [‘04/04/2019’, '50030', 'campaign_00', 'video_0', '0.2']
```   
Last column is viewing_percentage = VP

### Calculate standard deviation (using S3 Select)
   Mean (run S3 Select query on FACT_UPDATED to calculate sum of all values in VP, then calculate number of all rows. 
   
   `Mean = sum_of_val_VP/total_row_count`
```SQL
SELECT avg(cast(S._4 as float)) VP_mean FROM s3object S where date> ‘06/01/2019’
```
   Now calculate std deviation.. (Run S3 Select query on FACT_UPDATED to do it)
```SQL
SELECT sum((cast(S._4 as float)- VP_mean)* (cast(S._4 as float)- VP_mean)) VP_mean_sum FROM s3object S where date> ‘06/01/2019’
```
`VP_std_dev = sqrt(VP_mean_sum/VP_count)`


# Test

      tables/FACT_IMPRESSIONS_UPDATED/B_0/FACT_IMPRESSIONS_UPDATED.B_0.campaign_00.updated_fact_from_dim.csv.gz
      crossix-test
      tables/FACT_IMPRESSIONS_UPDATED/
      S3 key:  tables/FACT_IMPRESSIONS_UPDATED/B_0/FACT_IMPRESSIONS_UPDATED.B_0.campaign_00.updated_fact_from_dim.csv.gz
      S3 key:  tables/FACT_IMPRESSIONS_UPDATED/B_1/FACT_IMPRESSIONS_UPDATED.B_1.campaign_00.updated_fact_from_dim.csv.gz
      S3 key:  tables/FACT_IMPRESSIONS_UPDATED/B_2/FACT_IMPRESSIONS_UPDATED.B_2.campaign_00.updated_fact_from_dim.csv.gz
      S3 key:  tables/FACT_IMPRESSIONS_UPDATED/B_3/FACT_IMPRESSIONS_UPDATED.B_3.campaign_00.updated_fact_from_dim.csv.gz
      S3 key:  tables/FACT_IMPRESSIONS_UPDATED/B_4/FACT_IMPRESSIONS_UPDATED.B_4.campaign_00.updated_fact_from_dim.csv.gz
      S3 key:  tables/FACT_IMPRESSIONS_UPDATED/B_5/FACT_IMPRESSIONS_UPDATED.B_5.campaign_00.updated_fact_from_dim.csv.gz
      S3 key:  tables/FACT_IMPRESSIONS_UPDATED/B_6/FACT_IMPRESSIONS_UPDATED.B_6.campaign_00.updated_fact_from_dim.csv.gz
      S3 key:  tables/FACT_IMPRESSIONS_UPDATED/B_7/FACT_IMPRESSIONS_UPDATED.B_7.campaign_00.updated_fact_from_dim.csv.gz
      VP cnt, sum: ['2', '0.6\n']
      VP cnt, sum: ['2', '0.4242424242424242\n']
      VP cnt, sum: ['2', '0.3333333333333333\n']
      VP cnt, sum: ['2', '0.27692307692307694\n']
      VP cnt, sum: ['2', '0.23809523809523808\n']
      VP cnt, sum: ['2', '0.2095238095238095\n']
      VP cnt, sum: ['2', '0.1875\n']
      VP cnt, sum: ['2', '0.16993464052287582\n']
      Total rows: 16.0 , sum: 2.4395525226407577
      Mean: 0.15247203266504736
      Sum executed in 0.74 seconds.
      VP cnt, square: ['2', '0.12352900229196570572376147394975029549\n']
      VP cnt, square: ['2', '0.036500805877071419603623708664217207643\n']
      VP cnt, square: ['2', '0.014291864157768743390423972286257298116\n']
      VP cnt, square: ['2', '0.0079665523926945334146743244339381149809\n']
      VP cnt, square: ['2', '0.0067695302436281042192323652445238045379\n']
      VP cnt, square: ['2', '0.0074550069401342117967235206082362826225\n']
      VP cnt, square: ['2', '0.0088496792406297783063180620859392\n']
      VP cnt, square: ['2', '0.010480767993370960680744330116208007637\n']
      Total rows: 16.0 , squares: 0.21584320913726343
      Stddev: 0.11614732270301785
      Stddev executed in 1.22 seconds.
