# large-datasets-AWS-lake-formation
This workshop teaches you how to build, secure, and manage a data lake using AWS Lake Formation. You'll work with the TPC-DS benchmark dataset to implement fine-grained access controls, column-level security, row-level filtering, and cross-account data sharing.

# AWS Lake Formation Workshop - TPC-DS Retail Analytics

A comprehensive hands-on workshop for learning AWS Lake Formation with real-world TPC-DS retail dataset.

[![AWS](https://img.shields.io/badge/AWS-Lake%20Formation-orange)](https://aws.amazon.com/lake-formation/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Level](https://img.shields.io/badge/Level-Intermediate-yellow)](.)

## 📋 Overview

This workshop teaches you how to build, secure, and manage a data lake using AWS Lake Formation. You'll work with the TPC-DS benchmark dataset to implement fine-grained access controls, column-level security, row-level filtering, and cross-account data sharing.

**Duration:** 4-5 hours  
**Cost:** ~$10-15 USD  
**Region:** us-east-1 (recommended)

## 🎯 Learning Objectives

- ✅ Deploy a complete data lake infrastructure using CloudFormation
- ✅ Configure AWS Lake Formation for centralized data governance
- ✅ Implement column-level security to protect PII data
- ✅ Create row-level filters for data access control
- ✅ Grant different permissions to multiple user personas
- ✅ Query data securely using Amazon Athena
- ✅ Share data across AWS accounts
- ✅ Monitor and audit data access

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    TPC-DS Data Sources                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │Web_Sales │  │ Customer │  │   Item   │  │Date_Dim  │   │
└───────┬──────────────┬──────────────┬──────────────┬────────┘
        │              │              │              │
        └──────────────┴──────────────┴──────────────┘
                       │
        ┌──────────────▼───────────────┐
        │    Amazon S3 Data Lake       │
        │  (TPC-DS Parquet Data)       │
        └──────────────┬───────────────┘
                       │
        ┌──────────────▼───────────────┐
        │   AWS Lake Formation         │
        │  • Data Catalog (Glue)       │
        │  • Fine-Grained Access       │
        │  • Row/Column Security       │
        └──────────────┬───────────────┘
                       │
        ┌──────────────┴───────────────┐
        │                              │
┌───────▼────────┐          ┌─────────▼─────────┐
│ Amazon Athena  │          │ Amazon SageMaker  │
│  (Analytics)   │          │  (ML Workloads)   │
└────────────────┘          └───────────────────┘
```

## 📊 TPC-DS Dataset

The workshop uses the TPC-DS benchmark dataset with 14 tables:

**Fact Table:**
- `web_sales` - 35 columns with sales transactions

**Dimension Tables:**
- `customer` - Customer master data
- `customer_demographics` - Demographic information
- `customer_address` - Address details
- `household_demographics` - Household info
- `income_band` - Income classification
- `date_dim` - Date dimension
- `time_dim` - Time dimension
- `item` - Product catalog
- `promotion` - Marketing campaigns
- `web_page` - Web page metadata
- `web_site` - Website information
- `warehouse` - Warehouse locations
- `ship_mode` - Shipping methods

## 🚀 Quick Start

### Prerequisites

- AWS Account with admin access
- EC2 Key Pair in us-east-1
- AWS CLI installed and configured
- Basic SQL and data warehousing knowledge

### Deploy Workshop (5 minutes)

```bash
# Clone repository
git clone https://github.com/NileshjVaghela/aws-lakeformation-workshop.git
cd aws-lakeformation-workshop

# Deploy CloudFormation stack
aws cloudformation create-stack \
  --stack-name lf-workshop \
  --template-body file://lf-workshop.template \
  --parameters \
    ParameterKey=EEKeyPair,ParameterValue=YOUR_KEY_PAIR \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# Wait for completion (~15-20 minutes)
aws cloudformation wait stack-create-complete \
  --stack-name lf-workshop
```

### Get User Credentials

```bash
# Get password for all workshop users
aws secretsmanager get-secret-value \
  --secret-id lf-workshop-lf-users-credentials \
  --query SecretString --output text | jq -r '.password'

# Get console login URL
aws cloudformation describe-stacks \
  --stack-name lf-workshop \
  --query 'Stacks[0].Outputs[?OutputKey==`ConsoleIAMLoginUrl`].OutputValue' \
  --output text
```

## 👥 User Personas

The workshop creates 4 IAM users with different roles:

| Username | Role | Permissions |
|----------|------|-------------|
| `lf-data-admin` | Data Lake Administrator | Full Lake Formation admin access |
| `lf-data-engineer` | Data Engineer | Create/manage databases and tables |
| `lf-data-analyst` | Data Analyst | Query with column restrictions |
| `lf-data-scientist` | Data Scientist | Access with row-level filters |

All users share the same password (stored in Secrets Manager).

## 📚 Workshop Modules

### Module 1: Environment Setup (30 min)
- Deploy CloudFormation stack
- Verify resources created
- Understand infrastructure components

### Module 2: Understanding Resources (15 min)
- Explore IAM users and roles
- Review S3 buckets and data
- Check Glue catalog

### Module 3: Data Catalog Setup (45 min)
- Run Glue crawler
- Verify catalog tables
- Explore table schemas

### Module 4: Lake Formation Configuration (60 min)
- Configure Lake Formation settings
- Register S3 locations
- Revoke IAM permissions

### Module 5: Data Engineer Permissions (45 min)
- Grant database permissions
- Grant table permissions
- Test access

### Module 6: Column-Level Security (60 min)
- Implement PII protection
- Grant selective column access
- Test restrictions

### Module 7: Row-Level Security (60 min)
- Create data filters
- Apply row-level filtering
- Verify filtered access

### Module 8: Analytics Queries (45 min)
- Run sales analysis queries
- Create analytical views
- Generate insights

### Module 9: Cross-Account Sharing (45 min)
- Share database with another account
- Accept resource share
- Query shared data

### Module 10: Monitoring & Auditing (30 min)
- Enable CloudTrail logging
- Query access logs
- Monitor data usage

## 🔍 Sample Queries

**Top Selling Items:**
```sql
SELECT 
    i.i_item_desc,
    SUM(ws.ws_quantity) as total_quantity,
    SUM(ws.ws_net_profit) as total_profit
FROM tpc.web_sales ws
JOIN tpc.item i ON ws.ws_item_sk = i.i_item_sk
GROUP BY i.i_item_desc
ORDER BY total_profit DESC
LIMIT 10;
```

**Customer Demographics Analysis:**
```sql
SELECT 
    cd.cd_gender,
    cd.cd_education_status,
    COUNT(DISTINCT c.c_customer_sk) as customer_count,
    AVG(ws.ws_net_profit) as avg_profit
FROM tpc.customer c
JOIN tpc.customer_demographics cd ON c.c_current_cdemo_sk = cd.cd_demo_sk
JOIN tpc.web_sales ws ON c.c_customer_sk = ws.ws_bill_customer_sk
GROUP BY cd.cd_gender, cd.cd_education_status;
```

## 🔒 Security Features Demonstrated

- **Column-Level Security:** Restrict access to PII columns (names, emails, birth dates)
- **Row-Level Security:** Filter data based on date ranges or regions
- **Tag-Based Access Control:** Use LF-Tags for scalable permissions
- **Cross-Account Sharing:** Securely share data with external accounts
- **Audit Logging:** Track all data access via CloudTrail

## 🧹 Cleanup

```bash
# Delete CloudFormation stack
aws cloudformation delete-stack --stack-name lf-workshop

# Verify deletion
aws cloudformation wait stack-delete-complete --stack-name lf-workshop
```

**Note:** Manual cleanup may be required for:
- S3 buckets (if not empty)
- Lake Formation permissions
- CloudTrail logs

## 📖 Documentation

- [Complete Workshop Guide](AWS-LakeFormation-Workshop-Complete.md) - Detailed step-by-step instructions
- [CloudFormation Template](lf-workshop.template) - Infrastructure as Code
- [TPC-DS Entity Diagram](tpc_data_entity.png) - Data model reference

## 🛠️ Troubleshooting

### Crawler Fails
```bash
# Check crawler logs
aws glue get-crawler --name "TPC Crawler" --query 'Crawler.LastCrawl'

# Verify IAM role permissions
aws iam get-role --role-name LF-GlueServiceRole
```

### Permission Denied in Athena
```bash
# List Lake Formation permissions
aws lakeformation list-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::ACCOUNT_ID:user/lf-data-analyst

# Check if IAMAllowedPrincipals still has access
aws lakeformation list-permissions \
  --resource-type DATABASE \
  --resource '{"Database": {"Name": "tpc"}}'
```

### Cannot See Shared Database
- Verify RAM invitation was accepted
- Check Lake Formation permissions in both accounts
- Ensure target account opted into Lake Formation

## 💡 Best Practices

1. **Always revoke IAMAllowedPrincipals** - Enforce Lake Formation permissions
2. **Use LF-Tags for scale** - Manage permissions across multiple resources
3. **Implement least privilege** - Grant only necessary permissions
4. **Enable CloudTrail** - Monitor all data access
5. **Test permissions** - Verify access controls work as expected
6. **Document data lineage** - Track data transformations

## 🤝 Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit changes (`git commit -am 'Add new feature'`)
4. Push to branch (`git push origin feature/improvement`)
5. Open a Pull Request

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- AWS Lake Formation team for the service
- TPC for the benchmark dataset specification
- AWS Workshop Studio for inspiration

## 📧 Support

- **Issues:** [GitHub Issues](https://github.com/YOUR_USERNAME/aws-lakeformation-workshop/issues)
- **Questions:** [GitHub Discussions](https://github.com/YOUR_USERNAME/aws-lakeformation-workshop/discussions)
- **AWS Support:** [AWS Support Center](https://console.aws.amazon.com/support/)

## 🔗 Additional Resources

- [AWS Lake Formation Documentation](https://docs.aws.amazon.com/lake-formation/)
- [AWS Glue Documentation](https://docs.aws.amazon.com/glue/)
- [Amazon Athena Documentation](https://docs.aws.amazon.com/athena/)
- [TPC-DS Specification](http://www.tpc.org/tpcds/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

---

**⭐ If you find this workshop helpful, please star the repository!**

**📢 Share your feedback and improvements via GitHub Issues**
