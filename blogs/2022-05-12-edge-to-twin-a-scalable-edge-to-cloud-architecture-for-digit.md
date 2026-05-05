---
title: "Edge to Twin: A scalable edge to cloud architecture for digital twins"
url: "https://aws.amazon.com/blogs/iot/edge-to-twin-a-scalable-edge-to-cloud-architecture-for-digital-twins/"
date: "Thu, 12 May 2022 16:12:29 +0000"
author: "Chris Azer"
feed_url: "https://aws.amazon.com/blogs/iot/tag/aws-iot-twinmaker/feed/"
---
<p>Are you seeking ways to get an immersive 3D view of your systems and operations to optimize efficiency, increase production, and improve performance? Perhaps you are generating all the data you need from various on-premise systems, but are unsure how to gain access to this information in a living virtual representation. In this blog post, you will learn how to build an end-to-end solution, from Edge to Twin, using AWS IoT TwinMaker. You will also learn how to configure various AWS services to pull telemetry data from an industrial mixer using Open Platform Communications United Architecture (OPC UA) at the edge and build its twin with AWS IoT TwinMaker.</p> 
<h2><strong>Overview</strong></h2> 
<p>This blog will only work with one data source, but as you acquire multiple streams of data from an industrial environment, AWS IoT TwinMaker can help save time with an automatically generated knowledge graph that binds your data sources. Data sources can range from AWS IoT SiteWise, time series historians, alarming databases, Manufacturing Execution Systems (MES), Enterprise Resource Planning (ERP) systems, and other business systems. Binding these data sources can enable you to create virtual replicas of physical systems to accurately model real-world environments. Over time, your digital twin can grow to adopt Machine Learning (ML) capabilities for anomaly detection and predictive maintenance. The image below is an example of how you can bind various data streams into AWS IoT TwinMaker, including video.</p> 
<div class="wp-caption alignnone" id="attachment_7541" style="width: 1034px;">
 <img alt="" class="size-large wp-image-7541" height="442" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/Overview_1-1024x442.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-7541">Example manufacturing dashboard for AWS IoT TwinMaker</p>
</div> 
<h3><strong>Scalable Architecture for an Industrial Edge to Twin</strong></h3> 
<div class="wp-caption alignnone" id="attachment_7542" style="width: 1034px;">
 <img alt="" class="size-large wp-image-7542" height="555" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/Edge-To-Twin-Blog-Architecture-1024x555.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-7542">This scalable architecture will be created in this blog and will allow you to manage thousands of entities and link their respective data sources</p>
</div> 
<h2><strong>Prerequisites</strong></h2> 
<p>An AWS account will be required to setup and execute the steps in this blog with real-time simulation. Several services will be configured for the edge which will stream this real-time simulated data through AWS IoT SiteWise to AWS IoT TwinMaker. It is recommended that you work in the Virginia region (us-east-1). You may incur cost on some of the following services:</p> 
<ul> 
 <li>Setup of OPC UA server on Amazon Elastic Compute Cloud (Amazon EC2) to simulate Mixer data</li> 
 <li>Setup of AWS IoT SiteWise gateway on AWS IoT Greengrass</li> 
 <li>Setup of AWS IoT SiteWise Asset Model and Asset for a Mixer</li> 
 <li>Setup of AWS IoT TwinMaker Workspace, Entities, and Scene</li> 
 <li>Visualize AWS IoT TwinMaker scene in Amazon Managed Grafana</li> 
</ul> 
<h2><strong>Running on Amazon EC2 with Ubuntu</strong></h2> 
<p>Let’s start with the telemetry simulation. For simplicity, you will use an Amazon EC2 instance, but this section may be replaced with your choice for edge computing. On the Amazon EC2 system, you will set up a gateway to ingest data from an asset using a common industrial protocol, OPC UA.</p> 
<p>The next task is to log into the instance and install node.js and Node-RED. You will also install an OPC UA server node to simulate data. This is solely for the purpose of simulation. In reality, industrial assets and systems on-premise will likely support OPC UA.</p> 
<h3><strong>Create the base Amazon EC2 image</strong></h3> 
<ol> 
 <li>Log in to the <a href="https://console.aws.amazon.com/ec2">Amazon EC2 console</a></li> 
 <li>Click <strong>Launch Instance</strong></li> 
 <li>Enter the name <strong>Edge Gateway</strong></li> 
 <li>Select the <strong>Ubuntu Quickstart</strong></li> 
 <li>Select the Instance Type – <strong>t2.medium</strong></li> 
 <li>Use an existing <strong>Key pair</strong> or <strong>Create a new key pair</strong> and click <strong>Download key pair</strong>. Your browser will save the .pem file. Keep that safe!</li> 
 <li>Click the <strong>Launch Instance</strong> button</li> 
 <li>After a couple of minutes your Amazon EC2 instance will be running.</li> 
 <li>Connect to your Instance using an SSH Client by following these steps <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html">here</a></li> 
 <li>Copy and run the command below to install node red<br /> <code>bash &lt;(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered) --confirm-root --confirm-install --skip-pi</code></li> 
 <li>Run these commands to install the OPC UA server node<br /> <code class="lang-bash">sudo systemctl start nodered.service<br /> sudo npm install --prefix ~/.node-red node-red-contrib-opcua-server &amp;&gt;/dev/null<br /> sudo systemctl restart nodered.service<br /> </code></li> 
 <li><span style="font-family: Georgia, 'Times New Roman', 'Bitstream Charter', Times, serif;"><span style="font-family: Georgia, 'Times New Roman', 'Bitstream Charter', Times, serif;">Now run these final commands to install and run the OPC UA server simulation<br /> <code class="lang-bash">curl -O https://iot-blog-files.s3.amazonaws.com/edge-to-twin/opcua_sim.json<br /> curl -vX POST http://127.0.0.1:1880/flows -d @opcua_sim.json --header "Content-Type: application/json"</code></span></span></li> 
 <li>Server should not be running and can proceed to the next step.</li> 
</ol> 
<h2><strong>Setup AWS IoT SiteWise Collector on AWS IoT Greengrass</strong></h2> 
<p>Next you will setup and install AWS IoT SiteWise Gateway. This gateway serves as the intermediary between your industrial assets and AWS IoT SiteWise. You can deploy the AWS IoT SiteWise gateway software on any <a href="https://docs.aws.amazon.com/greengrass/v2/developerguide/setting-up.html#greengrass-v2-supported-platforms">supported platform</a> for AWS IoT Greengrass.</p> 
<h3><strong>Create an AWS IoT SiteWise Gateway</strong></h3> 
<ol> 
 <li>Navigate to <a href="https://console.aws.amazon.com/iotsitewise">AWS IoT SiteWise</a></li> 
 <li>Navigate to <strong>Edge-&gt;Gateways</strong> on the left and click on <strong>Create gateway</strong> and select Greengrass v2<img alt="" class="alignnone size-large wp-image-7543" height="327" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/IoT_Sitewise_1-1024x327.png" width="1024" /></li> 
 <li>Select <strong>Default Setup</strong> and hit <strong>Next</strong> until you reach <strong>add data sources</strong></li> 
 <li>Click <strong>Add data source</strong></li> 
 <li>Configure the OPC Server as seen below and use <strong>opc.tcp://localhost:54845</strong> for the local endpoint of Node Red<img alt="" class="alignnone size-large wp-image-7544" height="836" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/IoT_Sitewise_2-1024x836.png" width="1024" /></li> 
 <li>Hit <strong>Add</strong> and then <strong>Next</strong></li> 
 <li>In the section, <strong>Review and generate an installer</strong>, change the <strong>Gateway device OS</strong> to <strong>Ubuntu</strong> and click on <strong>Generate</strong></li> 
</ol> 
<h3><strong>Install the AWS IoT SiteWise Gateway with AWS IoT Greengrass on the Amazon EC2 instance</strong></h3> 
<ol> 
 <li>Copy the installer script to the Amazon EC2 instance. You may use the scp command like below:<code class="lang-bash"><code class="lang-bash">scp -i "blog.pem" Gateway-xxxxxxxx.deploy.sh ubuntu@ec2-xx-xxx-x-xx.compute-1.amazonaws.com:</code></code></li> 
 <li>Run the shell script that was copied<code class="lang-bash">sudo bash ~/Gateway-xxxxxxxx.sh</code></li> 
 <li>Your AWS IoT SiteWise Gateway should be in sync within a couple minutes and collecting data. The image below shows data streams captured if the AWS IoT SiteWise gateway is in sync. If you do not see this, you can force the synchronization. To do so, edit the publisher configuration in your AWS SiteWise Gateway in the console, change the <strong>Publishing Order</strong> to <strong>Newest first</strong> to force a change. This will initiate a sync for the Publisher. If you still do not see streams, you may not have disassociated data ingestion enabled. To enable, go to Settings in the IoT SiteWise console, choose Data Ingestion, and then enable Disassociated data ingestion. You can verify that data streams are ingested in the AWS IoT SiteWise Console to proceed.<img alt="" class="alignnone size-large wp-image-7545" height="380" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/IoT_Sitewise_3-1024x380.png" width="1024" /></li> 
</ol> 
<h2><strong>Setup AWS IoT SiteWise Model and Asset</strong></h2> 
<p>You have completed the configuration of the Edge node and are now collecting telemetry data from an OPC UA server. This is streamed to AWS IoT SiteWise in the cloud. Next, you will create models and assets and associate these unclaimed streams of data.</p> 
<h3><strong>Create an Asset Model</strong></h3> 
<ol> 
 <li>Navigate to <a href="https://console.aws.amazon.com/iotsitewise">AWS IoT SiteWise</a> in the AWS Console.</li> 
 <li>Navigate to <strong>Build-&gt;Models</strong></li> 
 <li>Click <strong>Create model</strong></li> 
 <li>For the Name, type <strong>Mixer Model</strong></li> 
 <li>Add 3 measurements: 
  <ul> 
   <li>Name: rpm, Unit: RPM, Data Type: Double</li> 
   <li>Name: temperature, Unit: Fahrenheit, Data Type: Double</li> 
   <li>Name: state, Unit: Leave_Blank, Data Type: String<img alt="" class="alignnone size-large wp-image-7546" height="943" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/IoT_Sitewise_4-1024x943.png" width="1024" /></li> 
  </ul> </li> 
 <li>Click <strong>Create Model</strong></li> 
</ol> 
<h3><strong>Create an Asset from the Model and assign data streams</strong></h3> 
<ol> 
 <li>After the model is created, you will see a section for <strong>Assets</strong> at the bottom of the Mixer Model. Click <strong><strong>Create asset</strong></strong><img alt="" class="alignnone size-large wp-image-7547" height="818" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/IoT_Sitewise_5-1024x818.png" width="1024" /></li> 
 <li>Select the Mixer Model template and provide a name, <strong>Mixer_A</strong></li> 
 <li>Return to the Data Streams section in AWS IoT SiteWise, select the 3 data streams and click on <strong>Manage Data Streams</strong></li> 
 <li>Choose each Measurement on the left and select the corresponding measurement on the right. Then click <strong>Choose</strong>.<img alt="" class="alignnone size-large wp-image-7548" height="558" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/IoT_Sitewise_6-1024x558.png" width="1024" /></li> 
 <li>Click <strong>Update</strong> once complete</li> 
 <li>Now return to your asset <strong>Mixer_A</strong>. You can verify that your asset is receiving data by checking the <strong>Latest value</strong> column<img alt="" class="alignnone size-large wp-image-7549" height="697" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/IoT_Sitewise_7-1024x697.png" width="1024" /></li> 
</ol> 
<h2><strong>Building your Twin</strong></h2> 
<p>Now that you have connected to an asset at the Edge and streamed this data into your AWS IoT SiteWise data source, you can now use AWS IoT TwinMaker to build your twin. Keep in mind, AWS IoT SiteWise is not a requirement for AWS IoT TwinMaker. Sources of telemetry data may come from other systems. For this blog, you are specifically handling telemetry data but an industrial twin is expected to be comprised of several data sources.</p> 
<h3><strong>Create a workspace</strong></h3> 
<ol> 
 <li>Navigate to <a href="https://console.aws.amazon.com/iottwinmaker/home">AWS IoT TwinMaker</a></li> 
 <li>Click <strong>Create workspace</strong></li> 
 <li>Provide a name <strong>MixerWorkspace</strong></li> 
 <li>Select to create a new S3 bucket</li> 
 <li>Select to Auto-generate a new role, then click <strong>Skip to review and create</strong></li> 
 <li>Click <strong>Create Workspace</strong></li> 
</ol> 
<h3><strong>Create an Entity</strong></h3> 
<ol> 
 <li>Navigate to <strong>Workspaces → Entities</strong></li> 
 <li>Click <strong>Create Entity</strong></li> 
 <li>Provide the name <strong>Mixers</strong></li> 
 <li>Click on the entity your created and create a child entity</li> 
 <li>Provide the name <strong>Mixer_A</strong>. When building out a twin for a facility, you may create a hierarchy that is representative of your physical layout or process.</li> 
 <li>Select the <strong>Mixer_A</strong> entity and click to <strong>Add component</strong></li> 
 <li>Provide the name <strong>SiteWise</strong> and select the type <strong>com.amazon.iotsitewise.connector</strong></li> 
 <li>Once the type is selected, you will see that you can select the asset model and asset that you configured in AWS IoT SiteWise. Just select the <strong>Mixer Model</strong> and <strong>Mixer_A</strong> asset. That’s it! The data is now linked.</li> 
 <li>Scroll down and click <strong>Add component</strong>. See expected results below<img alt="" class="alignnone size-large wp-image-7550" height="838" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/IoT_Twinmaker_1-1024x838.png" width="1024" /></li> 
</ol> 
<h3><strong>Add a Resource</strong></h3> 
<ol> 
 <li>Download this CAD <a href="https://github.com/aws-samples/aws-iot-twinmaker-samples/blob/main/src/workspaces/cookiefactory/scenes/CookieFactoryMixer.glb">model</a> from our Cookie Factory demo for a mixer.</li> 
 <li>Navigate to <strong>Workspace→Resource</strong> library</li> 
 <li>Click <strong>Add resource</strong></li> 
 <li>Chose the GLB model file you downloaded. You will use this later when creating the Scene for this mixer.</li> 
</ol> 
<h3><strong>Add a Scene</strong></h3> 
<ol> 
 <li>Navigate to <strong>Workspace→Scenes</strong></li> 
 <li>Provide a Scene name <strong>Mixers</strong> and create the scene</li> 
 <li>Within the scene editor, click on plus <strong>“+”</strong> sign to add a 3D model<img alt="" class="alignnone size-large wp-image-7551" height="573" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/IoT_Twinmaker_2-1024x573.png" width="1024" /></li> 
 <li>Rename the model from <strong>CookieFactoryMixer</strong> to <strong>Mixer_A</strong></li> 
 <li>Click the plus sign again and select <strong>Add light</strong>. Feel free to adjust angle of the light and intensity<img alt="" class="alignnone size-large wp-image-7552" height="661" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/IoT_Twinmaker_3-1024x661.png" width="1024" /></li> 
 <li>Select <strong>Mixer_A</strong> from the left and click the plus sign to select <strong>Add model shader</strong></li> 
 <li>On the right, select Mixer_A for the entity_id, SiteWise for the ComponentName, State for the PropertyName, and sampleTimeSeriesColorRule for the Rule Id<img alt="" class="alignnone size-large wp-image-7553" height="717" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/IoT_Twinmaker_4-1024x717.png" width="1024" /></li> 
 <li>Select the <strong>Rules</strong> tab</li> 
 <li>Select the existing rule <strong>sampleTimeSeriesColorRule</strong></li> 
 <li>Change the 3 expressions as seen below and adjust colors to your preference. Proceed to visualize this scene in Grafana<br /> <img alt="" class="alignnone wp-image-7554" height="618" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/IoT_Twinmaker_5.png" width="308" /></li> 
</ol> 
<h2><strong>Visualize your Twin in Grafana</strong></h2> 
<p>AWS IoT TwinMaker supports Grafana integration through an application plugin. The AWS IoT TwinMaker plugin provides custom panels, dashboard templates, and a data source to connect to your digital twin.</p> 
<h3><strong>Setup Managed Grafana Workspace</strong></h3> 
<ol> 
 <li>Navigate to <a href="https://console.aws.amazon.com/grafana/home">Amazon Managed Grafana</a></li> 
 <li>Complete the <a href="https://docs.aws.amazon.com/grafana/latest/userguide/getting-started-with-AMG.html">Getting Started</a> guide to setup a workspace with SSO authentication</li> 
 <li>Use this inline policy (<a href="https://iot-blog-files.s3.amazonaws.com/edge-to-twin/inline_policy.json">inline_policy.json</a>) to provide Grafana with permission to call APIs for AWS IoT SiteWise, AWS IoT TwinMaker, and Amazon S3. Replace the {account id} with your AWS account id and the Universally Unique Identifier (UUID) for both the Asset Model and Asset in Sitewise<img alt="" class="alignnone size-large wp-image-7555" height="557" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/grafana_IAM_role-1024x557.png" width="1024" /></li> 
</ol> 
<h3><strong>Setup AWS IoT TwinMaker Data source in Grafana</strong></h3> 
<ol> 
 <li>Click on the workspace Grafana URL</li> 
 <li>In Grafana, setup your data source<img alt="" class="alignnone size-full wp-image-7556" height="686" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/grafana_1.png" width="742" /></li> 
 <li>Click <strong>Add data source</strong></li> 
 <li>Select <strong>AWS IoT TwinMaker</strong></li> 
 <li>Select the <strong>MixerWorkspace</strong> click <strong><strong>Save &amp; test</strong></strong><img alt="" class="alignnone size-full wp-image-7557" height="939" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/grafana_2.png" width="957" /></li> 
</ol> 
<h3><strong>Create an AWS IoT TwinMaker dashboard</strong></h3> 
<ol> 
 <li>Create a new dashboard in Grafana.</li> 
 <li>In a new Panel, select <strong>AWS IoT TwinMaker Scene Viewer</strong></li> 
 <li>Select the Scene <strong>Mixers</strong></li> 
 <li>In order to animate the shading of colors configured in the scene rules, you will need to add a query for the mixer state property value history as shown below. Animations will not work without this step</li> 
 <li>That’s it! Feel free to create other panels to view data trends<img alt="" class="alignnone size-large wp-image-7558" height="567" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/06/grafana_3-1024x567.png" width="1024" /><br /> Note: In Grafana, you can take advantage of creating Variables in order to allow the dynamic switching between Entities you select within the dashboard.</li> 
</ol> 
<h2><strong>Clean Up</strong></h2> 
<p>Be sure to clean up the work in this blog to avoid charges. Delete the following resources when finished in this order</p> 
<ol> 
 <li><a href="https://console.aws.amazon.com/grafana/home">AWS Managed Grafana</a> Workspace</li> 
 <li><a href="https://console.aws.amazon.com/iottwinmaker/home">AWS IoT TwinMaker</a> Scene, Resources, Entities, and Workspace</li> 
 <li><a href="https://console.aws.amazon.com/iotsitewise">AWS IoT SiteWise</a> Gateway, Asset, and Asset Model</li> 
 <li><a href="https://console.aws.amazon.com/greengrass">AWS IoT Greengrass</a> Core device under menu Greengrass→Core devices</li> 
 <li>Core Device Thing in <a href="https://console.aws.amazon.com/iot">AWS IoT Core</a> under the menu Manage-&gt;Things</li> 
 <li>Terminate the <a href="https://console.aws.amazon.com/ec2">Amazon EC2</a> Instance</li> 
</ol> 
<h2><strong>Conclusion</strong></h2> 
<p>In this blog, you worked from Edge to Twin, creating a connection to an OPC UA server to collect industrial data through AWS IoT SiteWise on AWS IoT Greengrass. You streamed this data to AWS IoT SiteWise in the cloud and accessed this data from AWS IoT TwinMaker to create a Twin entity of your mixer that you later visualized in Amazon Managed Grafana. This was a simplified walkthrough to connect one entity and data tag. In common practice, this can be scaled out to thousands of entities and data sources allowing you to access streams of data ingested for telemetry, video, machine learning inference results for predictive maintenance, manufacturing execution data, and more. As your Digital Twin grows in capability, you can begin to marry data between various business systems giving your operations insights into your production performance, utilization, and quality of product.</p> 
<p>AWS IoT TwinMaker is now generally available. To learn more about AWS IoT TwinMaker, visit some of the websites below for more details:<br /> <a href="https://aws.amazon.com/iot-twinmaker">https://aws.amazon.com/iot-twinmaker</a><br /> <a href="https://aws.amazon.com/blogs/aws/aws-iot-twinmaker-is-now-generally-available/">https://aws.amazon.com/blogs/aws/aws-iot-twinmaker-is-now-generally-available/</a></p> 
<h2>About the author</h2> 
<p><img alt="" class="alignleft wp-image-7766 size-full" height="150" src="https://d2908q01vomqb2.cloudfront.net/f6e1126cedebf23e1463aee73f9df08783640400/2022/05/12/azer_150px.png" width="150" /><a href="https://www.linkedin.com/in/chrisazer/">Chris Azer</a> is a Principal IoT Specialist Solutions Architect helping customers with their digital twin initiatives. Chris has worked in various roles at AWS since 2017 supporting partners and customers with architecting IoT solutions. This includes a broad set of use cases covering the DoD, Manufacturing, State and Local Government, Federal and Civilian, Smart Cities, Partners, and others. His career in Industrial Automation dates back to 2004 where he continues to assist enterprises today with their smart manufacturing journey.</p>
