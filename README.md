# AWS Serverless CRUD App Using Lambda, DynamoDB, API Gateway, Cognito, and Cloudfront

This project demonstrates how to build a serverless application for a coffee shop using AWS services. The coffee shop owner can authenticate using AWS Cognito and manage inventory (perform CRUD operations) through a React frontend.

**Check App:** [url]

## Why Serverless?

AWS-managed serverless services like Lambda, Cognito, and API Gateway scale down to zero when idle and scale up rapidly as demand grows. The term "serverless" does not mean there are no servers involved; instead, it signifies that AWS handles provisioning, managing, and scaling servers for us, charging based on resource utilization. This reduces operational overhead and costs.

## Architecture Overview

![AWS Serverless Architecture](.\img\architecture.webp)

The application architecture consists of:

- **DynamoDB**: Stores coffee shop inventory data
- **Lambda Functions**: Process CRUD operations via a Lambda layer for shared dependencies
- **API Gateway**: Exposes Lambda functions as RESTful APIs, secured by Cognito
- **Cognito**: Manages user authentication
- **Cloudfront**: Serves the React frontend and acts as a CDN for the API Gateway
- **S3**: Hosts the React frontend build

Optionally, Cloudflare and AWS Certificate Manager can be used to add a custom domain to the Cloudfront distribution.

## Step-by-Step Guide

### Step 1: Create DynamoDB Table
![Dynamo-table](img\DynamoDB.png)
1. Navigate to the DynamoDB section in the AWS Console
2. Create a table named `CoffeeShop` with `coffeeId` as the partition key
3. Add a sample item for testing:

```json
{
    "coffeeId": "c123",
    "name": "new cold coffee",
    "price": 456,
    "available": true
}
```

### Step 2: Create IAM Role for Lambda Functions
1[IAM-Role](img\IAM-Role.png)
Create an IAM role named `CoffeeShopRole` with the following permissions:

- CRUD access to the CoffeeShop DynamoDB table
- Permissions to create CloudWatch logs

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "dynamodb:PutItem",
                "dynamodb:DeleteItem",
                "dynamodb:GetItem",
                "dynamodb:Scan",
                "dynamodb:UpdateItem"
            ],
            "Resource": "arn:aws:dynamodb::<DYNAMODB_TABLE_NAME>"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

### Step 3: Create Lambda Layer and Lambda Functions
![Lambda-funcs](img\Lambda.png)
#### Create a Lambda Layer:

1. Create a directory named `nodejs`
2. Initialize a Node.js project and install dependencies:

```bash
cd nodejs
npm init -y
npm i @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```

3. Create a `utils.mjs` file with DynamoDB client initialization:

```javascript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
    DynamoDBDocumentClient,
    ScanCommand,
    GetCommand,
    PutCommand,
    UpdateCommand,
    DeleteCommand
} from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

const createResponse = (statusCode, body) => {
    return {
        statusCode,
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(body),
    };
};

export {
    docClient,
    createResponse,
    ScanCommand,
    GetCommand,
    PutCommand,
    UpdateCommand,
    DeleteCommand
};
```

4. Zip the `nodejs` directory and upload it as a Lambda layer named `Dynamo-Layer`:

```bash
cd ..
zip -r layer.zip nodejs
```

#### Create Lambda Functions:

1. Create four functions: `getCoffee`, `createCoffee`, `updateCoffee`, and `deleteCoffee`
2. Attach the `Dynamo-Layer` layer and `CoffeeShopRole` IAM role to each

Example code for `getCoffee`:

```javascript
import { docClient, GetCommand, ScanCommand, createResponse } from '/opt/nodejs/utils.mjs';

const tableName = process.env.tableName || "CoffeeShop";

export const getCoffee = async (event) => {
    const { pathParameters } = event;
    const { id } = pathParameters || {};

    try {
        let command;
        if (id) {
            command = new GetCommand({
                TableName: tableName,
                Key: { "coffeeId": id },
            });
        } else {
            command = new ScanCommand({
                TableName: tableName,
            });
        }
        const response = await docClient.send(command);
        return createResponse(200, response);
    } catch (err) {
        console.error("Error fetching data from DynamoDB:", err);
        return createResponse(500, { error: err.message });
    }
};
```

Similar implementations exist for `createCoffee`, `updateCoffee`, and `deleteCoffee` (refer to the full code in the query for details).

### Step 4: Create API Gateway to Expose Lambda Functions
![api-gateway](img\api-gateway.webp)
1. Create an HTTP API Gateway
2. Add the following routes:
   - `GET /coffee` → `getCoffee`
   - `GET /coffee/{id}` → `getCoffee`
   - `POST /coffee` → `createCoffee`
   - `PUT /coffee/{id}` → `updateCoffee`
   - `DELETE /coffee/{id}` → `deleteCoffee`
3. Test the APIs using tools like Postman or Thunderclient

### Step 5: Create Cognito UserPool and API Gateway Authorizer
![cognito](img\cognito.webp)
1. Create a Cognito UserPool with a public client (SPA App)
2. Configure it as a JWT authorizer for all API Gateway routes

### Step 6: Setup React Application and Upload Build to S3 Bucket
1. Create a React app and configure it with Cognito. Sample files are available at: https://github.com/omar99elnemr/aws-serverless-coffee-shop/tree/main/FrontendWithAuth
2. Create an S3 bucket
3. Build the React app and upload the `dist` folder to the S3 bucket
![S3-bucket](img\bucket.png)

### Step 7: Create Cloudfront Distribution with Behaviors for S3 and API Gateway
![cloudfront](img\cloudfront.png)
1. Create a Cloudfront distribution with the S3 bucket as the origin, using Origin Access Control (OAC) for private bucket access
2. Add another origin for the API Gateway
3. Create a behavior to redirect `/coffee*` routes to the API Gateway origin

### Step 8: Test App
![Login-test](img\test0.png)
![Login-test](img\test1.png)
![Login-test](img\test2.png)

### ~Step 9~: Attach Custom Domain Name to CDN (Optional)

1. Use AWS Certificate Manager to issue an SSL certificate for your custom domain
2. Update the Cloudfront distribution to use the custom domain and SSL certificate
3. Create a CNAME record in your domain's DNS settings (e.g., Cloudflare) to point to the Cloudfront distribution URL

### Step 9: Clean Up All Resources

To avoid unnecessary costs, delete all created resources:
![Delete-Resource](img\delete-resources.png)
- Cloudfront distribution
- DynamoDB table
- API Gateway
- Lambda functions and layer
- IAM role
- ACM certificate
- Cognito UserPool

## Conclusion

This README provides a complete guide to building a serverless CRUD application for a coffee shop using AWS services. Follow these steps to create a scalable, cost-effective solution with minimal server management.
