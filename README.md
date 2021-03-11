# ec2lambda

aws configure 

aws iam create-role --role-name lambda-ex --assume-role-policy-document '{"Version": "2012-10-17","Statement": [{ "Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"}]}'
#keep the arn from the response (ex:  "Arn": "arn:aws:iam::445318494978:role/lambda-ex",


#create ec2 policy
echo '{  "Version": "2012-10-17",  "Statement": [    {      "Effect": "Allow",      "Action": [        "ec2:StartInstances",        "ec2:StopInstances"      ],      "Resource": "arn:aws:ec2:*:*:instance/*",      "Condition": {        "StringEquals": {          "ec2:ResourceTag/Owner": "${aws:username}"        }      }    },    {      "Effect": "Allow",      "Action": "ec2:DescribeInstances",      "Resource": "*"    }  ]}' > policy.json

aws iam create-policy --policy-name ec2_all --policy-document file://policy.json
#you need the policy arn


aws iam attach-role-policy --role-name lambda-ex --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam attach-role-policy --role-name lambda-ex --policy-arn arn:aws:iam::445318494978:policy/ec2_all


#deployment package with dependencies
pip3.8 install --target ./package godaddypy boto3

cd package
zip -r ../lambda_function_package.zip .

cd ..
zip -g lambda_function_package.zip lambda_function.py

#delete
aws lambda delete-function --function-name start_ec2
#create
aws lambda create-function --function-name start_ec2 --zip-file fileb://lambda_function_package.zip --handler lambda_function.lambda_handler --runtime python3.8 --role arn:aws:iam::445318494978:role/lambda-ex 

#update
zip -g lambda_function_package.zip lambda_function.py; aws lambda update-function-code --function-name start_ec2 --zip-file fileb://lambda_function_package.zip 


#invoke
aws lambda invoke --cli-binary-format raw-in-base64-out --function-name start_ec2 --payload '{"instanceType": "t3.small"}' output.txt  --log-type Tail --query 'LogResult' --output text |  base64 -d
