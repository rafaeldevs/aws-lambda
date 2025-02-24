# AWS Lambda Amazon FBA and Shopify Inventory Sychronization set it and forget it Report Generation

## Synopsis

FBA Invenory + Shopify Inventory --> Combined Inventory

---

By following these steps, you’ll have an AWS Lambda function that processes inventory CSV files stored in S3, performs the reconciliation and discrepancy flagging, and writes the audit report back to S3. This setup can be integrated into automated workflows to help manage large datasets and trigger invoicing policies based on data volume.

---

Below is an example of an AWS Lambda function written in Python that uses pandas to audit inventory data from two CSV files stored in S3, and detailed documentation on how to set it up.

## AWS Lambda Function Code

This sample code does the following:
- Reads two CSV files (`fba_inventory.csv` and `shopify_inventory.csv`) from an input S3 bucket.
- Normalizes the SKU columns.
- Merges the datasets and flags discrepancies.
- Writes the resulting audit report CSV to an output S3 bucket.

```python
import boto3
import pandas as pd
import io

s3 = boto3.client('s3')

def lambda_handler(event, context):
    # Define S3 bucket names and object keys
    # (These can also be passed via environment variables)
    input_bucket = "your-input-bucket"
    output_bucket = "your-output-bucket"
    fba_key = "fba_inventory.csv"
    shopify_key = "shopify_inventory.csv"
    output_key = "audit_report.csv"
    
    # Retrieve FBA CSV file from S3
    fba_obj = s3.get_object(Bucket=input_bucket, Key=fba_key)
    fba_data = fba_obj['Body'].read().decode('utf-8')
    fba_df = pd.read_csv(io.StringIO(fba_data))
    
    # Retrieve Shopify CSV file from S3
    shopify_obj = s3.get_object(Bucket=input_bucket, Key=shopify_key)
    shopify_data = shopify_obj['Body'].read().decode('utf-8')
    shopify_df = pd.read_csv(io.StringIO(shopify_data))
    
    # Normalize SKU columns
    fba_df['SKU'] = fba_df['SKU'].str.strip().str.upper()
    shopify_df['SKU'] = shopify_df['SKU'].str.strip().str.upper()
    
    # Merge datasets on SKU using an outer join
    merged_df = pd.merge(shopify_df, fba_df, on='SKU', how='outer', suffixes=('_shopify', '_fba'))
    
    # Flag discrepancies and missing data
    merged_df['Status'] = 'Match'
    merged_df.loc[merged_df['Shopify_Quantity'] != merged_df['FBA_Quantity'], 'Status'] = 'Mismatch'
    merged_df.loc[merged_df['FBA_Quantity'].isna(), 'Status'] = 'Missing in FBA'
    merged_df.loc[merged_df['Shopify_Quantity'].isna(), 'Status'] = 'Missing in Shopify'
    
    # Convert merged dataframe to CSV
    csv_buffer = io.StringIO()
    merged_df.to_csv(csv_buffer, index=False)
    
    # Upload audit report to S3
    s3.put_object(Bucket=output_bucket, Key=output_key, Body=csv_buffer.getvalue())
    
    return {
        'statusCode': 200,
        'body': 'Audit report generated successfully!'
    }
```

---

## Documentation: How to Set Up the AWS Lambda Function

### Prerequisites
- **AWS Account:** You must have an active AWS account.
- **S3 Buckets:** Create at least two S3 buckets:
  - One for input files (e.g., `your-input-bucket`) containing the CSV files.
  - One for the output file (e.g., `your-output-bucket`) where the audit report will be saved.
- **AWS CLI:** (Optional) Install and configure the AWS CLI for deployment.
- **Python Dependencies:** The function uses `pandas`. Since it’s not included by default in AWS Lambda’s Python runtime, you’ll need to package it with your code or add it as a Lambda layer.

### Step 1: Prepare the Deployment Package

1. **Create a Project Directory:**
   ```bash
   mkdir lambda_inventory_audit
   cd lambda_inventory_audit
   ```

2. **Create a Virtual Environment and Install Dependencies:**
   ```bash
   python3 -m venv v-env
   source v-env/bin/activate
   pip install pandas boto3
   ```

3. **Add Your Lambda Function Code:**
   Save the code provided above as `lambda_function.py` in your project directory.

4. **Package the Code and Dependencies:**
   - Create a directory to hold the package:
     ```bash
     mkdir package
     cd package
     ```
   - Copy installed libraries:
     ```bash
     cp -r ../v-env/lib/python3.*/site-packages/* .
     ```
   - Copy your function code into the package:
     ```bash
     cp ../lambda_function.py .
     ```
   - Zip the contents:
     ```bash
     zip -r9 ../lambda_inventory_audit.zip .
     cd ..
     ```

   *Alternatively, you can use a Lambda layer for pandas if you prefer keeping your deployment package small.*

### Step 2: Create an IAM Role for Lambda

1. **Open the IAM Console** in AWS.
2. **Create a New Role:**
   - Choose **AWS service** as the trusted entity.
   - Select **Lambda** as the service that will use this role.
3. **Attach the Following Policies:**
   - **AWSLambdaBasicExecutionRole** (provides CloudWatch Logs access)
   - **AmazonS3ReadOnlyAccess** (for input S3 bucket)
   - A custom policy (or additional managed policy) that grants `s3:PutObject` permission on your output bucket. For example, a custom policy like:
     ```json
     {
         "Version": "2012-10-17",
         "Statement": [
             {
                 "Effect": "Allow",
                 "Action": "s3:PutObject",
                 "Resource": "arn:aws:s3:::your-output-bucket/*"
             }
         ]
     }
     ```
4. **Save the Role** and note its ARN.

### Step 3: Create the Lambda Function

1. **Go to the AWS Lambda Console** and click **Create function**.
2. **Select "Author from scratch":**
   - **Function name:** e.g., `InventoryAuditFunction`
   - **Runtime:** Choose Python 3.8 (or later)
   - **Permissions:** Under **Change default execution role**, select **Use an existing role** and choose the role you created in Step 2.
3. **Upload the Deployment Package:**
   - In the function code section, select **Upload from > .zip file** and upload the `lambda_inventory_audit.zip` you created.
4. **Set the Handler:**
   - Ensure that the Handler is set to `lambda_function.lambda_handler` (assuming your file is named `lambda_function.py` and your function is named `lambda_handler`).

### Step 4: Configure Environment Variables (Optional)

- You may set environment variables for S3 bucket names and file keys so you don’t need to hardcode them:
  - `INPUT_BUCKET` : `your-input-bucket`
  - `OUTPUT_BUCKET` : `your-output-bucket`
  - `FBA_KEY` : `fba_inventory.csv`
  - `SHOPIFY_KEY` : `shopify_inventory.csv`
  - `OUTPUT_KEY` : `audit_report.csv`

*Update your Lambda function code to use these environment variables if needed:*

```python
import os
input_bucket = os.environ.get("INPUT_BUCKET")
output_bucket = os.environ.get("OUTPUT_BUCKET")
fba_key = os.environ.get("FBA_KEY")
shopify_key = os.environ.get("SHOPIFY_KEY")
output_key = os.environ.get("OUTPUT_KEY")
```

### Step 5: Test the Lambda Function

1. **Create a Test Event:**
   - In the Lambda console, click **Test**.
   - Create a new test event (the event body can be empty if you’re not triggering based on event data).
2. **Run the Test:**
   - Verify that the function executes without errors.
   - Check the output S3 bucket for the generated `audit_report.csv`.
3. **Review CloudWatch Logs** for debugging or any errors.

### Step 6: Set Up a Trigger (Optional)

If you’d like the Lambda to run automatically (for example, when files are uploaded to S3):
1. **In the Lambda Function’s Designer**, click **Add trigger**.
2. **Select S3** as the trigger.
3. **Configure the trigger:**
   - Choose the input S3 bucket.
   - Set an event type (e.g., ObjectCreated).
   - Optionally, add a prefix/suffix filter (such as `.csv`).
4. **Save the Trigger** and test the end-to-end flow.

---

By following these steps, you’ll have an AWS Lambda function that processes inventory CSV files stored in S3, performs the reconciliation and discrepancy flagging, and writes the audit report back to S3. This setup can be integrated into automated workflows to help manage large datasets and trigger invoicing policies based on data volume.