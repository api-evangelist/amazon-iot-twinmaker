---
title: "How to use your own data source in AWS IoT TwinMaker"
url: "https://aws.amazon.com/blogs/iot/own-data-source-aws-iot-twinmaker/"
date: "Tue, 18 Oct 2022 16:17:13 +0000"
author: "Ali Benfattoum"
feed_url: "https://aws.amazon.com/blogs/iot/tag/aws-iot-twinmaker/feed/"
---
<h1>Introduction</h1> 
<p>AWS IoT TwinMaker makes it easier for developers to create digital twins of real-world systems such as buildings and factories with the ability to use existing data from multiple sources.</p> 
<p>AWS IoT TwinMaker uses a <a href="https://docs.aws.amazon.com/iot-twinmaker/latest/guide/data-connector-interface.html" rel="noopener noreferrer" target="_blank">connector-based architecture</a> that you can connect data from your own data source to AWS IoT TwinMaker without needing to re-ingest or move the data to another location. AWS IoT TwinMaker provides built-in data connectors for AWS services such as <a href="https://aws.amazon.com/iot-sitewise" rel="noopener noreferrer" target="_blank">AWS IoT SiteWise</a> and <a href="https://aws.amazon.com/kinesis/video-streams/" rel="noopener noreferrer" target="_blank">Amazon Kinesis Video Streams</a>. You can also create custom data connectors to use with other AWS or third-party data sources, such as Amazon Timestream, Amazon DynamoDB, Snowflake, and Siemens Mindsphere.</p> 
<p>In this blog, you will learn how to use your own data source in AWS IoT TwinMaker using the AWS IoT TwinMaker data connector interface.</p> 
<h2>Overview</h2> 
<p>The connection between a data source and AWS IoT TwinMaker is described in <a href="https://docs.aws.amazon.com/iot-twinmaker/latest/guide/twinmaker-component-types.html" rel="noopener noreferrer" target="_blank">Components</a>. A component accesses an external data source by using a Lambda connector. A Lambda connector is a Lambda function that you specify in the component definition.</p> 
<p>Here are the steps to create a data connector for Amazon DynamoDB using a <a href="https://docs.aws.amazon.com/iot-twinmaker/latest/guide/data-connector-interfaces.html#SchemaInitializer-connector" rel="noopener noreferrer" target="_blank">Schema initializer connector</a> to fetch the properties from the underlying data source and a <a href="https://docs.aws.amazon.com/iot-twinmaker/latest/guide/data-connector-interfaces.html#DataReader-connector" rel="noopener noreferrer" target="_blank">DataReader connector</a> to get the time-series values of these properties. Once the data connector is created, you will receive direction on how to create a component for this data connector and attach it to entities.</p> 
<p><img alt="" class="size-large wp-image-10762 aligncenter" height="457" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/10/17/architecture-1-1024x457.png" width="1024" /></p> 
<p>Amazon DynamoDB is used as data source in this post but the concepts described are applicable for any other data source.</p> 
<h2>Prerequisites</h2> 
<p>To setup and execute the steps in this blog, you need the following:</p> 
<ul> 
 <li>An AWS account. If you don’t have one, see<a href="https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/" rel="noopener noreferrer" target="_blank"> Set up an AWS account</a>.</li> 
 <li>An AWS IAM Identity Center (successor to AWS Single Sign-On) user with the permissions to create the resources described in the blog.</li> 
 <li>Read the section <a href="https://docs.aws.amazon.com/iot-twinmaker/latest/guide/what-is-twinmaker.html" rel="noopener noreferrer" target="_blank">What is AWS IoT TwinMaker?</a> of the documentation to understand the key concepts of AWS IoT TwinMaker.</li> 
</ul> 
<h2>Walkthrough</h2> 
<p>In this walkthrough, you will perform 6 steps in order to connect your Amazon DynamoDB data source to AWS IoT TwinMaker:</p> 
<ol> 
 <li>Create a DynamoDB table. This table is only for the purpose of this post. You can easily adapt the instructions to use an existing database.</li> 
 <li>Create a Lambda Function for the Schema initializer connector.</li> 
 <li>Create a Lambda Function for the DataReader. You will need to give to the function’s execution role the permissions to read from the table.</li> 
 <li>Create a TwinMaker Workspace. You will need to add to the workspace role the permissions to invoke both functions.</li> 
 <li>Create a TwinMaker Component.</li> 
 <li>Test the component. Before testing the component, you will create a TwinMaker entity and attach the component to the entity.</li> 
</ol> 
<h3>Step 1: Create a DynamoDB table</h3> 
<p>For the purpose of this post, you will create a DynamoDB table named <code>TwinMakerTable</code> that contains the key <code>thingName</code> of type <code>String</code> as partition key and the key <code>timestamp</code> of type <code>Number</code> as Sort key. See <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/getting-started-step-1.html" rel="noopener noreferrer" target="_blank">how to create a DynamoDB table</a> for more information.</p> 
<p><img alt="Dynamodb table creation" class="size-full wp-image-10764 aligncenter" height="621" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/10/17/twinmaker-customdataconnector-tablecreation.png" width="838" /></p> 
<p>The table you created would store air quality measurements from sensors. You will keep it simple for this post and create items in the table corresponding to measurements from a sensor identified by its name (stored as partition key <code>thingName</code>). In addition to the name of the sensor, each measurement has the following properties of type <code>Number</code>: <code>timestamp</code> (stored as sort key <code>timestamp</code> that is the <a href="https://en.wikipedia.org/wiki/Unix_time" rel="noopener noreferrer" target="_blank">Unix timestamp</a> in <strong>milliseconds</strong> of the measurement); <code>temperature</code>, <code>humidity</code> and <code>co2</code>.</p> 
<p>Let’s create 5 items in the table, corresponding to 5 measurements of a sensor named <code>airTwin</code>. For the timestamp you can receive the current timestamp in milliseconds from this <a href="https://currentmillis.com/" rel="noopener noreferrer" target="_blank">website</a> and then derive 5 timestamps by subtracting 10000 per measurement. You can then enter random values for the properties: <code>temperature</code>, <code>humidity</code> and <code>co2</code>.&nbsp; See <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/getting-started-step-2.html" rel="noopener noreferrer" target="_blank">Write data to a table using the console</a> to learn more.</p> 
<p><img alt="Item creation in DynamoDB" class="size-full wp-image-10765 aligncenter" height="852" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/10/17/twinmaker-customdataconnector-itemcreation.png" width="920" /></p> 
<p>Now that you have the table created with data, you will create two Lambda functions. The first one for the Schema initializer connector and the second one for the DataReader connector.</p> 
<h3>Step 2: Create a Schema initializer connector</h3> 
<p>The Schema initializer connector is a Lambda function used in the component type or entity lifecycle to fetch the component type or component properties from the underlying data source. You will create a Lambda function that will return the schema of the <code>TwinMakerTable</code>.</p> 
<p>You create a Node.js Lambda function using the Lambda console.</p> 
<ul> 
 <li>Open the <a href="https://console.aws.amazon.com/lambda/home#/functions" rel="noopener noreferrer" target="_blank">Functions page</a>.</li> 
 <li>On the Lambda console, choose <strong>Create function</strong>.</li> 
 <li>Under <strong>Basic information</strong>, do the following: 
  <ul> 
   <li>For <strong>Function name</strong>, enter <code>TwinMakerDynamoSchemaInit</code>.</li> 
   <li>For <strong>Runtime</strong>, confirm that <strong>Node.js 16.x</strong> is selected.</li> 
  </ul> </li> 
 <li>Choose <strong>Create function</strong>.</li> 
 <li>Under <strong>Function code</strong>, in the inline code editor, copy/paste the following code and choose <strong>Deploy</strong>:</li> 
</ul> 
<pre><code class="lang-js">exports.handler = async (event) =&gt; {
    let result = {
          properties: {
                temperature: {
                  definition: {
                      dataType: {
                          type: "DOUBLE"
                      },
                      isTimeSeries: true
                  }
                },
                humidity: {
                  definition: {
                      dataType: {
                          type: "DOUBLE"
                      },
                      isTimeSeries: true
                  }
                },
                co2: {
                  definition: {
                      dataType: {
                          type: "DOUBLE"
                      },
                      isTimeSeries: true
                  }
                },
              
          }
        }
    
    return result
}
</code></pre> 
<p>This function sends the definition of each property of our table and specifies the type. In this case all properties are of type “DOUBLE” and are time-series data. You can check the valid types in the <a href="https://docs.aws.amazon.com/iot-twinmaker/latest/apireference/API_DataType.html" rel="noopener noreferrer" target="_blank">documentation</a>.</p> 
<p><em>Note: here the properties are hard-coded in the function. You could design a function that retrieves automatically the properties and their types from an Item for example.</em></p> 
<p>Now let’s create the DataReader connector.</p> 
<h3>Step 3: Create a DataReader connector</h3> 
<p>DataReader is a data plane connector that is used to get the time-series values of properties in a single component.</p> 
<p>You create a Node.js Lambda function using the Lambda console.</p> 
<ul> 
 <li>Open the <a href="https://console.aws.amazon.com/lambda/home#/functions" rel="noopener noreferrer" target="_blank">Functions page</a>.</li> 
 <li>On the Lambda console, choose <strong>Create function</strong>.</li> 
 <li>Under <strong>Basic information</strong>, do the following: 
  <ul> 
   <li>For <strong>Function name</strong>, enter <code>TwinMakerDynamoDataReader</code>.</li> 
   <li>For <strong>Runtime</strong>, confirm that <strong>Node.js 16.x</strong> is selected.</li> 
  </ul> </li> 
 <li>Choose <strong>Create function</strong>.</li> 
 <li>Under <strong>Function code</strong>, in the inline code editor, copy/paste the following code and choose <strong>Deploy</strong>:</li> 
</ul> 
<pre><code class="lang-js">const TABLE = 'TwinMakerTable'
const aws = require('aws-sdk')
const dynamo = new aws.DynamoDB.DocumentClient()


exports.handler = async (event) =&gt; {
    try {
 
        let {workspaceId, entityId, componentName, selectedProperties, startTime, endTime } = event
        
        
        // QUERY THE DATABASE WITH THE SELECTED PROPERTIES
        const {Items} = await dynamo.query({
            TableName: TABLE,
            ProjectionExpression: `${selectedProperties}, #tmsp`,
            KeyConditionExpression: `thingName = :hashKey AND #tmsp BETWEEN :startTime AND :endTime`,
            ExpressionAttributeNames: {
                '#tmsp': 'timestamp'
            },
            ExpressionAttributeValues: {
                ':hashKey': entityId,
                ':startTime': (new Date(startTime)).getTime(), 
                ':endTime': (new Date(endTime)).getTime() 
            }
        }).promise()

        let results = { propertyValues: [] }
        let res = []
        Items.forEach(item =&gt; {
    
            selectedProperties.forEach(prop =&gt; {
                if(!res[prop]){
                    res[prop] = {
                        entityPropertyReference:{
                            propertyName: prop,
                            componentName,
                                   entityId: event.entityId

                        },
                        values: []
                    }
                }
                res[prop].values.push({
                    time: (new Date(item['timestamp'])).toISOString(),
                    value: {doubleValue: item[prop]}
                })
            })
    
        })
    
        for (let key in res){
            results.propertyValues.push(res[key])
        }
    
        console.log(results)
        return results
    } catch (e) {
        console.log(e)
    }

}
</code></pre> 
<p>The TwinMaker component will use this DataReader connector to fetch the data from the DynamoDB table. The component provides in the request two properties startTime and endTime (ISO-8601 timestamp format) that are used by the connector to fetch only the data in this time range. You can check the request and response interfaces in the <a href="https://docs.aws.amazon.com/iot-twinmaker/latest/guide/data-connector-interfaces.html" rel="noopener noreferrer" target="_blank">Data connectors</a> section of the documentation.</p> 
<p>Before moving to the next step, you need to grant the function the access to the table. See <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_lambda-access-dynamodb.html" rel="noopener noreferrer" target="_blank">Allows a Lambda function to access an Amazon DynamoDB table</a> to learn more.</p> 
<p><img alt="Adding permissions to the Lambda function to access the table" class="size-large wp-image-10767 aligncenter" height="482" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/10/17/twinmaker-customdataconnector-twinmakerole-1024x482.png" width="1024" /></p> 
<p>Now you can move to the step of creating a workspace in AWS IoT TwinMaker.</p> 
<h3>Step 4: Create a Workspace in AWS IoT TwinMaker</h3> 
<p>On the <a href="https://console.aws.amazon.com/iottwinmaker/home" rel="noopener noreferrer" target="_blank">AWS IoT TwinMaker console</a>, create a workspace named <code>AirWorkspace</code>. You can follow the instructions of the section <a href="https://docs.aws.amazon.com/iot-twinmaker/latest/guide/twinmaker-gs-workspace.html" rel="noopener noreferrer" target="_blank">Create a workspace</a> of the AWS IoT TwinMaker documentation.</p> 
<p>Once the workspace is created, you should have an Amazon Simple Storage Service (Amazon S3) bucket created. AWS IoT TwinMaker will use this bucket to store information and resources related to the workspace.</p> 
<p>You should also have an IAM Identity Center role created. This role allows the workspace to access resources in other services on your behalf.</p> 
<p><img alt="" class="size-full wp-image-10768 aligncenter" height="505" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/10/17/twinmaker-customdataconnector-workspace.png" width="864" /></p> 
<p>Before creating the component, you must provide permissions to invoke both lambda functions (created in the previous steps) to the workspace role. See <a href="https://docs.aws.amazon.com/iot-twinmaker/latest/guide/twinmaker-gs-service-role.html#twinmaker-gs-service-role-external" rel="noopener noreferrer" target="_blank">Permissions for a connector to an external data source</a> for an example of giving permission to the service role to use a Lambda function.</p> 
<pre><code class="lang-json">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": [
                "arn:aws:lambda:{{AWS_REGION}}:{{ACCOUNT_ID}}:function:TwinMakerDynamoDataReader",
                "arn:aws:lambda:{{AWS_REGION}}:{{ACCOUNT_ID}}:function:TwinMakerDynamoSchemaInit"
            ]
        }
    ]
}</code></pre> 
<p>You can now create your component.</p> 
<h3>Step 5: Create an AWS IoT TwinMaker component</h3> 
<p>Select the workspace you have created. In the workspace, choose <strong>Component types</strong> and then choose <strong>Create component type</strong>.</p> 
<p><img alt="Twinmaker component types" class="size-large wp-image-10771 aligncenter" height="435" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/10/17/twinmaker-customdataconnector-components-1024x435.png" width="1024" /></p> 
<p>Copy the following JSON document in the <strong>Request</strong> section and <strong>replace the ARN</strong> of the DataReader and Schema initializer functions respectively with the ones you created before:</p> 
<pre><code class="lang-json">{
  "componentTypeId": "com.dynamodb.airQuality",
  "description": "Connector for DynamoDB – Use case Air Quality",
  "propertyDefinitions":{
   },
  "functions": {
      "dataReader": { 
            "implementedBy": {
                             "lambda": {
                                    "arn": "arn:aws:lambda:{{AWS_REGION}}:{{ACCOUNT_ID}}:function:TwinMakerDynamoDataReader"
                              }
             }
      },
      "schemaInitializer": {
            "implementedBy": {
                             "lambda": { 
                                   "arn": "arn:aws:lambda:{{AWS_REGION}}:{{ACCOUNT_ID}}:function:TwinMakerDynamoSchemaInit"
                               }
            }
       }
  }
}
</code></pre> 
<p>Choose <strong>Create component type</strong>. Now the component is created, you can create an entity to test the component.</p> 
<h3>Step 6: Create an entity and test the component</h3> 
<p>You will now create an entity and attach the component you created to it.</p> 
<ul> 
 <li>On the <strong>Workspaces</strong> page, choose your workspace, and then in the left pane choose <strong>Entities</strong>.</li> 
 <li>On the <strong>Entities</strong> page, choose <strong>Create</strong>, and then choose <strong>Create entity</strong>.</li> 
 <li>In the <strong>Create an entity</strong> window, enter airTwin for the entity name <strong>and also for the entity ID</strong> of your entity.</li> 
 <li>Choose <strong>Create entity</strong>.</li> 
</ul> 
<p><img alt="Create entity in TwinMaker" class="size-large wp-image-10773 aligncenter" height="622" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/10/17/twinmaker-customdataconnector-entity-1024x622.png" width="1024" /></p> 
<ul> 
 <li>On the <strong>Entities</strong> page, choose the entity you just created, and then choose <strong>Add component</strong>.</li> 
 <li>Enter a name for the component. You can call it <code>dynamoAirComponent</code>.</li> 
 <li>In <strong>Type</strong>, select the component <code>com.dynamodb.airQuality</code> created previously.</li> 
 <li>Choose <strong>Add component</strong>.</li> 
</ul> 
<p><img alt="Add a component in TwinMaker" class="size-large wp-image-10774 aligncenter" height="332" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/10/17/twinmaker-customdataconnector-addcomponent-1024x332.png" width="1024" /></p> 
<p>The component is attached to the entity with the ID <code>airTwin</code>. Now the only step that remains, is to test the component. When testing the component (or when calling the <a href="https://docs.aws.amazon.com/iot-twinmaker/latest/apireference/API_GetPropertyValueHistory.html" rel="noopener noreferrer" target="_blank">GetPropertyValueHistory</a> API action), the component will send to the DataReader Lambda connector a request including the ID for the entity. The Lambda connector will use the ID to query the measurements of the sensor with the name corresponding to the ID. In this case, it will be measurements from the <code>airTwin</code> sensor.</p> 
<ul> 
 <li>On the <strong>Entities</strong> page, choose the entity <code>airTwin</code>, and then select the component <code>com.dynamodb.airQuality</code>.</li> 
 <li>Then choose <strong>Actions</strong> and <strong>View component details</strong>.</li> 
 <li>In the tab <strong>Test</strong>, select the properties you want to retrieve and a time range. Make sure that the time range selected includes the timestamp of the measurements.</li> 
 <li>Finally, choose<strong> Run test</strong> to test our component.</li> 
</ul> 
<p>You should see the measurements of your sensors in the <strong>Time-series result</strong> section.</p> 
<p><img alt="Time-series result in TwinMaker" class="size-large wp-image-10775 aligncenter" height="672" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/10/17/twinmaker-customdataconnector-results-1024x672.png" width="1024" /></p> 
<p>You can now call the <a href="https://docs.aws.amazon.com/iot-twinmaker/latest/apireference/API_GetPropertyValueHistory.html" rel="noopener noreferrer" target="_blank">GetPropertyValueHistory</a> API action to retrieve the measurements from your sensors stored in your DynamoDB table.</p> 
<p><img alt="" class="size-full wp-image-10776 aligncenter" height="452" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/10/17/twinmaker-customdataconnector-api-results.png" width="849" /></p> 
<h2>Cleaning up</h2> 
<p>To avoid incurring future charges, delete the resources created during this walk-through.</p> 
<h2>Conclusion</h2> 
<p>AWS IoT TwinMaker provides a unified data access API to read from and write to your digital twin’s source data. You can use your existing data sources without the need to move your data.</p> 
<p>In this blog, you learned how to connect an Amazon DynamoDB table to AWS IoT TwinMaker. The concepts described are applicable to your other data sources. You can also combine multiple data sources to enrich your digital twin applications.</p> 
<p>If you want to see an example of a solution using AWS IoT TwinMaker and Amazon S3 as data source, watch the video <a href="https://youtu.be/iSFtl46h6Vw" rel="noopener noreferrer" target="_blank">Build a Digital Twin using the Smart Territory Framework and AWS IoT TwinMaker</a> on Youtube. You can also visit the related <a href="https://github.com/aws-samples/aws-stf-dc-twinmaker" rel="noopener noreferrer" target="_blank">GitHub repository</a> to check the code.</p> 
<h2>About the Author</h2> 
<table> 
 <tbody> 
  <tr> 
   <td><img alt="" class="alignleft size-full wp-image-10824" height="150" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/10/18/abenfat1.png" width="150" /><a href="https://www.linkedin.com/in/alibenfattoum/" rel="noopener noreferrer" target="_blank">Ali</a> is a Technology Evangelist for IoT and Smart Cities at Amazon Web Services. With over 12 years of experience in IoT and Smart Cities, Ali brings his technical expertise to enable and help AWS partners and customers to accelerate their IoT and Smart Cities projects. Ali also holds an executive MBA, giving him the ability to zoom out and help customers and partners at a strategic level.</td> 
  </tr> 
 </tbody> 
</table> 
<p></p>
