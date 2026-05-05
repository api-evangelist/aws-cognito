---
title: "Reduce risks of user sign-up fraud and SMS pumping with Amazon Cognito user pools"
url: "https://aws.amazon.com/blogs/security/reduce-risks-of-user-sign-up-fraud-and-sms-pumping-with-amazon-cognito-user-pools/"
date: "Fri, 13 Sep 2024 13:46:19 +0000"
author: "Edward Sun"
feed_url: "https://aws.amazon.com/blogs/security/tag/amazon-cognito/feed/"
---
<blockquote>
 <p><strong>September 10, 2025:</strong> We’ve updated this post to reflect changes in suggested mitigation approaches.</p>
</blockquote> 
<blockquote>
 <p><strong>December 16, 2024:</strong> We’ve updated this post to reflect changes in suggested mitigation approaches.</p>
</blockquote> 
<hr /> 
<p>If you have a customer facing application, you might want to enable self-service sign-up, which allows potential customers on the internet to create an account and gain access to your applications. While it’s necessary to allow valid users to sign up to your application, self-service options can open the door to unintended use or sign-ups. Bad actors might leverage the user sign-up process for unintended purposes, launching large-scale distributed denial of service (DDoS) attacks to disrupt access for legitimate users or committing a form of telecommunications fraud known as SMS pumping. SMS pumping occurs when bad actors use bots and other measures to coerce unsuspecting services, often those generate a one-time codes or promotional links, into sending a large volume of SMS messages and unexpected SMS charges.</p> 
<p><a href="https://aws.amazon.com/cognito/" rel="noopener" target="_blank">Amazon Cognito</a> is a managed OpenID Connect (OIDC) identity provider (IdP) that you can use to add self-service sign-up, sign-in, and control access features to your web and mobile applications. AWS customers who use Cognito might encounter SMS pumping if SMS functions are enabled to send SMS messages, for example, perform user phone number verification during the registration process, to facilitate SMS multi-factor authentication (MFA) flows, or to support account recovery using SMS. In this blog post, we explore how SMS pumping may be perpetrated and options to reduce risks, including blocking unexpected user registration, detecting anomalies, and responding to risk events with your Cognito user pool.</p> 
<h2>Cognito user sign-up process</h2> 
<p>After a user has signed up in your application with an Amazon Cognito user pool, their account is placed in the <em>Registered (unconfirmed)</em> state in your user pool and the user won’t be able to sign in yet. You can use the Cognito-assisted verification and confirmation process to verify user-provided attributes (such as email or phone number) and then confirm the user’s status. This verified attribute is also used for MFA and account recovery purposes. If you choose to verify the user’s phone number, Cognito sends SMS messages with a one-time password (OTP). After a user has provided the correct OTP, their email or phone number is marked as verified and the user can sign in to your application.</p> 
<div class="wp-caption aligncenter" id="attachment_35579" style="width: 790px;">
 <img alt="Figure 1: Amazon Cognito sign-up process" class="size-full wp-image-35579" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/08/27/img1_v2-1.png" style="border: 1px solid #bebebe;" width="780" />
 <p class="wp-caption-text" id="caption-attachment-35579">Figure 1: Amazon Cognito sign-up process</p>
</div> 
<p>If the sign-up process isn’t protected, bad actors can create scripts or deploy bots to sign up a large number of accounts, resulting in a significant volume of SMS messages sent in a short period of time. We dive deep into detection, prevention, and remediation mechanisms and strategies that you can apply to help protect against SMS pumping based on your use case.</p> 
<h2>Detect SMS pumping</h2> 
<p>When you’re considering the various options for mitigations, it’s important to set up detection mechanisms to identify SMS pumping as it arises. In this section, we show you how to use <a href="https://aws.amazon.com/cloudtrail/" rel="noopener" target="_blank">AWS CloudTrail</a> and <a href="https://aws.amazon.com/cloudwatch/" rel="noopener" target="_blank">Amazon CloudWatch</a> to monitor your Amazon Cognito user pool and detect anomalies that could lead to SMS pumping. Note that building a detection mechanism based on anomalies requires knowing your average or baseline traffic and the difference in metrics that represent regular activity and metrics that can indicate unauthorized or unintended activity.</p> 
<h3>Service quotas dashboard and CloudWatch alarms</h3> 
<p>Bad actors may attempt to use either the sign-up confirmation or the reset password functionality of Amazon Cognito. As shown in Figure 1, when a new user signs up to your Cognito user pool, the <a href="https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_SignUp.html" rel="noopener" target="_blank">SignUp</a> API operation is invoked. When the user provides the OTP confirmation code, the <a href="https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_ConfirmSignUp.html" rel="noopener" target="_blank">ConfirmSignUp</a> API operation is invoked. The call rate of both APIs is tracked collectively under <em>Rate of UserCreation</em> requests under the Amazon Cognito service in the <a href="https://console.aws.amazon.com/servicequotas/home?region=us-east-1#!/dashboard" rel="noopener" target="_blank">service quotas dashboard</a>.</p> 
<p>You can <a href="https://docs.aws.amazon.com/servicequotas/latest/userguide/configure-cloudwatch.html" rel="noopener" target="_blank">set up Amazon CloudWatch alarms</a> to monitor and issue notifications when you’re close to a quota value threshold. These alarms could be an early indication of a sudden usage increase, and you can use them to triage potential incidents. You can also use CloudWatch alarms to monitor for an excessive amount of text messages sent in a short period of time. For example, see <a href="https://docs.aws.amazon.com/sms-voice/latest/userguide/monitoring-sms-cw.html" rel="noopener" target="_blank">Create CloudWatch Alarms for AWS End User Messaging SMS metrics</a> to learn how to create an alarm if more than 1,000 text messages are sent in 1 hour. </p> 
<p>Additionally, when your services are sending SMS messages, those transactions count towards the <a href="https://aws.amazon.com/sns" rel="noopener" target="_blank">Amazon Simple Notification Service (Amazon SNS)</a> service quota. You can set up alarms to monitor the <em>Transactional SMS Message Delivery Rate per Second quota</em> and the <em>SMS Message Spending in USD quota</em>.</p> 
<h3>CloudTrail event history</h3> 
<p>When bad actors plan SMS pumping, they are likely attempting to trick you to send as many SMS messages as possible rather than completing the user confirmation process. Under the context of a user sign-up event, you might notice in the CloudTrail event history that there are more <a href="https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_SignUp.html" rel="noopener" target="_blank">SignUp</a> and <a href="https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_ResendConfirmationCode.html" rel="noopener" target="_blank">ResendConfirmationCode</a> events—which send out SMS messages—than <a href="https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_ConfirmSignUp.html" rel="noopener" target="_blank">ConfirmSignUp</a> operations, indicating a user has initiated but has not completed the sign-up process. You can use <a href="https://aws.amazon.com/athena" rel="noopener" target="_blank">Amazon Athena</a> or <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/analyzingcteventscwinsight.html" rel="noopener" target="_blank">CloudWatch Logs Insights to search and analyze your Amazon Cognito CloudTrail events</a> and identify if there’s a significant reduction in completion of the user sign-up process.</p> 
<div class="wp-caption aligncenter" id="attachment_35573" style="width: 790px;">
 <img alt="Figure 2: SignUp API logged in CloudTrail event history" class="size-full wp-image-35573" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/08/27/img2-5.png" style="border: 1px solid #bebebe;" width="780" />
 <p class="wp-caption-text" id="caption-attachment-35573">Figure 2: SignUp API logged in CloudTrail event history</p>
</div> 
<p>Similarly, you can apply this observability towards the user password reset flow by analyzing the <a href="https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_ForgotPassword.html" rel="noopener" target="_blank">ForgotPassword</a> API and <a href="https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_ConfirmForgotPassword.html" rel="noopener" target="_blank">ConfirmForgotPassword</a> API operations for deviations. Note that the slight deviations in user completion flow in the CloudTrail event history alone might not be an indication of unauthorized activity; however, a substantial deviation above the regular baseline might be a signal of unintended use.</p> 
<h3>Monitor excessive billing</h3> 
<p>Another opportunity for detecting and identifying unauthorized Amazon Cognito activity is by using <a href="https://aws.amazon.com/aws-cost-management/aws-cost-explorer/" rel="noopener" target="_blank">AWS Cost Explorer</a>. You can use this interface to visualize, understand, and manage your AWS costs and usage over time, which might assist by highlighting the source of excessive billing in your AWS account. Be aware that charges in your account can take up to 24 hours to display, so although this method can help provide some assistance in identifying SMS pumping activity, you should use it only as a supplement to other detection methods.</p> 
<p><strong>To use Cost Explorer:</strong></p> 
<ol> 
 <li>Open the <a href="https://console.aws.amazon.com/console/home" rel="noopener" target="_blank">AWS Management Console</a> and choose <strong>Billing and Cost Management</strong>.</li> 
 <li>In the navigation pane, under <strong>Cost Analysis</strong>, choose <strong>Cost Explorer</strong>.</li> 
 <li>In the <strong>Cost and Usage Report</strong>, under <strong>Report Parameters</strong>, select <strong>Date Range</strong> to include the start and end date of the time period that you want to apply a filter to. In Figure 3, we use an example date range between 2024-07-03 and 2024-07-17.</li> 
 <li>In the same <strong>Report Parameter</strong> area, under <strong>Filters</strong>, for <strong>Service</strong>, select <strong>SNS (Simple Notification Service)</strong>. Because Amazon Cognito uses Amazon SNS for delivery of SMS messages, filtering on SNS can help you identify excessive billing.</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_35574" style="width: 790px;">
 <img alt="Figure 3: Review billing charges by service" class="size-full wp-image-35574" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/08/27/img3-3.png" style="border: 1px solid #bebebe;" width="780" />
 <p class="wp-caption-text" id="caption-attachment-35574">Figure 3: Review billing charges by service</p>
</div> 
<h2>Protect the sign-up flow</h2> 
<p>In this section, we review several strategies to help protect against SMS sign-up frauds and help reduce the amount of SMS messages sent to bad actors.</p> 
<h3>Implement bot mitigation</h3> 
<p>Implementing bot mitigation techniques, such as CAPTCHA, can be effective in helping to prevent simple bots from pumping user creation flows. You can integrate a CAPTCHA framework on your application’s frontend and validate that the client initiating the sign-up request is operated by a human user. If the user has passed the verification, you then pass the CAPTCHA user response token in <code style="color: #000000;">ClientMetadata</code> together with user attributes to an Amazon Cognito <code style="color: #000000;"><a href="https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_SignUp.html" rel="noopener" target="_blank">SignUp</a></code> API call. As part of the sign-up process, Cognito invokes an <a href="https://aws.amazon.com/lambda/" rel="noopener" target="_blank">AWS Lambda</a> function called <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-pre-sign-up.html" rel="noopener" target="_blank">pre sign-up Lambda trigger</a>, which you can use to reject sign-up requests if there isn’t a valid CAPTCHA token presented. This will slow down bots and help reduce unintended account creation in your Cognito user pool.</p> 
<h3>Apply AWS WAF bot control rule group</h3> 
<p>Amazon Cognito customers can <a href="https://aws.amazon.com/blogs/security/protect-your-amazon-cognito-user-pool-with-aws-waf/" rel="noopener noreferrer" target="_blank">protect their user pools with AWS WAF</a>. As an AWS managed rule group, AWS WAF offers a bot control rule group. It is recommended to apply a <em>targeted</em> bot control managed rule group, which includes common protections and adds targeted detection for sophisticated bots that don’t self-identify. For more information about how bot control rule groups work, see <a href="https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-bot.html#aws-managed-rule-groups-bot-using" rel="noopener noreferrer" target="_blank">Considerations for using this rule group</a> and <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-waf.html" rel="noopener noreferrer" target="_blank">Associate an AWS WAF web ACL with a user pool</a>.</p> 
<h3>Apply SMS country-level allow, block, or filter with End User Messaging Protect feature</h3> 
<p>If you use <a href="https://aws.amazon.com/sns/" rel="noopener noreferrer" target="_blank">Amazon Simple Notification Service (Amazon SNS)</a> to send text messages in Amazon Cognito, you can use <a href="https://aws.amazon.com/end-user-messaging/" rel="noopener noreferrer" target="_blank">AWS End User Messaging</a> SMS Protect feature to <a href="https://aws.amazon.com/blogs/messaging-and-targeting/defending-against-sms-pumping-new-aws-features-to-help-combat-artificially-inflated-traffic/" rel="noopener noreferrer" target="_blank">defend against SMS pumping</a>. You can use SMS Protect to set up country-level rules to allow, block, or filter outbound SMS messages.</p> 
<p>To set up a new protect configuration:</p> 
<ol> 
 <li>Open the <a href="https://console.aws.amazon.com/sms-voice/" rel="noopener noreferrer" target="_blank">AWS End User Messaging Console</a>, and choose <strong>SMS</strong>.</li> 
 <li>In the navigation pane, under <strong>Protect</strong>, choose <strong>Protect configurations</strong>.</li> 
 <li>On the <strong>Protect configurations</strong> page, choose <strong>Create protect configuration</strong>.</li> 
 <li>On the <strong>Create protect configuration</strong> page, do the following: 
  <ol type="a"> 
   <li>Under <strong>SMS country rules</strong>, define SMS country rules with <strong>Allow</strong>, <strong>Block</strong>, or <strong>Filter</strong>.</li> 
   <li>Under <strong>Protect configuration associations</strong>, choose <strong>Account default</strong> to apply configuration towards SMS message sent by Cognito.</li> 
   <li>Choose <strong>Create protect configuration</strong>.</li> 
  </ol> </li> 
</ol> 
<p>When setting SMS Protect strategies, consider enabling <strong>block</strong> for countries where you don’t have any customer bases and don’t need to send text messages to and enabling <strong>allow</strong> for countries that are generally not common targets for SMS pumping, such as the United States and Canada. You can consider using <strong>filter</strong> mode for other countries, that might have higher risk of SMS pumping or higher per-message SMS prices, where you have customer bases and need to send text messages for communication.</p> 
<p>For more information about SMS Protect, see <a href="https://docs.aws.amazon.com/sms-voice/latest/userguide/protect.html" rel="noopener noreferrer" target="_blank">Protect in AWS End User Messaging SMS</a>. Note that there’s an additional SMS Protect charge for messages sent under filter mode in addition to SMS fees, while there is no charge if the message is blocked. For more details, see <a href="https://aws.amazon.com/end-user-messaging/pricing/#SMS_Protect" rel="noopener noreferrer" target="_blank">SMS Protect pricing</a>.</p> 
<p>As an example, shown in Figure 4, we created an example configuration rule to block all SMS messages sent to countries in Africa, Antarctica, and South America, while allowing SMS messages to be sent to Asia and North America. For SMS messages sent to Europe, we applied a monitor rule and a mix of monitor and filter rules for countries in Oceania. You can adjust the country rules based on your specific requirements.</p> 
<div class="wp-caption aligncenter" id="attachment_40494" style="width: 1912px;">
 <img alt="Figure 4: Example SMS Protect configuration details" class="size-full wp-image-40494" height="1136" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/11/17/figure-4.png" style="border: 1px solid #bebebe;" width="1902" />
 <p class="wp-caption-text" id="caption-attachment-40494">Figure 4: Example SMS Protect configuration details</p>
</div> 
<p>Note that when using SMS Protect with Amazon Cognito, Cognito doesn’t have visibility into whether a message is filtered or blocked by SMS Protect. Therefore, if user’s sign-up confirmation code is blocked and not delivered, the user won’t be able to use the code to complete the confirmation process, as shown in Figure 1 step 3.3. The user will be created in your Cognito user pool and remain in Unconfirmed status. You can view blocked message metrics in the Amazon SMS console, see <a href="https://docs.aws.amazon.com/sms-voice/latest/userguide/filter-and-monitor-messages-monitor.html" rel="noopener noreferrer" target="_blank">View protect metrics</a> for more information.</p> 
<h3>Validate phone number before user sign-up</h3> 
<p>Another method of mitigation is to identify the bad actor’s phone number early in your application’s sign-up process. You can validate the user provided phone number in the backend to catch incorrectly formatted phone numbers and add logic to help filter out unwanted phone numbers prior to sending text messages. In addition to SMS protect feature, End User Messaging service offers a <a href="https://docs.aws.amazon.com/pinpoint/latest/apireference/phone-number-validate.html" rel="noopener" target="_blank">Phone Number Validate</a> feature that can help you determine if a user-provided phone number is valid, determine phone number type (such as mobile, landline, or VoIP), and identify the country and service provider the phone number is associated with. The returned phone number metadata can be used to decide whether the user will continue the sign-up process and send an SMS message to that user. Note that there’s an additional charge for using the phone number validation service. For more information, see <a href="https://aws.amazon.com/end-user-messaging/pricing/#Phone_number_validate" rel="noopener" target="_blank">phone number validate pricing</a>.</p> 
<p>To build this validation check into the Amazon Cognito sign-up process, you can customize the <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-pre-sign-up.html" rel="noopener" target="_blank">pre sign-up Lambda trigger</a>, which Cognito uses to invoke your code before allowing users to sign-up and sending out an SMS OTP. The Lambda trigger invokes the Amazon Pinpoint phone number validate API, and based on the validation response, you can build a custom pattern that fits your application to continue or reject the user sign-up. For example, you can reject user sign-ups with VoIP numbers or reject users who provide a phone number that’s associated with countries that you don’t operate in, or even reject certain cellular service providers. After you reject a user sign-up using the Lambda trigger, Cognito will deny the user sign-up request and will not invoke user confirmation flow nor send out an SMS message.</p> 
<p><strong>Example validation command using AWS CLI</strong></p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">aws pinpoint phone-number-validate --number-validate-request PhoneNumber=+155501001234</code></pre> 
</div> 
<p>When you send a request to the Amazon Pinpoint phone number validation service, it returns the following metadata about the phone number. The following example represents a valid mobile phone number data set:</p> 
<pre class="unlimited-height-code"><code class="lang-json">{
    "NumberValidateResponse": {
        "Carrier": "ExampleCorp Mobile",
        "City": "Seattle",
        "CleansedPhoneNumberE164": "+155501001234",
        "CleansedPhoneNumberNational": "55501001234",
        "Country": "United States",
        "CountryCodeIso2": "US",
        "CountryCodeNumeric": "1",
        "OriginalPhoneNumber": "+155501001234",
        "PhoneType": "MOBILE",
        "PhoneTypeCode": 0,
        "Timezone": "America/Seattle",
        "ZipCode": "98109"
    }
}</code></pre> 
<p>Note that <code style="color: #000000;">PhoneType</code> includes type MOBILE, LANDLINE, VOIP, INVALID, or OTHER. INVALID phone numbers don’t include information about the carrier or location associated with the phone number and are unlikely to belong to actual recipients. This helps you decide when to reject user sign-ups and reduces SMS messages to undesired phone numbers. You can see details about other responses in the <a href="https://docs.aws.amazon.com/pinpoint/latest/developerguide/validate-phone-numbers.html#validate-phone-numbers-example-responses" rel="noopener" target="_blank">Amazon Pinpoint developer guide</a>.</p> 
<p><strong>Example pre sign-up Lambda function to block user sign-up except with a valid MOBILE number</strong></p> 
<p>The following pre sign-up Lambda function example invokes the Amazon Pinpoint phone number validation service and rejects user sign-ups unless the validation service returns a valid mobile phone number.</p> 
<pre><code class="lang-js">import { PinpointClient, PhoneNumberValidateCommand } from "@aws-sdk/client-pinpoint"; // ES Modules import

const validatePhoneNumber = async (phoneNumber) =&gt; {
  const pinpoint = new PinpointClient();
  const input = { // PhoneNumberValidateRequest
    NumberValidateRequest: { // NumberValidateRequest
      PhoneNumber: phoneNumber,
    },
  };
  const command = new PhoneNumberValidateCommand(input);
  const response = await pinpoint.send(command);

  return response;
};

const handler = async (event, context, callback) =&gt; {

  const phoneNumber = event.request.userAttributes.phone_number;
  const validationResponse = await validatePhoneNumber(phoneNumber);

  if (validationResponse.NumberValidateResponse.PhoneType != "MOBILE") {
    var error = new Error("Cannot register users without a mobile number");
    // Return error to Amazon Cognito
    callback(error, event);
  }
  // Return to Amazon Cognito
  callback(null, event);
};

export { handler };</code></pre> 
<h3>Reject sign-up requests from a specific area code</h3> 
<p>In an SMS pumping scheme, bad actors often purchase blocks of cell phone numbers from a wireless service provider and use phone numbers with the same area code. If you observe a pattern and identify that these attempts use the same area code, you can modify your pre sign-up Lambda function to reject sign-up requests containing those area code patterns.</p> 
<p><strong>Example pre sign-up Lambda function to block user sign-up based on the area code pattern</strong></p> 
<p>The following pre sign-up Lambda function performs a matching function and rejects user sign-ups if the user provides a phone number with a specific country code and area code that is part of the <code style="color: #000000;">countryAreaCodeBlocked</code> list (in this example, <code style="color: #000000;">+1404</code>, <code style="color: #000000;">+1555</code>, <code style="color: #000000;">+4420</code>, <code style="color: #000000;">+441904</code>, <code style="color: #000000;">+9122</code>, and <code style="color: #000000;">+9133</code>).</p> 
<pre><code class="lang-js">const handler = async (event, context, callback) =&gt; {
  const phoneNumber = event.request.userAttributes.phone_number;
  const normalizeNumber = phoneNumber.slice(1);
  
  // List of country codes and area codes to block
  const countryAreaCodeBlocked = {
    "1": ["404","555"],
    "44": ["20","1904"],
    "91": ["22","33"]
  };
  
  const blockedPrefix = Object.entries(countryAreaCodeBlocked).flatMap(
    ([countryCode, areaCodes]) =&gt; areaCodes.map((areaCode) =&gt; `${countryCode}${areaCode}`)
  );

  const isBlocked = blockedPrefix.some((prefix) =&gt; normalizeNumber.startsWith(prefix));

  if (isBlocked) {
    var error = new Error("Phone Number not allowed");
    // Return error to Amazon Cognito
    callback(error, event);
  } else {
    // Return to Amazon Cognito
    callback(null, event);
  }
};

export { handler };</code></pre> 
<h3>Use a custom user-initiated confirmation flow or alternative OTP delivery method</h3> 
<p>In your user pool configurations, you can opt out of using Amazon Cognito-assisted verification and confirmation to send SMS messages to confirm users. Instead, you can build a custom reverse OTP flow to ask your users to initiate the user confirmation process. For example, instead of automatically sending SMS messages to a user when they sign up, your application can display an OTP and direct the user to initiate the SMS conversation by texting the OTP to your service number. After your application has received the SMS message and confirmed the correct OTP is provided, invoke a service such as a Lambda function to call the <a href="https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_AdminConfirmSignUp.html" rel="noopener" target="_blank">AdminConfirmSignUp</a> administrative API operation to confirm user, then call <a href="https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_AdminUpdateUserAttributes.html" rel="noopener" target="_blank">AdminUpdateUserAttributes</a> to set the <code style="color: #000000;">phone_number_verified</code> attribute as <code style="color: #000000;">true</code> to indicate that the user phone number is verified.</p> 
<p>You can also choose to deliver an OTP using other methods, such as email, especially if your application doesn’t require the user’s phone number. During the user sign-up process, you can configure a <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-custom-sms-sender.html" rel="noopener" target="_blank">custom SMS sender Lambda trigger</a> in Amazon Cognito to send a user verification code through email or another method. Additionally, you can use the <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pool-settings-advanced-security-email-mfa.html" rel="noopener" target="_blank">Cognito email MFA feature</a> to send MFA codes through email.</p> 
<h2>Apply AWS WAF rules as mitigation approaches</h2> 
<p>It’s recommended that you <a href="https://aws.amazon.com/blogs/security/protect-your-amazon-cognito-user-pool-with-aws-waf/" rel="noopener" target="_blank">apply AWS WAF with your Amazon Cognito user pool</a> to help protect against common threats. In this section, we show you a few advanced options using <a href="https://aws.amazon.com/waf" rel="noopener" target="_blank">AWS WAF</a> rules to block or throttle specific bad actor’s traffic when you have observed irregular sign-up attempts and suspect they were part of fraudulent activities.</p> 
<h3>Target a specific bad actor’s IP address</h3> 
<p>When building AWS WAF remediation strategies, you can start by building an IP deny list to block traffic from known malicious IP addresses. This method is straightforward and can be highly effective in mitigating unwanted access. For detailed instructions on how to set up an IP deny list, see <a href="https://docs.aws.amazon.com/waf/latest/developerguide/waf-ip-set-creating.html" rel="noopener" target="_blank">Creating an IP set</a>.</p> 
<h3>Target a specific bad actor’s client fingerprint</h3> 
<p>Another method is to examine an actor’s TLS traffic. If your application UI is hosted using <a href="https://aws.amazon.com/cloudfront/" rel="noopener" target="_blank">Amazon CloudFront</a> or <a href="https://aws.amazon.com/elasticloadbalancing/application-load-balancer/" rel="noopener" target="_blank">Application Load Balancer (ALB)</a>, you can build AWS WAF rules to match the client’s JA3 and JA4 fingerprint for more comprehensive client identification. The JA3 fingerprint is a 32-character MD5 hash derived from the TLS three-way handshake when the client sends a <code style="color: #000000;">ClientHello</code> packet to the server. It serves as a unique identifier for the client’s TLS configuration because various attributes such as TLS version, cipher suites, and extensions are derived to calculate the fingerprint, allowing for the unique detection of clients even when the source IP and other commonly used identification information might have changed. Similarly, a JA4 TLS client fingerprint contains a 36-character long fingerprint of the TLS Client Hello, which is used to initiate a secure connection from clients. The JA4 fingerprint extends this concept by incorporating additional TLS handshake characteristics including Application-Layer Protocol Negotiation (ALPN) details and Server Name Indication (SNI), resulting in more granular client identification that can better differentiate between legitimate users and sophisticated automated threats attempting to mimic normal browser behavior.</p> 
<p>Fraudulent activities, such as SMS pumping, are typically carried out using automated tools and scripts. These tools often have a consistent SSL/TLS handshake pattern, resulting in a unique JA3 fingerprint. By configuring an AWS WAF web ACL rule to match the JA3 fingerprint associated with this traffic, you can identify clients with a high degree of accuracy, even if they change other attributes, such as IP addresses.</p> 
<p>AWS WAF has introduced support for <a href="https://aws.amazon.com/about-aws/whats-new/2025/03/aws-waf-ja4-fingerprinting-aggregation-ja3-ja4-fingerprints-rate-based-rules/" rel="noopener" target="_blank">JA3 and JA4 fingerprint matching</a>, which you can use to identify and differentiate clients based on the way they initiate TLS connections, enabling you to inspect incoming requests for their JA3 or JA4 fingerprints. You can build the remediation strategy by first evaluating AWS WAF logs to extract JA3 and JA4 fingerprints for potential malicious hosts, then proceed with creating rules to block requests where the fingerprint matches the malicious fingerprints associated with previous exploits.</p> 
<p><strong>To configure an AWS WAF web ACL to block using JA3 fingerprint matching for CloudFront resources:</strong></p> 
<ol> 
 <li>Open the AWS WAF console.</li> 
 <li>In the navigation pane, under <strong>AWS WAF</strong>, choose <strong>WAF ACLs</strong>.</li> 
 <li>Choose <strong>Create web ACL</strong>. Under <strong>Web ACL details</strong>, select <strong>Amazon CloudFront distributions</strong>. Under <strong>Associated AWS resources</strong>, select <strong>Add AWS resources</strong>, and select your CloudFront distribution. Choose <strong>Next</strong>.</li> 
 <li>On the <strong>Add rules and rule groups</strong> page, choose <strong>Add rules</strong>, <strong>Add my own rules and rule groups</strong>, and <strong>Rule builder</strong>.</li> 
 <li>In <strong>Rule builder</strong>: 
  <ol> 
   <li>For <strong>If a request</strong>, select <strong>matches the statement</strong>.</li> 
   <li>For <strong>Inspect</strong>, select <strong>JA3 fingerprint</strong> (later you can add one for JA4 fingerprint).</li> 
   <li>For <strong>Match type</strong>, keep <strong>Exactly matches string</strong>.</li> 
   <li>For <strong>String to match</strong>, enter the JA3 fingerprint that you want to block.</li> 
   <li>For <strong>Text transformation</strong>, choose <strong>None</strong>.</li> 
   <li>For <strong>Fallback for missing fingerprint</strong>, select a fallback match status for cases where no JA3 fingerprint is detected. We recommend choosing <strong>No match</strong> to help prevent unintended traffic blocking.</li> 
   <li>If you need to block multiple fingerprints, include each one in the rule and for <strong>If a request</strong> select <strong>matches at least one of the statements (OR)</strong>. <p></p>
    <div class="wp-caption aligncenter" id="attachment_35576" style="width: 710px;">
     <img alt="Figure 4: Creating an AWS WAF statement for a JA3 fingerprint" class="size-full wp-image-35576" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/08/27/img5-2.png" style="border: 1px solid #bebebe;" width="700" />
     <p class="wp-caption-text" id="caption-attachment-35576">Figure 5: Creating an AWS WAF statement for a JA3 fingerprint</p>
    </div></li> 
   <li>Under <strong>Action</strong>, select <strong>Block</strong>, and choose <strong>Add rule</strong>. You can choose other actions such as COUNT or CAPTCHA that suit your use case.</li> 
  </ol> </li> 
 <li>Continue with <strong>Set rule priority</strong> and <strong>Configure metrics</strong>, then choose <strong>Create web ACL</strong>.</li> 
</ol> 
<p>Note that JA3 fingerprints can change over time due to the randomization of TLS <code style="color: #000000;">ClientHello</code> messages by modern browsers. It’s important to dynamically update your web ACL rules or manually review logs to update the JA3 fingerprint search string in your match rule when applicable.</p> 
<h2>Considerations</h2> 
<p>By using pre sign-up Lambda triggers in Amazon Cognito and remediation approaches in AWS WAF, you can help block potential threats by providing mechanisms to filter out malicious traffic. However, it’s essential to continually review the effectiveness of these mechanisms to minimize the risk of blocking legitimate sources and make dynamic adjustments to the rules when you detect new bad actors and patterns.</p> 
<h2>Summary</h2> 
<p>In this blog post, we introduced mechanisms that you can use to detect and help protect your Amazon Cognito user pool against unintended user sign-up and SMS pumping. By implementing these strategies, you can enhance the security of your web and mobile applications and help to safeguard your services from potential abuse and financial loss. We suggest that you apply a combination of these detection and mitigation approaches to help protect your Cognito user pools.</p> 
<p>If you have feedback about this post, submit comments in the<strong> Comments</strong> section below. If you have questions about this post, <a href="https://console.aws.amazon.com/support/home" rel="noopener noreferrer" target="_blank">contact AWS Support</a>.</p> 
<footer> 
 <div class="blog-author-box">
  <img alt="Edward Sun" class="alignleft size-full" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/01/09/Edward_Sun.jpg" style="width: 93.750px; height: 125px; margin: 12px 18px 6px 12px;" />
  <span class="lb-h4" style="line-height: 2.1em; padding-top: 12px; margin-top: 24px;">Edward Sun</span>
  <br /> Edward is a Security Specialist Solutions Architect focused on identity and access management. He loves helping customers throughout their cloud transformation journey with architecture design, security best practices, migration, and cost optimizations. Outside of work, Edward enjoys hiking, golfing, and cheering for his alma mater, the Georgia Bulldogs.
 </div> 
 <div class="blog-author-box">
  <img alt="Steve de Vera" class="alignleft size-full" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2022/12/21/Steve-de-vera-author.jpg" style="width: 93.750px; height: 125px; margin: 12px 18px 6px 12px;" />
  <span class="lb-h4" style="line-height: 2.1em; padding-top: 12px; margin-top: 24px;">Steve de Vera</span>
  <br /> Steve is a manager in the AWS Customer Incident Response Team (CIRT). He is passionate about American-style BBQ and is a certified competition BBQ judge. He has a dog named Brisket.
 </div> 
 <div class="blog-author-box">
  <img alt="Tony Suarez" class="alignleft size-full" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/08/27/suarito.jpg" style="width: 93.750px; height: 125px; margin: 12px 18px 6px 12px;" />
  <span class="lb-h4" style="line-height: 2.1em; padding-top: 12px; margin-top: 24px;">Tony Suarez</span>
  <br /> Tony is a San Diego, CA based Solutions Architect with over 15 years of experience in IT operations. As a member of the AWS VMware technical field community, Tony enjoys helping customers solve challenging problems in innovative ways. Enabling customers to efficiently manage, automate, and orchestrate large-scale hybrid infrastructure projects is Tony’s passion.
 </div> 
</footer>
