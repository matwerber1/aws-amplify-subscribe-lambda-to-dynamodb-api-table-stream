# Purpose

AWS Amplify allows you to create Lambda functions, as well as AWS AppSync GraphQL APIs backed by DynamoDB (DDB) tables. 

When creating an AWS AppSync API backed by DynamoDB, Amplify will configure the table with an event stream. 

However, there's no "out of the box" Amplify CLI command to configure one of your Amplify functions to receive and process records from your API table's stream.

This project shows examples of how to customize your project to have a Lambda function subscribe to a stream, based on the guidance from this Git issue:

https://github.com/aws-amplify/amplify-cli/issues/987

## Prerequisites

1. Existing Amplify CLI project
2. Existing Amplify GraphQL API (or add a new one with `amplify add api`)
3. Existing Amplify Lambda function to act as your stream processor (or add a new one with `amplify add function`)

## Subscribing your function to your table's stream

1. Within your amplify project directory, open the CloudFormation (CF) template at `amplify/backend/function/FUNCTION_NAME/FUNCTION_NAME-cloudformation-template.json`:

  1. In the `Parameters:` section of the CF template, verify that you see a parameter similar to the one below: 
  
    ```
    "Parameters": {
      "apiXXXXXXGraphQLAPIIdOutput": {
        "Type": "String",
        "Default": "apiXXXXXXGraphQLAPIIdOutput"
     },
    ```
    
    Note: the `XXXXXX` above should have the name of the API you used during `amplify add api`.
  
  2. In the `Resources:` section, add the following: 
  
  ```
  "EventSourceMapping": {
			"Type": "AWS::Lambda::EventSourceMapping",
			"DependsOn": [
				"LambdaExecutionRole",
				"CustomPolicyForTableStreamSubscription"
			],
			"Properties": {
				"EventSourceArn": {
					"Fn::ImportValue": {
						"Fn::Sub": "${apiXXXXXXGraphQLAPIIdOutput}:GetAtt:YYYYYYTable:StreamArn"
					}
				},
				"FunctionName": {
					"Fn::GetAtt": [
						"LambdaFunction",
						"Arn"
					]
				},
				"StartingPosition": "TRIM_HORIZON"
			}
		},
  ```
  
  Note: again, `XXXXXX` above should be replaced with your API name. In addition, `YYYYYY` should be replaced with the friendly Amplify name (not the actual name in DynamoDB, which will contain additional random characters) of the table you want to subscribe to. This will likely match the resource name in your Amplify GraphQL schema specification (`amplify/backend/api/YOUR_API_NAME/schema.graphql`), but you can also double-check by looking in the [Exports section of the CloudFormation console](https://console.aws.amazon.com/cloudformation/home?#/exports)
