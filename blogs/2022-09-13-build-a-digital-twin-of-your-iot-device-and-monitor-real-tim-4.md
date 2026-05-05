---
title: "Build a digital twin of your IoT device and monitor real-time sensor data using AWS IoT TwinMaker (Part 1 of 2)"
url: "https://aws.amazon.com/blogs/iot/build-a-digital-twin-of-your-iot-device-and-monitor-real-time-sensor-data-using-aws-iot-twinmaker-part-1-of-2/"
date: "Tue, 13 Sep 2022 22:58:16 +0000"
author: "Angelo Postiglione"
feed_url: "https://aws.amazon.com/blogs/iot/tag/aws-iot-twinmaker/feed/"
---
<h1>Introduction</h1> 
<p>A digital twin is a living digital representation of a physical system that is dynamically updated to mimic the structure, state, and behavior of the physical system to drive business outcomes. Building one is usually not an easy task – to solve this challenge, we launched a new service, <a href="https://aws.amazon.com/iot-twinmaker/"><strong>AWS IoT TwinMaker</strong></a>. AWS IoT TwinMaker connects data from a variety of sources like equipment sensors, video feeds, and business applications and creates a knowledge graph to model real-world systems and generate real-time insights from the digital twin.</p> 
<p>Some <a href="https://aws.amazon.com/iot-twinmaker/customers/?nc=sn&amp;loc=6">customers</a> are using AWS IoT TwinMaker to optimize maintenance schedules by quickly pinpointing and addressing equipment and process anomalies, or to give field workers a consolidated view of all asset and operational data. <a href="https://aws.amazon.com/blogs/iot/improving-building-operational-performance-with-cognizant-1facility-and-aws-iot-twinmaker/">Another common use case</a> is enhancing the user experience and improving occupant comfort in buildings by monitoring live and historical data about temperature, occupancy or air quality within rooms.</p> 
<p>This blog post series addresses this last scenario: monitoring temperature and humidity of a room in real time while being able to control the location and status of sensors.</p> 
<p>To simulate this scenario, you will learn how to use AWS IoT TwinMaker to create a digital twin of a <a href="https://www.raspberrypi.com/">Raspberry Pi</a> device connected to a sensor that collects temperature and humidity data. You will integrate with an Amazon Managed Grafana dashboard to visualize the 3D environment where the digital twin lives together with collected data that influences the device status and 3D model representation in real time.</p> 
<p>Part 1 (this blog post) covers an overview of the solution, the setup of the time-series database, which will host your data, and the configuration of the IoT Thing. It also covers the wiring of the sensor to the Raspberry Pi device.</p> 
<p>In <a href="https://aws.amazon.com/blogs/iot/build-a-digital-twin-of-your-iot-device-and-monitor-real-time-sensor-data-using-aws-iot-twinmaker-part-2-of-2/">Part 2</a>, you will continue with the setup of the <a href="https://aws.amazon.com/grafana/">Amazon Managed Grafana</a> dashboard that will be used to visualize data. You will create the <a href="https://aws.amazon.com/lambda/">AWS Lambda</a> function to read data from the time-series database and most importantly, you will setup AWS IoT TwinMaker and integrate it with the Amazon Managed Grafana dashboard to display your digital twin together with real time data you will collect.</p> 
<h2>Solution Overview</h2> 
<p>The diagram below shows a <strong>high-level architecture</strong> overview. Data generated from the sensor attached to the&nbsp;Raspberry Pi device is sent via a Python script to <strong><a href="https://aws.amazon.com/iot-core/">AWS IoT Core</a></strong>, that easily and securely connects devices through the <a href="https://docs.aws.amazon.com/iot/latest/developerguide/protocols.html">MQTT and HTTPS protocols</a>. From here, using <a href="https://docs.aws.amazon.com/iot/latest/developerguide/iot-rules.html">AWS IoT Core rules</a>, data is streamed to an <a href="https://aws.amazon.com/timestream/"><strong>Amazon Timestream</strong></a> database. On the <strong>AWS IoT TwinMaker</strong> side, you will create the workspace environment&nbsp;where a virtual entity is defined together with its 3D model representation. A component will also be created, which uses a&nbsp;<a href="https://aws.amazon.com/lambda/"><strong>Lambda function</strong></a> to read data from Amazon Timestream, so that your digital twin is in sync with data arriving from the sensor. For the visualization part, you will leverage the&nbsp;<a href="https://docs.aws.amazon.com/iot-twinmaker/latest/guide/grafana-integration.html">AWS IoT TwinMaker Grafana dashboard integration</a>&nbsp;to create a dashboard which presents data together with the 3D&nbsp;model. The dashboard is accessible in SSO via <a href="https://aws.amazon.com/iam/identity-center/">AWS IAM Identity Center (successor to AWS SSO).</a> Finally, you will create&nbsp;AWS IoT TwinMaker&nbsp;rules&nbsp;to be able to easily see changes in the dashboard whenever the temperature or humidity goes below or above the thresholds defined.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Architecture of our solution" class="wp-image-9756" height="397" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/09/Screenshot-2022-09-09-at-18.17.53-1024x517.png" width="787" /><br /> <span style="color: #808080;">Figure 1: The high-level architecture of the solution</span><br /> </span></p> 
<h2>Prerequisites</h2> 
<ul> 
 <li>An AWS account</li> 
 <li><a href="https://docs.aws.amazon.com/iot/latest/developerguide/connecting-to-existing-device.html">Raspberry Pi 4 Model B</a> with a pre-installed Operating System SD card and a power cable</li> 
 <li>DHT sensor (or similar, to retrieve temperature/humidity data)</li> 
 <li>Breadboard with male to male (M2M) jumper wires, and resistor. You would also need female to female (F2F) jumper wires if you’re not going to use an extension board like I did (see note in section 3 – Raspberry Pi setup)</li> 
</ul> 
<h2>Implementation</h2> 
<p>Below are the macro-steps you will perform in this blog post series:</p> 
<p>(Part1)</p> 
<ol> 
 <li>Setup of the time-series database <strong>Amazon Timestream</strong>, which will store your temperature and humidity data</li> 
 <li>Setup of <strong>AWS IoT Thing</strong> with certificates and rules to make sure that data collected will be sent to the database</li> 
 <li>Configuration of <strong>Raspberry Pi</strong> device: wiring of the sensor and creation of the Python file used to send data to AWS</li> 
</ol> 
<p>(<a href="https://aws.amazon.com/blogs/iot/build-a-digital-twin-of-your-iot-device-and-monitor-real-time-sensor-data-using-aws-iot-twinmaker-part-2-of-2/">Part2</a>)</p> 
<ol> 
 <li>Setup of the <strong>Amazon Managed Grafana</strong> dashboard that will be used to visualize data</li> 
 <li>Creation of <strong>AWS Lambda</strong> function to read data from Timestream</li> 
 <li>Setup of <strong>AWS IoT TwinMaker</strong> role, policy, bucket destination and workspace. Definition of the telemetry component to read from database. Import of the 3D model and definition of the scene</li> 
 <li>Creation of the dashboard in Amazon Managed Grafana with AWS IoT TwinMaker plugin.</li> 
</ol> 
<p><strong><span style="text-decoration: underline;">IMPORTANT</span></strong>: Some of the services used are not yet available in all the AWS regions. Make sure you create all your resources in us-east-1 or eu-west-1 (depending if you prefer US or Europe region).</p> 
<h2>1. Setup of Amazon Timestream Database</h2> 
<p>Let’s start by configuring the database for the temperature and humidity data. The use case is clearly related to collecting time-series data, so the right “tool” in this case is <a href="https://aws.amazon.com/timestream/"><strong>Amazon Timestream</strong></a>.</p> 
<p>To <strong>create a database</strong>:</p> 
<ol> 
 <li>Choose <strong>Timestream</strong> in the AWS Management Console.</li> 
 <li>Choose <strong>Create database</strong>.</li> 
 <li>Enter the following information: 
  <ul> 
   <li>Configuration: Standard database</li> 
   <li>Name: <em>TempHumidityDatabase</em></li> 
  </ul> </li> 
 <li>Confirm by choosing <strong>Create database</strong>.</li> 
</ol> 
<p style="text-align: center;"><span style="color: #666699;"><img alt="" class="wp-image-9763 alignnone" height="243" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/09/1.png" width="917" /><br /> <span style="color: #808080;">Figure 2: Creation of Timestream database</span><br /> </span></p> 
<p>Then,&nbsp;<strong>create a table</strong>.</p> 
<ol> 
 <li>Choose the <strong>Tables</strong> tab, and choose&nbsp;<strong>Create table</strong>.</li> 
 <li>Set the following values: 
  <ul> 
   <li>Database name: <em>TempHumidityDatabase</em></li> 
   <li>Table name: <em>TempHumidity</em></li> 
   <li>Data retention: 
    <ul> 
     <li>Memory: 1 Day</li> 
     <li>Magnetic: 1 Month(s)</li> 
    </ul> </li> 
  </ul> </li> 
 <li><strong>Create table</strong></li> 
</ol> 
<p>The database is ready to receive data.</p> 
<h2>2. Connecting Raspberry Pi to AWS IoT Core</h2> 
<p>In this section, you will prepare the “connection” between the Raspberry Pi device and AWS IoT Core by creating a policy and certificates, then registering the device as a “Thing”. Next, you will define the rules to send data to the Amazon Timestream database and route potential errors to logs in <a href="https://aws.amazon.com/cloudwatch/">Amazon CloudWatch</a>.</p> 
<h3><strong>Create a policy</strong></h3> 
<p><a href="https://docs.aws.amazon.com/iot/latest/developerguide/iot-policies.html">AWS IoT Core policies</a> are JSON documents and follow the same conventions as <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html#access_policies-json">AWS Identity and Access Management (IAM) policies</a>. AWS IoT Core policies allow you to control access to the AWS IoT Core data plane. The AWS IoT Core data plane consists of operations that allow you to connect to the AWS IoT Core message broker, send and receive MQTT messages, and receive or update an AWS IoT thing’s device shadow.</p> 
<p>You will now create a policy to allow the publication and subscription of a specific IoT topic (<em>/raspberry/temphumid</em>). This policy will be attached to a certificate used by the AWS IoT thing.</p> 
<ol> 
 <li>Open the AWS Management Console of your AWS account.</li> 
 <li>Navigate to the <strong>AWS IoT Core</strong> service, then from the left menu choose <strong>Manage &gt; Security &gt; Policies</strong> section.</li> 
 <li>Choose <strong>Create Policy</strong>.</li> 
 <li>Enter the following values: 
  <ul> 
   <li>Policy properties → Policy name: <em>TempHumidityPolicy</em></li> 
   <li>Policy statements → Policy document → select JSON</li> 
  </ul> </li> 
 <li>Paste the following Json replacing your AWS region and AWS account</li> 
</ol> 
<pre class="unlimited-height-code"><code class="lang-json">{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:Publish",
      "Resource": "arn:aws:iot:<strong>[AWS_REGION]</strong>:<strong>[AWS_ACCOUNT]</strong>:topic/raspberry/temphumid"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Subscribe",
      "Resource": "arn:aws:iot:<strong>[AWS_REGION]</strong>:<strong>[AWS_ACCOUNT]</strong>:topicfilter/raspberry/temphumid"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Connect",
      "Resource": "*"
    }
  ]
}</code></pre> 
<ol start="6"> 
 <li>choose <strong>Create</strong></li> 
</ol> 
<h3>Create an AWS IoT Thing and generate a certificate</h3> 
<ol> 
 <li>In the console, choose <strong>AWS IoT Core</strong>.</li> 
 <li>From the <strong>Manage &gt; All devices &gt; Things</strong> section on the left, choose <strong>Create Things</strong>.</li> 
 <li>Select <strong>Create a single thing</strong>.</li> 
 <li>Enter a name for the thing (eg <em>RaspberryPi</em>) and leave the Device Shadow section with <strong>No Shadow</strong>. Choose <strong>Next</strong>.</li> 
 <li>In <strong>Configure Device Certificate</strong>, select <strong>Auto-generate a new certificate (recommended)</strong> and choose <strong>Next</strong>.</li> 
 <li>In Policies, select the policy created before (<em>TempHumidityPolicy</em>) and choose <strong>Create Thing</strong>.</li> 
 <li>Download the <strong>Device certificate</strong> (crt) along with <strong>public</strong> and <strong>private</strong> key (pem.key) and <strong>Root CA 1 certificate</strong> in a folder on your local machine. You will use the certificate and private keys later, on your Raspberry Pi device. You won’t need the public key or the Amazon Root CA 3.</li> 
 <li>When the download is complete, choose <strong>Done</strong>.</li> 
</ol> 
<p style="text-align: center;"><span style="color: #666699;"><img alt="Figure 3: An IoT Thing has been created to represent your Raspberry Pi" class="wp-image-9771 size-full" height="308" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/09/2.png" width="879" /><br /> <span style="color: #808080;">Figure 3: An IoT thing has been created to represent your Raspberry Pi</span><br /> </span></p> 
<h3>Create a rule to send data to Amazon Timestream and errors to Amazon CloudWatch</h3> 
<ol> 
 <li>Choose <strong>AWS IoT Core</strong> in the console.</li> 
 <li>In the <strong>Manage &gt; Message Routing &gt; Rules</strong> section, choose <strong>Create rule</strong>, and then enter the following: 
  <ul> 
   <li>Name: <em>TempHumidityRule</em></li> 
   <li>Description: <em>Rule to handle temperature and humidity message</em>. Choose <strong>Next</strong>.</li> 
   <li>Rule query statement: <pre class="unlimited-height-code"><code>SELECT * FROM 'raspberry/temphumid'</code></pre> </li> 
   <li>Choose <strong>Next</strong>.</li> 
  </ul> </li> 
 <li>In the “Rule actions” panel, choose <strong>Timestream table – Write a message into the Timestream table</strong>. Next, select the Timestream database <em>TempHumidDatabase</em> and table <em>TempHumidity</em> you created before.</li> 
</ol> 
<p style="text-align: center;"><span style="color: #666699;"><img alt="Figure 4: The IoT Core rule to write data on Timestream" class="alignnone wp-image-9773" height="406" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/09/3.png" width="991" /><br /> <span style="color: #808080;">Figure 4: The IoT Core rule to write data in Amazon Timestream</span><br /> </span></p> 
<ol start="4"> 
 <li>You need to enter the dimension name (minimum of 1 is required). Define a dimension with dimension name <em>DeviceName</em> and dimension value <em>Rpi4</em>.</li> 
 <li>Next, you need to create an IAM role to allow the AWS IoT Core service to access the database. Choose <strong>Create new role</strong>, and then enter the following name: <em>TempHumidityDBRole</em></li> 
 <li>In the “Error action” panel, choose <strong>Add error action</strong>. Select <strong>CloudWatch logs – Send message data to CloudWatch logs</strong>.</li> 
 <li>Choose <strong>Create CloudWatch Log group</strong> – you’ll be redirected in a new tab to <a href="https://aws.amazon.com/cloudwatch/">CloudWatch</a>. Create a log group named <em>TempHumidityRuleErrors</em>. You can access log groups from the left menu Logs &gt; Logs group. You can leave Expiration to <em>never</em> as default.</li> 
 <li>Go back to AWS IoT Core and refresh the <strong>Log group name</strong> list and select the newly created log group.</li> 
 <li>Create an IAM role to allow the service to access CloudWatch: choose <strong>Create new role</strong>, then enter the following name <em>TempHumidityCloudwatchRole</em></li> 
</ol> 
<p style="text-align: center;"><span style="color: #666699;"><img alt="Figure 5: The IoT Core error rule action will send errors to Amazon CloudWatch" class="size-full wp-image-9777 alignnone" height="402" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/09/4.png" width="606" /><br /> <span style="color: #808080;">Figure 5: The IoT Core error rule action will send errors to Amazon CloudWatch</span><br /> </span></p> 
<ol start="10"> 
 <li>Choose <strong>Next</strong>.</li> 
 <li>Review the inputs and confirm the rule creation by choosing <strong>Create</strong></li> 
</ol> 
<p>You now have a valid IoT rule that will route temperature and humidity data sent by the sensor to the Timestream database. Errors will be sent to Amazon CloudWatch logs.</p> 
<h2>3. Raspberry Pi setup</h2> 
<p>Now that you have defined the database and prepared the AWS IoT thing that represents your Raspberry Pi, it’s time to wire the&nbsp;sensor to the Raspberry Pi and then send some data to AWS IoT Core.</p> 
<h3>Wire the sensor</h3> 
<p>In this post we use a DHT11 sensor to collect temperature and humidity data from the environment. The DHT11 is a basic, low-cost digital temperature and humidity sensor. It uses a capacitive humidity sensor and a thermistor to measure the surrounding air and generates a digital signal on the data pin</p> 
<p><em><span style="text-decoration: underline;">Note</span>: This blog was created using a Raspberry Pi 4 Model B mounted on a case box kit. This box kit neatly packages the Pi and prevents damage, but is not required. The same is true for the expansion board, which makes it easier to work on the breadboard rather than using the Raspberry Pi pins directly. You don’t need it if you want to connect wires directly to your device.</em></p> 
<p>The DHT11 sensor is connected to the breadboard as shown in the pictures following.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 6: Raspberry Pi wiring of the DHT11 sensor" class="aligncenter" height="235" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/09/5.png" width="641" /><span style="color: #808080;">Figure 6: Raspberry Pi wiring of the DHT11 sensor</span><br /> </span></p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 7: Raspberry Pi with extension board and DHT11 sensor wired" class="aligncenter" height="560" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/09/6.png" width="420" /><span style="color: #808080;">Figure 7: Raspberry Pi with extension board and DHT11 sensor wired</span><br /> </span></p> 
<h3>Send data from Raspberry Pi to AWS IoT Core</h3> 
<p>Now that you have the sensor correctly wired to your Raspberry Pi device, you will try to send some temperature and humidity data to AWS. First, you need to copy the Raspberry Pi <strong>certificates</strong> that you downloaded when you configured the AWS IoT thing, so that Raspberry Pi knows where to connect and send the generated data. There are several options to connect with Raspberry Pi and copy over files. You could use SFTP and save the certificates in a Raspberry Pi safe folder, where you can reference them in the code later.</p> 
<p>Once you have the certificates in place, you can move to the next step. You will create a Python&nbsp;file in the Raspberry Pi that will include the Python code below to collect data from the DHT11 sensor. This script will collect data and send it to AWS.</p> 
<p>On your Raspberry Pi device, create a file named <strong>temphumid.py</strong> and paste the Python code below</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-python">from AWSIoTPythonSDK.MQTTLib import AWSIoTMQTTClient
import json
import time
import board
import adafruit_dht

# MQTT config (clientID must be unique within the AWS account)
clientID = "XXXX-XXXX-XXXXX-XXXXX"
endpoint = "XXXXXXXX.[AWS_REGION].amazonaws.com" #Use the endpoint from the settings page in the IoT console
port = 8883
topic = "raspberry/temphumid"

# Init MQTT client
mqttc = AWSIoTMQTTClient(clientID)
mqttc.configureEndpoint(endpoint,port)
mqttc.configureCredentials("certs/AmazonRootCA1.pem","certs/raspberry-private.pem.key","certs/raspberry-certificate.pem.crt")

# Send message to the iot topic
def send_data(message):
    mqttc.publish(topic, json.dumps(message), 0)
    print("Message Published")

# Loop until terminated
def loop():
     #Init the dht device with data pin connected
     dhtDevice = adafruit_dht.DHT11(board.D17)

     while(True):
          try:
               temperature = dhtDevice.temperature
               humidity = dhtDevice.humidity
               print("Temp: {:.1f} C    Humidity: {}% ".format(temperature, humidity)) 
               
               message = {
                         'temperature': temperature,
                         'humidity': humidity
                         }
                    
               # Send data to topic
               send_data(message)

               time.sleep(3)
          except RuntimeError as error:     # Errors happen fairly often, DHT's are hard to read, just keep going
               print(error.args[0])

# Main
if __name__ == '__main__':
    print("Starting program...")
    try:
        # Connect
        mqttc.connect()
        print("Connect OK!")

        # Main loop called
        loop()
    except KeyboardInterrupt:
        mqttc.disconnect()
        exit()</code></pre> 
</div> 
<p>Let’s examine the code above:</p> 
<ol> 
 <li>Imported some libraries (time, json) plus the library specific for the DHT sensor – in this case Adafruit_DHT. You will need to install the dependency (e.g. with <code>sudo pip3 install Adafruit_DHT</code>)</li> 
 <li>Connected the Raspberry Pi to AWS IoT Core by setting up a name for MQTT client, which needs to be unique for the AWS account.</li> 
 <li>Defined the source for certificates, make sure you have them in your Raspberry device.</li> 
 <li>Created a loop to: 
  <ul> 
   <li>Read temperature and humidity data.</li> 
   <li>Create a message composed by temperature and humidity.</li> 
   <li>Send data to the MQTT topic.</li> 
  </ul> </li> 
</ol> 
<p>As you probably noticed, the code needs to be adjusted to include your unique <code>clientID</code> (you can choose a unique string) and most importantly, your IoT <code>endpoint</code>. To find it, in the console, select <strong>AWS IoT Core</strong> and then navigate to <strong>Settings&nbsp;&gt;&nbsp;Device data endpoint</strong>. Copy the endpoint URL you see in the page and paste it in the Python script.</p> 
<p>Once you followed the steps above in the temphumid.py file,&nbsp;do not forget to <strong>save</strong> it, then <strong>run</strong> it. If everything is set up correctly, you will see a message like the following:</p> 
<p style="text-align: center;"><span style="color: #666699;"><img alt="" class="alignnone size-full wp-image-9968" height="220" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/13/8-1.png" width="400" /><br /> <span style="color: #808080;">Figure 8: Output example of the temphumid.py python script</span><br /> </span></p> 
<p>To make sure that data sent to AWS IoT Core is correctly routed to the Amazon Timestream database, open Amazon Timestream in the console and check that the table&nbsp;<em>TempHumidity</em> contains data. You can navigate to the <strong>query editor</strong> in Amazon Timestream to have a preview of your data by choosing the three dots close to your table name and choose <strong>Preview Data</strong> – by default, it will retrieve the last 15 minutes of data.</p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 9: Preview data in Amazon Timestream" class="wp-image-9788 " height="420" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/09/9.png" width="480" /><br /> <span style="color: #808080;">Figure 9: Preview data in Amazon Timestream</span><br /> </span></p> 
<p style="text-align: center;"><span style="color: #666699;"><br /> <img alt="Figure 10: Humidity and temperature sent to Amazon Timestream via AWS IoT Core" class="alignnone size-full wp-image-9939" height="241" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/13/10_bis.png" width="868" /><br /> <span style="color: #808080;">Figure 10: Humidity and temperature sent to Amazon Timestream via AWS IoT Core</span><br /> </span></p> 
<p>If data is not there, refer to the error logs you have in the CloudWatch log group you created before.</p> 
<p>For now, you can stop the Python script. You will run it again later.</p> 
<h2>Conclusion</h2> 
<p>In <strong>part 1</strong> of this series, you have laid the foundations for the creation of a digital twin of your device. You have completed the setup of the Amazon Timestream database and you have created the AWS IoT Thing with certificates and rules to make sure that temperature and humidity data collected by your device is sent to the database. You also have configured your Raspberry device by wiring the sensor and using a Python script to send data to AWS.</p> 
<p>In <a href="https://aws.amazon.com/blogs/iot/build-a-digital-twin-of-your-iot-device-and-monitor-real-time-sensor-data-using-aws-iot-twinmaker-part-2-of-2/"><strong>part 2</strong></a>, you will continue with the setup of the Amazon Managed Grafana dashboard that will be used to visualize data. You will create the AWS Lambda function to read data from Timestream database and most importantly you will setup AWS IoT TwinMaker and integrate it with the Amazon Managed Grafana dashboard to display your digital twin 3D Model together with real time data you will collect.</p> 
<hr /> 
<h3><strong>About the author</strong></h3> 
<table> 
 <tbody> 
  <tr> 
   <td><strong><img alt="" class="alignleft wp-image-9997" height="117" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/09/13/pic4-150x150.png" width="117" />Angelo Postiglione</strong> is a Senior Solutions Architect at AWS. He’s currently based in Copenhagen, where he helps customers adopt cloud technologies to build scalable and secure solutions using AWS. In his spare time, he likes to discover new places in the world, have long walks in the nature and play guitar and drums.</td> 
  </tr> 
 </tbody> 
</table> 
<p></p>
