# POC_SNS_SQS
## AWS SNS + SQS + DOCKER
<br/>

> LET'S SEND A SIMPLE 'HELLO WORLD' MESSAGE TO AN SNS TOPIC AND CONSUME A SUBSCRIBED SQS QUEUE IN THIS TOPIC

> RUN: ```sudo apt update```

> INSTALL AWS-CLI ```sudo apt install -y awscli```

> CHECK THE VERSION
~~~bash
> aws --version
aws-cli/1.18.69 Python/3.8.10 Linux/5.15.79.1-microsoft-standard-WSL2 botocore/1.16.19
~~~

> RUN: ```aws configure```
~~~bash
# set Access Key, Secret Access Key, Default region and Default output format
> aws configure
AWS Access Key ID [None]: 123456
AWS Secret Access Key [None]: qwerty
Default region name [None]: us-east-1
Default output format [None]: json
~~~

> CHECK THE GENERATED CREDENTIALS
~~~bash
cat ~/.aws/credentials
# OUTPUT
[default]
aws_access_key_id = 123456
aws_secret_access_key = qwerty
~~~

> RUN: ```docker-compose -f docker-compose-lite.yml up```

> CHECK IF EVERYTHING IS UP
~~~bash
> curl -v http://localhost:4566
# OUTPUT
*   Trying 127.0.0.1:4566...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 4566 (#0)
> GET / HTTP/1.1
> Host: localhost:4566
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200
~~~

> CREATE A TEST PROFILE
~~~bash
> aws configure set aws_access_key_id "dummy" --profile test-profile
> aws configure set aws_secret_access_key "dummy" --profile test-profile
> aws configure set region "us-east-1" --profile test-profile
> aws configure set output "json" --profile test-profile
~~~

> CREATE THE SNS TOPIC
~~~bash
> aws --endpoint-url=http://localhost:4566 sns create-topic --name product-creation-events --region us-east-1 --profile test-profile --output json | cat
# OUTPUT
{
    "TopicArn": "arn:aws:sns:us-east-1:000000000000:product-creation-events"
}
~~~

> CREATE THE SQS QUEUE
~~~bash
> aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name product-created-queue --profile test-profile --region us-east-1 --output json | cat
# OUTPUT
{
    "QueueUrl": "http://localhost:4566/000000000000/product-created-queue"
}
~~~

> GET MORE DETAILS ABOUT THE SQS QUEUE, WE'RE GONNA NEED THE 'QueueArn' SOON
~~~bash
> aws --endpoint-url=http://localhost:4566 sqs get-queue-attributes --queue-url https://localhost:4566/000000000000/product-created-queue --attribute-names All
# OUTPUT
{
    "Attributes": {
        "ApproximateNumberOfMessages": "0",
        "ApproximateNumberOfMessagesNotVisible": "0",
        "ApproximateNumberOfMessagesDelayed": "0",
        "CreatedTimestamp": "1673314625",
        "DelaySeconds": "0",
        "LastModifiedTimestamp": "1673314625",
        "MaximumMessageSize": "262144",
        "MessageRetentionPeriod": "345600",
        "QueueArn": "arn:aws:sqs:us-east-1:000000000000:product-created-queue",
        "ReceiveMessageWaitTimeSeconds": "0",
        "VisibilityTimeout": "30",
        "SqsManagedSseEnabled": "false"
    }
}
~~~

> SUBSCRIBE TO THE SNS TOPIC
~~~bash
> aws --endpoint-url=http://localhost:4566 sns subscribe --region us-east-1 --topic-arn arn:aws:sns:us-east-1:000000000000:product-creation-events --profile test-profile --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:000000000000:product-created-queue --output json | cat
# OUTPUT
{
    "SubscriptionArn": "arn:aws:sns:us-east-1:000000000000:product-creation-events:ece65573-6bcb-4a9f-9c47-78117d201121"
}
~~~

> PUBLISHING TO THE SNS TOPIC
~~~bash
> aws sns publish --endpoint-url=http://localhost:4566 --topic-arn arn:aws:sns:us-east-1:000000000000:product-creation-events --message "Hello World" --profile test-profile --region us-east-1 --output json | cat
# OUTPUT
{
    "MessageId": "2d3c3e2e-2e8d-49ed-ac04-5cea02789dd7"
}
~~~

> CONSUMING FROM SQS QUEUE
~~~bash
> aws --endpoint-url=http://localhost:4566 sqs receive-message --queue-url https://localhost:4566/000000000000/product-created-queue --profile test-profile --region us-east-1 --output json | cat
# OUTPUT
{
    "Messages": [
        {
            "MessageId": "106c3e01-6f76-47db-a22f-c8dcc35c1f7e",
            "ReceiptHandle": "Yzg4OTI3YzMtMzA0Ny00MzI0LWI0OWEtZTQ4ODBkYjJiMDQzIGFybjphd3M6c3FzOnVzLWVhc3QtMTowMDAwMDAwMDAwMDA6cHJvZHVjdC1jcmVhdGVkLXF1ZXVlIDEwNmMzZTAxLTZmNzYtNDdkYi1hMjJmLWM4ZGNjMzVjMWY3ZSAxNjczMzE4MjkzLjQ2MTUxNzY=",
            "MD5OfBody": "5654e4c0e972b416bb150aaeefb7682e",
            "Body": "{\"Type\": \"Notification\", \"MessageId\": \"2d3c3e2e-2e8d-49ed-ac04-5cea02789dd7\", \"TopicArn\": \"arn:aws:sns:us-east-1:000000000000:product-creation-events\", \"Message\": \"Hello World\", \"Timestamp\": \"2023-01-10T02:37:55.076Z\", \"SignatureVersion\": \"1\", \"Signature\": \"EXAMPLEpH+..\", \"SigningCertURL\": \"https://sns.us-east-1.amazonaws.com/SimpleNotificationService-0000000000000000000000.pem\", \"UnsubscribeURL\": \"http://localhost:4566/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:us-east-1:000000000000:product-creation-events:ece65573-6bcb-4a9f-9c47-78117d201121\"}"
        }
    ]
}
~~~