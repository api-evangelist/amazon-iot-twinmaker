---
title: "Build a digital twin of your IoT device and monitor real-time sensor data using AWS IoT TwinMaker (Part 2 of 2)"
url: "https://aws.amazon.com/blogs/iot/build-a-digital-twin-of-your-iot-device-and-monitor-real-time-sensor-data-using-aws-iot-twinmaker-part-2-of-2/"
date: "Tue, 13 Sep 2022 23:01:18 +0000"
author: "Angelo Postiglione"
feed_url: "https://aws.amazon.com/blogs/iot/tag/aws-iot-twinmaker/feed/"
---
<h1>Introduction</h1> 
<p>This post is the second of the series on how to use <strong><a href="https://aws.amazon.com/iot-twinmaker/">AWS IoT TwinMaker</a></strong> to create a digital twin of a Raspberry Pi device connected to a sensor that collects temperature and humidity data, and integrate it with an <strong><a href="https://aws.amazon.com/grafana/">Amazon Managed Grafana</a></strong> dashboard. This allows users to visualize the 3D environment where the digital twin lives together with collected data that influences the device status and 3D model representation in real time.</p> 
<p>In <a href="https://aws.amazon.com/blogs/iot/build-a-digital-twin-of-your-iot-device-and-monitor-real-time-sensor-data-using-aws-iot-twinmaker-part-1-of-2/"><strong>Part 1</strong></a>, we introduced the idea together with the general architecture shown below. If you followed Part 1 already, you have completed the configuration of the <strong><a href="https://aws.amazon.com/timestream/">Amazon Timestream</a></strong> database that will host your data, the setup of the IoT Thing, and the wiring of the sensor that collects and transmits temperature and humidity to your Raspberry Pi device.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 1: The high-level architecture of the solution" class="wp-image-9756" height="397" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/09/Screenshot-2022-09-09-at-18.17.53-1024x517.png" width="787" /><br /> <span style="color: #808080;">Figure 1: The high-level architecture of the solution</span></span></p> 
<p>In this <strong>second</strong> part, you will continue with the setup of the Amazon Managed Grafana dashboard that will be used to visualize data. You will create the AWS Lambda function to read data from the Amazon Timestream database and most importantly, you will setup AWS IoT TwinMaker and integrate it with the dashboard in Amazon Managed Grafana to display your 3D Model together with real time data you will collect.</p> 
<h2>1. Setup of Amazon Managed Grafana Dashboard workspace</h2> 
<p>From the console, search and select <strong>Amazon Managed Grafana</strong>&nbsp;from the list of AWS services. Choose <strong>Create Workspace</strong>. Use <em>TempHumidTwinmaker</em>&nbsp;as name for your Grafana workspace and optionally provide a description. Choose <strong>Next</strong>.</p> 
<p>For <strong>Step 2 Configure Settings</strong>, from the&nbsp;<strong>Authentication access section</strong>, select&nbsp;<strong>AWS IAM Identity Center (successor to AWS SSO)</strong>. For the <strong>Permission Type</strong> choose <strong>Service managed</strong>. Note that you may need to create a user if this is the first time you have configured SSO.</p> 
<p>Choose <strong>Next</strong> and leave the default (<em>Current account</em>, without selecting any data source) in the next page.&nbsp;Choose <strong>Next</strong> then <strong>Create Workspace</strong>.</p> 
<p><em><span style="text-decoration: underline;"><strong>Note</strong></span>: AWS&nbsp;IoT TwinMaker is not listed as data source. However, the plugin is already installed on all Amazon Managed Grafana workspaces. You will add it later on.</em></p> 
<p>Wait a few minutes for the workspace to be created. When the Amazon Managed Grafana workspace is ready, it will have created an IAM service role. You will be able to see it from the <strong>Summary</strong>&nbsp;tab once you have selected your workspace. Take note of this IAM service role as you will need it later. It should be something like&nbsp;<strong>arn:aws:iam::[YOUR-AWS-ACCOUNT-ID]:role/service-role/AmazonGrafanaServiceRole-[1234abcd]</strong></p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 2: The Amazon Managed Grafana workspace is ready. IAM role is displayed in the top right corner" class="alignnone size-full wp-image-9933" height="385" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/13/2_2-1.png" width="916" /><br /> <span style="color: #808080;">Figure 2: The Amazon Managed Grafana workspace is ready. IAM role is displayed in the top right corner</span></span></p> 
<p>Next, create&nbsp;the user that will access the dashboard.</p> 
<p>If you have already performed these actions in your AWS account, you can skip this step. Otherwise, select <strong>AWS IAM Identity Center (successor to AWS SSO)</strong> from the console search bar, then choose <strong>Users → Add user</strong>. Enter your username, choose how you would like to get the password, enter your email address along with your first and last name.</p> 
<p>Choose <strong>Next</strong>, skip the Groups section by choosing <strong>Next</strong> (as you don’t want to assign this user to one or more groups) and&nbsp;confirm by choosing <strong>Add user</strong>. You will receive an invitation email at the address specified and you will need to accept the invitation in order to access the AWS SSO portal. When the invitation is accepted, you will be asked to change your password depending on the setting you chose when creating the user.</p> 
<p>Back in the Amazon Managed Grafana workspace, choose the <strong>Authentication</strong> section and then <strong>Configure users and user groups</strong>&nbsp;in the&nbsp;<strong>AWS IAM Identity Center (successor to AWS SSO) </strong>section. Select the user you just created and choose <strong>Assign users and groups</strong>.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 3: Assign a user to the Amazon Managed Grafana workspace" class="alignnone size-full wp-image-9905" height="229" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_4-1.png" width="879" /><br /> <span style="color: #808080;">Figure 3: Assign a user to the Amazon Managed Grafana workspace</span></span></p> 
<p>You now have a user to access the dashboard you’re going to create at the end of this post. You need to make them an admin so they will be able to change settings in Amazon Managed Grafana and use the AWS IoT TwinMaker plugin. To do so, select the user and choose the <strong>Make Admin</strong> button.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 4: Making your user an admin" class="alignnone size-full wp-image-9870" height="266" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_5.png" width="879" /><br /> <span style="color: #808080;">Figure 4: Making your user an admin</span></span></p> 
<h2>2. Creation of Lambda function to read data from Amazon Timestream</h2> 
<p>Next you will need to create a Lambda function to retrieve data from your Amazon Timestream database. This Lambda function will be used within AWS IoT TwinMaker when you create an AWS IoT TwinMaker component.</p> 
<p>First create the <strong>IAM role</strong> required for the Lambda function to access Amazon Timestream and Amazon CloudWatch logs.&nbsp;From the console open the <strong>IAM</strong> service, then move to <strong>Roles</strong>. Choose <strong>Create Role</strong>. Choose&nbsp;<strong>AWS Service</strong>&nbsp;as Trusted Entity Type and select <strong>Lambda</strong> from the&nbsp;<strong>Use Case</strong> section. Choose <strong>Next</strong>&nbsp;and add the&nbsp;<em>AWSLambdaBasicExecutionRole</em> and&nbsp;<em>AmazonTimestreamReadOnlyAccess</em> permissions policies.&nbsp;Choose <strong>Next</strong>, give the role the name <em>ReadTimestreamFromLambda</em>&nbsp;then review the details and click <strong>Create Role</strong>.</p> 
<p><em><span style="text-decoration: underline;"><strong>Note</strong></span>:&nbsp;For this&nbsp;blog, the AmazonTimestreamReadOnlyAccess policy was used, which allow read operations to Timestream. As a best practice, you would restrict read access only to the TimeStream database (and even table) you have created.</em></p> 
<p>Next, create the Lambda function: from the Lambda homepage choose <strong>Create function</strong> then select the <strong>Author from scratch </strong>option<strong>. </strong>Name the function <em>timestreamReader</em> and select Python 3.7 as Runtime. In the <strong>Permissions</strong>&nbsp;tab, choose “Use an existing role” and select the role <em>ReadTimestreamFromLambda</em>&nbsp;created before. Choose <strong>Create function</strong>.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 5: Creating the Lambda function to read data from Amazon Timestream" class="alignnone size-full wp-image-9872" height="483" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_7.png" width="879" /><br /> <span style="color: #808080;">Figure 5: Creating the Lambda function to read data from Amazon Timestream</span></span></p> 
<p>When the function is created, move to the <strong>Configuration</strong> section and in the <strong>General configuration</strong>&nbsp;change the Memory to 256 MB and timeout to 15 min. Remember to <strong>Save</strong>.</p> 
<p>Still in the <strong>Configuration</strong> section, choose <strong>Environment variables</strong> and add the following four environment variables:</p> 
<ul> 
 <li>Key: <em>TIMESTREAM_DATABASE_NAME</em>, value <em>TempHumidityDatabase</em></li> 
 <li>Key: <em>TIMESTREAM_TABLE_NAME</em>, value <em>TempHumidity</em></li> 
 <li>Key: <em>TWINMAKER_COMPONENT_NAME </em>with no value as we will add it later</li> 
 <li>Key: <em>TWINMAKER_ENTITY_ID</em> with no value as we will add it later</li> 
</ul> 
<p>Now move to the <strong>code</strong> section. Copy and paste the following python code.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">import logging
import json
import os
import boto3

from datetime import datetime

LOGGER = logging.getLogger()
LOGGER.setLevel(logging.INFO)

# Get db and table name from Env variables as well as TwinMaker component name and entityId
DATABASE_NAME = os.environ['TIMESTREAM_DATABASE_NAME']
TABLE_NAME = os.environ['TIMESTREAM_TABLE_NAME']
TM_COMPONENT_NAME = os.environ['TWINMAKER_COMPONENT_NAME']
TM_ENTITY_ID = os.environ['TWINMAKER_ENTITY_ID']

# Python boto client for AWS Timestream
QUERY_CLIENT = boto3.client('timestream-query')


# Utility function: parses a timestream row into a python dict for more convenient field access
def parse_row(column_schema, timestream_row):
    """
    Example:
    column=[
        {'Name': 'TelemetryAssetId', 'Type': {'ScalarType': 'VARCHAR'}},
        {'Name': 'measure_name', 'Type': {'ScalarType': 'VARCHAR'}},
        {'Name': 'time', 'Type': {'ScalarType': 'TIMESTAMP'}},
        {'Name': 'measure_value::double', 'Type': {'ScalarType': 'DOUBLE'}},
        {'Name': 'measure_value::varchar', 'Type': {'ScalarType': 'VARCHAR'}}
    ]
    row={'Data': [
        {'ScalarValue': 'Mixer_15_7e3c0bdf-3b1c-46b9-886b-14f9d0b9df4d'},
        {'ScalarValue': 'alarm_status'},
        {'ScalarValue': '2021-10-15 20:45:43.287000000'},
        {'NullValue': True},
        {'ScalarValue': 'ACTIVE'}
    ]}

    -&gt;

    {
        'TelemetryAssetId': 'Mixer_15_7e3c0bdf-3b1c-46b9-886b-14f9d0b9df4d',
        'measure_name': 'alarm_status',
        'time': '2021-10-15 20:45:43.287000000',
        'measure_value::double': None,
        'measure_value::varchar': 'ACTIVE'
    }
    """
    data = timestream_row['Data']
    result = {}
    for i in range(len(data)):
        info = column_schema[i]
        datum = data[i]
        key, val = parse_datum(info, datum)
        result[key] = val
    return result

# Utility function: parses timestream datum entries into (key,value) tuples. Only ScalarTypes currently supported.
def parse_datum(info, datum):
    """
    Example:
    info={'Name': 'time', 'Type': {'ScalarType': 'TIMESTAMP'}}
    datum={'ScalarValue': '2021-10-15 20:45:25.793000000'}

    -&gt;

    ('time', '2021-10-15 20:45:25.793000000')
    """
    if datum.get('NullValue', False):
        return info['Name'], None
    column_type = info['Type']
    if 'ScalarType' in column_type:
        return info['Name'], datum['ScalarValue']
    else:
        raise Exception(f"Unsupported columnType[{column_type}]")

# This function extracts the timestamp from a Timestream row and returns in ISO8601 basic format
def get_iso8601_timestamp(str):
    #  e.g. '2022-04-06 00:17:45.419000000' -&gt; '2022-04-06T00:17:45.419000000Z'
    return str.replace(' ', 'T') + 'Z'

# Main logic
def lambda_handler(event, context):
    selected_property = event['selectedProperties'][0]

    LOGGER.info("Selected property is %s", selected_property)

    # 1. EXECUTE THE QUERY TO RETURN VALUES FROM DATABASE
    query_string = f"SELECT measure_name, time, measure_value::bigint" \
        f" FROM {DATABASE_NAME}.{TABLE_NAME} " \
        f" WHERE time &gt; from_iso8601_timestamp('{event['startTime']}')" \
        f" AND time &lt;= from_iso8601_timestamp('{event['endTime']}')" \
        f" AND measure_name = '{selected_property}'" \
        f" ORDER BY time ASC"
            
    try:
        query_page = QUERY_CLIENT.query(
            QueryString = query_string
        )
    except Exception as err:
        LOGGER.error("Exception while running query: %s", err)
        raise err

    # Query result structure: https://docs.aws.amazon.com/timestream/latest/developerguide/API_query_Query.html

    next_token = None
    if query_page.get('NextToken') is not None:
       next_token = query_page['NextToken']
    schema = query_page['ColumnInfo']

    # 2. PARSE TIMESTREAM ROWS
    result_rows = []
    for row in query_page['Rows']:
        row_parsed = parse_row(schema,row)
        #LOGGER.info('row parsed: %s', row_parsed)
        result_rows.append(row_parsed)

    # 3. CONVERT THE QUERY RESULTS TO THE FORMAT TWINMAKER EXPECTS

    # There must be one entityPropertyReference for Humidity OR one for Temperature
    entity_property_reference_temp = {}
    entity_property_reference_temp['componentName'] = TM_COMPONENT_NAME
    entity_property_reference_temp['propertyName'] = 'temperature'
    entity_property_reference_temp['entityId'] = TM_ENTITY_ID


    entity_property_reference_hum = {}
    entity_property_reference_hum['componentName'] = TM_COMPONENT_NAME
    entity_property_reference_hum['propertyName'] = 'humidity'
    entity_property_reference_hum['entityId'] = TM_ENTITY_ID


    values_temp = []
    values_hum = []

    for result_row in result_rows:
        ts = result_row['time']
        measure_name = result_row['measure_name']
        measure_value = result_row['measure_value::bigint']

        time = get_iso8601_timestamp(ts)
        value = { 'doubleValue' : str(measure_value) }

        if measure_name == 'temperature':
            values_temp.append({
                'time': time,
                'value':  value
            })
        elif measure_name == 'humidity':
             values_hum.append({
                'time': time,
                'value':  value
            })

    # The final structure "propertyValues"
    property_values = []

    if(measure_name == 'temperature'):
        property_values.append({
            'entityPropertyReference': entity_property_reference_temp,
            'values': values_temp
        })
    elif(measure_name == 'humidity'):
        property_values.append({
            'entityPropertyReference': entity_property_reference_hum,
            'values': values_hum
        })
    LOGGER.info("property_values: %s", property_values)

    # marshall propertyValues and nextToken into final response
    return_obj = {
       'propertyValues': property_values,
       'nextToken': next_token
       }

    return return_obj</code></pre> 
</div> 
<p><span style="text-decoration: underline;"><strong>Note</strong></span>: The code contains references to <strong>TM_COMPONENT_NAME</strong>&nbsp; and <strong>TM_ENTITY_ID</strong> which are respectively the name of the TwinMaker component and the Id of the TwinMaker entity representing your sensor. You are going to create both in the next section, and then update the Lambda environment variables.</p> 
<p>This code is the implementation of an AWS IoT TwinMaker connector against Timestream. Remember to <strong>Deploy</strong> your Lambda function.</p> 
<h2>3. Configuration of AWS IoT TwinMaker</h2> 
<p>In the previous paragraph you created and configured an Amazon Managed Grafana workspace and a Lambda function to read data from the Amazon Timestream database. You can now move to the configuration of the digital twin.</p> 
<h3>Configure the IAM policy and roles that will be used by AWS IoT TwinMaker</h3> 
<p>From the console select the <strong>IAM</strong> service, then move to <strong>Roles</strong>. Choose <strong>Create Role</strong>. Choose <strong>Custom trust policy</strong>&nbsp;and paste the policy below. AWS IoT TwinMaker requires that you use a service role to allow it to access resources in other services on your behalf. Choose <strong>Next</strong>.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-json">{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "iottwinmaker.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}</code></pre> 
</div> 
<p>In the next step, choose <strong>Create Policy</strong>, a new tab opens up. Select JSON tab and paste the following code:</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
"Version": "2012-10-17",
"Statement": [
{
    
    "Action": [
        "iottwinmaker:*", 
        "s3:*", 
        "iotsitewise:*", 
        "kinesisvideo:*"
    ],
    "Resource": [ "*" ],
    "Effect": "Allow"
},
{
    "Action": [   
        "lambda:invokeFunction"
    ],
    "Resource": ["*"],
    "Effect": "Allow"
},
{
    "Condition": {
    "StringEquals": {
        "iam:PassedToService": "lambda.amazonaws.com"
        }
    },
    "Action": ["iam:PassRole"],
    "Resource": ["*"],
    "Effect": "Allow"
}]}</code></pre> 
 <p>You are giving AWS IoT TwinMaker the ability to work with Amazon Simple Storage Service (Amazon S3), AWS IoT SiteWise, and Amazon Kinesis services and also the ability to invoke the Lambda function to read data from the database. You can modify this policy to be more restrictive in a production environment.</p> 
 <p>Choose <strong>Next (Tags)</strong>, then <strong>Next (Review)</strong>. Give this policy the name <em>TwinMakerWorkspacePolicy</em> and choose <strong>Create Policy</strong>. When done, go back to the page of the role you were creating and look for your new policy in the list. Choose <strong>Refresh</strong> if you don’t see it immediately. Choose <strong>Next</strong>, and give the role the name <em>TwinMakerWorkspaceRole</em> then review the details and click <strong>Create Role.</strong></p> 
</div> 
<h3>Create the AWS IoT TwinMaker workspace</h3> 
<p>From the console, search and select <strong>AWS IoT TwinMaker</strong> from the list of AWS services. Choose <strong>Create workspace</strong>. When creating the workspace, you first need to provide some basic information: type “<em>TempHumidWorkspace</em>” as Workspace Name and insert an optional Description. From the Amazon S3 bucket dropdown, select <strong>Create a new S3 bucket</strong>. From the <strong>Execution Role</strong> dropdown, select the&nbsp;<em>TwinMakerWorkspaceRole</em> role you created in the previous step.&nbsp;Choose <strong>Next</strong>.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 6: Creating the AWS IoT TwinMaker workspace" class="alignnone size-full wp-image-9878" height="370" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_9.png" width="879" /><br /> <span style="color: #808080;">Figure 6: Creating the AWS IoT TwinMaker workspace</span></span></p> 
<p>Now you will refer to the Grafana dashboard that you created before. From the <em>Dashboard management</em> page, select <strong>Amazon Managed Grafana</strong>. From&nbsp;the <strong>Grafana authentication provider</strong> dropdown, select the Grafana service role that has been created before — the one with name&nbsp;like&nbsp;<em>AmazonGrafanaServiceRole-[1234abc]</em>.&nbsp;Click <strong>Next</strong>.</p> 
<p>From the <em>Dashboard</em> role page, leave <strong>No video permissions</strong>&nbsp;selected. You will create an IAM policy and role to be used by the dashboard to access the AWS IoT TwinMaker workspace’s Amazon S3 bucket and resources. Copy the policy code provided in the page, then click <strong>Create Policy in IAM</strong>.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 7: Creating the Amazon Managed Grafana dashboard role and policy" class="alignnone size-full wp-image-9879" height="477" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_10.png" width="879" /><br /> <span style="color: #808080;">Figure 7: Creating the Amazon Managed Grafana dashboard role and policy</span></span></p> 
<p>In the new page, select <strong>JSON</strong> tab and paste the code for the policy you just copied. Choose <strong>Next (Tags)</strong>, then <strong>Next (Review)</strong>.&nbsp;Give this policy the name <em>TempHumidWorkspaceDashboardPolicy</em> and choose <strong>Create Policy</strong>.</p> 
<p>Go back to the AWS IoT TwinMaker workspace creation page and choose <strong>Create dashboard role in IAM</strong>.&nbsp;In the new page, select <strong>Custom trust policy</strong> and paste the following trust policy JSON:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-json">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": { 
                "AWS": "*"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
</code></pre> 
</div> 
<p>Choose <strong>Next</strong> and then select the IAM policy you just created (named&nbsp;<em>TempHumidWorkspaceDashboardPolicy</em>). Choose the refresh button if you don’t see it immediately.&nbsp;Choose <strong>Next</strong>, then give this role the name&nbsp;<em>TwinMakerDashboardRole</em> and choose <strong>Create Role</strong>. You will receive an alert that the trust policy is overly permissive, but you will change that later. For now, choose <strong>Continue</strong>.</p> 
<p>When done, go back to&nbsp;the AWS IoT TwinMaker workspace creation page and select the dashboard role you just created from the list. Choose the refresh button if you don’t see it immediately.</p> 
<p>Next, copy the code provided in the&nbsp;<strong>Update dashboard role policy</strong> tab. You are going to apply this policy in the <em>TwinMakerDashboardRole</em> you have just created. Choose <strong>Update trust policy in IAM</strong>&nbsp;and paste the code replacing what is present already to apply the trust policy in the role.&nbsp;With this, you are changing the overly permissive permission by applying it only to a specific AWS Principal, the service role used by Amazon Managed Grafana. Choose <strong>Update Policy</strong>.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 8: Creating the Amazon Managed Grafana dashboard role and policy (continued)" class="alignnone size-full wp-image-9881" height="558" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_11-1.png" width="779" /><span style="color: #808080;">Figure 8: Creating the Amazon Managed Grafana dashboard role and policy (continued)</span></span></p> 
<p>When done, go back to&nbsp;the AWS IoT TwinMaker workspace creation page and choose <strong>Next</strong>, then <strong>Create Workspace</strong>.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 9: Review of the Amazon Managed Grafana workspace creation" class="alignnone size-full wp-image-9882" height="449" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_12.png" width="879" /><br /> <span style="color: #808080;">Figure 9: Review of the Amazon Managed Grafana workspace creation</span></span></p> 
<p>You now have the AWS IoT TwinMaker workspace ready with all the permissions required to use it from the Grafana dashboard.</p> 
<h3>Create AWS IoT TwinMaker Component &amp; Entity</h3> 
<p>With your AWS IoT TwinMaker workspace selected, move to <strong>Component types</strong> section to create a component type. AWS IoT TwinMaker components provide context for properties and data for their associated entities. You will now create a component that will access your device’s temperature and humidity data. The component will be the link between the AWS IoT TwinMaker workspace and the Lambda function used to read values from the Timestream database.</p> 
<p>Choose <strong>Create component type</strong> and paste the following code in the <strong>Request</strong> section.</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
  "workspaceId": "<strong>[YOUR_TWINMAKER_WORKSPACE]</strong>",
  "isSingleton": false,
  "componentTypeId": "com.blog.DHT11sensor",
  "propertyDefinitions": {
    "humidity": {
      "dataType": {
        "type": "DOUBLE"
      },
      "isTimeSeries": true,
      "isRequiredInEntity": false,
      "isExternalId": false,
      "isStoredExternally": true,
      "isImported": false,
      "isFinal": false,
      "isInherited": false
    },
    "temperature": {
      "dataType": {
        "type": "DOUBLE"
      },
      "isTimeSeries": true,
      "isRequiredInEntity": false,
      "isExternalId": false,
      "isStoredExternally": true,
      "isImported": false,
      "isFinal": false,
      "isInherited": false
    }
  },
  "functions": {
    "dataReader": {
      "implementedBy": {
        "lambda": {
          "arn": "<strong>[YOUR_LAMBDA_ARN]</strong>"
        },
        "isNative": false
      },
      "isInherited": false
    }
  },
  "creationDateTime": "2022-05-19T14:58:42.140Z",
  "updateDateTime": "2022-05-19T14:58:42.140Z",
  "arn": "arn:aws:iottwinmaker:<strong>[YOUR_REGION]</strong>:<strong>[YOUR_AWS_ACCOUNT]</strong>:workspace/<strong>[YOUR_TWINMAKER_WORKSPACE]</strong>/component-type/com.blog.DHT11sensor",
  "isAbstract": false,
  "isSchemaInitialized": false,
  "status": {
    "state": "ACTIVE",
    "error": {}
    }
  }
</code></pre> 
</div> 
<p><em><strong><span style="text-decoration: underline;">IMPORTANT</span></strong>: Make sure you replace the value between brackets [YOUR_TWINMAKER_WORKSPACE], [YOUR_LAMBDA_ARN], [YOUR_REGION] and [YOUR_AWS_ACCOUNT].</em></p> 
<p>When ready, choose <strong>Create component type</strong>.</p> 
<p>Now it’s time to create an <strong>Entity</strong> for your sensor. You could potentially create a structure or hierarchy of entities representing your environment, but in this case for simplicity, only a single entity representing the device/sensor will be created. To do so, move to the <strong>Entities</strong>&nbsp;section and choose <strong>Create entity</strong>. Give the entity a name (eg&nbsp;<em>TempHumiditySensor</em>) and choose <strong>Create entity</strong>.</p> 
<p><strong><span style="text-decoration: underline;">IMPORTANT</span></strong>: This entity is the one you need to use in your Lambda function. Copy the Entity id and add it as value of the <strong>[TWINMAKER_ENTITY_ID]</strong> environment variable you have in your Lambda function, created before.</p> 
<p>Select now the Entity from the list of Entities and choose <strong>Add component</strong> on the right.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 10: Creating the AWS IoT TwinMaker entity" class="alignnone size-full wp-image-9884" height="443" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_13.png" width="879" /><br /> <span style="color: #808080;">Figure 10: Creating the AWS IoT TwinMaker entity</span></span></p> 
<p>Select the type&nbsp;<strong>com.blog.DHT11sensor</strong> and give the component a name.</p> 
<p><strong><span style="text-decoration: underline;">IMPORTANT</span></strong>: Remember to update the <strong>[TWINMAKER_COMPONENT_NAME]</strong> environment variable you have in your Lambda function with the value you choose here as component name.</p> 
<p>You will see the properties of the components in the table (like temperature and humidity). When done, choose <strong>Add component</strong>.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 11: Adding the AWS IoT TwinMaker component to the Entity" class="alignnone wp-image-9885 size-full" height="375" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_14.png" width="737" /><br /> <span style="color: #808080;">Figure 11: Adding the AWS IoT TwinMaker component to the Entity</span></span></p> 
<h3>Create AWS IoT TwinMaker Resource &amp; Scene</h3> 
<p>Next, import the 3D model to represent your device or sensor in the virtual environment. AWS IoT TwinMaker supports a variety of files like BIN, GLB, GLTF, PNG, PDF, JPG, JPEG, and MP4. In this case, a Raspberry Pi4 model was used. You should be able to find free models in websites like <a href="https://www.cgtrader.com/">CGTrader</a>,&nbsp;<a href="https://sketchfab.com/">Sketchfab</a>&nbsp;or <a href="https://www.turbosquid.com/">TurboSquid</a>.</p> 
<p>With your workspace selected, go to <strong>Resource library</strong> and choose <strong>Add resources</strong> and upload your file.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 12: Uploading the 3D model" class="alignnone size-full wp-image-9886" height="302" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_15.png" width="879" /><br /> <span style="color: #808080;">Figure 12: Uploading the 3D model</span></span></p> 
<p>Finally, you will setup your Scene. With your workspace selected, go to <strong>Scenes</strong>&nbsp;and choose <strong>Create scene</strong>. Give it an ID (name) and choose <strong>Create scene</strong>. Once created, you will be presented with a view containing 3 main sections (see screenshot below). On the left, a section containing 3 tabs:</p> 
<ul> 
 <li><strong>Hierarchy</strong>: your objects in the scene</li> 
 <li><strong>Rules</strong>: how to change items in the scene depending on the data received (we’ll use it in this exercise)</li> 
 <li><strong>Settings</strong></li> 
</ul> 
<p>In the center part, there is a section containing the <strong>3D world</strong>, with the possibility to move around, pan, zoom, tilt the view etc. On the right, you have a section with the <strong>Inspector</strong>, to see details of what is selected in the scene.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 13: The UI of the AWS IoT TwinMaker scene" class="alignnone size-full wp-image-9887" height="377" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_16.png" width="879" /><br /> <span style="color: #808080;">Figure 13: The UI of the AWS IoT TwinMaker scene</span></span></p> 
<p>You will start by creating the Rules for the item in the scene. Choose the <strong>Rules</strong> section on the left side panel and check the rules that are already present. Create two new rules, one for humidity data and another for temperature data. Define&nbsp;<em>temperatureIconRule</em> as RuleID and choose <strong>Add New Rule</strong>. Select the rule and click <strong>Add new statement</strong> to define some expressions to have the target icon change from <em>Info</em> to <em>Warning</em> to <em>Error</em> as shown below.</p> 
<p><strong><span style="text-decoration: underline;">IMPORTANT</span></strong>: Make sure that the Expression you write uses the exact word that you used to name the property coming from the sensor and stored in the database (i.e. “<em>temperature</em>” and “<em>humidity</em>”).</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 14: Rules that will make your tag change color or icon depending on the data received" class="alignnone wp-image-9888" height="622" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_17.png" width="451" /><br /> <span style="color: #808080;">Figure 14: Rules that will make your tag change color or icon depending on the data received</span></span></p> 
<p>When you are done with temperature rules, repeat the same process adding a new rule for humidity.</p> 
<p>Next, add the 3D model. In the center part of the screen, click on the + icon and select <strong>Add 3d model</strong>, selecting from the resource library the 3d object that you uploaded before.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 15: Adding your 3D model to the scene" class="alignnone size-full wp-image-9889" height="447" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_18.png" width="879" /><br /> <span style="color: #808080;">Figure 15: Adding your 3D model to the scene</span></span></p> 
<p>Once loaded, you can scale the model with the <strong>Transform</strong>&nbsp;section in the right panel. Most likely once you add it in the scene, the object will be dark. To fix that, you can adjust the lighting by clicking on <strong>Settings</strong> and choosing an <strong>Environmental Preset</strong>. Another way to add a light would be clicking on the + icon and selecting&nbsp;<strong>Add light</strong>. You can then select it and move it around with your mouse to light up your scene and the 3d model imported.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 16: Make sure your model has proper lighting" class="alignnone size-full wp-image-9890" height="556" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_19.png" width="879" /><br /> <span style="color: #808080;">Figure 16: Make sure your model has proper lighting</span></span></p> 
<p>Finally, you will add a <strong>tag</strong> to handle the humidity and temperature data and make sure that what is received affects what is shown in the scene. Click on the + icon and choose <strong>Add tag</strong>. Using the Inspector section, define its name as Temperature and choose a <strong>Default&nbsp;Icon</strong>. Select your Entity as <strong>EntityId</strong> and your component as <strong>ComponentName</strong>. Select temperature as <strong>PropertyName</strong> and <em>temperatureIconRule</em> as <strong>RuleId</strong>. Repeat the same action creating a new tag for Humidity with humidity as <strong>PropertyName</strong> and <em>humidityIconRule</em> as <strong>RuleId</strong>.</p> 
<p><strong><span style="text-decoration: underline;">Note</span></strong>: Move the two tags close to the 3D model but distant enough to make them visible in the scene.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 17: Positioning tags in the scene" class="alignnone size-full wp-image-9891" height="377" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_20.png" width="879" /><br /> <span style="color: #808080;">Figure 17: Positioning tags in the scene</span></span></p> 
<h2>4. Create an Amazon Managed Grafana dashboard with AWS IoT TwinMaker plugin</h2> 
<p>You are finally ready to create a dashboard in Amazon Managed Grafana to visualize the digital twin and your data. From the console, select<strong> Amazon Managed Grafana</strong> and then your Amazon Managed Grafana workspace. Access your Amazon Managed Grafana workspace from the link provided in the page at <strong>Amazon Managed Grafana workspace URL</strong>.&nbsp;You’ll be asked to access with the credentials you have setup when configuring the AWS IAM Identity Center (successor to AWS SSO). Since your user was set in admin, you should be able to access the Amazon Managed Grafana settings page.</p> 
<p>First, you need to add the AWS IoT TwinMaker data source. To do so, go to <strong>Configuration</strong> and choose <strong>Add data source</strong>, then search for <em>TwinMaker</em> and select <strong>AWS IoT TwinMaker</strong>.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 18: Configure AWS IoT TwinMaker as datasource for your dashboard" class="alignnone wp-image-9893" height="222" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_21.png" width="957" /><br /> <span style="color: #808080;">Figure 18: Configure AWS IoT TwinMaker as datasource for your dashboard</span></span></p> 
<p>Then, make sure that all the <strong>Connection Details</strong> are correct in the Settings of the data source. This includes the authentication provider and the ARN of the role that AWS IoT TwinMaker assumes to access the dashboard and the AWS region (<em>TwinMakerDashboardRole</em>). Here also is configured the AWS IoT TwinMaker workspace.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 19: Connection details" class="alignnone wp-image-9894" height="568" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_22.png" width="678" /><br /> Figure 19: Connection details</span></p> 
<p>Choose <strong>Save &amp; Test</strong> to verify that the connection between Grafana and the AWS IoT TwinMaker workspace is setup correctly.</p> 
<p>Then, move to the creation of the dashboard. From the left sidebar, click <strong>Create → Dashboard</strong>. We’ll start by adding an <strong>empty panel</strong> first. Choose <strong>Add a new panel</strong>.</p> 
<p>On the right, select the type of Visualization to use. In the search bar type <em>TwinMaker</em> and select <strong>AWS IoT TwinMaker Scene Viewer</strong>. Using the controls in the right part of the screen, give this panel a name and select the AWS IoT TwinMaker workspace and Scene. Your 3D model should appear in the preview.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 20: Adding AWS IoT TwinMaker Scene Viewer to the dashboard" class="alignnone wp-image-9895" height="437" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_23.png" width="960" /><br /> <span style="color: #808080;">Figure 20: Adding AWS IoT TwinMaker Scene Viewer to the dashboard</span></span></p> 
<p>Now you make sure that the connection between what is shown in the dashboard and your data is defined. To do so, create two <strong>queries</strong>, one for the temperature data and the other for the humidity. These queries will use the AWS IoT TwinMaker component you created, which in turn, uses the Lambda function to read from the Timestream database.</p> 
<p>In the query section, make sure that <strong>AWS IoT TwinMaker</strong> is selected as Data source and define a new query of type <strong>Get Property Value History by Entity</strong>. Select your Entity (<em>TempHumiditySensor</em>) and Component (<em>DHTComponent</em>), then choose the <strong>temperature</strong> property. Repeat the same adding a new query of same Type and with same Entity and Component but this time selecting the <strong>humidity</strong> property. When done, save your panel and click <strong>Apply</strong> then <strong>Save</strong>.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 21: The query needed to read data with the component" class="alignnone size-full wp-image-9896" height="347" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_24.png" width="879" /><br /> <span style="color: #808080;">Figure 21: The query needed to read data with the component</span></span></p> 
<p>Aside from the AWS IoT TwinMaker panel, you can also create other panels to represent your data in various visualization formats, for example a <strong>Gauge</strong> or <strong>Time series</strong>, to show your temperature and humidity data. You will need to configure the same query mechanism to make sure you are able to retrieve data. The little red corner on the upper-left of each panel will inform you in case of issues with the component reading data. In this case, it just alerts us that no data is coming – that’s because you haven’t started the python script in your Raspberry Pi to send data to cloud.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 22: Adding panels to the dashboard" class="alignnone size-full wp-image-9897" height="443" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_25.png" width="879" /><br /> <span style="color: #808080;">Figure 22: Adding panels to the dashboard</span></span></p> 
<h2>5. The final result</h2> 
<p>If you now start the Python script again in your Raspberry Pi device, you should be able to see temperature and humidity data populating your dashboard’s panels. Since you have defined rules in your AWS IoT TwinMaker workspace, the tags associated with the Entity represented in the dashboard (the two blue dots) will change icons (info, warning, error), or color if you define a color-based rule, whenever the temperature or humidity data received is above/below the threshold defined in your rules.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 23: The final result. Temperature tag is showing a warning icon as the threshold defined in the rule was 23°" class="alignnone size-full wp-image-9898" height="493" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/12/2_26.png" width="977" /><br /> <span style="color: #808080;">Figure 23: The final result. Temperature tag is showing a warning icon as the threshold defined in the rule was 23°</span></span></p> 
<h2>Cleaning up</h2> 
<p>If you followed along with this solution, complete the following steps to avoid incurring unwanted charges to your AWS account.</p> 
<h4>AWS IoT Core</h4> 
<ul> 
 <li>In the Manage → All devices, delete the Thing and Thing type.</li> 
 <li>In the Manage→ Security section, remove the Policy and Certificate.</li> 
 <li>In the Manage → Message Routing section, clean up the Rule.</li> 
</ul> 
<h4>Amazon Timestream</h4> 
<ul> 
 <li>Delete the table and the database.</li> 
</ul> 
<h4>Amazon Managed Grafana</h4> 
<ul> 
 <li>Delete the Amazon Managed Grafana workspace.</li> 
</ul> 
<h4>AWS IAM</h4> 
<ul> 
 <li>Delete the roles created along the way.</li> 
</ul> 
<h4>AWS IoT TwinMaker</h4> 
<ul> 
 <li>Delete the AWS IoT TwinMaker workspace</li> 
</ul> 
<h2>Conclusion</h2> 
<p>In this blog post series, you learned how to set up a simple end-to-end solution to monitor the temperature and humidity data creating a digital twin using a Raspberry Pi device and DHT sensor. The solution is achieved by first connecting the Raspberry Pi to AWS IoT Core using MQTT followed by forwarding messages from the topic stream using AWS IoT rules and putting records on an Amazon Timestream database. You also used AWS IoT TwinMaker to create a digital twin of the device/sensor and Amazon Managed Grafana to build a real time interactive dashboard.</p> 
<p>You can create more complex scenarios, involving a multitude of devices and sensors and recreating your real environment. For a more complex use case, check out the AWS IoT TwinMaker <a href="https://github.com/aws-samples/aws-iot-twinmaker-samples">Cookie Factory sample project</a>. Also, visit <a href="https://aws.amazon.com/blogs/iot/tag/aws-iot-twinmaker/">The Internet of Things on AWS – Official Blog</a> to learn more about AWS IoT TwinMaker or see <a href="https://aws.amazon.com/iot-twinmaker/customers/?nc=sn&amp;loc=6">what our customers built with it</a>.</p> 
<hr /> 
<h3><strong>About the author</strong></h3> 
<table> 
 <tbody> 
  <tr> 
   <td><strong><img alt="" class="alignleft size-thumbnail wp-image-9997" height="150" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/13/pic4-150x150.png" width="150" />Angelo Postiglione</strong> is a Senior Solutions Architect at AWS. He’s currently based in Copenhagen, where he helps customers adopt cloud technologies to build scalable and secure solutions using AWS. In his spare time, he likes to discover new places in the world, have long walks in the nature and play guitar and drums.</td> 
  </tr> 
 </tbody> 
</table> 
<p>&nbsp;</p>
