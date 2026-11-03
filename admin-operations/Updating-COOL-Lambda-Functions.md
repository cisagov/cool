# Updating COOL Lambda Functions

**Purpose:** A step-by-step guide for deploying a new release of a COOL
Lambda function to every account that runs it.

Terraform creates each Lambda function below from a deployment package in the
COOL instance's Lambda deployment bucket, in the Terraform account, and refers
to the package only by its bucket and S3 key. A new release is uploaded under
the same key, so Terraform does not see the change. This runbook updates each
function's code directly. Later Terraform applies leave the new code in place,
and any function created afterwards uses the latest package in the bucket.

| Lambda function | Source repository | Package (S3 key) | Deployed in |
|---|---|---|---|
| `disable-inactive-iam-users` | [`cisagov/disable-inactive-iam-users-lambda`](https://github.com/cisagov/disable-inactive-iam-users-lambda) | `disable_inactive_iam_users.zip` | Every account bootstrapped by `cisagov/cool-accounts`, including each `env<X>` account, plus any User Services or CyHy accounts |
| `ec2-reaper` | [`cisagov/ec2-reaper-lambda`](https://github.com/cisagov/ec2-reaper-lambda) | `ec2_reaper.zip` | Images |

**NOTE:** Deploy a release to a non-production COOL instance and verify it
there before deploying it to Production.

## A. Prerequisites

1. Docker and its Compose plugin are installed.
1. The Lambda's source repository is cloned into `~/code/cisagov`, and the
   release you are deploying (e.g. `v1.1.0`) has been published.
1. Your AWS credentials file for the COOL instance has the
   `cool-terraform-provisionaccount` profile, plus a
   `cool-<account>-provisionaccount` profile for every account whose function
   you will update (e.g. `cool-images-provisionaccount`,
   `cool-env0-provisionaccount`). If you are missing any of them, request
   them from the DevSecOps team.
1. You know the instance's Lambda deployment bucket name, referred to below as
   `<lambda_bucket>`. It is the `lambda_bucket_name` value in the instance's
   `cool-accounts` Terraform variables files.

## B. Building the deployment package

1. Check out the release:

    ```console
    cd ~/code/cisagov/<lambda_repository>
    git fetch --tags
    git checkout <release_tag>
    ```

1. Build the package, named for its S3 key in the table above (e.g.
   `ec2_reaper.zip`). The `--build` flag is required: the Lambda's code is
   copied into the Docker image when the image is built, and without the flag
   an image from an earlier build is reused, along with its old code:

    ```console
    BUILD_FILE_NAME=<package> docker compose up --build build_deployment_package
    ```

1. Record the package's SHA-256 checksum. Lambda reports the same value as the
   function's `CodeSha256` once the function has been updated:

    ```console
    openssl dgst -sha256 -binary <package> | openssl base64
    ```

## C. Uploading the package

1. Point the AWS CLI at the credentials file for the COOL instance you are
   updating (see [How to read these runbooks](README.md#how-to-read-these-runbooks)),
   and set the region:

    ```console
    export AWS_DEFAULT_REGION=us-east-1
    ```

1. Upload the package, replacing the previous one:

    ```console
    aws s3 cp <package> s3://<lambda_bucket>/ --profile cool-terraform-provisionaccount
    ```

## D. Updating the functions

1. Update each account's function from the uploaded package. Each command
   prints the function's new `CodeSha256`, which should match the checksum
   from section B.
    - `ec2-reaper`:

        ```console
        aws lambda update-function-code --function-name ec2-reaper \
          --s3-bucket <lambda_bucket> --s3-key ec2_reaper.zip \
          --profile cool-images-provisionaccount --query CodeSha256 --output text
        ```

    - `disable-inactive-iam-users`: replace `<last_env_number>` with the
      number of the instance's highest-numbered environment account (e.g.
      `11`), so that `seq` lists `env0` through `env11`. If any environment
      numbers in that range don't exist, list the environment accounts
      explicitly instead (e.g. `env0 env2 env3`). For any User Services or
      CyHy accounts, run the same `aws lambda update-function-code` command
      with those accounts' profiles.

        ```console
        for p in audit dns $(seq -f 'env%g' 0 <last_env_number>) images logarchive master sharedservices terraform users; do
          printf '%-18s ' "$p"
          aws lambda update-function-code --function-name disable-inactive-iam-users \
            --s3-bucket <lambda_bucket> --s3-key disable_inactive_iam_users.zip \
            --profile "cool-${p}-provisionaccount" --query CodeSha256 --output text
        done
        ```

1. Confirm that each update finished. Each command prints the update's
   status, followed by the function's `CodeSha256`. The status is
   `InProgress` for a few seconds after an update and should then be
   `Successful`.
    - `ec2-reaper`:

        ```console
        aws lambda get-function --function-name ec2-reaper \
          --profile cool-images-provisionaccount \
          --query 'Configuration.[LastUpdateStatus,CodeSha256]' --output text
        ```

    - `disable-inactive-iam-users`: use the same `<last_env_number>` as in
      the previous step, and check any User Services or CyHy accounts the
      same way.

        ```console
        for p in audit dns $(seq -f 'env%g' 0 <last_env_number>) images logarchive master sharedservices terraform users; do
          printf '%-18s ' "$p"
          aws lambda get-function --function-name disable-inactive-iam-users \
            --profile "cool-${p}-provisionaccount" \
            --query 'Configuration.[LastUpdateStatus,CodeSha256]' --output text
        done
        ```

    If a status is `Failed`, that function keeps running its previous code.
    Check its `Configuration.LastUpdateStatusReason` for the cause.
1. Remove the package and return the source repository to its default branch:

    ```console
    rm <package>
    git checkout develop
    ```

## E. Rolling back

To roll back, repeat sections B through D with the previous release's tag.
The Lambda deployment bucket keeps earlier versions of each package, but the
`ProvisionAccount` roles can read only the current one, so a function can't be
pointed at an earlier version directly.
