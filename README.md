# Stack is stuck in XX_IN_PROGRESS
## Troubleshooting steps: 
1. Identify the stuck resources: 

- Open the CloudFormation console.
- In the navigation pane, choose Stacks, and then select the stack that's in a stuck state.
- Choose the Resources tab.
- In the Resources section, refer to the Status column. Find any resources that are stuck in state CREATE_IN_PROGRESS, UPDATE_IN_PROGRESS, or DELETE_IN_PROGRESS.

2. Check errors in CloudTrail. 
3. Resource failed to stabilize.

    If there is no error in CloudTrail, it’s usually due to the resource failed to stabilize. CloudFormation performs additional verification in order to ensure the provisioned resources are started correctly. For example, although you observe the ECS service is created successfully in ECS console, the number of running tasks does not reach the desired number of tasks. That means the task running failed. The creation of ECS service would be stuck in CFN.

--------

## Sample - ECS service stuck in 3 hours
- Check `Resources` tab: Identify the stuck resource ==> AWS::ECS::Service
- Check events in [Ctrail](https://command-center.support.aws.a2z.com/troubleshooting?CSGroupId=d1123308-10b6-4bda-9e2a-aa4104127a08#/script-runner/K2-cloudtrail_contributionmodelasset_Dante-curated-cloudtrail/?_inputParameters=%7B%22region%22%3A%22ap-southeast-2%22%2C%22customerEntity%22%3A%22502261777658%22%2C%22displayName%22%3A%22CTrail%22%7D&region=ap-southeast-2).
- Check `Events` tab: 
  Eventual consistency check initiated ===> 
  > What does it mean? During this phase, the service performs internal consistency checks to ensure the resource is fully operational and meets service stabilization criteria defined by each AWS service. [CFN playbook](https://w.amazon.com/bin/view/AmazonWebServices/SalesSupport/DeveloperSupport/Internal/Deployment_Management/Docs/CloudFormation/CloudFormationPlaybook/Troubleshooting_Scenarios/#HResourcefailedtostabilize)
- Go to the service console to manually modify the service to meet the stabilization criteria ==> `ECS console` ==> Set Desired Task Count as 0. 

### TSC links: 

  [stack: ECS stuck in 3 hours: ECSStabilization-Broken](https://command-center.support.aws.a2z.com/troubleshooting?CSGroupId=80ed20f3-eade-4555-859a-a3a0a49e18ab#/dashboardv2/?accountId=502261777658&toolName=ChronosCFNv2-TSC&chronosRoutePath=stackInfo&chronosRouteParam=arn%253Aaws%253Acloudformation%253Aap-southeast-2%253A502261777658%253Astack%252FECSStabilization-Broken%252Ffa091f90-cb0d-11ef-90a1-0608c69f9535%2CstackEvents&displayName=CFN)

  [Stack: How to bypass the 3 hours: ECSbypass-timeout]()

  [ECS service]()

--------
## Sample - CFN did not receive a response from your custom resource - stuck in 1 hour
### What is Custom Resources?
Custom resources provide a way for you to write custom provisioning logic into your CloudFormation templates. This can be useful when your provisioning requirements involve complex logic or workflows that can't be expressed with CloudFormation's built-in resource types.

### Sample for Custom Resource: Delete non-empty S3 bucket
1. Create a CFN stack with S3 bucket. `cfnTrainingBucketFailed`
2. Manually upload a file into the S3 bucket. 
3. Delete the CFN stack. 

==> DELETE_FAILED with error ==> Block cx's deployment process.
```bash
Resource handler returned message: "The bucket you tried to delete is not empty"
```
[How do I delete an AWS CloudFormation stack that's stuck in DELETE_FAILED status?](https://repost.aws/knowledge-center/cloudformation-stack-delete-failed)


4. Create a lambda-backed custom resource, define empty bucket command in lambda function, execute the lambda function before deleting the bucket. `cfnTrainingBucketCR`
```yaml
  S3CustomResource:
    Type: Custom::S3CustomResource
    Properties:
      ServiceToken: !GetAtt AWSLambdaFunction.Arn
      bucket_name: !Ref SampleS3Bucket

  AWSLambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      Description: "Empty an S3 bucket!"
      FunctionName: !Sub "${AWS::StackName}-${AWS::Region}-lambda"
      Handler: index.handler
      Role: !GetAtt AWSLambdaExecutionRole.Arn
      Timeout: 360
      Runtime: python3.8
      Code:
        ZipFile: |
          import boto3
          import cfnresponse
          ### cfnresponse module help in sending responses to CloudFormation
          ### instead of writing your own code

          def handler(event, context):
              # Get request type
              the_event = event['RequestType']
              print("The event is: ", str(the_event))
              print ('REQUEST BODY:\n' + str(event))

              response_data = {}
              s3 = boto3.client('s3')
              physical_id = 'TheOnlyCustomResourceSomethingElse'

              # Retrieve parameters (bucket name)
              bucket_name = event['ResourceProperties']['bucket_name']

              try:
                  if the_event == 'Delete':
                      print("Deleting S3 content...")
                      b_operator = boto3.resource('s3')
                      b_operator.Bucket(str(bucket_name)).objects.all().delete()

                  # Everything OK... send the signal back
                  print("Execution succesfull!")
                  cfnresponse.send(event, context, cfnresponse.SUCCESS, attributes, physical_id, True)
              except Exception as e:
                  print("Execution failed...")
                  print(str(e))
                  response_data['Data'] = str(e)
                  cfnresponse.send(event, context, cfnresponse.FAILED, response_data)
```

### Sample: Stuck in 1 hour & how to bypass the timeout

[CRStuck](https://command-center.support.aws.a2z.com/troubleshooting?CSGroupId=d1123308-10b6-4bda-9e2a-aa4104127a08#/dashboardv2/?toolName=ChronosCFNv2-TSC&accountId=502261777658&chronosRoutePath=stackInfo&chronosRouteParam=arn%253Aaws%253Acloudformation%253Aap-southeast-2%253A502261777658%253Astack%252FCRStuck%252F53453ae0-cb21-11ef-8aa4-0673592cad2f%2CstackEvents&displayName=CFN)

- Error: 
```
CloudFormation did not receive a response from your Custom Resource. Please check your logs for requestId [25d1c921-2cec-4f67-9a8d-7f0307342ed0]. If you are using the Python cfn-response module, you may need to update your Lambda function code so that CloudFormation can attach the updated version.
```

- Find the related info in CloudWatch log group and send below command manually:
``` bash
curl -H 'Content-Type: ''' -X PUT -d '{
    "Status": "SUCCESS",
    "StackId": "arn:aws:cloudformation:ap-southeast-2:502261777658:stack/CRStuck-1/c9f70460-cbdb-11ef-ac24-06dd21145b8f",
    "RequestId": "c61d335b-cb7b-4f71-a4f8-deaefe2d7339",
    "PhysicalResourceId": "TheOnlyCustomResourceSomethingElse",
    "LogicalResourceId": "S3CustomResource",
    "Reason": "fei testing for failure sending reason."
}' 'https://cloudformation-custom-resource-response-apsoutheast2.s3-ap-southeast-2.amazonaws.com/arn%3Aaws%3Acloudformation%3Aap-southeast-2%3A502261777658%3Astack/CRStuck-1/c9f70460-cbdb-11ef-ac24-06dd21145b8f%7CS3CustomResource%7Cc61d335b-cb7b-4f71-a4f8-deaefe2d7339?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20250106T031646Z&X-Amz-SignedHeaders=host&X-Amz-Expires=7200&X-Amz-Credential=AKIA6MM33IIZU35UNEER%2F20250106%2Fap-southeast-2%2Fs3%2Faws4_request&X-Amz-Signature=e758f9a8c62e79e663af8306cf957f29d954195cfa1f3ece9dbaee796787be01'
```

## Reference
1. [Troubleshooting Scenarios](https://w.amazon.com/bin/view/AmazonWebServices/SalesSupport/DeveloperSupport/Internal/Deployment_Management/Docs/CloudFormation/CloudFormationPlaybook/Troubleshooting_Scenarios/#HStackStuckIn)
2. [Why is my CloudFormation stack stuck in an IN_PROGRESS state?](https://repost.aws/knowledge-center/cloudformation-stack-stuck-progress)
3. [How can I stop my Amazon ECS service from failing to stabilize in AWS CloudFormation?](https://repost.aws/knowledge-center/cloudformation-ecs-service-stabilize)
4. [How do I delete a Lambda-backed custom resource that's stuck in DELETE_FAILED status or DELETE_IN_PROGRESS status in CloudFormation?](https://repost.aws/knowledge-center/cloudformation-lambda-resource-delete)
5. [Create custom provisioning logic with custom resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)
6. [How do I delete an AWS CloudFormation stack that's stuck in DELETE_FAILED status?](https://repost.aws/knowledge-center/cloudformation-stack-delete-failed)
7. [Extend your template's capabilities with CloudFormation-supplied resource types](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-supplied-resource-types.html)
