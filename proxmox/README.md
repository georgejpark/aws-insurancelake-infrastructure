# aws-insurancelake-infrastructure on Proxmox

| | |
|---|---|
| Upstream | aws-solutions-library-samples/aws-insurancelake-infrastructure (MIT-0) — mirror snapshot 2025-07-31 |
| Kind | infra-as-code (AWS CDK v2, Python) |
| Cloud verdict | REQUIRED — this repository *is* AWS CDK code; it provisions AWS services and cannot run on Proxmox |
| GPU | none |
| Target host | none (pve5 LXC only to run its unit tests) |
| Effort | L for a port (it would be a rewrite); S to run the tests |

## What it is
The infrastructure half of AWS InsuranceLake (the ETL half is a sibling repo). `app.py` builds CDK Pipelines stacks per environment (Dev/Test/Prod) that deploy an S3 bucket-zones stack (Collect/Cleanse/Consume buckets + access-log bucket + KMS key), a VPC stack (multi-AZ VPC, security groups, VPC endpoints, flow logs) and an optional CodeCommit mirror, all driven from a central deployment account through CodePipeline/CodeBuild. It is AWS-native by design, not an application that happens to use AWS.

## Cloud dependency evidence
| File:line | What | Needed for core use? |
|---|---|---|
| `app.py:5` | `import aws_cdk as cdk`; `cdk.App()` / `app.synth()` | Yes — the whole program is a CDK app |
| `cdk.json:2` | `"app": "python3 app.py"` for the `cdk` CLI | Yes |
| `requirements.txt` | `aws-cdk-lib`, `boto3`, `cdk-nag` | Yes |
| `lib/configuration.py:4,74` | `boto3.client('sts').get_caller_identity()` at config load — even `cdk synth` needs AWS credentials | Yes |
| `lib/pipeline_stack.py:5-11,105` | `aws_cdk.pipelines.CodePipeline`, `aws_s3`, `aws_iam`, `aws_logs`, `aws_codepipeline`, `aws_codebuild`, `aws_codecommit` | Yes |
| `lib/s3_bucket_zones_stack.py:142,247,319` | `kms.Key`, `s3.Bucket` ×4 | Yes |
| `lib/vpc_stack.py:62,73` | `ec2.Vpc`, `logs.LogGroup` | Yes |
| `lib/code_commit_stack.py:5-6` | `aws_iam`, `aws_codecommit` | Optional stack, still AWS |
| `lib/prerequisites/bootstrap_deployment_account.sh`, `bootstrap_target_account.sh` | `aws sts get-caller-identity`, `cdk bootstrap` | Yes |

## Runtime requirements
- Python **>=3.9** (`setup.py:21`), pip; `aws-cdk-lib`, `boto3`, `cdk-nag`; Node.js for the `cdk` CLI (`npm install -g aws-cdk`); AWS CLI with credentials for a deployment account plus Dev/Test/Prod target accounts; `cdk bootstrap` in each.
- Env vars: `ENV` (`Dev`/`Test`/`Prod`, `app.py:46,65,84`), `IS_BOOTSTRAP` (`app.py:23`), `CDK_NEW_BOOTSTRAP` (bootstrap scripts), standard `AWS_PROFILE` / `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_DEFAULT_REGION`. Secrets: AWS credentials; optional CodeConnections ARN in `lib/configuration.py`.
- Ports/databases: none locally. In AWS: S3, KMS, VPC, CodePipeline, CodeBuild, CodeCommit, CloudWatch Logs (plus Glue/Lambda/Step Functions/DynamoDB/Athena from the ETL repo).
- Unit tests: `requirements-dev.txt` (`pytest`, `pytest-cov`); `test/boto_mocking_helper.py:1-12` mocks the STS call, so `pytest` should run without AWS credentials ("to verify" for every test module).

## Proxmox plan
There is nothing to deploy on Proxmox: `cdk deploy` creates AWS resources only. What can run locally is the test suite, and a **port** would be a new project rather than a configuration change:

| AWS piece | Proxmox substitute (if you rebuild it) |
|---|---|
| 3 S3 zone buckets + access-log bucket, SSE-KMS | MinIO on pve5 (4 buckets, SSE-S3 with `MINIO_KMS_SECRET_KEY`) |
| VPC, subnets, security groups, endpoints | Proxmox bridge/VLAN + nftables on the containers |
| CodePipeline / CodeBuild / CodeCommit | Gitea + Gitea Actions (or Jenkins) in an LXC |
| CloudWatch Logs / flow logs | journald + Loki/Grafana or plain rsyslog |
| Glue / Lambda / Step Functions / DynamoDB / Athena (ETL repo) | Spark on a VM + Airflow + PostgreSQL + Trino — a rewrite of the ETL code |

Test-only container: LXC on **pve5**, Debian 13, 2 vCPU, 2 GB RAM, 8 GB disk, unprivileged.
1. Install:
   ```bash
   apt-get update && apt-get install -y python3 python3-venv python3-pip git
   git clone https://github.com/aws-solutions-library-samples/aws-insurancelake-infrastructure.git /opt/insurancelake && cd /opt/insurancelake
   python3 -m venv .venv && . .venv/bin/activate
   pip install -r requirements-dev.txt
   ```
2. Run the tests:
   ```bash
   pytest
   ```
3. Do not run `cdk synth` / `cdk deploy` here: `lib/configuration.py:74` calls STS and the bootstrap scripts target real accounts.

## Cloud later
This repo only makes sense with an AWS account: follow the upstream Quickstart (`cdk bootstrap` through `lib/prerequisites/bootstrap_deployment_account.sh` and `bootstrap_target_account.sh`, set account IDs/regions in `lib/configuration.py`, then `ENV=Dev cdk deploy`). Expect pipeline, bucket and VPC-endpoint charges even before the ETL repo is added.

## Test plan
- `pytest` in the LXC passes `test/test_configuration.py`, `test_pipeline_stack.py`, `test_s3_bucket_zones_stack.py`, `test_vpc_stack.py`, `test_codecommit_stack.py`, `test_tagging.py` with `--durations=5 -vv` output (`pytest.ini`), no network needed.
- `python -c "import aws_cdk; print(aws_cdk.__version__)"` prints the installed CDK lib version.
- Nothing else is provable without an AWS account.

## Open questions
- Whether every test is fully mocked (the helper only implements `sts`; any test touching another client raises `RuntimeError`).
- George's goal for this mirror: reference architecture reading, or a real intention to build an on-prem data lake (which would start from the substitute table, not from this code).
