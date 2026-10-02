# AWS CloudFormation Connection Lookup Resource

> [!IMPORTANT]
> **This repository is archived and replaced by [jamesward/cfn-extras-resource](https://github.com/jamesward/cfn-extras-resource).**
> Its **Connection lookup** resource (`Handler: cfn_extras.connection_lookup.handler`) resolves the CodeConnections ARN behind a stack's CloudFormation Git sync (`Fn::GetAtt <Resource>.ConnectionArn`). It ships with the other resources as one
> Lambda artifact in a public, versioned S3 bucket, so there's nothing to build. See that repository's
> README for the CloudFormation snippet and the per-release `S3ObjectVersion`.
>
> The original code and documentation below are kept for reference.

This project provides a custom CloudFormation resource that resolves the ARN of
the AWS CodeConnections (GitHub App) connection that a stack uses for
**CloudFormation Git sync**.

## Why

CloudFormation Git sync deploys a stack directly from a Git repository via a
CodeConnections connection — there is no CodePipeline to read the connection ARN
from. A stack that wants to *reuse* that same (already-authorized) connection —
for example to drive a CodeBuild deploy — has a chicken-and-egg problem:
CloudFormation has no native way to turn a connection name into its ARN, and
hardcoding the ARN is brittle.

This resource breaks that cycle. Given the deploying stack's name it
triangulates to the connection ARN:

```
stack name
  --get_sync_configuration(SyncType=CFN_STACK_SYNC)-->  RepositoryLinkId
  --get_repository_link----------------------------->   ConnectionArn
```

The lookup is deterministic and scoped to the deploying stack, so it does not
depend on connection names or on there being exactly one connection.

## IAM

The Lambda's role needs read access to the Git sync metadata:

- `codeconnections:GetSyncConfiguration`
- `codeconnections:GetRepositoryLink`

These are read-only and can be scoped to `*` (they operate on sync
configuration / repository link metadata rather than the connection resource).

## CloudFormation usage

```yaml
  ConnectionLookupFunction:
    Type: AWS::Lambda::Function
    Properties:
      Code:
        S3Bucket:
          Ref: ConnectionLookupBuildBucket
        S3Key: function.zip
      Handler: index.handler
      Role:
        Fn::GetAtt:
          - ConnectionLookupRole
          - Arn
      Runtime: python3.13
      Timeout: 30

  ConnectionLookup:
    Type: AWS::CloudFormation::CustomResource
    Properties:
      ServiceTimeout: 90
      ServiceToken:
        Fn::GetAtt:
          - ConnectionLookupFunction
          - Arn
      # Resolve the connection of the very stack being deployed.
      StackName:
        Ref: AWS::StackName

  # Then reference the resolved ARN anywhere:
  #   Fn::GetAtt: [ConnectionLookup, ConnectionArn]
```

### Properties

- `StackName` (required) — the resource whose Git sync connection to resolve;
  normally `Ref AWS::StackName`.
- `SyncType` (optional) — defaults to `CFN_STACK_SYNC`.

### Return values (`Fn::GetAtt`)

- `ConnectionArn` — the ARN of the resolved CodeConnections connection.

## Development

```
nix-shell
pytest
```

The AWS calls sit behind the `ConnectionResolver` interface, so the resolution
logic is unit-tested with a fake and no live AWS access.
