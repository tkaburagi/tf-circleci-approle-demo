# tf-circleci-approle-demo

This is a demo for showing deployin AWS resources with Circle CI and Vault. Vault AWS Secret Engine dynamically generate AWS keys and allows you to prevent from exposing or writing diretly to configurations file. 

In the CircleCI pileline, Vault uses AppRole authentication(pull mode) to generate Vault token. This alos enable users to hide Vault Token in the pipeline configuration file. 

## Preparations

1. Create Circle CI account and connect GitHub repositories which is forked from here.
2. Sign up to terraform Cloud
3. Set the Environment Var in Circle CI.
	* TF_ORG_NAME
	* TF_TOKEN
	* TF_WORKSPACE_ID
	* TF_WORKSPACE_NAME
	* VAULT_ADDR
	* VAULT_INIT_TOKEN

## Set up Vault

Vault should be internet-facing(can be connected with Circle CI).

```shell
vault auth enable approle
vault secrets enable aws
```

```shell
vault write aws/config/root \
    access_key=******* \
    secret_key==*******+ \
    region=ap-northeast-1
```

```shell
vault write aws/roles/tf-demo-role \
    credential_type=iam_user \
    policy_document=-<<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "rds:*",
                "ec2:*",
                "elasticloadbalancing:*"
            ],
            "Resource": "*"
        }
    ]
}
EOF
```

```shell
cat << EOF > approle.hcl
path "auth/approle/*" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}
EOF
```

```shell
vault policy write approle -policy=approle.hcl
```

```shell
vault token create -policy=approle -use-limit=1
```

```shell
cat << EOF > aws.hcl
path "aws/*" {
  capabilities = [ "read" ]
}
EOF
```

```shell
vault policy write aws -policy=aws.hcl
```

```shell
vault write auth/approle/role/aws policies="aws" token_ttl=10m token_max_ttl=1h
```