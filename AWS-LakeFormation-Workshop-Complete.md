# AWS Lake Formation Workshop - TPC-DS Retail Analytics
## Complete Hands-On Exercise

---

## Workshop Overview

This workshop demonstrates building a secure data lake using AWS Lake Formation with TPC-DS (Transaction Processing Performance Council - Decision Support) benchmark dataset for retail analytics.

**Duration:** 4-5 hours  
**Cost:** ~$10-15  
**Level:** Intermediate

---

## Architecture

The workshop creates:
- **VPC** with public/private subnets
- **S3 Data Lake** with TPC-DS parquet data
- **AWS Glue** for data catalog
- **Lake Formation** for fine-grained access control
- **4 User Personas** with different permissions
- **Amazon Athena** for querying
- **Amazon SageMaker** for ML workloads

---

## Prerequisites

1. AWS Account with admin access
2. EC2 Key Pair in your region
3. AWS CLI installed and configured
4. Basic understanding of SQL and data warehousing

---

## Module 1: Deploy Workshop Infrastructure (30 minutes)

### 1.1 Download CloudFormation Template

Save the CloudFormation template as `lf-workshop.template`

### 1.2 Deploy Stack via AWS Console

**Step 1:** Navigate to CloudFormation Console
```
https://console.aws.amazon.com/cloudformation
```

**Step 2:** Create Stack
- Click "Create stack" → "With new resources"
- Choose "Upload a template file"
- Upload `lf-workshop.template`
- Click "Next"

**Step 3:** Configure Stack Parameters
```
Stack name: lf-workshop
Database Name: tpc (default)
Master Username: tpcadmin (default)
Master Password: BigData26! (default)
EC2 Key Pair: [Select your key pair]
```

**Step 4:** Deploy
- Click "Next" through options
- Check "I acknowledge that AWS CloudFormation might create IAM resources"
- Click "Create stack"

**Wait Time:** 15-20 minutes for stack creation

### 1.3 Deploy via AWS CLI (Alternative)

```bash
aws cloudformation create-stack \
  --stack-name lf-workshop \
  --template-body file://lf-workshop.template \
  --parameters \
    ParameterKey=TPCDBName,ParameterValue=tpc \
    ParameterKey=DBMasterUser,ParameterValue=tpcadmin \
    ParameterKey=DBMasterPassword,ParameterValue=BigData26! \
    ParameterKey=EEKeyPair,ParameterValue=YOUR_KEY_PAIR \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# Monitor stack creation
aws cloudformation describe-stacks \
  --stack-name lf-workshop \
  --query 'Stacks[0].StackStatus'
```

### 1.4 Verify Stack Outputs

```bash
# Get stack outputs
aws cloudformation describe-stacks \
  --stack-name lf-workshop \
  --query 'Stacks[0].Outputs' \
  --output table
```

**Key Outputs:**
- `LFDataLakeBucketName`: lf-data-lake-{account-id}
- `LFWorkshopBucketName`: lf-workshop-{account-id}
- `AthenaQueryResultLocation`: s3://lf-workshop-{account-id}/athena-results/
- `LFUsersCredentials`: Link to Secrets Manager for user passwords
- `ConsoleIAMLoginUrl`: IAM login URL

---

## Module 2: Understanding Workshop Resources (15 minutes)

### 2.1 IAM Users Created

The stack creates 4 user personas:

| User | Username | Role | Permissions |
|------|----------|------|-------------|
| **Data Admin** | lf-data-admin | Administrator | Full Lake Formation admin |
| **Data Engineer** | lf-data-engineer | ETL Developer | Create/manage databases and tables |
| **Data Analyst** | lf-data-analyst | Analyst | Query specific tables with restrictions |
| **Data Scientist** | lf-data-scientist | ML Engineer | Access for ML workloads |

### 2.2 Get User Passwords

```bash
# Get password from Secrets Manager
aws secretsmanager get-secret-value \
  --secret-id lf-workshop-lf-users-credentials \
  --query SecretString \
  --output text | jq -r '.password'
```

Or via Console:
```
Navigate to: Secrets Manager → lf-workshop-lf-users-credentials → Retrieve secret value
```

### 2.3 S3 Buckets Created

```bash
# List data lake bucket contents
aws s3 ls s3://lf-data-lake-${ACCOUNT_ID}/tpcparquet/

# Expected folders:
# - customer/
# - customer_address/
# - customer_demographics/
# - date_dim/
# - household_demographics/
# - income_band/
# - item/
# - promotion/
# - ship_mode/
# - time_dim/
# - warehouse/
# - web_page/
# - web_sales/
# - web_site/
```

### 2.4 IAM Roles Created

```bash
# List created roles
aws iam list-roles --query 'Roles[?contains(RoleName, `LF`) || contains(RoleName, `Glue`)].RoleName'
```

**Roles:**
- `LF-GlueServiceRole` - Main Glue/Lake Formation service role
- `DA-GlueServiceRole` - Data Analyst Glue role
- `DE-GlueServiceRole` - Data Engineer Glue role
- `LF-KinesisServiceRole` - Kinesis Firehose role
- `EC2Role` - EC2 instance role

---

## Module 3: Data Catalog Setup (45 minutes)

### 3.1 Login as Data Admin

```
URL: https://{account-id}.signin.aws.amazon.com/console
Username: lf-data-admin
Password: [From Secrets Manager]
```

### 3.2 Run Glue Crawler

**Step 1:** Navigate to AWS Glue Console
```
Services → AWS Glue → Crawlers
```

**Step 2:** Run TPC Crawler
- Find crawler: "TPC Crawler"
- Click "Run crawler"
- Wait for completion (~5-10 minutes)

**Via CLI:**
```bash
# Start crawler
aws glue start-crawler --name "TPC Crawler"

# Check status
aws glue get-crawler --name "TPC Crawler" \
  --query 'Crawler.State'

# Wait for READY status
```

### 3.3 Verify Catalog Tables

```bash
# List databases
aws glue get-databases

# List tables in tpc database
aws glue get-tables --database-name tpc \
  --query 'TableList[].Name' \
  --output table
```

**Expected Tables:**
- customer
- customer_address
- customer_demographics
- date_dim
- household_demographics
- income_band
- item
- promotion
- ship_mode
- time_dim
- warehouse
- web_page
- web_sales
- web_site

### 3.4 Explore Table Schema

```bash
# Get web_sales table schema
aws glue get-table \
  --database-name tpc \
  --name web_sales \
  --query 'Table.StorageDescriptor.Columns' \
  --output table
```

---

## Module 4: Lake Formation Configuration (60 minutes)

### 4.1 Configure Lake Formation Settings

**Step 1:** Navigate to Lake Formation Console
```
Services → Lake Formation → Settings
```

**Step 2:** Add Data Lake Administrators
- Click "Add" under "Data lake administrators"
- Select IAM user: `lf-data-admin`
- Click "Save"

**Via CLI:**
```bash
aws lakeformation put-data-lake-settings \
  --data-lake-settings '{
    "DataLakeAdmins": [
      {
        "DataLakePrincipalIdentifier": "arn:aws:iam::'${AWS_ACCOUNT_ID}':user/lf-data-admin"
      }
    ],
    "CreateDatabaseDefaultPermissions": [],
    "CreateTableDefaultPermissions": []
  }'
```

### 4.2 Register S3 Locations

**Step 1:** Register Data Lake Location
```
Lake Formation → Register and ingest → Data lake locations → Register location
```

**Settings:**
- S3 path: `s3://lf-data-lake-{account-id}/tpcparquet`
- IAM role: `LF-GlueServiceRole`
- Click "Register location"

**Via CLI:**
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws lakeformation register-resource \
  --resource-arn arn:aws:s3:::lf-data-lake-${ACCOUNT_ID}/tpcparquet \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/LF-GlueServiceRole \
  --use-service-linked-role
```

### 4.3 Revoke IAM Permissions (Important!)

By default, IAM principals have "Super" permissions. We need to revoke these to enforce Lake Formation permissions.

**Step 1:** Navigate to Lake Formation → Permissions → Data permissions

**Step 2:** Revoke IAMAllowedPrincipals permissions on database

```bash
# Revoke database permissions
aws lakeformation batch-revoke-permissions \
  --entries '[
    {
      "Id": "1",
      "Principal": {
        "DataLakePrincipalIdentifier": "IAMAllowedPrincipals"
      },
      "Resource": {
        "Database": {
          "Name": "tpc"
        }
      },
      "Permissions": ["ALL"]
    }
  ]'
```

---

## Module 5: Grant Permissions to Data Engineer (45 minutes)

### 5.1 Grant Database Permissions

**Via Console:**
```
Lake Formation → Permissions → Grant
```

**Settings:**
- IAM users and roles: `lf-data-engineer`
- Database: `tpc`
- Database permissions: `CREATE_TABLE`, `DESCRIBE`
- Click "Grant"

**Via CLI:**
```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::${ACCOUNT_ID}:user/lf-data-engineer \
  --permissions "CREATE_TABLE" "DESCRIBE" \
  --resource '{
    "Database": {
      "Name": "tpc"
    }
  }'
```

### 5.2 Grant Table Permissions

**Grant SELECT on all tables:**
```bash
# Get all table names
TABLES=$(aws glue get-tables --database-name tpc --query 'TableList[].Name' --output text)

# Grant permissions on each table
for table in $TABLES; do
  aws lakeformation grant-permissions \
    --principal DataLakePrincipalIdentifier=arn:aws:iam::${ACCOUNT_ID}:user/lf-data-engineer \
    --permissions "SELECT" "DESCRIBE" "ALTER" "INSERT" "DELETE" \
    --resource '{
      "Table": {
        "DatabaseName": "tpc",
        "Name": "'$table'"
      }
    }'
done
```

### 5.3 Test as Data Engineer

**Login as lf-data-engineer and run query in Athena:**

```sql
-- This should work
SELECT COUNT(*) FROM tpc.web_sales;

-- Check customer data
SELECT * FROM tpc.customer LIMIT 10;
```

---

## Module 6: Column-Level Security for Data Analyst (60 minutes)

### 6.1 Grant Limited Access to Data Analyst

**Scenario:** Data Analyst should only see specific columns from customer table (no PII data)

**Step 1:** Grant database DESCRIBE permission
```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::${ACCOUNT_ID}:user/lf-data-analyst \
  --permissions "DESCRIBE" \
  --resource '{
    "Database": {
      "Name": "tpc"
    }
  }'
```

**Step 2:** Grant column-level SELECT on customer table
```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::${ACCOUNT_ID}:user/lf-data-analyst \
  --permissions "SELECT" \
  --resource '{
    "TableWithColumns": {
      "DatabaseName": "tpc",
      "Name": "customer",
      "ColumnNames": [
        "c_customer_sk",
        "c_customer_id",
        "c_current_cdemo_sk",
        "c_current_hdemo_sk",
        "c_current_addr_sk",
        "c_first_shipto_date_sk",
        "c_first_sales_date_sk",
        "c_birth_country",
        "c_login",
        "c_preferred_cust_flag"
      ]
    }
  }'
```

**Note:** Excluded PII columns: `c_first_name`, `c_last_name`, `c_email_address`, `c_birth_day`, `c_birth_month`, `c_birth_year`

### 6.2 Grant Full Access to web_sales

```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::${ACCOUNT_ID}:user/lf-data-analyst \
  --permissions "SELECT" \
  --resource '{
    "Table": {
      "DatabaseName": "tpc",
      "Name": "web_sales"
    }
  }'
```

### 6.3 Test Column-Level Security

**Login as lf-data-analyst:**

```sql
-- This should work (allowed columns)
SELECT 
    c_customer_sk,
    c_customer_id,
    c_birth_country,
    c_preferred_cust_flag
FROM tpc.customer
LIMIT 10;

-- This should FAIL (restricted columns)
SELECT 
    c_first_name,
    c_last_name,
    c_email_address
FROM tpc.customer
LIMIT 10;
```

**Expected Error:**
```
Insufficient Lake Formation permission(s) on c_first_name, c_last_name, c_email_address
```

---

## Module 7: Row-Level Security with Data Filters (60 minutes)

### 7.1 Create Data Filter for Specific Region

**Scenario:** Data Scientist should only see web sales from specific date range

**Step 1:** Create data filter
```bash
aws lakeformation create-data-cells-filter \
  --table-data '{
    "TableCatalogId": "'${ACCOUNT_ID}'",
    "DatabaseName": "tpc",
    "TableName": "web_sales",
    "Name": "recent_sales_filter",
    "RowFilter": {
      "FilterExpression": "ws_sold_date_sk >= 2451545"
    },
    "ColumnNames": ["ws_sold_date_sk", "ws_item_sk", "ws_bill_customer_sk", "ws_quantity", "ws_sales_price", "ws_net_profit"],
    "ColumnWildcard": {}
  }'
```

### 7.2 Grant Filtered Access

```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::${ACCOUNT_ID}:user/lf-data-scientist \
  --permissions "SELECT" \
  --resource '{
    "DataCellsFilter": {
      "TableCatalogId": "'${ACCOUNT_ID}'",
      "DatabaseName": "tpc",
      "TableName": "web_sales",
      "Name": "recent_sales_filter"
    }
  }'
```

### 7.3 Test Row-Level Security

**Login as lf-data-scientist:**

```sql
-- Should only return filtered rows
SELECT 
    ws_sold_date_sk,
    COUNT(*) as order_count,
    SUM(ws_net_profit) as total_profit
FROM tpc.web_sales
GROUP BY ws_sold_date_sk
ORDER BY ws_sold_date_sk;

-- Verify filter is applied
SELECT MIN(ws_sold_date_sk), MAX(ws_sold_date_sk)
FROM tpc.web_sales;
```

---

## Module 8: Analytics Queries (45 minutes)

### 8.1 Sales Analysis Queries

**Query 1: Top Selling Items**
```sql
SELECT 
    i.i_item_id,
    i.i_item_desc,
    COUNT(ws.ws_order_number) as order_count,
    SUM(ws.ws_quantity) as total_quantity,
    SUM(ws.ws_net_profit) as total_profit
FROM tpc.web_sales ws
JOIN tpc.item i ON ws.ws_item_sk = i.i_item_sk
GROUP BY i.i_item_id, i.i_item_desc
ORDER BY total_profit DESC
LIMIT 20;
```

**Query 2: Customer Demographics Analysis**
```sql
SELECT 
    cd.cd_gender,
    cd.cd_marital_status,
    cd.cd_education_status,
    COUNT(DISTINCT c.c_customer_sk) as customer_count,
    AVG(ws.ws_net_profit) as avg_profit
FROM tpc.customer c
JOIN tpc.customer_demographics cd ON c.c_current_cdemo_sk = cd.cd_demo_sk
JOIN tpc.web_sales ws ON c.c_customer_sk = ws.ws_bill_customer_sk
GROUP BY cd.cd_gender, cd.cd_marital_status, cd.cd_education_status
ORDER BY customer_count DESC;
```

**Query 3: Time-Based Sales Trends**
```sql
SELECT 
    d.d_year,
    d.d_moy as month,
    COUNT(ws.ws_order_number) as orders,
    SUM(ws.ws_sales_price * ws.ws_quantity) as revenue,
    SUM(ws.ws_net_profit) as profit
FROM tpc.web_sales ws
JOIN tpc.date_dim d ON ws.ws_sold_date_sk = d.d_date_sk
WHERE d.d_year IN (2000, 2001, 2002)
GROUP BY d.d_year, d.d_moy
ORDER BY d.d_year, d.d_moy;
```

**Query 4: Promotional Campaign Effectiveness**
```sql
SELECT 
    p.p_promo_name,
    p.p_channel_email,
    p.p_channel_dmail,
    COUNT(ws.ws_order_number) as promo_orders,
    SUM(ws.ws_ext_sales_price) as total_sales,
    SUM(ws.ws_net_profit) as net_profit
FROM tpc.web_sales ws
JOIN tpc.promotion p ON ws.ws_promo_sk = p.p_promo_sk
GROUP BY p.p_promo_name, p.p_channel_email, p.p_channel_dmail
HAVING COUNT(ws.ws_order_number) > 100
ORDER BY net_profit DESC;
```

### 8.2 Create Views for Common Queries

```sql
-- Create view for sales summary
CREATE VIEW tpc.sales_summary AS
SELECT 
    d.d_date,
    i.i_category,
    SUM(ws.ws_quantity) as quantity_sold,
    SUM(ws.ws_sales_price * ws.ws_quantity) as revenue
FROM tpc.web_sales ws
JOIN tpc.date_dim d ON ws.ws_sold_date_sk = d.d_date_sk
JOIN tpc.item i ON ws.ws_item_sk = i.i_item_sk
GROUP BY d.d_date, i.i_category;
```

---

## Module 9: Cross-Account Data Sharing (Optional - 45 minutes)

### 9.1 Share Database with Another Account

**Prerequisites:** You need a second AWS account

**Step 1:** Create Resource Share
```bash
TARGET_ACCOUNT_ID=123456789012  # Replace with target account

aws lakeformation create-lake-formation-opt-in \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::${TARGET_ACCOUNT_ID}:root \
  --resource '{
    "Database": {
      "Name": "tpc"
    }
  }'
```

**Step 2:** Grant Permissions to External Account
```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::${TARGET_ACCOUNT_ID}:root \
  --permissions "DESCRIBE" \
  --resource '{
    "Database": {
      "Name": "tpc"
    }
  }'

aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::${TARGET_ACCOUNT_ID}:root \
  --permissions "SELECT" \
  --resource '{
    "Table": {
      "DatabaseName": "tpc",
      "Name": "web_sales"
    }
  }'
```

### 9.2 Accept Share in Target Account

**In target account:**
```bash
# List pending invitations
aws ram get-resource-share-invitations

# Accept invitation
aws ram accept-resource-share-invitation \
  --resource-share-invitation-arn <invitation-arn>
```

---

## Module 10: Monitoring and Auditing (30 minutes)

### 10.1 Enable CloudTrail Logging

```bash
# Create S3 bucket for CloudTrail logs
aws s3 mb s3://lf-audit-logs-${ACCOUNT_ID}

# Create CloudTrail
aws cloudtrail create-trail \
  --name lf-audit-trail \
  --s3-bucket-name lf-audit-logs-${ACCOUNT_ID}

# Start logging
aws cloudtrail start-logging --name lf-audit-trail
```

### 10.2 Query Access Logs

```sql
-- Create Athena table for CloudTrail logs
CREATE EXTERNAL TABLE cloudtrail_logs (
    eventversion STRING,
    useridentity STRUCT<
        type:STRING,
        principalid:STRING,
        arn:STRING,
        accountid:STRING,
        invokedby:STRING,
        accesskeyid:STRING,
        userName:STRING>,
    eventtime STRING,
    eventsource STRING,
    eventname STRING,
    awsregion STRING,
    sourceipaddress STRING,
    useragent STRING,
    errorcode STRING,
    errormessage STRING,
    requestparameters STRING,
    responseelements STRING
)
PARTITIONED BY (region STRING, year STRING, month STRING, day STRING)
ROW FORMAT SERDE 'com.amazon.emr.hive.serde.CloudTrailSerde'
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://lf-audit-logs-${ACCOUNT_ID}/AWSLogs/${ACCOUNT_ID}/CloudTrail/';

-- Query Lake Formation access
SELECT 
    useridentity.arn as user,
    eventname,
    eventsource,
    COUNT(*) as access_count
FROM cloudtrail_logs
WHERE eventsource = 'lakeformation.amazonaws.com'
GROUP BY useridentity.arn, eventname, eventsource
ORDER BY access_count DESC;
```

---

## Validation and Testing

### Test 1: Verify User Permissions

```bash
# Test script
cat > test_permissions.sh << 'EOF'
#!/bin/bash

echo "Testing lf-data-engineer permissions..."
aws athena start-query-execution \
  --query-string "SELECT COUNT(*) FROM tpc.web_sales" \
  --query-execution-context Database=tpc \
  --result-configuration OutputLocation=s3://lf-workshop-${ACCOUNT_ID}/athena-results/

echo "Testing lf-data-analyst permissions..."
aws athena start-query-execution \
  --query-string "SELECT c_customer_sk FROM tpc.customer LIMIT 10" \
  --query-execution-context Database=tpc \
  --result-configuration OutputLocation=s3://lf-workshop-${ACCOUNT_ID}/athena-results/

echo "Testing restricted access..."
aws athena start-query-execution \
  --query-string "SELECT c_first_name FROM tpc.customer LIMIT 10" \
  --query-execution-context Database=tpc \
  --result-configuration OutputLocation=s3://lf-workshop-${ACCOUNT_ID}/athena-results/
EOF

chmod +x test_permissions.sh
./test_permissions.sh
```

### Test 2: Verify Data Filters

```sql
-- As lf-data-scientist
SELECT 
    MIN(ws_sold_date_sk) as min_date,
    MAX(ws_sold_date_sk) as max_date,
    COUNT(*) as total_rows
FROM tpc.web_sales;

-- Verify only filtered data is visible
```

---

## Cleanup

### Step 1: Delete CloudFormation Stack

```bash
# Delete stack
aws cloudformation delete-stack --stack-name lf-workshop

# Monitor deletion
aws cloudformation describe-stacks \
  --stack-name lf-workshop \
  --query 'Stacks[0].StackStatus'
```

### Step 2: Manual Cleanup (if needed)

```bash
# Empty S3 buckets
aws s3 rm s3://lf-data-lake-${ACCOUNT_ID} --recursive
aws s3 rm s3://lf-workshop-${ACCOUNT_ID} --recursive

# Delete buckets
aws s3 rb s3://lf-data-lake-${ACCOUNT_ID}
aws s3 rb s3://lf-workshop-${ACCOUNT_ID}

# Deregister Lake Formation resources
aws lakeformation deregister-resource \
  --resource-arn arn:aws:s3:::lf-data-lake-${ACCOUNT_ID}
```

---

## Troubleshooting

### Issue 1: Crawler Fails

**Problem:** Glue crawler fails to create tables

**Solution:**
- Check IAM role permissions
- Verify S3 bucket has data
- Check CloudWatch logs for crawler

### Issue 2: Permission Denied in Athena

**Problem:** User gets "Insufficient permissions" error

**Solution:**
```bash
# Verify Lake Formation permissions
aws lakeformation list-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::${ACCOUNT_ID}:user/lf-data-analyst

# Check if IAMAllowedPrincipals still has permissions
aws lakeformation list-permissions \
  --resource-type DATABASE \
  --resource '{"Database": {"Name": "tpc"}}'
```

### Issue 3: Cannot See Shared Database

**Problem:** Cross-account sharing not working

**Solution:**
- Verify RAM invitation was accepted
- Check Lake Formation permissions in both accounts
- Ensure target account has opted into Lake Formation

---

## Key Takeaways

1. **Lake Formation provides centralized governance** for data lakes
2. **Column-level security** protects sensitive PII data
3. **Row-level security** enables data filtering based on business rules
4. **Cross-account sharing** enables secure data collaboration
5. **IAM permissions must be revoked** to enforce Lake Formation controls

---

## Next Steps

- Implement data quality checks with AWS Glue Data Quality
- Set up automated ETL pipelines
- Integrate with Amazon QuickSight for visualization
- Explore Lake Formation governed tables
- Implement data lineage tracking

---

## Additional Resources

- [AWS Lake Formation Documentation](https://docs.aws.amazon.com/lake-formation/)
- [TPC-DS Benchmark Specification](http://www.tpc.org/tpcds/)
- [AWS Glue Best Practices](https://docs.aws.amazon.com/glue/latest/dg/best-practices.html)

---

**Workshop Version:** 1.0  
**Last Updated:** January 2026  
**License:** MIT
