---
title: DynamoDB Global Secondary Index Autoscaling with Cloudformation
author: Alexander Erben
date: 2018-02-08 18:18:03
tags: 
- aws
---

Recently, I wanted to bake my DynamoDB table setup into a CloudFormation template. The table contained a global secondary index and
is an autoscaling target for read and write capacity scaling. Registering auto scaling with a DynamoDB table in Cloudformation isn't hard at all.
Yet it isn't immediately obvious how to register the index as scalable target. The [documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-dynamodb-table.html) did not go into detail about index autoscaling, so how do we do it?

First, we will register the table itself as scalable target for write capacity and then attach a policy to it. This isn't strictly necessary
to enable scaling for the index, but why would you want to auto-scale the index and not the table itself?
The following CloudFormation snipped will do the trick:

```yaml
  Table:
    Type: 'AWS::DynamoDB::Table'
    Properties:
      TableName: !Ref TableName
      AttributeDefinitions:
      - AttributeName: 'id'
        AttributeType: 'S'
      - AttributeName: 'timestamp'
        AttributeType: 'N'
      - AttributeName: 'productgroup'
        AttributeType: 'S'
      KeySchema:
      - KeyType: 'HASH'
        AttributeName: 'id'
      - KeyType: 'RANGE'
        AttributeName: 'timestamp'
      GlobalSecondaryIndexes:
      - IndexName: !Sub '${TableName}-index'
        KeySchema:
        - KeyType: 'HASH'
          AttributeName: 'productgroup'
        Projection:
          ProjectionType: 'KEYS_ONLY'
        ProvisionedThroughput:
          ReadCapacityUnits: 2
          WriteCapacityUnits: 2
      ProvisionedThroughput:
        ReadCapacityUnits: 2
        WriteCapacityUnits: 2

  WriteScalableTarget:
    DependsOn: Table
    Type: "AWS::ApplicationAutoScaling::ScalableTarget"
    Properties:
      MaxCapacity: 10
      MinCapacity: 2
      ResourceId: !Sub 'table/${Table}'
      RoleARN: !GetAtt ScalingRole.Arn
      ScalableDimension: dynamodb:table:WriteCapacityUnits
      ServiceNamespace: dynamodb

  WriteScalingPolicy:
    DependsOn: Table
    Type: "AWS::ApplicationAutoScaling::ScalingPolicy"
    Properties:
      PolicyName: WriteScalingPolicy
      PolicyType: TargetTrackingScaling
      ScalingTargetId: !Ref WriteScalableTarget
      TargetTrackingScalingPolicyConfiguration:
        TargetValue: 50.0
        ScaleInCooldown: 70
        ScaleOutCooldown: 70
        PredefinedMetricSpecification:
          PredefinedMetricType: DynamoDBWriteCapacityUtilization
```

That will scale the write capacity between two to ten units targeting a value of 50% consumption. 
But what about the global secondary index? The given template snipped won't register it as scalable target.
Two questions arise: what is the scalable dimension of a global secondary index, and what is its resource identifier?
The answers are in the following snippet:

```yaml
  IndexWriteScalableTarget:
    DependsOn: Table
    Type: "AWS::ApplicationAutoScaling::ScalableTarget"
    Properties:
      MaxCapacity: 10
      MinCapacity: 2
      ResourceId: !Sub 'table/${Table}/index/${TableName}-index'
      RoleARN: !GetAtt ScalingRole.Arn
      ScalableDimension: dynamodb:index:WriteCapacityUnits
      ServiceNamespace: dynamodb

  IndexWriteScalingPolicy:
    DependsOn: Table
    Type: "AWS::ApplicationAutoScaling::ScalingPolicy"
    Properties:
      PolicyName: IndexWriteScalingPolicy
      PolicyType: TargetTrackingScaling
      ScalingTargetId: !Ref IndexWriteScalableTarget
      TargetTrackingScalingPolicyConfiguration:
        TargetValue: 50.0
        ScaleInCooldown: 70
        ScaleOutCooldown: 70
        PredefinedMetricSpecification:
          PredefinedMetricType: DynamoDBWriteCapacityUtilization
```

The resource identifier has the format `table/${Table}/index/MyIndex`, the scalable dimension is `dynamodb:index:WriteCapacityUnits`
for write capacity scaling and `dynamodb:index:ReadCapacityUnits` for read capacity scaling.
You can find the full template [here](https://github.com/aerben/aerben.github.io-samples/blob/master/cloudformation-samples/dynamodb-index-autoscaling.yaml), where the `ScalingRole` needed for the scaling to work is defined as well.

So that's it, thank you for reading!