---
title: "Aws Api Gateway Rekognition"
date: 2022-12-23T15:58:30-07:00
draft: true
---

Amazon Rekognition is a powerful service that allows you to perform image and video analysis. With AWS API Gateway, you can easily create and manage APIs that allow you to access Rekognition's capabilities from your own applications.

To use the AWS API Gateway with Rekognition, you will need to create an API Gateway API and configure it to use the x-amazon-apigateway-integration request parameter. This parameter allows you to specify the integration type and integration URI for your API.

Here's a step-by-step guide on how to set up an AWS API Gateway API for Rekognition using the x-amazon-apigateway-integration parameter:

Go to the AWS API Gateway console and create a new API.

In the API Gateway console, click on the "Resources" tab and create a new resource. This resource will represent the endpoint for your Rekognition API.

In the resource's "Methods" tab, create a new method and select "POST" as the HTTP method.

In the "Integration Request" section, select "AWS Service" as the integration type and enter the ARN of the Rekognition service in the "AWS Region" field.

In the "AWS Service" section, enter the name of the Rekognition operation you want to perform (e.g. "DetectLabels") in the "Action" field.

In the "HTTP Headers" section, add the x-amazon-apigateway-integration header with the following value: { "type": "aws", "uri": "arn:aws:apigateway:REGION:rekognition:action/OPERATION", "httpMethod": "POST" }, where REGION is the region where your Rekognition service is located and OPERATION is the name of the Rekognition operation you want to perform (e.g. "DetectLabels").

Save your changes and deploy your API.

Now you can use your API Gateway API to perform Rekognition operations by sending a POST request to the endpoint you created. The x-amazon-apigateway-integration header will ensure that the request is forwarded to the Rekognition service and the response will be returned to the client.

I hope this helps! Let me know if you have any questions.