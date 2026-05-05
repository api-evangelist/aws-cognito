---
title: "How to set up SAML federation in Amazon Cognito using IdP-initiated single sign-on, request signing, and encrypted assertions"
url: "https://aws.amazon.com/blogs/security/how-to-set-up-saml-federation-in-amazon-cognito-using-idp-initiated-single-sign-on-request-signing-and-encrypted-assertions/"
date: "Thu, 16 May 2024 16:57:51 +0000"
author: "Vishal Jakharia"
feed_url: "https://aws.amazon.com/blogs/security/tag/amazon-cognito/feed/"
---
<p>When an identity provider (IdP) serves multiple service providers (SPs), IdP-initiated single sign-on provides a consistent sign-in experience that allows users to start the authentication process from one centralized portal or dashboard. It helps administrators have more control over the authentication process and simplifies the management.</p> 
<p>However, when you support IdP-initiated authentication, the SP (<a href="https://aws.amazon.com/cognito" rel="noopener" target="_blank">Amazon Cognito</a> in this case) can’t verify that it has solicited the SAML response that it receives from IdP because there is no SAML request initiated from the SP. To accept unsolicited SAML assertions in your user pool, you must consider its effect on your app security. Although your user pool can’t verify an IdP-initiated sign-in session, Amazon Cognito validates your request parameters and SAML assertions.</p> 
<p>Amazon Cognito has recently enhanced support for the SAML 2.0 protocol by adding support to IdP-initiated single sign-on (SSO), SAML request signing and accepting encrypted SAML responses.</p> 
<p>Amazon Cognito acts as the SP representing your application and generates a token after federation that can be used by the application to access protected backends. The SAML provider acts as an IdP, where the user identities and credentials are stored, and is responsible for authenticating the user.</p> 
<p>This post describes the steps to integrate a SAML IdP, Microsoft Entra ID, with an <a href="https://aws.amazon.com/cognito/" rel="noopener" target="_blank">Amazon Cognito</a> user pool and use SAML IdP-initiated SSO flow. It also describes steps to enable signing authentication requests and accepting encrypted SAML responses.</p> 
<h2>IdP-initiated authentication flow using SAML federation</h2> 
<div class="wp-caption aligncenter" id="attachment_34188" style="width: 790px;">
 <img alt="Figure 1: High-level diagram for SAML IdP-initiated authentication flow in a web or mobile app" class="size-full wp-image-34188" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img1-6.png" style="border: 1px solid #bebebe;" width="780" />
 <p class="wp-caption-text" id="caption-attachment-34188">Figure 1: High-level diagram for SAML IdP-initiated authentication flow in a web or mobile app</p>
</div> 
<p>As shown in Figure 1, the high-level flow diagram of an application with federated authentication typically involves the following steps:</p> 
<ol> 
 <li>An enterprise user opens their SSO portal and signs in. This usually opens a portal with several applications that the user has access to. When the user selects an Amazon Cognito protected application from their SSO portal, an IdP-initiated SSO flow is initiated.</li> 
 <li>When the user launches an application from the SSO portal, Entra ID sends a SAML assertion to the Cognito endpoint to federate the user.</li> 
 <li>Amazon Cognito validates the SAML assertion and creates the user in Cognito if this is first-time federation for the user or updates the user’s record if user has signed in before from this IdP. Cognito then generates an authorization code and redirects the user to the application URL with this authorization code. The application exchanges the authorization code for tokens from the Cognito token endpoint.</li> 
 <li>After the application has tokens, it uses them to authorize access within the application stack as needed.</li> 
</ol> 
<p>The SAML response contains claims or assertions that contain user-specific data. The SAML response is transferred over HTTPS to protect confidentiality of the data, but you can also enable encryption to further protect the confidentiality of transferred user information. This enables trusted parties who have the decryption key to decrypt the data. It protects the confidentiality of the data after it’s received by the SP.</p> 
<h2>Setting up SAML federation between Amazon Cognito and Entra ID</h2> 
<p>To set up SAML federation and use IdP-initiated SSO, you will complete the following steps:</p> 
<ol> 
 <li>Create an Amazon Cognito user pool.</li> 
 <li>Create an app client in the Cognito user pool.</li> 
 <li>Add Cognito as an enterprise application in Entra ID.</li> 
 <li>Add Entra ID as the SAML IdP and enable IdP-initiated SSO in Cognito.</li> 
 <li>Add the newly created SAML IdP to your user pool app client.</li> 
 <li>Enable encrypting the SAML response.</li> 
 <li>Add <span style="font-family: courier;">RelayState</span> in Entra ID SAML SSO.</li> 
</ol> 
<h3>Prerequisites</h3> 
<p>To implement the solution, you must have the necessary permissions to perform these tasks in Azure portal and in your AWS account.</p> 
<h2>Step 1: Create an Amazon Cognito user pool</h2> 
<p><a href="https://docs.aws.amazon.com/cognito/latest/developerguide/tutorial-create-user-pool.html" rel="noopener" target="_blank">Create a new user pool</a> in Amazon Cognito with the default settings. Make a note of the user pool ID, for example, <span style="font-family: courier;">us-east-1_abcd1234</span>. You will need this value for the next steps.</p> 
<h3>Add a domain name to user pool</h3> 
<p>The Cognito user pool’s hosted UI can be used as the OAuth 2.0 authorization server with a customizable web interface for sign-up and sign-in. Cognito OAuth 2.0 endpoints are accessible from a domain name that must be added to the user pool. There are two options for adding a domain name to a user pool. You can either use a Cognito domain or a domain name that you own. This solution uses a Cognito domain, which will look like the following:</p> 
<p style="font-family: courier;">https://<span style="font-family: courier; color: #ff0000;">&lt;yourDomainPrefix&gt;</span>.auth.<span style="font-family: courier; color: #ff0000;">&lt;aws-region&gt;</span>.amazoncognito.com</p> 
<h4>To add a domain name to a user pool:</h4> 
<ol> 
 <li>In the AWS Management Console for Amazon Cognito, navigate to the <strong>App integration</strong> tab for your user pool.</li> 
 <li>On the right side of the pane, choose <strong>Actions</strong> and select <strong>Create Cognito domain</strong>. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34189" style="width: 750px;">
   <img alt="Figure 2: Create a Cognito domain" class="size-full wp-image-34189" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img2-6.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-34189">Figure 2: Create a Cognito domain</p>
  </div><p></p> </li> 
 <li>Enter an available domain prefix (for example <span style="font-family: courier;">example-corp-prd</span>) to use with the Cognito domain. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34190" style="width: 750px;">
   <img alt="Figure 3: Add a domain prefix" class="size-full wp-image-34190" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img3-6.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-34190">Figure 3: Add a domain prefix</p>
  </div><p></p> </li> 
 <li>Choose <strong>Create Cognito domain</strong>.</li> 
</ol> 
<h2>Step 2: Create an app client in the Cognito user pool</h2> 
<p>Before you can use Amazon Cognito in your web application, you must register your app with Amazon Cognito as an app client. The IdP-initiated SAML flow can’t be enabled on one app client with the other SP-initiated authentication SAML IdPs or social IdPs. IdP-initiated SAML introduces additional risks that other SSO providers aren’t subject to. For example, it’s not possible to add a <span style="font-family: courier;">state</span> parameter, which is usually used for cross-site request forgery (CSRF) mitigation. Because of this, you can’t add IdPs that aren’t SAML, including the user pool itself, to an app client that uses a SAML provider with IdP-initiated SSO.</p> 
<h4>To create an app client:</h4> 
<ol> 
 <li>In the Amazon Cognito console, navigate to the <strong>App integration</strong> tab for the same user pool and locate <strong>App clients</strong>. Choose <strong>Create an app client</strong>.</li> 
 <li>Select an <strong>Application type</strong>. For this example, create a public client.</li> 
 <li>Enter an <strong>App client name</strong>.</li> 
 <li>Choose <strong>Don’t generate client secret</strong>.</li> 
 <li>Keep the rest of the settings as default.</li> 
 <li>Under <strong>Hosted UI settings</strong>, add<strong> Allowed callback URLs</strong> for your app client. This is where you will be directed after authentication.</li> 
 <li>Choose <strong>Authorization code grant </strong>for OAuth 2.0 grant types.</li> 
 <li>You can keep the remaining configuration as default and choose <strong>Create app client</strong>.</li> 
</ol> 
<p>After the app client is successfully created, capture the <strong>app client ID</strong> from the <strong>App integration</strong> tab of the user pool.</p> 
<h3>Prepare information for the Entra ID setup</h3> 
<p>Prepare the <strong>Identifier (Entity ID)</strong> and <strong>Reply URL</strong>, which are required to add Amazon Cognito as an enterprise application in Entra ID (Step 3).</p> 
<p>Create values for <strong>Identifier (Entity ID) </strong>and <strong>Reply URL</strong> according to the following formats:</p> 
<p>For <strong>Identifier (Entity ID)</strong>, the format is:<br /> <span style="font-family: courier;">urn:amazon:cognito:sp:<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;yourUserPoolID&gt;</span></span></p> 
<p>For example: <span style="font-family: courier;">urn:amazon:cognito:sp:us-east-1_abcd1234</span></p> 
<p>For <strong>Reply URL</strong>, the format is: <br /> <span style="font-family: courier;">https://<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;yourDomainPrefix&gt;</span>.auth.<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;aws-region&gt;</span>.amazoncognito.com/saml2/idpresponse</span></p> 
<p>For example: <span style="font-family: courier;">https://example-corp-prd.auth.us-east-1.amazoncognito.com/saml2/idpresponse</span></p> 
<p>The reply URL is the endpoint where Entra ID will send the SAML assertion to Amazon Cognito during user authentication.</p> 
<p>For more information, see <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-saml-idp.html" rel="noopener" target="_blank">Adding SAML identity providers to a user pool</a>.</p> 
<h2>Step 3: Add Amazon Cognito as an enterprise application in Entra ID</h2> 
<p>With the user pool and app client created and the information for Entra ID prepared, you can add Amazon Cognito as an application in Entra ID. To complete this step, you will add Cognito as an enterprise application and set up SSO.</p> 
<h4>To add Cognito as an enterprise application</h4> 
<ol> 
 <li>Sign in to the Azure portal.</li> 
 <li>In the search box, search for the service <strong>Microsoft Entra ID</strong>.</li> 
 <li>In the left sidebar, select <strong>Enterprise applications</strong>.</li> 
 <li>Choose <strong>New application</strong>.</li> 
 <li>On the <strong>Browse Microsoft Entra Gallery</strong> page, choose <strong>Create your own application</strong>. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34191" style="width: 708px;">
   <img alt="Figure 4: Create an application in Entra ID" class="size-full wp-image-34191" height="217" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img4-6.png" style="border: 1px solid #bebebe;" width="698" />
   <p class="wp-caption-text" id="caption-attachment-34191">Figure 4: Create an application in Entra ID</p>
  </div><p></p> </li> 
 <li>Under <strong>What’s the name of your app?</strong>, enter a name for your application and select <strong>Integrate any other application you don’t find in the gallery (Non-gallery)</strong>, as shown in Figure 4. Choose <strong>Create</strong>.</li> 
 <li>It will take few seconds for the application to be created in Entra ID, and then you should be redirected to the <strong>Overview</strong> page for the newly added application.</li> 
</ol> 
<h4>To set up SSO using SAML:</h4> 
<ol> 
 <li>On the <strong>Getting started</strong> page, in the <strong>Set up single sign on </strong>tile, choose <strong>Get started</strong>, as shown in Figure 5. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34192" style="width: 708px;">
   <img alt="Figure 5: Choose Set up single sign-on in Getting Started" class="size-full wp-image-34192" height="224" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img5-3.png" style="border: 1px solid #bebebe;" width="698" />
   <p class="wp-caption-text" id="caption-attachment-34192">Figure 5: Choose Set up single sign-on in Getting Started</p>
  </div><p></p> </li> 
 <li>On the next screen, select <strong>SAML</strong>.</li> 
 <li>In the middle pane under <strong>Set up Single Sign-On with SAML</strong>, in the <strong>Basic SAML Configuration</strong> section, choose the edit icon.</li> 
 <li>In the right pane under <strong>Basic SAML Configuration</strong>, replace the default <strong>Identifier ID (Entity ID)</strong> with the identifier (entity ID) you created in Step 2. Replace <strong>Reply URL (Assertion Consumer Service URL)</strong> with the reply URL you created in Step 2. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34193" style="width: 750px;">
   <img alt="Figure 6: Add the identifier (entity ID) and reply URL" class="size-full wp-image-34193" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img6-2.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-34193">Figure 6: Add the identifier (entity ID) and reply URL</p>
  </div><p></p> </li> 
 <li>Now go to <strong>Attributes &amp; Claims</strong> and note the claims, as shown in Figure 7. You’ll need these when creating attribute mapping in Amazon Cognito. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34194" style="width: 750px;">
   <img alt="Figure 7: Entra ID Attributes &amp; Claims" class="size-full wp-image-34194" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img7-2.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-34194">Figure 7: Entra ID Attributes &amp; Claims</p>
  </div><p></p> </li> 
 <li>Scroll down to the <strong>SAML Certificates</strong> section and copy the <strong>App Federation Metadata Url</strong> by choosing the copy into clipboard icon. Make a note of this URL to use in the next step. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34195" style="width: 750px;">
   <img alt="Figure 8: Copy SAML metadata URL from Entra ID" class="size-full wp-image-34195" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img8-2.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-34195">Figure 8: Copy SAML metadata URL from Entra ID</p>
  </div><p></p> </li> 
</ol> 
<h2>Step 4: Add Entra ID as SAML IdP in Amazon Cognito</h2> 
<p>In this step, you’ll add Entra ID as a SAML IdP to your user pool and download the signing and encryption certificates.</p> 
<h4>To add the SAML IdP:</h4> 
<ol> 
 <li>In the Amazon Cognito console, navigate to the <strong>Sign-in experience</strong> tab of the same user pool. Locate <strong>Federated identity provider sign-in </strong>and choose <strong>Add an Identity provider</strong>.</li> 
 <li>Choose a SAML IdP.</li> 
 <li>Enter a <strong>Provider name</strong>, for example, <span style="font-family: courier;">EntraID</span>.</li> 
 <li>Under <strong>IdP-initiated SAML sign-in</strong>, choose <strong>Accept SP-initiated and IdP-initiated SAML assertions</strong>.</li> 
 <li>Under <strong>Metadata document source</strong>, enter the metadata document endpoint URL you captured in Step 3. </li> 
 <li>(Optional) Under <strong>SAML signing and encryption</strong>, select <strong>Require encrypted SAML assertion from this provider</strong>. <p>Enable <strong>Required encrypted SAML assertion from this provider</strong> only if you can turn on token encryption in the Entra ID application. See Step 6.</p> </li> 
 <li>Under <strong>Map attributes between your SAML provider and your user pool</strong> to map SAML provider attributes to the user profile in your user pool. Include your user pool required attributes in your attribute map. <p>For example, when you choose <strong>User pool attribute</strong> email, enter the SAML attribute name as it appears in the SAML assertion from your IdP. In our case it will be <a href="http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress" rel="noopener" target="_blank">http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress</a>.</p> <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34198" style="width: 750px;">
   <img alt="Figure 9: Enter the SAML attribute name" class="size-full wp-image-34198" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img9-1.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-34198">Figure 9: Enter the SAML attribute name</p>
  </div><p></p> </li> 
 <li>Choose <strong>Add identity provider</strong>.</li> 
</ol> 
<p>After the IdP has been created, you can navigate to the recently added EntraID IdP in the user pool for downloading the SAML signing and encryption certificate. These certificates must be imported into the Entra ID enterprise application.</p> 
<h4>To download the certificates</h4> 
<ol> 
 <li>To download the SAML signing certificate, Choose <strong>View signing certificate</strong> and <strong>Download as .crt</strong></li> 
 <li>To download the SAML encryption certificate, Choose <strong>View encryption certificate</strong> and <strong>Download as .crt</strong>.</li> 
</ol> 
<h2>Step 5: Add the newly created SAML IdP to your user pool app client</h2> 
<p>Before you can use Amazon Cognito in your web application, you must add the SAML IdP created in Step 4 to your app client.</p> 
<h4>To add the SAML IdP:</h4> 
<ol> 
 <li>In the Amazon Cognito console, navigate to the <strong>App integration</strong> tab for the same user pool and locate <strong>App clients</strong>.</li> 
 <li>Choose the app client you created in Step 2.</li> 
 <li>Locate the <strong>Hosted UI</strong> section and choose <strong>Edit</strong>.</li> 
 <li>Under <strong>Identity providers</strong>, select the identity provider you created in Step 4 and choose <strong>Save changes</strong>. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34199" style="width: 629px;">
   <img alt="Figure 10: Enabling the Entra ID SAML identity provider in the Cognito app client" class="size-full wp-image-34199" height="791" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img10.png" style="border: 1px solid #bebebe;" width="619" />
   <p class="wp-caption-text" id="caption-attachment-34199">Figure 10: Enabling the Entra ID SAML identity provider in the Cognito app client</p>
  </div><p></p> </li> 
</ol> 
<p>At this stage, the Amazon Cognito OAuth 2.0 server is up and running and the web interface is accessible and ready to use. You can access the Cognito hosted UI from your app client using the Cognito console to test it further.</p> 
<h2>Step 6: Enable encrypting the SAML response in EntraID</h2> 
<p>For additional security and privacy of user data, enable encrypting the SAML response. Amazon Cognito and your IdP can establish confidentiality in SAML responses when users sign in and sign out. Cognito assigns a public-private RSA key pair and a certificate to each external SAML provider that you configure in your user pool. You will use the SAML encryption certificate downloaded in step 4.</p> 
<h4>To enable encrypting the SAML response:</h4> 
<ol> 
 <li>Navigate to your Enterprise application in Entra ID and in the left menu, under <strong>Security</strong>, select <strong>Token encryption</strong>.</li> 
 <li>Import the SAML encryption certificate you have already downloaded in step 4. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34200" style="width: 750px;">
   <img alt="Figure 11: Import the Cognito encryption certificate to Entra ID" class="size-full wp-image-34200" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img11.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-34200">Figure 11: Import the Cognito encryption certificate to Entra ID</p>
  </div><p></p> </li> 
 <li>After the certificate is imported, it’s inactive by default. To activate it, right-click on the certificate and select <strong>Activate token encryption certificate</strong>. This enables the encrypted SAML response. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34201" style="width: 750px;">
   <img alt="Figure 12: Activate the token encryption certificate in Entra ID" class="size-full wp-image-34201" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img12-1.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-34201">Figure 12: Activate the token encryption certificate in Entra ID</p>
  </div><p></p> </li> 
</ol> 
<h2>Step 7: Add RelayState in Entra ID SAML SSO</h2> 
<p>A <span style="font-family: courier;">RelayState</span> parameter is required when using SAML IdP-initiated authentication flow. Set this up in Entra ID for the Amazon Cognito user pool and the enabled app client ID.</p> 
<h4>To add RelayState in Entra ID SAML SSO:</h4> 
<ol> 
 <li>Sign in to the Azure portal and open the enterprise application created in Step 3.</li> 
 <li>In the left sidebar, choose <strong>Single sign-on</strong>.</li> 
 <li>In the middle pane under <strong>Set up Single Sign-On with SAML</strong>, in the <strong>Basic SAML Configuration</strong> section, choose the edit icon.</li> 
 <li>In the right pane under <strong>Basic SAML Configuration</strong>, apply the value as the format below to the <strong>Relay State (Optional)</strong> field. 
  <div class="hide-language"> 
   <pre><code class="lang-text">identity_provider=&lt;IDProviderName&gt;&amp;client_id=&lt;ClientId&gt;&amp;redirect_uri=&lt;callbackURL&gt;&amp;response_type=code&amp;scope=openid+email+phone</code></pre> 
  </div> 
  <ol> 
   <li>Replace <span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;IDProviderName&gt;</span> with the name you previously used for ID provider.</li> 
   <li>Replace <span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;ClientId&gt;</span> with the app client’s ClientID created in Step 2.</li> 
   <li>Replace <span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;ecallbackURL&gt;</span> with the URL of your web application that will receive the authorization code. It must be an HTTPS endpoint, except for in a local development environment where you can use <a href="http://localhost:PORT_NUMBER" rel="noopener" target="_blank">http://localhost:PORT_NUMBER</a>.</li> 
  </ol> <p>For example:</p> 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">identity_provider=EntraID&amp;client_id=abcd1234567&amp;redirect_uri=https://example.com&amp;response_type=code&amp;scope=openid+email+phone</code></pre> 
  </div> <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34202" style="width: 750px;">
   <img alt="Figure 13: Set RelayState in Entra ID single sign-on" class="size-full wp-image-34202" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img13-1.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-34202">Figure 13: Set RelayState in Entra ID single sign-on</p>
  </div><p></p> </li> 
</ol> 
<h2>Test the IdP-initiated flow</h2> 
<p>Next, do a quick test to check if everything is configured properly.</p> 
<ol> 
 <li>Sign in to the Azure portal and open the Enterprise application created in Step 3.</li> 
 <li>In the left sidebar, choose <strong>Users and groups</strong>.</li> 
 <li>On the right side, choose <strong>Add user/group</strong>. This will show the <strong>Add Assignment</strong> page.</li> 
 <li>From the left side of the page, choose <strong>None Selected </strong>.</li> 
 <li>Select a user from the right of the screen and follow the prompt to assign the user for this application.</li> 
 <li>Once the user is assigned successfully, open <a href="https://myapps.microsoft.com/" rel="noopener" target="_blank">https://www.microsoft365.com/apps</a> and sign in as the assigned user.</li> 
 <li>After you are signed in, choose the application icon registered as the IdP-initiated SSO. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34207" style="width: 745px;">
   <img alt="Figure 14: Testing IdP-initiated SSO from an Office 365 application" class="size-full wp-image-34207" height="721" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img14-1.png" style="border: 1px solid #bebebe;" width="735" />
   <p class="wp-caption-text" id="caption-attachment-34207">Figure 14: Testing IdP-initiated SSO from an Office 365 application</p>
  </div><p></p> </li> 
 <li>The application will start the IdP-initiated authentication flow and the user will be redirected to the application as a signed-in user.</li> 
</ol> 
<h2>Signing an authentication request in case of SP-initiated flow</h2> 
<p>The preceding authentication flow that you tested uses IdP-initiated SSO. If you’re using an SP-initiated flow, you can enable signing of the SAML request that is sent from the SP (Amazon Cognito) to the IdP (Entra ID) for additional security and integrity of communication between them.</p> 
<p>You can enable the authentication request signing in Cognito while creating the IdP or by updating your existing IdP.</p> 
<h4>To enable signing of the SAML request:</h4> 
<ol> 
 <li>In the Amazon Cognito console, when you create or edit your SAML identity provider, under<strong> SAML signing and encryption</strong>, select the box <strong>Sign SAML requests to this provider</strong> and choose <strong>Save changes</strong>. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_34208" style="width: 616px;">
   <img alt="Figure 15: Enabling signing SAML request" class="size-full wp-image-34208" height="712" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/img15.png" style="border: 1px solid #bebebe;" width="606" />
   <p class="wp-caption-text" id="caption-attachment-34208">Figure 15: Enabling signing SAML request</p>
  </div><p></p> </li> 
 <li>Sign in to the Azure portal and access your Entra ID enterprise application. Go to <strong>Set up single sign on</strong> and edit <strong>Verification certificates (optional)</strong>.</li> 
 <li>Select the checkbox <strong>Require verification certificates</strong> and upload the Cognito user pool SAML signing certificate already downloaded in Step 4 with a <span style="font-family: courier;">.cer</span> file extension. You must convert the .crt file to a .cer file because Entra ID requires a verification certificate in a .cer extension.</li> 
</ol> 
<h4>To convert the .crt certificate extension to .cer:</h4> 
<ol> 
 <li>Right-click the .crt file and choose <strong>Open</strong>.</li> 
 <li>Navigate to the <strong>Details</strong> tab.</li> 
 <li>Select <strong>Copy to File</strong>… and choose <strong>Next</strong>.</li> 
 <li>Select <strong>Base-64 encoded X.509 (.CER)</strong> and choose <strong>Next</strong>.</li> 
 <li>Give your export file a name (for example, <span style="font-family: courier;">Entra ID.cer</span>) and choose <strong>Save</strong>.</li> 
 <li>Choose <strong>Next</strong>.</li> 
 <li>Confirm the details and choose <strong>Finish</strong>.</li> 
</ol> 
<h2>Test the SP-initiated flow</h2> 
<p>Next, do a quick test to check if everything is configured properly.</p> 
<ol> 
 <li>In the Amazon Cognito console, navigate to the <strong>App integration</strong> tab for the same user pool and locate <strong>App clients</strong>.</li> 
 <li>Choose the app client you created in Step 2.</li> 
 <li>Locate the <strong>Hosted UI</strong> section and choose <strong>View Hosted UI</strong>.</li> 
 <li>From the hosted UI, authenticate yourself using Entra ID as the identity provider.</li> 
 <li>After authentication is completed successfully, you will be redirected to the callback URL you configured in your app client with the authorization code.</li> 
</ol> 
<p>If you capture the SAML request, you will see that Amazon Cognito is sending a cryptographic signature with the signing certificate in the SAML request to the IdP, and the IdP will match the cryptographic signature with the uploaded certificate to ensure the integrity of the request.</p> 
<h2>Conclusion</h2> 
<p>In this post, you learned the benefits of using IdP-initiated single sign-on. It helps centralize administration and lowers dependency on service provider applications. Also, you learned how to integrate an Amazon Cognito user pool with Microsoft Entra ID as an external SAML IdP using IdP-initiated SSO so your users can use their corporate ID to sign in to web or mobile applications. Also, you learned about how to enable signed authentication requests when using an SP-initiated flow and encrypting SAML responses for additional security between Cognito and the SAML IdP.</p> 
<p>&nbsp;<br />If you have feedback about this post, submit comments in the<strong> Comments</strong> section below. If you have questions about this post, <a href="https://console.aws.amazon.com/support/home" rel="noopener noreferrer" target="_blank">contact AWS Support</a>.</p> 
<p><strong>Want more AWS Security news? Follow us on <a href="https://twitter.com/AWSsecurityinfo" rel="noopener noreferrer" target="_blank" title="Twitter">Twitter</a>.</strong></p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Vishal Jakharia" class="aligncenter size-full wp-image-34187" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/05/14/vishjakh.jpg" width="120" /> 
  </div> 
  <p class="lb-h4">Vishal Jakharia</p> 
  <p>Vishal is a cloud support engineer based in New Jersey, USA. He is an Amazon Cognito subject matter expert who loves to work with customers and provide them solutions for implementing authentication and authorization. He helps customers migrate and build secure scalable architecture on the AWS Cloud.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Yungang Wu" class="aligncenter size-full wp-image-31490" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2023/10/20/Yungang_Wu.jpg" width="120" /> 
  </div> 
  <p class="lb-h4">Yungang Wu</p> 
  <p>Yungang is a senior cloud support engineer who specializes in the Amazon Cognito service. He helps AWS customers troubleshoot issues and suggests well-designed application authentication and authorization implementations.</p> 
 </div> 
</footer>
