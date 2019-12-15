# Purpose

AWS Amplify allows you to create Lambda functions, as well as AWS AppSync GraphQL APIs backed by DynamoDB (DDB) tables. 

When creating an AWS AppSync API backed by DynamoDB, Amplify will configure the table with an event stream. 

However, as of this writing (Dec 17, 2019), there's no "out of the box" Amplify CLI command to configure one of your Amplify functions to receive and process records from your API table's stream.

This project provides an example of how to manually customize your Amplify project's generated CloudFormation templates to subscribe a Lambda function to your table's stream. This guide was based on the guidance from https://github.com/aws-amplify/amplify-cli/issues/987

## Prerequisites

1. Existing Amplify CLI project
2. Existing Amplify GraphQL API (or add a new one with `amplify add api`)
3. Existing Amplify Lambda function to act as your stream processor (or add a new one with `amplify add function`)

## Subscribing your function to your table's stream

1. Within your amplify project directory, open the CloudFormation (CF) template at `amplify/backend/function/FUNCTION_NAME/FUNCTION_NAME-cloudformation-template.json`:

2. In the `Parameters:` section of the CF template, verify that you see a parameter similar to the one below: 
  
	```
	"Parameters": {
		"apiXXXXXXGraphQLAPIIdOutput": {
			"Type": "String",
			"Default": "apiXXXXXXGraphQLAPIIdOutput"
		},
	}
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

	Note: again, `XXXXXX` above should be replaced with your API name. In addition, `YYYYYY` should be replaced with the friendly Amplify name (not the actual name in DynamoDB, which will contain additional random characters) of the table you want to subscribe to. This will likely match the resource name in your Amplify GraphQL schema specification (`amplify/backend/api/YOUR_API_NAME/schema.graphql`), but you can also double-check by looking in the [Exports section of the CloudFormation console](https://console.aws.amazon.com/cloudformation/home?#/exports) for a value in the **Export Value** column with a name similar to `yyyyyTable`.

3. In the `Resources:` section of the CloudFormation template, add a new resource to grant your Lambda function permission to access the DynamoDB table's stream: 

	```
	"CustomPolicyForTableStreamSubscription": {
		"Type": "AWS::IAM::Policy",
		"DependsOn": [
			"LambdaExecutionRole"
		],
		"Properties": {
			"PolicyName": "dynamodb-stream-permission",
			"Roles": [
				{
					"Ref": "LambdaExecutionRole"
				}
			],
			"PolicyDocument": {
				"Version": "2012-10-17",
				"Statement": [
					{
						"Effect": "Allow",
						"Action": [
							"dynamodb:GetRecords",
							"dynamodb:GetShardIterator",
							"dynamodb:DescribeStream",
							"dynamodb:ListStreams"
						],
						"Resource": [
							{
								"Fn::ImportValue": {
									"Fn::Sub": "${apiXXXXXXGraphQLAPIIdOutput}:GetAtt:YYYYYYTable:StreamArn"
								}
							}
						]
					}
				]
			}
		}
	}
	```

	Note - again, replace `XXXXXX` and `YYYYYY` above with the appropriate API name and table name, respectively. 
	
4. Edit your Lambda function code (e.g. `amplify/backend/function/YOUR_FUNCTION/src/index.js` as needed. You could use this just as a starting point for testing: 

	```
	console.log('Loading function');

	exports.handler = async (event, context) => {
		//console.log('Received event:', JSON.stringify(event, null, 2));
		for (const record of event.Records) {
			console.log(record.eventID);
			console.log(record.eventName);
			console.log('DynamoDB Record: %j', record.dynamodb);
		}
		return `Successfully processed ${event.Records.length} records.`;
	};
	```

5. Run `amplify push` to deploy your changes to the cloud. 

6. Add, edit, or delete an item in your DynamoDB table, then check your Lambda function's CloudWatch logs to validate that it successfully received the streamed change event. If you're not seeing the CloudWatch log results after a minute or so, navigate to the [DynamoDB console](https://console.aws.amazon.com/dynamodb/home), click your table, then click the **Triggers** tab and confirm that your Lambda function is in-fact shown as a trigger, the status is `Enabled`, and the `Last Result` is `OK`. If not, troubleshooting may be needed.

7. If this document could be improved, please share feedback!
