![cloudcover-part-of-sttc-color-dark-logo.png](.img/cloudcover-part-of-sttc-color-dark-logo.png) & ![cloudcustodian-logo.png](.img/cloudcustodian-logo.png)

---

# Cloud Custodian Policies for AWS
> powered by GitHub Actions

# Prerequisites
* Quotas limit increased to `200` (minimum) for CloudWatch Event Rules, in each region:
  ```shell
  aws service-quotas request-service-quota-increase \
    --service-code events \
    --quota-code L-244521F2 \
    --desired-value 200
  ```
* Alerts policies filenames should start with `a-`
* Remediation policies filenames should start with `r-`

---

# Custodian Policy Structure
* Ensure `Account {account_id} - ` is present in the `description` to identify multiple accounts independently.
* Ensure `name` is short and sweet. If the `name` is too long, validation check will fail.
* Ensure role name is not changed as it deployed via terraform in prerequisites.

---

# Using with GitHub Actions
> Ensure [validation.yml](.github/workflows/validation.yml) is being used for sanity

To use the policies with an AWS account, we have to refer the [sample workflow](.github/sample-workflow.yml) file.
The below sample workflow is for `cloudtrail` based policies. Similarly, the workflows can be created for hourly/daily frequencies.

```yaml
name: sample-workflow
on:
  push:
    branches:
      - main
    paths:
      - policies/cloudtrail/**

defaults:
  run:
    shell: bash
    working-directory: policies/cloudtrail/

env:
  AWS_ACCOUNT_ID: "123456789012"
  REGION_LIST: |
    (
    "us-east-1"
    "ap-southeast-1"
    )
  ROLE_NAME: custodian-sample-role

jobs:
  CustodianDeployer:
    name: Deploy Lambda for CloudTrail Events
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Install python3.8
        uses: actions/setup-python@v2
        with:
          python-version: '3.8'

      - name: Install Custodian
        run: |
          pip install c7n

      - name: Configure AWS credentials from ${{env.AWS_ACCOUNT_ID}} account
        run: |
          CREDS=( $(aws sts assume-role --role-arn "arn:aws:iam::${{env.AWS_ACCOUNT_ID}}:role/${{env.ROLE_NAME}}" --role-session-name "${{env.ROLE_NAME}}" --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' --duration-seconds 5400 --output text) )
          unset AWS_ACCESS_KEY_ID
          unset AWS_SECRET_ACCESS_KEY
          unset AWS_SESSION_TOKEN
          AWS_ACCESS_KEY_ID=${CREDS[0]}
          echo "::add-mask::$AWS_ACCESS_KEY_ID"
          echo AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID >> $GITHUB_ENV    
          AWS_SECRET_ACCESS_KEY=${CREDS[1]}
          echo "::add-mask::$AWS_SECRET_ACCESS_KEY"
          echo AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY >> $GITHUB_ENV
          AWS_SESSION_TOKEN=${CREDS[2]}
          echo "::add-mask::$AWS_SESSION_TOKEN"
          echo AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN >> $GITHUB_ENV

      - name: Check Access
        run: |
          aws sts get-caller-identity

      - name: Deploy regional policies
        run: |
            find . \( -iname "*.yml" -o -iname "*.yaml" \) | { grep -v "route53\|cloudfront\|iam\|s3" || true; } | while read POLICY; do
              sed -i 's,REPLACE_WEBHOOK_HERE,${{ secrets.CUSTODIAN_SLACK_WEBHOOK }},g' "$POLICY"
              array=${{ env.REGION_LIST }}
              for REGION in ${array[*]}; do
                custodian run -s /tmp/ -v --cache-period 0 -c "$POLICY" --region "$REGION"
              done
            done

      - name: Deploy global policies
        run: |
            find . \( -iname "*.yml" -o -iname "*.yaml" \) | { grep "route53\|cloudfront\|iam\|s3" || true; } | while read POLICY; do
              sed -i 's,REPLACE_WEBHOOK_HERE,${{ secrets.CUSTODIAN_SLACK_WEBHOOK }},g' "$POLICY"
              custodian run -s /tmp/ -v --cache-period 0 -c "$POLICY" --region us-east-1
            done
```

### Notes
* Ensure to change `REGION_LIST` inside the workflow file.
* Individual workflow files for each frequency.
* Slack Webhook. This repo will use `CUSTODIAN_SLACK_WEBHOOK` GitHub Secret to replace webhook while deployment.

---

