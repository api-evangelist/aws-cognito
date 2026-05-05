---
title: "How to use WhatsApp to send Amazon Cognito notification messages"
url: "https://aws.amazon.com/blogs/security/how-to-use-whatsapp-to-send-amazon-cognito-notification-messages/"
date: "Mon, 13 May 2024 13:29:24 +0000"
author: "Nideesh K T"
feed_url: "https://aws.amazon.com/blogs/security/tag/amazon-cognito/feed/"
---
<p>While traditional channels like email and SMS remain important, businesses are increasingly exploring alternative messaging services to reach their customers more effectively. In recent years, WhatsApp has emerged as a simple and effective way to engage with users. According to <a href="https://www.statista.com/statistics/258749/most-popular-global-mobile-messenger-apps/" rel="noopener" target="_blank">statista</a>, as of 2024, WhatsApp is the most popular mobile messenger app worldwide and has reached over two billion monthly active users in January 2024.</p> 
<p><a href="https://aws.amazon.com/cognito/" rel="noopener" target="_blank">Amazon Cognito</a> lets you add user sign-up and authentication to your mobile and web applications. Among many other features, Cognito provides a <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-custom-sms-sender.html" rel="noopener" target="_blank">custom SMS sender AWS Lambda trigger</a> for using third-party providers to send notifications. In this post, we’ll be using WhatsApp as the third-party provider to send verification codes or multi-factor authentication (MFA) codes instead of SMS during Cognito user pool sign up.</p> 
<blockquote>
 <p><strong>Note</strong>: WhatsApp is a third-party service subject to additional terms and charges. <a href="https://aws.amazon.com/aws" rel="noopener" target="_blank">Amazon Web Services (AWS)</a> isn’t responsible for third-party services that you use to send messages with a custom SMS sender in Amazon Cognito.</p>
</blockquote> 
<h2>Overview</h2> 
<p>By default, Amazon Cognito uses <a href="https://aws.amazon.com/sns/" rel="noopener" target="_blank">Amazon Simple Notification Service (Amazon SNS</a>) for delivery of SMS text messages. Cognito also supports custom triggers that will allow you to invoke an <a href="https://aws.amazon.com/lambda/" rel="noopener" target="_blank">AWS Lambda</a> function to support additional providers such as WhatsApp.</p> 
<p>The architecture shown in Figure 1 depicts how to use a custom SMS sender trigger and WhatsApp to send notifications. The steps are as follows:</p> 
<ol> 
 <li>A user signs up to an Amazon Cognito user pool.</li> 
 <li>Cognito invokes the custom SMS sender Lambda function and sends the user’s attributes, including the phone number and a one-time code to the Lambda function. This one-time code is encrypted using a custom symmetric encryption <a href="https://aws.amazon.com/kms/" rel="noopener" target="_blank">AWS Key Management Service (AWS KMS)</a> key that you create.</li> 
 <li>The Lambda function decrypts the one-time code using a Decrypt API call to your AWS KMS key.</li> 
 <li>The Lambda function then obtains the WhatsApp access token from <a href="https://aws.amazon.com/secrets-manager/" rel="noopener" target="_blank">AWS Secrets Manager</a>. The WhatsApp access token needs to be generated through Meta Business Settings (which are covered in the next section) and added to Secrets Manager. Lambda also parses the phone number, user attributes, and encrypted secrets.</li> 
 <li>Lambda sends a POST API call to the WhatsApp API and WhatsApp delivers the verification code to the user as a message. The user can then use the verification code to verify their contact information and <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/signing-up-users-in-your-app.html#allowing-users-to-sign-up-and-confirm-themselves" rel="noopener" target="_blank">confirm the sign-up</a>.</li> 
</ol> 
<p style="line-height: 1.25em;"></p>
<div class="wp-caption aligncenter" id="attachment_34119" style="width: 685px;">
 <img alt="Figure 1: Custom SMS sender trigger flow" class="size-full wp-image-34119" height="422" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/08/img1_v3.png" style="border: 1px solid #bebebe;" width="675" />
 <p class="wp-caption-text" id="caption-attachment-34119">Figure 1: Custom SMS sender trigger flow</p>
</div>
<p></p> 
<h3>Prerequisites</h3> 
<ul> 
 <li><a href="https://portal.aws.amazon.com/gp/aws/developer/registration/index.html" rel="noopener" target="_blank">Create an AWS account</a> if you don’t already have one and sign in. The <a href="https://aws.amazon.com/iam" rel="noopener" target="_blank">AWS Identity and Access Management (IAM)</a> role that you use must have sufficient permissions to make the necessary AWS service calls and manage AWS resources such as creating and updating Lambda functions, Amazon Cognito user pools, Secrets Manager, AWS KMS keys, and IAM roles.</li> 
 <li>A Meta (Facebook) developer account. For more details go to the <a href="https://developers.facebook.com/" rel="noopener" target="_blank">Meta for Developers console</a>.</li> 
 <li><a href="https://git-scm.com/book/en/v2/Getting-Started-Installing-Git" rel="noopener" target="_blank">Git</a> installed.</li> 
 <li><a href="https://docs.aws.amazon.com/cdk/latest/guide/cli.html" rel="noopener" target="_blank">AWS Cloud Development Kit (AWS CDK) Toolkit</a> installed and configured.</li> 
 <li><a href="https://nodejs.org/en/download/" rel="noopener" target="_blank">Node.js with NPM</a> installed.</li> 
 <li><a href="https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-docker.html" rel="noopener" target="_blank">Docker</a> installed and running.</li> 
</ul> 
<h2>Implementation</h2> 
<p>In the next steps, we look at how to create a Meta app, create a new system user, get the WhatsApp access token and create the template to send the WhatsApp token.</p> 
<h3>Create and configure an app for WhatsApp communication</h3> 
<p>To get started, create a Meta app with WhatsApp added to it, along with the customer phone number that will be used to test.</p> 
<h4>To create and configure an app</h4> 
<ol> 
 <li>Open the <a href="https://developers.facebook.com/" rel="noopener" target="_blank">Meta for Developers</a> console, choose <strong>My Apps</strong> and then choose <strong>Create App</strong> (or choose an existing Business type app and skip to step 4).</li> 
 <li>Select <strong>Other</strong> choose <strong>Next</strong> and then select<strong> Business</strong> as the app type and choose <strong>Next</strong>.</li> 
 <li>Enter an <strong>App name</strong>, <strong>App contact email</strong>, choose whether or not to attach a <strong>Business portfolio</strong> and choose <strong>Create app</strong>.</li> 
 <li>Open the app <strong>Dashboard</strong> and in the <strong>Add product to your app</strong> section, under <strong>WhatsApp</strong>, choose <strong>Set up</strong>.</li> 
 <li>Create or select an existing <strong>Meta business portfolio</strong> and choose <strong>Continue</strong>.</li> 
 <li>In the left navigation pane, under <strong>WhatsApp</strong>,<strong> </strong>choose<strong> API Setup</strong>.</li> 
 <li>Under <strong>Send and receive messages</strong>, take a note of the<strong> Phone number ID</strong>, which will be needed in the AWS CDK template later.</li> 
 <li>Under <strong>To</strong>, add the customer phone number you want to use for testing. Follow the instructions to add and verify the phone number.</li> 
</ol> 
<blockquote>
 <p><strong>Note</strong>: You must have WhatsApp registered with the number and the WhatsApp client installed on your mobile device.</p>
</blockquote> 
<h3>Create a user for accessing WhatsApp</h3> 
<p>Create a system user in Meta’s Business Manager and assign it to the app created in the previous step. The access tokens generated for this user will be used to make the WhatsApp API calls.</p> 
<h4>To create a user</h4> 
<ol> 
 <li>Open Meta’s <a href="https://business.facebook.com/settings/" rel="noopener" target="_blank">Business Manager </a>and select the business you created or associated your application with earlier from the dropdown menu under <strong>Business settings</strong>.</li> 
 <li>Under <strong>Users</strong>, select <strong>System users</strong> and then choose <strong>Add</strong> to create a new system user.</li> 
 <li>Enter a name for the <strong>System Username</strong> and set their role as <strong>Admin</strong> and choose <strong>Create system user</strong>.</li> 
 <li>Choose <strong>Assign assets</strong>.</li> 
 <li>From the <strong>Select asset type</strong> list, select <strong>Apps</strong>. Under <strong>Select assets</strong>, select your WhatsApp application’s name. Under <strong>Partial access</strong>, turn on the <strong>Test app </strong>option for the user. Choose <strong>Save Changes</strong> and then choose <strong>Done</strong>.</li> 
 <li>Choose <strong>Generate New Token</strong>, select the WhatsApp application created earlier, and leave the default<strong> 60 days</strong> as the token expiration. Under <strong>Permissions</strong> select <strong>WhatsApp_business_messaging</strong> and <strong>WhatsApp_business_management</strong> and choose <strong>Generate Token</strong> at the bottom.</li> 
 <li>Copy and save your access token. You will need this for the AWS CDK template later. Choose <strong>OK</strong>. For more details on creating the access token, see <a href="https://developers.facebook.com/docs/whatsapp/business-management-api/get-started/" rel="noopener" target="_blank">WhatsApp’s Business Management API Get Started guide</a>.</li> 
</ol> 
<h3>Create a template in WhatsApp</h3> 
<p>Create a template for the verification messages that will be sent by WhatsApp.</p> 
<h4>To create a template</h4> 
<ol> 
 <li>Open Meta’s <a href="https://business.facebook.com/wa/manage/home/" rel="noopener" target="_blank">WhatsApp Manager</a>.</li> 
 <li>On the left icon pane, under <strong>Account tools</strong>, choose <strong>Message template</strong> and then choose <strong>Create Template</strong>.</li> 
 <li>Select <strong>Authentication</strong> as the category.</li> 
 <li>For the <strong>Name</strong>, enter <span style="font-family: courier;">otp_message</span>.</li> 
 <li>For <strong>Languages</strong>, enter <span style="font-family: courier;">English</span>.</li> 
 <li>Choose <strong>Continue</strong>.</li> 
 <li>In the next screen, select <strong>Copy code</strong> and choose <strong>Submit</strong>.</li> 
</ol> 
<blockquote>
 <p><strong>Note</strong>: It’s possible that Meta might change the process or the UI. See the Meta documentation for specific details.</p>
</blockquote> 
<p>For more information on WhatsApp templates, see <a href="https://developers.facebook.com/docs/whatsapp/business-management-api/message-templates" rel="noopener" target="_blank">Create and Manage Templates</a>.</p> 
<h3>Create a Secrets Manager secret</h3> 
<p>Use the Secrets Manager console to create a Secrets Manager secret and set the secret to the WhatsApp access token.</p> 
<h4>To create a secret</h4> 
<ol> 
 <li>Open the AWS Management Console and go to Secrets Manager. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34098" style="width: 750px;">
   <img alt="Figure 2: Open the Secrets Manager console" class="size-full wp-image-34098" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/07/img2-1.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-34098">Figure 2: Open the Secrets Manager console</p>
  </div><p></p> </li> 
 <li>Choose <strong>Store a new secret</strong>. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34099" style="width: 750px;">
   <img alt="Figure 3: Store a new secret" class="size-full wp-image-34099" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/07/img3-1.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-34099">Figure 3: Store a new secret</p>
  </div><p></p> </li> 
 <li>Under <strong>Choose a secret type</strong>, choose <strong>Other type of secret</strong> and under <strong>Key/value pairs</strong>, select the <strong>Plaintext</strong> tab and enter <span style="font-family: courier;">Bearer</span> followed by the WhatsApp access token (<span style="font-family: courier;">Bearer</span> <span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;WhatsApp access token&gt;</span>). <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34100" style="width: 750px;">
   <img alt="Figure 4: Add the secret" class="size-full wp-image-34100" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/07/img4-1.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-34100">Figure 4: Add the secret</p>
  </div><p></p> </li> 
 <li>For the encryption key, you can use either the AWS KMS key that Secrets Manager creates or a customer managed AWS KMS key that you create and then choose <strong>Next</strong>.</li> 
 <li>Provide the secret name as the <strong>WhatsAppAccessToken</strong>, choose <strong>Next</strong>, and then choose <strong>Store</strong> to create the secret.</li> 
 <li>Note the secret Amazon Resource Name (ARN) to use in later steps.</li> 
</ol> 
<h3>Deploy the solution</h3> 
<p>In this section, you clone the GitHub repository and deploy the stack to create the resources in your account.</p> 
<h4>To clone the repository</h4> 
<ol> 
 <li>Create a new directory, navigate to that directory in a terminal and use the following command to clone the GitHub repository that has the Lambda and AWS CDK code: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">git clone <a href="https://github.com/aws-samples/amazon-cognito-whatsapp-otp" rel="noopener" target="_blank">https://github.com/aws-samples/amazon-cognito-whatsapp-otp</a></code></pre> 
  </div> </li> 
 <li>Change directory to the pattern directory: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">cd amazon-cognito-whatsapp-otp</code></pre> 
  </div> </li> 
</ol> 
<h4>To deploy the stack</h4> 
<ol> 
 <li>Configure the phone number ID obtained from WhatsApp, the secret name, secret ARN, and the <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html#cognito-user-pools-features" rel="noopener" target="_blank">Amazon Cognito user pool self-service sign-up</a> option in the <span style="font-family: courier;">constants.ts</span> file. <p>Open the <span style="font-family: courier;">lib/constants.ts</span> file and edit the fields. The <span style="font-family: courier;">SELF_SIGNUP</span> value must be set to <span style="font-family: courier;">true</span> for the purpose of this proof of concept. The <span style="font-family: courier;">SELF_SIGNUP</span> value represents the Boolean value for the Amazon Cognito user pool sign-up option, which when set to true allows public users to sign up.</p> 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">export const PHONE_NUMBER_ID = '<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;phone number ID&gt;</span>'; 
export const SECRET_NAME = '<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;WhatsAppAccessToken&gt;</span>'; 
export const SECRET_ARN = 'arn:aws:secretsmanager:<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;AWSRegion&gt;</span>:<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;phone number ID&gt;</span>:secret:<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;WhatsAppAccessToken&gt;</span>'; 
export const SELF_SIGNUP = <span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;true&gt;</span>;</code></pre> 
  </div> 
  <blockquote>
   <p><strong>Warning</strong>: If you activate user sign-up (enable self-registration) in your user pool, anyone on the internet can sign up for an account and sign in to your applications.</p>
  </blockquote> </li> 
 <li>Install the AWS CDK required dependencies by running the following command: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">npm install</code></pre> 
  </div> </li> 
 <li>This project uses typescript as the client language for AWS CDK. Run the following command to compile typescript to JavaScript: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">npm run build</code></pre> 
  </div> </li> 
 <li>From the command line, configure AWS CDK (if you have not already done so): 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">cdk bootstrap <span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;account number&gt;</span>/<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;AWS Region&gt;</span></code></pre> 
  </div> </li> 
 <li><a href="https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-docker.html" rel="noopener" target="_blank">Install and run Docker</a>. We’re using the <a href="https://docs.aws.amazon.com/cdk/api/v2/docs/aws-lambda-python-alpha-readme.html" rel="noopener" target="_blank">aws-lambda-python-alpha</a> package in the AWS CDK code to build the Lambda deployment package. The deployment package installs the required modules in a Lambda compatible Docker container.</li> 
 <li>Deploy the stack: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">cdk synth
cdk deploy --all</code></pre> 
  </div> </li> 
</ol> 
<h2>Test the solution</h2> 
<p>Now that you’ve completed implementation, it’s time to test the solution by signing up a user on Amazon Cognito and confirming that the Lambda function is invoked and sends the verification code.</p> 
<h4>To test the solution</h4> 
<ol> 
 <li>Open <a href="https://aws.amazon.com/cloudformation" rel="noopener" target="_blank">AWS CloudFormation</a> console.</li> 
 <li>Select the <strong>WhatsappOtpStack</strong> that was deployed through AWS CDK.</li> 
 <li>On the <strong>Outputs</strong> tab, copy the value of <strong>cognitocustomotpsenderclientappid</strong>.</li> 
 <li>Run the following <a href="https://aws.amazon.com/cli" rel="noopener" target="_blank">AWS Command Line Interface (AWS CLI)</a> command, replacing the client ID with the output of cognitocustomotpsenderclientappid, username, password, email address, name, phone number, and AWS Region to sign up a new Amazon Cognito user. 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">aws cognito-idp sign-up --client-id <span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;cognitocustomsmssenderclientappid&gt;</span> --username <span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;TestUserPhoneNumber&gt;</span> --password <span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;Password&gt;</span> --user-attributes Name="email",Value="<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;TestUserEmail&gt;</span>" Name="name",Value="<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;TestUserName&gt;</span>" Name="phone_number",Value="<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;TestPhoneNumber&gt;</span>" --region <span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;AWS Region&gt;</span></code></pre> 
  </div> <p>Example:</p> 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">aws cognito-idp sign-up --client-id xxxxxxxxxxxxxx --username +12065550100  --password Test@654321 --user-attributes Name="email",Value="jane@example.com" Name="name",Value="Jane" Name="phone_number",Value=”+12065550100" --region us-east-1</code></pre> 
  </div> 
  <blockquote>
   <p><strong>Note</strong>: Password requirements are a minimum length of eight characters with at least one number, one lowercase letter, and one special character.</p>
  </blockquote> </li> 
</ol> 
<p>The new user should receive a message on WhatsApp with a verification code that they can use to complete their sign-up.</p> 
<h2>Cleanup</h2> 
<ol> 
 <li>Run the following command to delete the resources that were created. It might take a few minutes for the CloudFormation stack to be deleted. 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">cdk destroy --all</code></pre> 
  </div> </li> 
 <li>Delete the secret WhatsAppAccessToken that was created from the Secrets Manager console.</li> 
</ol> 
<h2>Conclusion</h2> 
<p>In this post, we showed you how to use an alternative messaging platform such as WhatsApp to send notification messages from Amazon Cognito. This functionality is enabled through the <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-custom-sms-sender.html" rel="noopener" target="_blank">Amazon Cognito custom SMS sender trigger</a>, which invokes a Lambda function that has the custom code to send messages through the WhatsApp API. You can use the same method to use other third-party providers to send messages.</p> 
<p>If you have feedback about this post, submit comments in the&nbsp;Comments&nbsp;section below. If you have questions about this post, start a new thread on the&nbsp;<a href="https://repost.aws/tags/TAkhAE7QaGSoKZwd6utGhGDA/amazon-cognito" rel="noopener" target="_blank">Amazon Cognito re:Post</a>&nbsp;or&nbsp;<a href="https://console.aws.amazon.com/support/home" rel="noopener" target="_blank">contact AWS Support</a>.</p> 
<p><strong>Want more AWS Security news? Follow us on <a href="https://twitter.com/AWSsecurityinfo" rel="noopener noreferrer" target="_blank" title="Twitter">X</a>.</strong></p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Nideesh K T" class="aligncenter size-full wp-image-34095" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/07/nideesht.png" width="120" /> 
  </div> 
  <p class="lb-h4">Nideesh K T</p> 
  <p>Nideesh is an experienced IT professional with expertise in cloud computing and technical support. Nideesh has been working in the technology industry for 8 years. In his current role as a Sr. Cloud Support Engineer, Nideesh provides technical assistance and troubleshooting for cloud infrastructure issues. Outside of work, Nideesh enjoys staying active by going to the gym, playing sports, and spending time outdoors.</p> 
  <p></p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Reethi Joseph" class="aligncenter size-full wp-image-34096" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/07/reethi.png" width="120" /> 
  </div> 
  <p class="lb-h4">Reethi Joseph</p> 
  <p>Reethi is a Sr. Cloud Support Engineer at AWS with 7 years of experience specializing in serverless technologies. In her role, she helps customers architect and build solutions using AWS services. When not delving into the world of servers and generative AI, she spends her time trying to perfect her swimming strokes, traveling, trying new baking recipes, gardening, and watching movies.</p> 
  <p></p>
 </div> 
</footer>
