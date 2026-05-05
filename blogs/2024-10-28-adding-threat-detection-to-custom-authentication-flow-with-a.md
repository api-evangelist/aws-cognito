---
title: "Adding threat detection to custom authentication flow with Amazon Cognito advanced security features"
url: "https://aws.amazon.com/blogs/security/adding-threat-detection-to-custom-authentication-flow-with-amazon-cognito-advanced-security-features/"
date: "Mon, 28 Oct 2024 22:06:21 +0000"
author: "Vishal Jakharia"
feed_url: "https://aws.amazon.com/blogs/security/tag/amazon-cognito/feed/"
---
<blockquote>
 <p><strong>January 28, 2025:</strong> The following blog post highlights how to add threat detection to your custom authentication flows by using Amazon Cognito. With the introduction of <a href="https://aws.amazon.com/blogs/aws/improve-your-app-authentication-workflow-with-new-amazon-cognito-features/" rel="noopener" target="_blank">new Cognito feature tiers</a>, threat protection features are now included as default features for Plus tier customers.</p> 
 <p>Customers using advanced security features (ASF) in Amazon Cognito should consider switching to the Plus tier, which includes the full range of threat protection capabilities, additional capabilities such as passwordless sign-in, and up to 60% savings compared to using ASF from other tiers.</p> 
</blockquote> 
<hr /> 
<p>Recently, passwordless authentication has gained popularity compared to traditional password-based authentication methods. Application owners can add user management to their applications while offloading most of the security heavy-lifting to <a href="https://aws.amazon.com/cognito" rel="noopener" target="_blank">Amazon Cognito</a>. You can use Amazon Cognito to customize user authentication flow by implementing passwordless authentication. Amazon Cognito enhances the security posture of your applications because it handles the storage and management of user information securely. Additionally, Amazon Cognito provides secure authentication flow and verifiable tokens.</p> 
<p>This post explores how you can use the advanced security features of Amazon Cognito to add threat detection to your passwordless authentication custom authentication flow, further strengthening your defenses against account takeover risks.</p> 
<h2 id="overview">Overview</h2> 
<p>Amazon Cognito is a customer identity and access management (CIAM) service that streamlines the process of building secure, scalable, and user-friendly authentication solutions. With Amazon Cognito, you can integrate user sign-up, sign-in, and access control functionalities into your web and mobile applications. One of the key features of Amazon Cognito is that it supports <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-challenge.html" rel="noopener" target="_blank">custom authentication flow</a>, which you can use to implement passwordless authentication for your users or you can require users to solve a CAPTCHA or answer a security question before being allowed to authenticate.</p> 
<p>Custom authentication flows, such as passwordless authentication, offer an improved user experience while enhancing security by using strong custom factors. In addition, it is recommended to implement additional measures to detect and mitigate potential risks. Amazon Cognito advanced security provides a suite of powerful features designed to detect risks and allows you to take action to protect your user accounts.</p> 
<p>For more information on the features offered by Amazon Cognito advanced security, see <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pool-settings-advanced-security.html" rel="noopener" target="_blank">User pool advanced security features</a>.</p> 
<p>By combining passwordless authentication with Amazon Cognito advanced security features, you can enhance your application’s overall security posture while providing a seamless and user-friendly authentication experience to your users.</p> 
<h2 id="advanced-security-support-for-custom-authentication-flow">Advanced security support for custom authentication flow</h2> 
<p>Amazon Cognito advanced security now supports custom authentication flows to provide additional threat detection, including passwordless authentication. You can improve the security of applications that use custom authentication factors by enabling risk detection and adaptive authentication.</p> 
<p>The custom authentication flow triggers three <a href="https://aws.amazon.com/lambda" rel="noopener" target="_blank">AWS Lambda</a> functions, as shown in Figure 1.</p> 
<div class="wp-caption aligncenter" id="attachment_36211" style="width: 703px;">
 <img alt="Figure 1: Custom authentication flow" class="size-full wp-image-36211" height="404" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/10/24/Adding-threat-detection-1.png" width="693" />
 <p class="wp-caption-text" id="caption-attachment-36211">Figure 1: Custom authentication flow</p>
</div> 
<p>The custom authentication flow depicted in Figure 1 includes the following steps:</p> 
<ol type="1"> 
 <li>A user initiates authentication from the custom sign-in page, which sends the authentication request to the Amazon Cognito user pool.</li> 
 <li>The user pool calls the <code style="color: #000000;">Define Auth Challenge</code> Lambda function. This function determines which custom challenge needs to be created. At the end, it reports back to Amazon Cognito to issue a token if authentication is successful. The function is invoked at the start of the custom authentication flow and after each completion of the <code style="color: #000000;">Verify Auth Challenge Response</code> Lambda trigger.</li> 
 <li>The user pool calls the <code style="color: #000000;">Create Auth Challenge</code> Lambda function. This function is invoked to create a unique challenge for the user based on the instruction of the <code style="color: #000000;">Define Auth Challenge</code> Lambda trigger.</li> 
 <li>The user responds to the challenge with their answer, which is sent by making a <code style="color: #000000;">RespondToAuthChallenge</code> API call to the Amazon Cognito user pool.</li> 
 <li>The user pool calls the <code style="color: #000000;">Verify Auth Challenge</code> Response Lambda function with the response from the user. The function determines if the answer is correct.</li> 
 <li>The user pool then calls the <code style="color: #000000;">Define Auth Challenge</code> Lambda function. This function verifies that the challenge has been successfully answered and that no further challenge is needed. It includes <code style="color: #000000;">issueTokens: true</code> in its response to the user pool.</li> 
 <li>When advanced security is enabled, Amazon Cognito performs risk analysis on the authentication request. If a risk is detected, it’s mitigated as configured in advanced security. The user pool now considers the user to be authenticated and sends the user a valid JSON Web Token (JWT) (in response to step 4, the authentication challenge).</li> 
</ol> 
<h2 id="how-to-configure-advanced-security-for-custom-authentication-flow">How to configure advanced security for custom authentication flow</h2> 
<p>In this section, you set up a custom passwordless authentication flow and then add advanced security features (ASF) to protect your existing authentication flow.</p> 
<p><strong>Configure advanced security features</strong></p> 
<ol type="1"> 
 <li>Start by <a href="https://catalog.workshops.aws/cognito-webauthn-passwordless/en-US" rel="noopener" target="_blank">implementing passwordless authentication with Amazon Cognito and WebAuthn</a>.</li> 
 <li>After setting up passwordless authentication, go to the AWS Management Console for Amazon Cognito and configure advanced security features for your passwordless authentication flow.</li> 
 <li>Navigate to the user pool that has been created for the passwordless authentication solution.</li> 
 <li>Choose the <strong>Advanced Security</strong> tab and choose <strong>Activate</strong>.</li> 
 <li>In the <strong>Included features and initial states</strong> pop-up, you’ll see the <strong>Threat protection for standard authentication</strong> and <strong>Threat protection for custom authentication</strong> have already been included in <strong>Audit-only mode</strong>, choose <strong>Activate</strong>.<br /> 
  <blockquote>
   <p><strong>Note</strong>: It’s recommended to run advanced security features in audit only mode initially to evaluate risk patterns and decide the appropriate settings for each risk level. </p>
  </blockquote> <p> </p>
  <div class="wp-caption aligncenter" id="attachment_36212" style="width: 826px;">
   <img alt="Figure 2: Activate advanced security features" class="size-full wp-image-36212" height="576" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/10/24/Adding-threat-detection-2.png" style="border: 1px solid #bebebe;" width="816" />
   <p class="wp-caption-text" id="caption-attachment-36212">Figure 2: Activate advanced security features</p>
  </div> </li> 
 <li>To set up full function mode and enforcement for <strong>Threat protection for custom authentication</strong>, choose <strong>Set up full-function mode</strong>.<br /> 
  <div class="wp-caption aligncenter" id="attachment_36213" style="width: 1288px;">
   <img alt="Figure 3: Activate threat protection for custom authentication flow" class="size-full wp-image-36213" height="518" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/10/24/Adding-threat-detection-3.png" style="border: 1px solid #bebebe;" width="1278" />
   <p class="wp-caption-text" id="caption-attachment-36213">Figure 3: Activate threat protection for custom authentication flow</p>
  </div> </li> 
 <li>For <strong>Custom authentication enforcement mode</strong>, you can select: 
  <ul> 
   <li><strong>No enforcement</strong> – Amazon Cognito doesn’t gather metrics on detected risks or automatically take preventive actions.</li> 
   <li><strong>Audit-only</strong> – Amazon Cognito gathers metrics on detected risks, but doesn’t take automatic action.</li> 
   <li><strong>Full-function</strong> – Amazon Cognito automatically takes preventive actions in response to different levels of risk that you configure for your user pool.</li> 
  </ul> <p> Select <strong>Full-function</strong>.<br /> </p>
  <div class="wp-caption aligncenter" id="attachment_36214" style="width: 1330px;">
   <img alt="Figure 4: Configure enforcement level" class="size-full wp-image-36214" height="363" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/10/24/Adding-threat-detection-4.png" style="border: 1px solid #bebebe;" width="1320" />
   <p class="wp-caption-text" id="caption-attachment-36214">Figure 4: Configure enforcement level</p>
  </div> </li> 
 <li>You can choose either <strong>Cognito defaults</strong> or <strong>Custom</strong> to respond to each level of risk when Amazon Cognito detects potential malicious activity. 
  <ol type="a"> 
   <li><strong>Cognito defaults</strong>&nbsp;will block sign-in attempts for low, medium, and high risks.<br /> 
    <div class="wp-caption aligncenter" id="attachment_36215" style="width: 1331px;">
     <img alt="Figure 5: Adaptive authentication configuration" class="size-full wp-image-36215" height="464" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/10/24/Adding-threat-detection-5.png" style="border: 1px solid #bebebe;" width="1321" />
     <p class="wp-caption-text" id="caption-attachment-36215">Figure 5: Adaptive authentication configuration</p>
    </div></li> 
   <li>If you choose <strong>Custom</strong>, you can customize the risk configuration for each risk level. 
    <ul> 
     <li><strong>Allow</strong> – Sign-in attempts will be allowed without additional authentication factors.</li> 
     <li><strong>Optional MFA</strong> – Amazon Cognito will send a multi-factor authentication (MFA) challenge to the user if the user is eligible for MFA. A user is eligible for MFA if: 
      <ol type="a"> 
       <li>They have configured an authenticator app and TOTP MFA is enabled for the user pool.</li> 
       <li>They have a phone number or email address, and SMS or email message MFA is enabled for the user pool.</li> 
      </ol> <p> If the user is eligible for MFA, they must respond correctly to the MFA challenge. If the user is not eligible for MFA, Cognito will allow sign-in without additional authentication factors. </p></li> 
     <li><strong>Require MFA</strong> – Amazon Cognito will send an MFA challenge to the user if the user is eligible for MFA. If the user is eligible for MFA, they must respond correctly to the MFA challenge. If the user is not eligible for MFA, Cognito will block the sign-in attempt.</li> 
     <li><strong>Block</strong> – Cognito blocks future sign-in attempts.</li> 
    </ul> </li> 
  </ol> </li> 
 <li>You can notify users when adaptive authentication detects potentially suspicious activity using a customized email message. This notification is sent to users to confirm their activity, and Amazon Cognito uses the user’s response to learn their behavior patterns over time. By customizing the notification message, you can provide a better user experience and make sure communication regarding the security measure is clear to your users.<br /> 
  <div class="wp-caption aligncenter" id="attachment_36216" style="width: 1254px;">
   <img alt="Figure 6: Adaptive authentication message template" class="size-full wp-image-36216" height="827" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/10/24/Adding-threat-detection-6.png" style="border: 1px solid #bebebe;" width="1244" />
   <p class="wp-caption-text" id="caption-attachment-36216">Figure 6: Adaptive authentication message template</p>
  </div> </li> 
 <li>Review the threat protection configuration.<br /> 
  <div class="wp-caption aligncenter" id="attachment_36217" style="width: 1330px;">
   <img alt="Figure 7: Custom auth flow threat protection configuration" class="size-full wp-image-36217" height="658" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/10/24/Adding-threat-detection-7.png" style="border: 1px solid #bebebe;" width="1320" />
   <p class="wp-caption-text" id="caption-attachment-36217">Figure 7: Custom auth flow threat protection configuration</p>
  </div> </li> 
</ol> 
<p><strong>Test the configuration</strong></p> 
<p>To test the configuration, sign in from multiple devices and locations. Amazon Cognito will calculate risk and take action based on your configuration. After you’ve signed in multiple times through different devices, you can view the <strong>User event history</strong>.</p> 
<ol type="1"> 
 <li>In the Amazon Cognito console, go to the user pool and search for the user you signed in as.</li> 
 <li>Select the user name and navigate to <strong>User event history</strong>.</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_36218" style="width: 1511px;">
 <img alt="Figure 8: User event history" class="size-full wp-image-36218" height="234" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/10/24/Adding-threat-detection-8.png" style="border: 1px solid #bebebe;" width="1501" />
 <p class="wp-caption-text" id="caption-attachment-36218">Figure 8: User event history</p>
</div> 
<p>You can see the user event history with the risk levels and actions taken by Amazon Cognito as shown in Figure 8. In the figure, Amazon Cognito advanced security has detected a high-risk event and has blocked the sign-in attempt.</p> 
<p>Amazon Cognito will associate a risk level with each sign-in attempt and based on your adaptive configuration; it will either allow the sign in, request an MFA response, or block the request.</p> 
<blockquote>
 <p><strong>Note</strong>: Populating <code style="color: #000000;">UserContextData</code> in the request is important to the functionality of the risk engine. Some SDKs, such as <a href="https://aws.amazon.com/amplify" rel="noopener" target="_blank">AWS Amplify</a>, will populate this object by default, but in custom code, you need to make sure <code style="color: #000000;">userContextData</code> is calculated and populated correctly in relevant events. See <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pool-settings-adaptive-authentication.html#user-pool-settings-adaptive-authentication-device-fingerprint" rel="noopener" target="_blank">Adding user device and session data to API requests</a> for more information about populating <code style="color: #000000;">userContextData</code>.</p>
</blockquote> 
<p>Additionally, you can <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pool-settings-adaptive-authentication.html#user-pool-settings-adaptive-authentication-event-user-history-exporting" rel="noopener" target="_blank">export user authentication event</a> history to <a href="https://aws.amazon.com/cloudwatch" rel="noopener" target="_blank">Amazon CloudWatch</a>, a <a href="https://aws.amazon.com/firehose" rel="noopener" target="_blank">Amazon Data Firehose</a> stream, or an <a href="https://aws.amazon.com/s3" rel="noopener" target="_blank">Amazon Simple Storage Service (Amazon S3)</a> bucket for further analysis of the security event.</p> 
<h2 id="conclusion">Conclusion</h2> 
<p>In this post, you learned how to enable threat detection for a custom authentication flow such as passwordless authentication in Amazon Cognito. Threat detection helps you to monitor user activity and enhances security measures even when your users sign in through a custom authentication flow.</p> 
<p>If you have feedback about this post, submit comments in the <strong>Comments</strong> section below.</p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Vishal Jakharia" class="aligncenter size-full wp-image-34187" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/vishjakh.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Vishal Jakharia</h3> 
  <p>Vishal is a Cloud Support Engineer based in New Jersey, USA. He’s an Amazon Cognito subject matter expert who loves to work with customers and provide them solutions for implementing secure authentication and authorization. He helps customers migrate and build secure scalable architecture on the AWS Cloud.</p> 
  <p></p>
 </div> 
</footer> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="" class="aligncenter size-full wp-image-33128" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/01/16/smvarun.png" width="120" /> 
  </div> 
  <h3 class="lb-h4">Varun Sharma</h3> 
  <p>Varun is a Senior AWS Cloud Security Engineer who wears his security cape proudly. With a knack for unravelling the mysteries of Amazon Cognito and IAM, Varun is a go-to subject matter expert for these services. When he’s not busy securing the cloud, you’ll find him in the world of security penetration testing. And when the pixels are at rest, Varun switches gears to capture the beauty of nature through the lens of his camera.</p> 
  <p></p>
 </div> 
</footer>
