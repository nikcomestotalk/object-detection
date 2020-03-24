# Category Predictor


This is a Category Predictor. It uses CNN algorithm to create a model and tell the correct category of given product image. This repository contains a [Jupiter Notebook](./ImageClassifier.ipynb) used to gather the data, train a model, and evaluate it. This notebook also contains a web app which can be deployed to predict category of image.

- [Report](./report.pdf)
- [Jupiter Notebook](./ImageClassifier.ipynb)

# Web App Setup Instructions

To run this Web App you must first create and deploy a model. There are a few requirements before you get started.

## Pre-requisites
 1. Python3.6
 2. SageMaker instance type ml.t2.2xlarge
 3. Disk space > 100GB
 4. jupyter notebook --NotebookApp.iopub_data_rate_limit=2000000.0
 5. Model training instance ml.p3.8xlarge [ Got many segmentation fault because of choosing smaller instance and your training takes too much time on this amount of dataset]
 6. im2rec.py to distribute data in test and train and creating RecordIO file for both test and train.

## SageMaker

You must also have an [Amazon Web Services](https://aws.amazon.com/) (AWS) account. Create an account and navigate to Amazon SageMaker from your AWS Console. Create a Notebook Instance. On the Create a Notebook Instance page. Finally, create the notebook instance. Create a role attached to SageMaker should have the RW access over S3 bucket.
### Create the Model

Once the Notebook instance is setup, open the `ImageClassifier` notebook. 

After that, you can run all the cells in the Notebook. After about 2 hour, you should have a model generated and evaluated.

### Setup Lambda

Now that you have a model trained, you will need to create a Lambda function to send data to the SageMaker endpoint and return the result. Copy and paste the code below into a Lambda function on AWS. You will need to enter your **Predictor Endpoint Name** in the code provided.

```
const AWS = require('aws-sdk');
var s3 = new AWS.S3();
exports.handler = (event, context, callback) => {
    let data = new Buffer.from(event.body.replace(/^data:image\/\w+;base64,/, ""),'base64');
    const type = event.body.split(';')[0].split('/')[1];
    var sagemakerruntime = new AWS.SageMakerRuntime();
    var invokingparams = 
    {
    "Body": data,
    "EndpointName":"[SAGEMAKER_MODEL_PREDICTION_ENDPOINT_NAME",
    "ContentType":"application/x-image"
    };
sagemakerruntime.invokeEndpoint(invokingparams, function (err, data) {
  if (err) console.log(err, err.stack); // an error occurred
  else     {
      
      console.log("done here");  
      var responseData = JSON.parse(Buffer.from(data.Body).toString('utf8'));
      var maxIndex = 0;
      var maxValue = responseData[0];
      var predictData = {}
      predictData[0] = responseData[0];
      for(var i=1;i<5;i++) {
          if(maxValue < responseData[i]) {
              maxIndex = i;
              maxValue = responseData[i];
             
          }
           predictData[i] = responseData[i];
      }
      //['books','car', 'furniture', 'mobile', 'motorcycle']
      var prediction = {
          0 : "books",
          1 : "car",
          2 : "furniture",
          3 : "mobile",
          4 : "motorcycle"
      }
      predictData['verdict'] = prediction[maxIndex];
      
      let response = {
        "statusCode": 200,
        "headers": {
            "my_header": "my_value"
        },
        "body": JSON.stringify(predictData),
        "isBase64Encoded": false
    };
    console.log(response)
    callback(null, response);
      }// successful response
});
};
```

### API Gateway

Finally, we can set up API Gateway to trigger the Lambda function we created and get Image category Predictions. To do this, create a new POST method and make sure Lambda Function is selected. Then, enter the name of your Lambda Function into the text box and click save. Finally, click the Actions dropdown to Deploy API.

You will need the **Invoke URL** to deploy your Web App and make sure you give access to API Gateway to call Lambda function.

## Web App

Now that you have an API available, you can start using it in a web app. I have made a very simple HTML and JavaScript file to interact with the API.
