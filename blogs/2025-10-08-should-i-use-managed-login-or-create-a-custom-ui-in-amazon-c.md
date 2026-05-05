---
title: "Should I use managed login or create a custom UI in Amazon Cognito?"
url: "https://aws.amazon.com/blogs/security/use-the-hosted-ui-or-create-a-custom-ui-in-amazon-cognito/"
date: "Wed, 08 Oct 2025 17:56:20 +0000"
author: "Joshua Du Lac"
feed_url: "https://aws.amazon.com/blogs/security/tag/amazon-cognito/feed/"
---
<blockquote>
 <p><strong>October 8, 2025:</strong> This blog post has been updated to include the Amazon Cognito managed login experience. The managed login experience has an updated look, additional features, and enhanced customization options.</p>
</blockquote> 
<blockquote>
 <p><strong>September 8, 2023:</strong> It’s important to know that if you activate user sign-up in your user pool, anyone on the internet can sign up for an account and sign in to your apps. Don’t enable self-registration in your user pool unless you want to open your app to allow users to sign up.</p>
</blockquote> 
<blockquote>
 <p><strong>June 9, 2023:</strong> Original publication date.</p>
</blockquote> 
<hr /> 
<div class="Page-articleBody"> 
 <div class="RichTextArticleBody RichTextBody"> 
  <p><span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/cognito/" rel="noopener" target="_blank">Amazon Cognito</a></span> is an authentication, authorization, and user management service for your web and mobile applications. Your users can sign in directly through many different authentication methods, such as user accounts within Amazon Cognito or through social providers such as Facebook, Amazon, Apple, or Google. You can also configure federation through a third-party <span class="LinkEnhancement"><a class="Link" href="https://openid.net/connect/" rel="noopener" target="_blank">OpenID Connect (OIDC)</a></span> or SAML 2.0 identity provider (IdP).</p> 
  <p><span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html" rel="noopener" target="_blank">Amazon Cognito user pools</a></span> are user directories that provide sign-up and sign-in functions for your application users, including federated authentication capabilities. A Cognito user pool has two primary UI options:</p> 
  <ul class="rte2-style-ul" id="rte-a98d9663-98b0-11f0-955d-3f714a6d9f45"> 
   <li><b>Managed login</b>: AWS hosts, preconfigures, maintains, and scales the UI—including <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-managed-login.html" rel="noopener" target="_blank">managed login</a></span> branding and classic Hosted UI branding—with a set of options that you can customize or configure for sign-up and sign-in for app users.</li> 
   <li><b>Custom UI</b>: You can configure an Amazon Cognito user pool with a completely custom UI by using the <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/cognito/dev-resources/" rel="noopener" target="_blank">SDK</a></span>. You’re accountable for hosting, configuring, maintaining, and scaling your custom UI as a part of your responsibility in the <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/compliance/shared-responsibility-model/" rel="noopener" target="_blank">AWS Shared Responsibility Model</a></span>.</li> 
  </ul> 
  <p>In this blog post, we review the benefits of using the managed login or creating a custom UI with the SDK and things to consider in determining which to choose for your application.</p> 
  <div class="RichTextHeading"> 
   <h2>Managed login</h2> 
   <p></p>
  </div> 
  <p>Managed login provides web interfaces for sign-up, sign-in, multi-factor authentication (MFA), password management, and passwordless and passkey sign-in capabilities in your user pool. The managed login provides an authorization server based on the OAuth 2.0 specification, and has a default implementation of user flows for sign-up and sign-in. Your application can redirect to the managed login, which will handle the user flows through the <span class="LinkEnhancement"><a class="Link" href="https://www.rfc-editor.org/rfc/rfc6749#page-24" rel="noopener" target="_blank">authorization code grant flow</a></span>. The managed login also supports sign-in through social providers and federation from OIDC-compliant and SAML 2.0 providers. Amazon Cognito offers two visual modes and branding and customization experiences: managed login branding with branding editor and hosted UI (classic) branding.</p> 
  <p><b>Managed login branding with branding editor</b><br /> Managed login branding provides an improved user experience with the most up-to-date authentication options for the user pool UI experience. Figure 1 shows managed login using the default branding settings.</p> 
  <div class="wp-caption alignnone" id="attachment_40099" style="width: 1440px;">
   <img alt="Figure 1: Managed login default branding settings" class="size-full wp-image-40099" height="784" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/10/03/Image-1.png" width="1430" />
   <p class="wp-caption-text" id="caption-attachment-40099">Figure 1: Managed login default branding settings</p>
  </div> 
  <p>The branding editor is a no-code visual editor that you can use to customize the look and feel of the entire user journey. You can customize each user pool application client individually, and preview screens in real-time with different screen sizes, as shown in Figure 2.</p> 
  <div class="wp-caption alignnone" id="attachment_40100" style="width: 1440px;">
   <img alt="Figure 2: Customization in the Amazon Cognito branding editor (Image credits)" class="size-full wp-image-40100" height="727" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/10/03/Image-2.png" width="1430" />
   <p class="wp-caption-text" id="caption-attachment-40100">Figure 2: Customization in the Amazon Cognito branding editor (<span class="LinkEnhancement"><a class="Link" href="https://stock.adobe.com/images/jumping-puppy-on-a-light-background-advertising-banner-template-for-pet-store-or-veterinary-clinic/854028815?prev_url=detail" rel="noopener" target="_blank">Image credits</a></span>)</p>
  </div> 
  <p>As shown in Figure 3, You can customize various components using the branding editor, including background, header and footer, buttons, focus state, icons, and more.</p> 
  <div class="wp-caption alignnone" id="attachment_40101" style="width: 1440px;">
   <img alt="Figure 3: Various components customization options" class="size-full wp-image-40101" height="926" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/10/03/Image-3.png" width="1430" />
   <p class="wp-caption-text" id="caption-attachment-40101">Figure 3: Various components customization options</p>
  </div> 
  <p>Additionally, managed login branding adds support for passwordless sign-in with passkeys, email one-time-passwords (OTP) and SMS OTPs, as shown in Figure 4. After you enable passwordless login in your user pool, managed login branding adapts to curated user flows with users’ preferred authentication methods.</p> 
  <div class="wp-caption alignnone" id="attachment_40102" style="width: 1440px;">
   <img alt="Figure 4: Sign in with passkey flow (left) and user-selected sign-in method flow (right)" class="size-full wp-image-40102" height="969" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/10/03/Image-4.png" width="1430" />
   <p class="wp-caption-text" id="caption-attachment-40102">Figure 4: Sign in with passkey flow (left) and user-selected sign-in method flow (right)</p>
  </div> 
  <p>Managed login branding also offers localization options in <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-managed-login.html#managed-login-localization" rel="noopener" target="_blank">several languages</a></span> (two are shown in Figure 5). You can add a <code>lang</code> query parameter in the link you distribute to users, and Amazon Cognito will set a cookie in users’ browsers with their language preference after the initial request.</p> 
  <div class="wp-caption alignnone" id="attachment_40093" style="width: 1440px;">
   <img alt="Figure 5: Cognito user sign up page in Japanese (left) and user sign in page in Simplified Chinese (right)" class="size-full wp-image-40093" height="910" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/10/03/Image-5.png" width="1430" />
   <p class="wp-caption-text" id="caption-attachment-40093">Figure 5: Cognito user sign up page in Japanese (left) and user sign in page in Simplified Chinese (right)</p>
  </div> 
  <p><b>Hosted UI (classic) branding</b><br /> For customers who prefer a traditional approach, Amazon Cognito continues to support the Hosted UI (classic) branding (shown in Figure 6) with basic customization where you can upload a CSS file to design the UI styling and upload a brand-specific logo. Hosted UI (classic) supports standard authentication flows with MFA and self-service sign up.</p> 
  <div class="wp-caption alignnone" id="attachment_40094" style="width: 518px;">
   <img alt="Figure 6: Hosted UI (classic) branding" class="size-full wp-image-40094" height="351" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/10/03/Image-6.png" width="508" />
   <p class="wp-caption-text" id="caption-attachment-40094">Figure 6: Hosted UI (classic) branding</p>
  </div> 
  <p>The managed login branding with branding editor is available to Amazon Cognito user pools with Essentials and Plus feature tiers, and Hosted UI (classic) branding is available to most Cognito user pools including Lite tier. To learn more about Cognito feature tiers, visit <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/cognito/pricing/" rel="noopener" target="_blank">Amazon Cognito pricing</a></span>.</p> 
  <p><b>Security and compliance capabilities</b></p> 
  <p>Both managed login branding and Hosted UI (classic) branding are designed to help you meet your compliance and security requirements and your users’ needs. Managed login supports custom OAuth scopes and OAuth 2.0 flows. If you want single sign-on (SSO), you can use managed login to support a single login across many application clients, with browser session cookies for the same domain. Actions are logged in <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/cloudtrail/" rel="noopener" target="_blank">AWS CloudTrail</a></span>, and you can use the logs for audit and reactionary automation. The managed login experience also supports the full suite of <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pool-settings-advanced-security.html" rel="noopener" target="_blank">threat protection</a></span> features for Amazon Cognito. For additional protection, managed login has support for <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-waf.html" rel="noopener" target="_blank">AWS WAF web ACLs</a></span> and for <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/waf/latest/developerguide/waf-captcha-and-challenge.html" rel="noopener" target="_blank">AWS WAF CAPTCHA</a></span>, which can help protect your Cognito user pools from web-based exploits and unwanted bots.</p> 
  <div class="wp-caption alignnone" id="attachment_40095" style="width: 1440px;">
   <img alt="Figure 7: Example default managed login with several login providers enabled" class="size-full wp-image-40095" height="760" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/10/03/Image-7.png" width="1430" />
   <p class="wp-caption-text" id="caption-attachment-40095">Figure 7: Example default managed login with several login providers enabled</p>
  </div> 
  <p>For federation, managed login<i> </i>supports federation with third-party IdPs that support OIDC and SAML 2.0, as well as social IdPs, as shown in Figure 7. Identity providers are connected to your Amazon Cognito user pool. In managed login, users use a button to select the federation source, and redirection is automatic. With SAML and OIDC IdPs, you can also configure mapping by using the domain in the user’s email address. In this case, a single text field is visible to your application users to enter an email address, as shown in Figure 8, and the lookup and redirect to the appropriate SAML IdP is automatic, as described in <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-managing-saml-idp-naming.html" rel="noopener" target="_blank">Choosing SAML identity provider names</a></span>.</p> 
  <div class="wp-caption alignnone" id="attachment_40096" style="width: 946px;">
   <img alt="Figure 8: Managed login that links to corporate IdP through an email domain" class="size-full wp-image-40096" height="567" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/10/03/Image-8.png" width="936" />
   <p class="wp-caption-text" id="caption-attachment-40096">Figure 8: Managed login that links to corporate IdP through an email domain</p>
  </div> 
  <p>Managed login integrates with <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/elasticloadbalancing/application-load-balancer/" rel="noopener" target="_blank">Application Load Balancer (ALB)</a></span> for web applications and works with <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/amplify/" rel="noopener" target="_blank">AWS Amplify</a></span> to enable social identity provider and enterprise federation (SAML and OIDC) capabilities. Beyond these integrations, Amazon Cognito user pools integrate with various AWS services (such as <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/appsync/latest/devguide/security-authz.html" rel="noopener" target="_blank">AWS AppSync</a></span>), that require user authentication and authorization, and <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/api-gateway/" rel="noopener" target="_blank">Amazon API Gateway</a></span> through <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-integrate-with-cognito.html" rel="noopener" target="_blank">Cognito authorizers</a></span> to secure your REST and HTTP endpoints.</p> 
  <p>You might choose to use managed login for many reasons. AWS fully manages the hosting, maintenance, and scaling of the managed login, which can contribute to the speed of go-to-market for customers. If your app requires OAuth 2.0 custom scopes, federation, social login, or native users with basic but customized branding and potentially numerous Amazon Cognito user pools, you might benefit from using managed login.</p> 
  <p>For more information about how to configure and use the hosted UI, see <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-integration.html" rel="noopener" target="_blank">Using the Amazon Cognito hosted UI for sign-up and sign-in</a></span>.</p> 
  <div class="RichTextHeading"> 
   <h2>Create a custom UI</h2> 
   <p></p>
  </div> 
  <p>Creating a custom UI using the SDK for Amazon Cognito provides a host of benefits and features that can help you completely customize the UI for your application users. With a custom UI, you have complete control over the look and feel of the UI that your application users will land on, including designing your app to support multiple languages, and you can build and design custom authentication flows.</p> 
  <p>There are numerous features that are supported when you build a custom UI. As with the managed login, the <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_Operations.html" rel="noopener" target="_blank">APIs invoked</a></span> from a custom UI using the <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-integrate-apps.html" rel="noopener" target="_blank">SDK</a></span> will create log entries in CloudTrail, and you can use the logs for audit and automation. You can also create a custom authentication flow for your users with a fully custom authentication experience beyond the those available in managed login.</p> 
  <p>In a custom UI, you can build custom session management and integrate with AWS WAF. A custom UI also works with the <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pool-settings-advanced-security.html" rel="noopener" target="_blank">threat protection features</a></span> of Amazon Cognito.</p> 
  <div class="wp-caption alignnone" id="attachment_40097" style="width: 634px;">
   <img alt="Figure 9: Example of a custom user interface" class="size-full wp-image-40097" height="313" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/10/03/Image-9.png" width="624" />
   <p class="wp-caption-text" id="caption-attachment-40097">Figure 9: Example of a custom user interface</p>
  </div> 
  <p>With a custom UI, such as the one shown in Figure 10, you can orchestrate a suite of sign-in options and sign-in flows for your users. For example, you can collect a user or tenant identifier at the beginning of the authentication flow and apply your own logic for user authentication flow, such as redirecting federated users to external IdPs, displaying a password prompt for local users, or directing users to create a new account if they don’t exist. You can also build flows to let a user choose alternative MFA methods if their preferred choices aren’t available.</p> 
  <div class="wp-caption alignnone" id="attachment_40098" style="width: 634px;">
   <img alt="Figure 10: Custom UI example" class="size-full wp-image-40098" height="818" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/10/03/Image-10.png" width="624" />
   <p class="wp-caption-text" id="caption-attachment-40098">Figure 10: Custom UI example</p>
  </div> 
  <p>When you build a custom UI, there is support for custom endpoints and proxies so that you have a wider range of options for management and consistency across application development as it relates to authentication. <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-authentication-flow.html#amazon-cognito-user-pools-custom-authentication-flow" rel="noopener" target="_blank">Custom authentication flows</a></span> are only available in applications with a custom UI, which gives you the ability to make customized challenge prompts and answers to help you meet custom security requirements by using <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/lambda/" rel="noopener" target="_blank">AWS Lambda</a></span> triggers. For example, you could use it to implement <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/blogs/security/implement-oauth-2-0-device-grant-flow-by-using-amazon-cognito-and-aws-lambda/" rel="noopener" target="_blank">OAuth 2.0 device grant flows</a></span>. Lastly, a custom UI supports a <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-device-tracking.html" rel="noopener" target="_blank">remember device</a></span> feature where you can add low-effort sign-in from trusted devices.</p> 
  <p>You might choose to build a custom UI with an SDK when full customization is a requirement or where you want to incorporate customized authentication flows using the custom authentication challenge Lambda triggers. A custom UI is a great choice if you aren’t required to use OAuth 2.0 flows and you have the resources to develop and implement a unique UI for your application users.</p> 
  <p>For more information about how to configure and use a custom UI, see <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-integration.html" rel="noopener" target="_blank">Using the Amazon Cognito managed login for sign-up and sign-in</a></span>. You can also visit the documentation on <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-integrate-apps.html#cognito-integrate-apps-amplify-ui" rel="noopener" target="_blank">Building custom UIs with Amplify</a></span>.</p> 
  <div class="RichTextHeading"> 
   <h2>Decision criteria matrix</h2> 
   <p></p>
  </div> 
  <p>When deciding between Amazon Cognito managed login branding options and a custom UI, there are some unique differences that can help you determine which UI is best for your application needs. Managed login offers a modern, customizable authentication experience with advanced features like no-code visual customization, dark mode themes, and support for passwordless options. It supports OAuth 2.0 flows, custom OAuth scopes, the ability to sign in one time and access many Cognito application clients (using SSO), and full use of the Cognito threat protection features. For applications requiring complete control over the authentication experience and UX—including custom authentication flows, device fingerprinting, and reduced token expiration—a custom UI is the better choice. This option allows for full UI customization, implementation of custom authentication flows, and integration with specific frameworks or libraries not supported by managed login.</p> 
  <p>When making your decision, consider factors such as the level of customization required, specific authentication features needed, development resources available, integration requirements with other AWS services, security and compliance needs, and user experience priorities. Remember that your application authentication requirements and customer experience should take precedence over other considerations. You can use the following table to help select the best UI for your requirements.</p> 
  <table border="1px" cellpadding="10px" style="border-collapse: separate; text-indent: initial; border-spacing: 2px; border-color: gray; width: 100%;"> 
   <tbody> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Requirements</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Managed login</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Hosted UI (classic)</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Custom UI (SDK)</b></p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>OAuth 2.0 flows</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Custom OAuth scopes</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Customization of UI</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>No-code branding designer</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Limited CSS customization</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Full custom control</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Custom user input forms</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Custom authentication flow</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Passwordless authentication flow</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Custom implementation available</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Localization with multiple languages</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Login once across many app clients</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Session expiration configurable under 1 hour</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Trusted-device authentication</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>AWS WAF integration</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Support for AWS WAF CAPTCHA</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>Ability to use a custom endpoint or proxy</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p><b>AWS Application Load Balancer integration</b></p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Supported</p> </td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"> <p>Not available</p> </td> 
    </tr> 
   </tbody> 
  </table> 
  <p><i>Figure 11: Decision criteria matrix</i></p> 
  <div class="RichTextHeading"> 
   <h2>Conclusion</h2> 
   <p></p>
  </div> 
  <p>In this post, you learned about using managed login, including its two branding options and creating a custom UI in Amazon Cognito and the many supported features and benefits of each. Each UI option targets a specific need. Choose from available options based on your list of requirements for authentication and the user sign-up and sign-in experience. You can use the information in this post as a reference as you add Amazon Cognito to your mobile and web applications for authentication.</p> 
  <p></p>
 </div> 
</div> 
<p>Have a question? <a href="https://aws.amazon.com/contact-us/?nc1=f_m" rel="noopener" target="_blank">Contact us</a> for general support services.</p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Author photo" class="aligncenter size-full wp-image-10795" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2019/06/11/du-lac-bio-photo.jpeg" width="119" /> 
  </div> 
  <h3 class="lb-h4">Joshua Du Lac</h3> 
  <p>Josh is a Senior Manager of Security Solutions Architects at AWS. He has advised hundreds of enterprise, global, and financial services customers to accelerate their journey to the cloud while improving their security along the way. Outside of work, Josh enjoys searching for the best tacos in Texas and practicing his handstands.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Jeremy Wave" class="aligncenter size-full wp-image-27951" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2022/12/08/Jeremy-Ware.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Jeremy Ware</h3> 
  <p>Jeremy is a Security Specialist Solutions Architect focused on Identity and Access Management. Jeremy and his team enable AWS customers to implement sophisticated, scalable, and secure IAM architecture and Authentication workflows to solve business challenges. With a background in Security Engineering, Jeremy has spent many years working to raise the Security Maturity gap at numerous global enterprises. Outside of work, Jeremy loves to explore the mountainous outdoors, and participate in sports such as snowboarding, wakeboarding, and dirt bike riding.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Edward Sun" class="aligncenter size-full wp-image-27951" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/01/09/Edward_Sun.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Edward Sun</h3> 
  <p>Edward is a Security Specialist Solutions Architect focused on identity and access management. He loves helping customers throughout their cloud transformation journey with architecture design, security best practices, migration, and cost optimizations. Outside of work, Edward enjoys hiking, golfing, and cheering for his alma mater, the Georgia Bulldogs.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Kiran Dongara" class="aligncenter size-full wp-image-27951" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/10/03/Kiran-Dongara-Author.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Kiran Dongara</h3> 
  <p>Kiran Dongara is a Solutions Architect at Amazon Web Services (AWS) in the Worldwide Public Sector, primarily supporting US state and local government (SLG) customers and partners. His expertise lies in designing scalable and efficient architectures that adhere to well-architected framework practices, maximizing value and return on investment for his customers. When not working, Kiran prioritizes family time, nature walks, and cycling.</p> 
 </div> 
</footer>
