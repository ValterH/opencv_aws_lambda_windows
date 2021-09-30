# AWS Lambda function for OpenCV 

This project illustrates how to create an AWS Lambda function using Python 3.7 and OpenCV (latest) to performe Canny Edge detection on an image in S3 and save it back to S3. The Python OpenCV library is published as a Lambda layer which reduces the size of the Lambda function and enables the function code to be rendered in the Lambda code viewer in the AWS console.

<b>BASED ON: [iandow](https://github.com/iandow)'s [opencv_aws_lambda](https://github.com/iandow/opencv_aws_lambda) </b>

<b> THIS TUTORIAL IS FOR WINDOWS USING POWERSHELL, LINUX USERS SHOULD REFER TO [ORIGINAL](https://github.com/iandow/opencv_aws_lambda).</b>

## USAGE:

### Preliminary AWS CLI Setup: 
1. Install [Docker](https://docs.docker.com/), the [AWS CLI](https://aws.amazon.com/cli/) on your workstation.
2. Setup credentials for AWS CLI (see http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html).
3. Create IAM Role with Lambda and S3 access:
Create role in [AWS Console](https://console.aws.amazon.com/iamv2/home#/roles) with the name *lambda-opencv_study*
```
# Create a role with S3 and Lambda exec access
$ROLE_NAME="lambda-opencv_study"

aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --role-name $ROLE_NAME
aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole --role-name $ROLE_NAME
```

### Build OpenCV library using Docker

AWS Lambda functions run in an [Amazon Linux environment](https://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html), so libraries should be built for Amazon Linux. You can build Python-OpenCV libraries for Amazon Linux using the provided Dockerfile, like this:

```
git clone https://github.com/ValterH/opencv_aws_lambda_windows
cd opencv_aws_lambda_windows
docker build --tag=lambda-layer-factory:latest .
docker create -ti --name temp lambda-layer-factory bash
docker cp temp:/packages/cv2-python37.zip .
```

### DEPLOY

1. Edit the Lambda function code to do whatever you want it to do.

2. Publish the OpenCV Python library as a Lambda layer.
```
$ACCOUNT_ID=YOUR-AWS-ACCOUNTID
$LAMBDA_LAYERS_BUCKET="lambda-layers-$ACCOUNT_ID"
$LAYER_NAME="cv2"
aws s3 mb s3://$LAMBDA_LAYERS_BUCKET
aws s3 cp cv2-python37.zip s3://$LAMBDA_LAYERS_BUCKET
aws lambda publish-layer-version --layer-name $LAYER_NAME --description "Open CV" --content S3Bucket=$LAMBDA_LAYERS_BUCKET,S3Key=cv2-python37.zip --compatible-runtimes python3.7
```

3. Create the Lambda function:
```
zip app.zip app.py
```

4. Deploy the Lambda function:
```
# Create the Lambda function:
$FUNCTION_NAME="opencv_layered"
$BUCKET_NAME="opencv-test-$ACCOUNT_ID"
$S3Key="images/my_image.jpg"
aws s3 mb s3://$BUCKET_NAME
aws s3 cp app.zip s3://$BUCKET_NAME
aws lambda create-function --function-name $FUNCTION_NAME --timeout 20 --role arn:aws:iam::${ACCOUNT_ID}:role/$ROLE_NAME --handler app.lambda_handler --region eu-central-1 --runtime python3.7 --environment "Variables={BUCKET_NAME=$BUCKET_NAME,S3_KEY=$S3_KEY}" --code S3Bucket="$BUCKET_NAME",S3Key="app.zip"
```

5. Attach the cv2 Lambda layer to our Lambda function:
```
aws lambda list-layer-versions --layer-name $LAYER_NAME
$LAYER=LAYERVERSIONARN-FROM-ABOVE-RESPONSE
aws lambda update-function-configuration --function-name $FUNCTION_NAME --layers $LAYER
```

### Test the Lambda function:
Our Lambda function requires an image as input. Copy an image to S3, like this:
```
aws s3 cp ./images/my_image.jpg s3://$BUCKET_NAME/images/my_image.jpg
```
Then invoke the Lambda function:
```
aws lambda invoke --function-name $FUNCTION_NAME --log-type Tail outputfile.txt
cat outputfile.txt
```

You should see output like this:
```
{"statusCode": 200, "body": "{\"message\": \"image saved to s3://$BUCKET_NAME/my_image-edges.jpg\"}"}
```

```
aws s3 cp s3://$BUCKET_NAME/my_image-edges.jpg .
```

<img src=images/my_image.jpg width="200"> <img src=images/my_image-edges.jpg width="200">

### Clean up resources
```
aws s3 rm s3://$BUCKET_NAME/my_image-edges.jpg
aws s3 rb s3://$BUCKET_NAME/
aws s3 rm s3://$LAMBDA_LAYERS_BUCKET/cv2-python37.zip
aws s3 rb s3://$LAMBDA_LAYERS_BUCKET
rm my_image-edges.jpg
rm app.zip
rm cv2-python37.zip

aws lambda delete-function --function-name $FUNCTION_NAME
aws lambda delete-layer-version --layer-name cv2 --version-number $LAYER_VERSION
aws iam detach-role-policy --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole --role-name $ROLE_NAME
aws iam detach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --role-name $ROLE_NAME
aws iam delete-role --role-name $ROLE_NAME