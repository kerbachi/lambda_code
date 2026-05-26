# SMUS Roles and permissions

## SMUS Domain Creation: Roles & Permissions

Roles to be pre-created and to be passed at SMUS Domain creation time

### (1) Domain Execution role
This is an IAM role that Amazon SageMaker Unified Studio requires to call APIs on behalf of authorized users, including those logged in to Amazon SageMaker Unified Studio.

**More info on role**: https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/AmazonSageMakerDomainExecution.html

Managed Policy to be attached (not to be created, managed by AWS): **SageMakerStudioDomainExecutionRolePolicy** (https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioDomainExecutionRolePolicy.html)

Role Trust Policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
      {
          "Effect": "Allow",
          "Principal": {
              "Service": "datazone.amazonaws.com"
          },
          "Action": [
              "sts:AssumeRole",
              "sts:TagSession",
              "sts:SetContext"
          ],
          "Condition": {
              "StringEquals": {
                  "aws:SourceAccount": "{{source_account_id}}"
              },
              "ForAllValues:StringLike": {
                  "aws:TagKeys": "datazone*"
              }
          }
      }
  ]
}
```

Note: your Amazon DataZone domain, metadata, and reporting data is encrypted by the AWS Key Management Service (KMS) using a key specific to your Amazon DataZone. You can specify whether you want to use an AWS owned key or choose a different AWS KMS key. In case of customer AWS Key, add an additional IAM policy to the `Domain Execution role`: https://docs.aws.amazon.com/datazone/latest/userguide/create-domain.html . 


### (2) Domain Service role

This is a service role for domain level actions performed by Amazon SageMaker Unified Studio.

More info on role: https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/AmazonSageMakerDomainService.html

Managed Policy to be attached (not to be created, managed by AWS): **SageMakerStudioDomainServiceRolePolicy** (https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioDomainServiceRolePolicy.html)

Role Trust Policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "datazone.amazonaws.com"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
            "aws:SourceAccount": "{{domain_account}}"
        }
      }
    }
  ]
}
```


## Tooling blueprint enablement: Roles & Permissions

To pass when enabling the Tooling Blueprint in one SMUS domain in one account.

### (1) Provisioning role
Amazon SageMaker Unified Studio uses this role to provision and manage resources defined in the selected blueprints in your account.

More info on role: https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/AmazonSageMakerProvisioning.html

Managed Policy to be attached (not to be created, managed by AWS): **SageMakerStudioProjectProvisioningRolePolicy** (https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioProjectProvisioningRolePolicy.html)

Role Trust Policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "datazone.amazonaws.com"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
            "aws:SourceAccount": "{{domain_account}}"
        }
      }
    }
  ]
}
```

Note: (https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/AmazonSageMakerProvisioning.html) If you are using your own query execution role (instead of the default AmazonSageMakerQueryExecution role), then you must modify the permissions of your provisioning role (whether you're using this default AmazonSageMakerProvisioning role or your own custom provisioning role) to include iam:PassRole and iam:GetRole permissions. These permissions enable your provisioning role to pass the query execution role to AWS LakeFormation during creation of federated connections. You can include these permissions by attaching the following inline policy to your provisioning role:

```
{
  "Version":"2012-10-17",
  "Statement": [
    {
      "Sid": "IamRolePermissionsForQueryExecution",
      "Effect": "Allow",
      "Action": [
        "iam:PassRole",
        "iam:GetRole"
      ],
      "Resource": "arn:aws:iam::*:role/{your-query-execution-role}"
    }
  ]
}
```

### (2) Manage Access role
This role grants Amazon SageMaker Unified Studio permissions to publish, grant access, and revoke access to Amazon SageMaker Lakehouse, AWS Glue Data Catalog and Amazon Redshift data. It also grants Amazon SageMaker Unified Studio to publish and manage subscriptions on Amazon SageMaker Catalog data and AI assets.

More info on role: https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/AmazonSageMakerManageAccess.html

Managed Policies to be attached (not to be created, managed by AWS):
- **AmazonDataZoneGlueManageAccessRolePolicy** (https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AmazonDataZoneGlueManageAccessRolePolicy.html)
- **AmazonDataZoneRedshiftManageAccessRolePolicy** (https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AmazonDataZoneRedshiftManageAccessRolePolicy.html)

Custom policy to be created and attached:

- **AmazonSageMakerManageAccessPolicy** Custom policy (e.g., AmazonSageMakerManageAccessPolicy) - needed if enabling the `LakeHouseDatabase` blueprint:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "RedshiftSecretStatement",
            "Effect": "Allow",
            "Action": "secretsmanager:GetSecretValue",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "secretsmanager:ResourceTag/AmazonDataZoneDomain": "<SMUS_DOMAIN_ID>"
                }
            }
        }
    ]
}
```

Role Trust Policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "datazone.amazonaws.com"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "aws:SourceAccount": "<AWS_ACCCOUNT_ID>"
                },
                "ArnEquals": {
                    "aws:SourceArn": "arn:aws:datazone:<AWS_REGION>:<AWS_ACCCOUNT_ID>:domain/<SMUS_DOMAIN_ID>"
                }
            }
        }
    ]
}
```

### (3) Query Execution role
This role is used while running a query execution. AWS LakeFormation assumes this role to vend credentials needed by Amazon Athena during query execution.

More info on role: https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/AmazonSageMakerQueryExecution.html

Managed Policy to be attached (not to be created, managed by AWS): **SageMakerStudioQueryExecutionRolePolicy**(https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioQueryExecutionRolePolicy.html

Role Trust Policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "lakeformation.amazonaws.com",
                    "glue.amazonaws.com"
                ]
            },
            "Action": [
                "sts:AssumeRole",
                "sts:SetContext"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:SourceAccount": "<AWS_ACCCOUNT_ID>"
                }
            }
        }
    ]
}
```



### (4) User role policy

Can be specified at tooling blueprint enablement  or later (updating the Tooling blueprint).

- Amazon SageMaker Unified Studio creates IAM roles for project users to perform data analytics, AI, and ML actions. You can attach your own AWS IAM policies to the role rather than using the default system-managed policy **SageMakerStudioProjectUserRolePolicy** (https://docs.aws.amazon.com/aws-managed-policy/latest/reference/SageMakerStudioProjectUserRolePolicy.html). This provides more granular control over permissions but requires knowledge of IAM policy configuration. The IAM policy must include all necessary permissions required for the service to function properly.

**When updated later with projects already created**: Project owners receive a notice to update their existing projects with the new permissions. Once a project is updated, the new permissions takes effect.



## Project Creation: Roles & Permissions

These roles are created as part of Project creation. One of each per project.
Using the new feature we can incorporate IAM permissions boundaries to created roles.

### (1) Project Role

Created as part of the Tooling blueprint.

Example: **datazone_usr_role_<project_id>_<domain_id>**

Managed policies attached:
- SageMakerStudioBedrockKnowledgeBaseServiceRolePolicy (https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioBedrockKnowledgeBaseServiceRolePolicy.html)
- SageMakerStudioProjectRoleMachineLearningPolicy (https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioProjectRoleMachineLearningPolicy.html)
- SageMakerStudioProjectUserRolePolicy (https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioProjectUserRolePolicy.html)

Role Trust Policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "auth.datazone.amazonaws.com",
                    "datazone.amazonaws.com"
                ]
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession",
                "sts:SetContext",
                "sts:SetSourceIdentity"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:SourceAccount": "<AWS_ACCCOUNT_ID>"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "airflow.amazonaws.com",
                    "athena.amazonaws.com",
                    "glue.amazonaws.com",
                    "sagemaker.amazonaws.com",
                    "bedrock.amazonaws.com",
                    "lambda.amazonaws.com",
                    "scheduler.amazonaws.com",
                    "airflow-env.amazonaws.com",
                    "emr-serverless.amazonaws.com",
                    "lakeformation.amazonaws.com",
                    "quicksight.amazonaws.com",
                    "access-grants.s3.amazonaws.com",
                    "airflow-serverless.amazonaws.com"
                ]
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession",
                "sts:SetContext",
                "sts:SetSourceIdentity"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:SourceAccount": "<AWS_ACCCOUNT_ID>"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "redshift-serverless.amazonaws.com",
                    "redshift.amazonaws.com"
                ]
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringLike": {
                    "sts:ExternalId": "arn:aws:redshift:*:<AWS_ACCCOUNT_ID>:dbuser:*/*"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "redshift-serverless.amazonaws.com",
                    "redshift.amazonaws.com"
                ]
            },
            "Action": [
                "sts:TagSession",
                "sts:SetContext",
                "sts:SetSourceIdentity"
            ]
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<AWS_ACCCOUNT_ID>:role/service-role/AmazonSageMakerProvisioning-<AWS_ACCCOUNT_ID>"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession",
                "sts:SetContext",
                "sts:SetSourceIdentity"
            ]
        }
    ]
}
```

### (2) Amazon Bedrock Service Role

Example: `AmazonBedrockServiceRole-<project_id>_<domain_id>`

Created as part of the Tooling blueprint.

Created if in the Project Profile we setup `true` for the `enableAmazonBedrockPermissions` Tooling Blueprint parameter (true by default).

Manged policies attached:
- SageMakerStudioBedrockAgentServiceRolePolicy (https://docs.aws.amazon.com/it_it/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioBedrockAgentServiceRolePolicy.html)
- SageMakerStudioBedrockEvaluationJobServiceRolePolicy (https://docs.aws.amazon.com/it_it/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioBedrockEvaluationJobServiceRolePolicy.html)
- SageMakerStudioBedrockFlowServiceRolePolicy (https://docs.aws.amazon.com/it_it/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioBedrockFlowServiceRolePolicy.html)
- SageMakerStudioBedrockKnowledgeBaseServiceRolePolicy (https://docs.aws.amazon.com/it_it/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioBedrockKnowledgeBaseServiceRolePolicy.html)

Role Trust Policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "bedrock.amazonaws.com"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "aws:SourceAccount": "<AWS_ACCCOUNT_ID>"
                },
                "ArnLike": {
                    "aws:SourceArn": "arn:aws:bedrock:<AWS_REGION>:<AWS_ACCCOUNT_ID>:*"
                }
            }
        }
    ]
}
```

### (3) Amazon Bedrock Lambda Execution Role

Example: `AmazonBedrockLambdaExecutionRole-<project_id>_<domain_id>`

Created if in the Project Profile we setup `true` for the `enableAmazonBedrockPermissions` Tooling Blueprint parameter (true by default).


Manged policies attached:
- AWSLambdaBasicExecutionRole (https://docs.aws.amazon.com/it_it/aws-managed-policy/latest/reference/AWSLambdaBasicExecutionRole.html)
- SageMakerStudioBedrockFunctionExecutionRolePolicy (https://docs.aws.amazon.com/it_it/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioBedrockFunctionExecutionRolePolicy.html)
- SageMakerStudioBedrockKnowledgeBaseCustomResourcePolicy (https://docs.aws.amazon.com/it_it/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-SageMakerStudioBedrockKnowledgeBaseCustomResourcePolicy.html)

Role Trust Policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "aws:SourceAccount": "<AWS_ACCCOUNT_ID>"
                },
                "ArnLike": {
                    "aws:SourceArn": "arn:aws:lambda:<AWS_REGION>:<AWS_ACCCOUNT_ID>:function:*"
                }
            }
        }
    ]
}
```


## Bedrock models enablement: Roles & Permissions

These roles are service roles created automatically by Amazon SageMaker Unified Studio when you enable Bedrock model access within a SageMaker project/domain.

Example of roles:
- **AmazonSageMakerBedrockModelConsumptionRole-1a89e1c83573e0**
- **AmazonSageMakerBedrockModelConsumptionRole-32d3755ee9bf81**
- **AmazonSageMakerBedrockModelVideoConsumptionRole-dfa33dfa57a0d3**

| Role Pattern | Purpose |
|---|---|
| `AmazonSageMakerBedrockModelConsumptionRole-*` | Allows SageMaker to **invoke Amazon Bedrock foundation models** (text/chat models like Claude, Titan, etc.) on behalf of users in a specific project or space. Each role is scoped to a particular model or set of models. |
| `AmazonSageMakerBedrockModelVideoConsumptionRole-*` | Same concept but specifically for **Bedrock video generation models** (e.g., Amazon Nova Reel, Stability AI video). |

SageMaker creates one role per Bedrock model (or model group) that's enabled in your domain/project. The hex suffix (e.g., -1a89e1c83573e0) is a hash that uniquely identifies the model + project combination. This is a least-privilege design — each role's trust policy only allows the specific SageMaker execution context to assume it, and each role's permission policy only grants bedrock:InvokeModel for that specific model ARN.

**AmazonDataZoneBedrockModelManagementRole**:
- More info: https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/AmazonDataZoneBedrockModelManagementRole.html
- Amazon SageMaker Unified Studio uses this role to create an inference profile for an Amazon Bedrock model in a project. The inference profile is required for the project to interact with the model. You can either let Amazon SageMaker Unified Studio automatically create a unique provisioning role, or you can provide a custom provisioning role.
- Manged policy: **AmazonDataZoneBedrockModelManagementPolicy** (https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/security-iam-awsmanpol-AmazonDataZoneBedrockModelManagementPolicy.html)




## Notes:
- Supported blueprints: https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/supported-blueprints.html
- Manage tolling blueprint parameters: https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/manage-tooling-blueprint.html (can be modified at project profile creation time)
- Append IAM policies with denies using a custom blueprint: https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/custom-blueprint.html
- GenAI assistence:
    - Data Agent (SMUS pricing) is used for:
        - Data Notebooks
        - SQL Editor helper
    - Amazon Q Developer (per-user free tier + per-user Kiro subscriptions) is used for:
        - Domain Chat
        - JupyterLab (e.g., chat, fix with AI)
        - Visual ETL GenAI generation
- In the SMUS domain configuration page we can enable/disable Data Agent and Amazon Q for GenAI support
- Amazon Q Developer Free Tier (https://aws.amazon.com/q/developer/pricing/ ): 
    - No subscription required - automatically enabled for all SageMaker Unified Studio domain users
    - Includes basic AI assistance capabilities for development tasks
- Amazon Q Developer Pro:
    - Needed if surpassing usage limits of free tier
    - To be managed in Kiro AWS Console section
- Amazon Q stores and processes your data in the N. Virginia (us-east-1) Region, even if the Amazon SageMaker Unified Studio domain is in a different AWS Region.
- Amazon Sagemaker provisioning role has an AWS managed policy with condition `aws:TagKeys` allowing tags to be created by this role only if the tag key begins with `AmazonDataZone`. See https://aws.amazon.com/blogs/big-data/use-amazon-sagemaker-custom-tags-for-project-resource-governance-and-cost-tracking/ to allow creation of tags with other prefixes + https://docs.aws.amazon.com/sagemaker-unified-studio/latest/adminguide/project-resource-tags.html