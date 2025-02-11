# JavaScript


The implementation is tailored towards using AWS Lambda to run the code. Since AWS Lambda does not have JavaScript AWS SDK v3 pre-installed, we'll be using AWS SDK v2. 

There are minimal modifications required between v2 and v3 including the differences between the importation of specific commands and naming convention of Redshift Data API client. 

For more information on the Redshift Data API Client, please check out [JavaScript AWS SDK v2](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/RedshiftData.html) or [JavaScript AWS SDK v3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-redshift-data/classes/redshiftdata.html).

If you would like to use JavaScript AWS SDK v3 and AWS Lambda, you will have to create a lambda layer and include JS AWS SDK as a node module. In addition, layers can only be used with Lambda functions deployed as .zip file archives [mentioned here](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html). 

## Pre-requisites

* The default Lambda function timeout is set to 3 seconds. Depending on your style of execution, you may have to increase this parameter.
* For this tutorial, we’ll be setting the Lambda Handler as `index.handler`
* IAM Role attached to your Redshift cluster having access to S3
* IAM Role used by your Lambda function is attached with a policy - `AmazonRedshiftDataFullAccess` see more [here](https://docs.aws.amazon.com/redshift/latest/mgmt/data-api.html#data-api-access).


## Walk-through

The AWS Lambda event handler JSON object should have this structure. Replace the values for placeholder according to your redshift cluster and IAM Role configuration. 

```json
{
    "redshift_cluster_id": "<your redshift cluster identifier>",
    "redshift_database": "<your redshift database name>",
    "redshift_user": "<your redshift database user>",
    "redshift_iam_role": "<your redshift IAM role with correct authorization and access>",
    "run_type": "<synchronous OR asynchronous>"
}
```

**JS AWS SDK v2 - Redshift Data API Client**


```JavaScript 
const AWS = require("aws-sdk");
// code

const redshiftDataApiClient = new AWS.RedshiftData({ region: "<your region>" });
```

* Additional information can be found [here](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/RedshiftData.html).

**JS AWS SDK v3 - Redshift Data API Client**


```JavaScript 
import { RedshiftData } from "@aws-sdk/client-redshift-data";

// code

const redshiftDataApiClient = new RedshiftData({ region: "<your region>" });
```

* Additional information on RedshiftData can be found [here](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-redshift-data/classes/redshiftdata.html).

* Additional information on RedshiftDataClient (parent to RedshiftData) can be found [here](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-redshift-data/classes/redshiftdataclient.html). 

* Documentation: [Creating Lambda Layers](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html)

* Documentation: [Deploy NodeJS Lambda Functions with Zip file archives](https://docs.aws.amazon.com/lambda/latest/dg/nodejs-package.html)



Setup the SQL statements you’re going to execute. We’re going to do CREATE, COPY, UPDATE, DELETE, SELECT on our Redshift Cluster. Additional SQL commands can be found [here](https://docs.aws.amazon.com/redshift/latest/dg/c_SQL_commands.html). 

```JavaScript
sqlStatements.set("SELECT", "SELECT r_regionkey, r_name FROM public.region;");
```

Use `executeStatement()` to access the Redshift Data API. 
* [JS AWS SDK v2 Documentation](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/RedshiftData.html#executeStatement-property) 
* [JS AWS SDK v3 Documentation](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-redshift-data/classes/redshiftdata.html#executestatement)

You can monitor events from your Redshift Data API in Amazon EventBridge. Data API events are sent when the ExecuteStatement API operation sets the WithEvent option to true.  The documentation can be found [here](https://docs.aws.amazon.com/redshift/latest/mgmt/data-api-monitoring-events.html).

You can leverage AWS Secrets Manager to enable access to your Amazon Redshift database. This parameter is required when authenticating. 

The documentation for `withEvent` and `SecretArn` parameter for your ExecuteStatement can be found [here](https://docs.aws.amazon.com/redshift-data/latest/APIReference/API_ExecuteStatement.html). 


```js
await redshiftDataApiClient.executeStatement(executeStatementInput).promise()
    .then((response) => {
        queryId = response.Id;
    })
    .catch((error) => {
        console.log('ExecuteStatement has failed.');
        throw new Error(error);
    });
```

We can set the run type to be synchronous or asynchronous. Please check the general ReadMe page for more information. 

If run type is synchronous, the while loop enforces each SQL execution to be completed before running the next. 

`MAX_WAIT_CYCLES` is set to 20 seconds as a timeout precaution. You may change this based on your requirements.


```JavaScript
while (attempts < MAX_WAIT_CYCLES) {
    attempts++;
    await sleep(1);

    ({ Status: queryStatus, ...describeStatementInfo } = await getDescribeStatement(redshiftDataApiClient, queryId));

    if (queryStatus === 'FAILED') {
        throw new Error(`SQL query failed: ${queryId}: \n Error: ${describeStatementInfo.Error}`);
    } else if (queryStatus === 'FINISHED') {
        console.log(`Query status is: ${queryStatus} for query id: ${queryId} and command: ${command}`);

        // Code

        break;
    } else {
        console.log(`Currently working... query status is ${queryStatus}`);
    }

    if (attempts >= MAX_WAIT_CYCLES) {
        throw new Error(`Limit for MAX_WAIT_CYCLES has been reached before the query was able to finish. We have exited out of the while-loop. You may increase the limit accordingly. \n Query status is: %s for query id: ${queryId} and command: ${command}`);
    }
}

```

If we run the sql statements synchronously, some statements might return results that we’d want to see. 
Add the following code into the while loop when a SQL query is finished. 

```JavaScript

// Print query response if available (typically from Select SQL statements)
if (describeStatementInfo.HasResultSet) {
    await redshiftDataApiClient.getStatementResult({ Id: queryId }).promise()
        .then((statementResult) => {
            console.log(`Printing response for query: ${command} --> ${JSON.stringify(statementResult.Records)}`);
        })
        .catch((error) => {
            console.log('GetStatementResult has failed.');
            throw new Error(error);
        });
}

```

Please review the JavaScript file for a holistic point of view. 

