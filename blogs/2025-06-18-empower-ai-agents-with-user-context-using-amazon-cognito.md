---
title: "Empower AI agents with user context using Amazon Cognito"
url: "https://aws.amazon.com/blogs/security/empower-ai-agents-with-user-context-using-amazon-cognito/"
date: "Wed, 18 Jun 2025 16:33:38 +0000"
author: "Abrom Douglas"
feed_url: "https://aws.amazon.com/blogs/security/tag/amazon-cognito/feed/"
---
<p><a href="https://aws.amazon.com/cognito/" rel="noopener noreferrer" target="_blank">Amazon Cognito</a> is a managed customer identity and access management (CIAM) service that enables seamless user sign-up and sign-in for web and mobile applications. Through <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html" rel="noopener noreferrer" target="_blank">user pools</a>, Amazon Cognito provides a user directory with strong authentication features, including <a href="https://fidoalliance.org/passkeys/" rel="noopener noreferrer" target="_blank">passkeys</a>, federation to external identity providers (IdPs), and <a href="https://aws.amazon.com/blogs/security/how-to-use-oauth-2-0-in-amazon-cognito-learn-about-the-different-oauth-2-0-grants/" rel="noopener noreferrer" target="_blank">OAuth 2.0 flows</a> for <a href="https://aws.amazon.com/blogs/security/how-to-monitor-optimize-and-secure-amazon-cognito-machine-to-machine-authorization/" rel="noopener noreferrer" target="_blank">secure machine-to-machine (M2M) authorization</a>.</p> 
<p>Amazon Cognito issues standard <a href="https://datatracker.ietf.org/doc/html/rfc7519" rel="noopener noreferrer" target="_blank">JSON Web Tokens (JWTs)</a> and supports the customization of identity and access tokens for user authentication by using the <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-pre-token-generation.html" rel="noopener noreferrer" target="_blank">pre token generation Lambda trigger</a>. Learn more about this in <a href="https://aws.amazon.com/blogs/security/how-to-customize-access-tokens-in-amazon-cognito-user-pools/" rel="noopener noreferrer" target="_blank">How to customize access tokens in Amazon Cognito user pools</a>. Amazon Cognito has extended token customization capabilities to support <a href="https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-cognito-access-token-m2m-authorization-flows/" rel="noopener noreferrer" target="_blank">access token customization for M2M</a>&nbsp;and the ability to <a href="https://aws.amazon.com/about-aws/whats-new/2025/04/amazon-cognito-context-machine-to-machine-flows/" rel="noopener noreferrer" target="_blank">pass metadata from the client during M2M authorization</a>.&nbsp;Application builders can use these two features to support multiple use cases, including customizing access tokens based on unique runtime policies, entitlements, environment, or passed metadata.&nbsp;This can simplify and enrich M2M authentication and authorization scenarios and opens up new possibilities for emerging use cases, such as identity and access management for <a href="https://aws.amazon.com/what-is/ai-agents/" rel="noopener noreferrer" target="_blank">AI agents</a>.</p> 
<p>This post demonstrates how Amazon Cognito enables AI agents to perform authorized actions on behalf of users through user-contextualized access tokens for OAuth 2.0-enabled resource servers. AI agents represent a class of autonomous services that require robust identity management and precise access control, especially when acting on behalf of users. By using the Amazon Cognito client credentials flow with access token customization, you can establish distinct identities for AI agents that carry critical information about their capabilities, scope of access, and intended use cases. This approach provides a foundation for more secure, auditable AI agent operations while maintaining clear boundaries around their authorized activities.</p> 
<p>The identity of an AI agent can be represented within Amazon Cognito as an <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-client-apps.html" rel="noopener noreferrer" target="_blank">app client</a>. The AI agent obtains an access token (JSON Web Token (JWT)) through an OAuth 2.0 client credentials grant. This JWT can be customized to contain claims that represent the authenticated human user whom the AI agent is acting on behalf of. This token can then be used to authorize access to other services that has established trust with the Amazon Cognito user pool by trusting the issuer and audience of the token. For example, this third-party service could be a claims processor, a travel agency service, or a scheduling service acting on behalf of a user. The focus of this post is on foundational building blocks using Amazon Cognito for AI agents and how to obtain a customized access token with user context.</p> 
<h2>Solution overview and reference architecture</h2> 
<p>Looking at an example architecture (Figure 1), a user signs in to a web or mobile application using an Amazon Cognito user pool, and tokens for the user are returned to the client. Here, the application could be a&nbsp;<a href="https://aws.amazon.com/blogs/machine-learning/create-an-end-to-end-serverless-digital-assistant-for-semantic-search-with-amazon-bedrock/" rel="noopener noreferrer" target="_blank">serverless digital assistant</a> using an <a href="https://aws.amazon.com/bedrock" rel="noopener noreferrer" target="_blank">Amazon Bedrock</a> agent that needs to gather and process data residing in a third-party cross-domain service. The AI agent obtains its own access token by performing an OAuth 2.0 client credentials grant while passing the user’s access token as context using the <code style="color: #000000;">aws_client_metadata</code>&nbsp;request parameter. The AI agent receives the user contextualized access token and calls an external, third-party, or cross-domain service that trusts the issuer and audience of the AI agent’s access token issued from an Amazon Cognito user pool. The cross-domain service can obtain the&nbsp;JSON Web Key Set (JWKS) to verify the token and extract claims presenting both the AI agent and most importantly, the underlying user. Authorization takes place within the cross-domain service using the claims of the customized access token and for fine-grain authorization, <a href="https://aws.amazon.com/verified-permissions/" rel="noopener noreferrer" target="_blank">Amazon Verified Permissions</a> is used. See Figure 1 for a detailed flow of this example.</p> 
<div class="wp-caption alignnone" id="attachment_38862" style="width: 1440px;">
 <img alt="Figure 1: AI agent identity reference architecture" class="size-full wp-image-38862" height="914" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/06/16/image-1-15.png" width="1430" />
 <p class="wp-caption-text" id="caption-attachment-38862">Figure 1: AI agent identity reference architecture</p>
</div> 
<ol> 
 <li>The user navigates to the application through the client.</li> 
 <li>There is no existing session or token for the user, so the user authentication flow with the Amazon Cognito user pool begins.</li> 
 <li>After a successful sign-in, Amazon Cognito returns access, ID, and refresh tokens to the client for the user.</li> 
 <li>As the user interacts with AI agent through the application, the client sends the user’s access token to an <a href="https://aws.amazon.com/api-gateway/" rel="noopener noreferrer" target="_blank">Amazon API Gateway</a> endpoint.</li> 
 <li>The API gateway integrates with the AI agent, which is using an <a href="https://aws.amazon.com/bedrock/agents/" rel="noopener noreferrer" target="_blank">Amazon Bedrock agent</a>. As an example, this can use several <a href="https://aws.amazon.com/pm/lambda" rel="noopener noreferrer" target="_blank">AWS Lambda</a> functions interacting with an <a href="https://aws.amazon.com/bedrock/knowledge-bases/" rel="noopener noreferrer" target="_blank">Amazon Bedrock&nbsp;Knowledge Base</a> or a <a href="https://aws.amazon.com/what-is/retrieval-augmented-generation/" rel="noopener noreferrer" target="_blank">Retrieval-Augmented Generation (RAG)</a> process.</li> 
 <li>The AI agent obtains its own access token from an Amazon Cognito user pool using an OAuth 2.0 client credentials grant. The user’s access token, obtained in step 1, is sent with the token request in the&nbsp;<code style="color: #000000;">aws_client_metadata</code>&nbsp;request parameter.</li> 
</ol> 
<blockquote>
 <p><strong>Note</strong>: You can use different Amazon Cognito user pools for user authentication and for agent (machine) authentication. This promotes separation and provides the ability to apply different settings and controls on each user pool if needed to meet security requirements.</p>
</blockquote> 
<ol start="7"> 
 <li>Amazon Cognito validates the client ID and secret from the AI agent and invokes the <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-pre-token-generation.html" rel="noopener noreferrer" target="_blank">pre token generation Lambda trigger</a> to customize the access token for the AI agent.</li> 
</ol> 
<blockquote>
 <p><strong>Note</strong>: Within the pre token generation Lambda trigger, the user’s access token is verified before returning a customized access token to the AI agent using the <a href="https://github.com/awslabs/aws-jwt-verify" rel="noopener noreferrer" target="_blank">aws-jwt-verify</a> library.</p>
</blockquote> 
<ol start="8"> 
 <li>The customized access token is returned to the AI agent, including custom claims representing the user.</li> 
 <li>The AI agent, using its own access token, calls the cross-domain service to perform the requested action on behalf of the user. For example, this can be a third-party reservation system or a photo sharing service.</li> 
 <li>The resource server in the cross-domain service verifies that the access token from the AI agent is valid. The resource server must be pre-configured to trust the user pool that issued the agent access token.</li> 
 <li>Coarse- and fine-grained authorization can happen either locally in the service code or using Verified Permissions.</li> 
 <li>A response from the cross-domain service flows back to the AI agent, if necessary.</li> 
 <li>A response from the AI agent to the user application or client is returned, if necessary.</li> 
 <li>Actions that take place throughout the flow are logged in <a href="https://aws.amazon.com/cloudtrail" rel="noopener noreferrer" target="_blank">AWS CloudTrail</a>, providing end-to-end logging and auditing.</li> 
</ol> 
<h2>Implementation details</h2> 
<p>Let’s take a deeper look into the three core components of this scenario:</p> 
<ol> 
 <li>The AI agent obtaining its own OAuth 2.0 access token</li> 
 <li>The Amazon Cognito pre token generation Lambda trigger used to enrich the AI agent’s access token with user context</li> 
 <li>The cross-domain resource server performing fine-grained authorization</li> 
</ol> 
<h3>AI agent</h3> 
<p></p>
<div class="wp-caption alignnone" id="attachment_38863" style="width: 1440px;">
 <img alt="Figure 2: AI agent obtaining a user access token from the frontend application through API Gateway" class="size-full wp-image-38863" height="888" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/06/16/image-2-13.png" width="1430" />
 <p class="wp-caption-text" id="caption-attachment-38863">Figure 2: AI agent obtaining a user access token from the frontend application through API Gateway</p>
</div>
<br /> 
<a href="https://aws.amazon.com/bedrock/agents/" rel="noopener noreferrer" target="_blank">Amazon Bedrock Agents</a> is used in this solution with a 
<a href="https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-bedrock-agents-custom-orchestration/" rel="noopener noreferrer" target="_blank">custom orchestration</a> configured to use Lambda. When the application interacts with the Amazon Bedrock agent, the custom orchestrator initiation begins with the agent passing the user’s access token to a Lambda function as part of the custom orchestration (shown in Figure 2). The Lambda function validates the user’s token to verify that it’s not expired and hasn’t been tampered with. This custom orchestrator begins the process for the agent to obtain its own OAuth access token and to access downstream and cross-domain resources on behalf of the user. The human user’s access token is included in the call from the application through the client. To learn more about Amazon Bedrock Agents custom orchestrator, see 
<a href="https://aws.amazon.com/blogs/machine-learning/getting-started-with-amazon-bedrock-agents-custom-orchestrator/" rel="noopener noreferrer" target="_blank">Getting started with Amazon Bedrock Agents custom orchestrator</a>. The following is an example of what a human user’s decoded access token provided through an API Gateway REST API might look like.
<p></p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
&nbsp;&nbsp;sub: "user-identity-4e4c-example-7cede8e609a2",
&nbsp;&nbsp;cognito:groups: 
&nbsp; &nbsp;&nbsp;[
&nbsp; &nbsp;&nbsp;"exampleChatApplicationAccess"
&nbsp; &nbsp;&nbsp;]
&nbsp;&nbsp;,
&nbsp;&nbsp;iss: https://cognito-idp.&lt;region&gt;.amazonaws.com/&lt;region&gt;_example,
&nbsp;&nbsp;version: 2,
&nbsp;&nbsp;client_id: "1example23456789",
&nbsp;&nbsp;origin_jti: "",
&nbsp;&nbsp;token_use: "access",
&nbsp;&nbsp;scope: "openid profile email",
&nbsp;&nbsp;auth_time: 499192140,
&nbsp;&nbsp;exp: 1445444940,
&nbsp;&nbsp;iat: 499192140,
&nbsp;&nbsp;jti: "",
&nbsp;&nbsp;username: "my-example-username"
}</code></pre> 
</div> 
<p>The following is a Node.js code sample that an AI agent can use to obtain its own access token from Amazon Cognito. This can be the Lambda function part of the custom orchestration for the Amazon Bedrock agent. Notice the <code style="color: #000000;">clientMetadata</code>&nbsp;variable being set, which will be passed to the Cognito&nbsp;<code style="color: #000000;">/token</code>&nbsp;endpoint using the <code style="color: #000000;">aws_client_metadata</code>&nbsp;request parameter. This request parameter is where the user’s access token is provided. In the following code example, you will find an attribute called <code style="color: #000000;">callerApp,</code>&nbsp;which is set to <code style="color: #000000;">ExampleChatApplication</code>, which serves as a unique identifier for the application. The callerApp value is preconfigured in the backend of the solution. This unique application identifier is included in the customized access token for the agent and used for additional authorization checks later. It’s a security best practice to use <a href="https://aws.amazon.com/secrets-manager/" rel="noopener noreferrer" target="_blank">AWS Secrets Manager</a> to store the client ID and client secret and obtain these credentials at runtime. As a security best practice, the user’s access token should be verified prior to passing it to the AI agent backend.</p> 
<div class="hide-language"> 
 <pre><code class="lang-javascript">async function getAccessToken() {
&nbsp;&nbsp; &nbsp;const clientId = 'exampleAiAgentClientId'; // use Secrets Manager
&nbsp;&nbsp; &nbsp;const clientSecret = 'exampleAiAgentClientSecret'; // use Secrets Manager
&nbsp;&nbsp; &nbsp;const tokenEndpoint = 'https://mydomain.auth.&lt;region&gt;.amazoncognito.com/oauth2/token';
&nbsp;&nbsp; &nbsp;const scope = 'crossDomainService/read userData/read';
&nbsp;&nbsp; &nbsp;const clientMetadata = '{"onBehalfOfToken":"&lt;HUMAN-USER-ACCESS-TOKEN&gt;", "callerApp":"ExampleChatApplication"}';
&nbsp;&nbsp;
&nbsp;&nbsp; &nbsp;const basicAuth = Buffer.from(`${clientId}:${clientSecret}`).toString('base64');
&nbsp;&nbsp;
&nbsp;&nbsp; &nbsp;const body = new URLSearchParams({
&nbsp;&nbsp; &nbsp; &nbsp;grant_type: 'client_credentials',
&nbsp;&nbsp; &nbsp; &nbsp;scope,
&nbsp;&nbsp; &nbsp; &nbsp;aws_client_metadata: clientMetadata
&nbsp;&nbsp; &nbsp;});
&nbsp;&nbsp;
&nbsp;&nbsp; &nbsp;const res = await fetch(tokenEndpoint, {
&nbsp;&nbsp; &nbsp; &nbsp;method: 'POST',
&nbsp;&nbsp; &nbsp; &nbsp;headers: {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;'Authorization': `Basic ${basicAuth}`,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;'Content-Type': 'application/x-www-form-urlencoded'
&nbsp;&nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp;body
&nbsp;&nbsp; &nbsp;});
&nbsp;&nbsp;
&nbsp;&nbsp; &nbsp;if (!res.ok) {
&nbsp;&nbsp; &nbsp; &nbsp;const error = await res.text();
&nbsp;&nbsp; &nbsp; &nbsp;throw new Error(`Token request failed: ${res.status} ${error}`);
&nbsp;&nbsp; &nbsp;}
&nbsp;&nbsp;
&nbsp;&nbsp; &nbsp;const { access_token } = await res.json();
&nbsp;&nbsp; &nbsp;console.log('Access Token:', access_token);
&nbsp;&nbsp;
&nbsp;&nbsp; &nbsp;return access_token;
&nbsp;&nbsp;}
&nbsp;&nbsp;
&nbsp;&nbsp;getAccessToken().catch(err =&gt; console.error('Error:', err.message));</code></pre> 
</div> 
<p>The access token for the AI agent is returned only if the client ID and secret are correct and the provided user access token is valid. However, before it’s returned, the AI agent’s access token is customized by the Amazon Cognito pre token generation Lambda trigger.</p> 
<h3>Amazon Cognito pre token generation Lambda trigger</h3> 
<div class="wp-caption alignnone" id="attachment_38864" style="width: 1439px;">
 <img alt="Figure 3: AI agent access token customization with Cognito pre token generation Lambda trigger" class="size-full wp-image-38864" height="883" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/06/16/image-3-14.png" width="1429" />
 <p class="wp-caption-text" id="caption-attachment-38864">Figure 3: AI agent access token customization with Cognito pre token generation Lambda trigger</p>
</div> 
<p>After the AI agent’s action calls the Amazon Cognito /token&nbsp;endpoint with a valid client ID and secret, Cognito invokes the pre token generation Lambda trigger. The following is an example Lambda function that takes the <code style="color: #000000;">aws_client_metadata</code>&nbsp;request parameter, which contains the access token of the user and the <code style="color: #000000;">callerApp</code>&nbsp;attribute that was defined while the user was authenticating. In the following Lambda function, the access token provided from the user is verified (shown in Figure 3). The&nbsp;<a href="https://github.com/awslabs/aws-jwt-verify" rel="noopener noreferrer" target="_blank">aws-jwt-verify</a>&nbsp;library is used to verify the token is not expired, the token has not been tampered with by verifying the signature, and it’s making sure that an access token was provided. The Lambda function is also pre-configured to accept user tokens from a specific issuer and audience, this protects against malicious context injection risks. This is also an opportunity to perform additional authorization. For example, check if the user is a member of certain groups.</p> 
<p>After the token is verified, the Lambda function customizes the access token to be returned to the AI agent.</p> 
<div class="hide-language"> 
 <pre><code class="lang-javascript">import { CognitoJwtVerifier } from "aws-jwt-verify";

// Initialize the JWT verifier to verify the user’s access token
// Provide the user pool ID, token use, and client ID 
const jwtVerifier = CognitoJwtVerifier.create({
&nbsp;&nbsp;userPoolId: process.env.USER_POOL_ID,  // user pool for user authentication
&nbsp;&nbsp;clientId: process.env.CLIENT_ID,
&nbsp; // groups: "exampleChatApplicationAccess",&nbsp;// optional group membership authorization
&nbsp;&nbsp;tokenUse:&nbsp;'access'
});

export const handler = async function(event, context) {
&nbsp;&nbsp;try {
&nbsp;&nbsp; &nbsp;const onBehalfOfToken = event.request.clientMetadata?.onBehalfOfToken || '';
    // It’s recommended that the provided “callerApp” value from the application is authorized for use with the app client for the AI agent
&nbsp;&nbsp; &nbsp;const callerApp = event.request.clientMetadata?.callerApp || '';

&nbsp;&nbsp; &nbsp;// The below console log will display the authenticated user’s JWT
&nbsp;&nbsp; &nbsp;// Keep this logging with caution in a production environment
&nbsp;&nbsp; &nbsp;console.log('Original event:', event);

&nbsp;&nbsp; &nbsp;// Verify the access token from the human user
&nbsp;&nbsp; &nbsp;// You could optionally also perform some authorization checks here as well
&nbsp;&nbsp; &nbsp;// Example: check for the membership of a group
&nbsp;&nbsp; &nbsp;let decodedJWT;
&nbsp;&nbsp; &nbsp;if (onBehalfOfToken) {
&nbsp;&nbsp; &nbsp; &nbsp;try {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;decodedJWT = await jwtVerifier.verify(onBehalfOfToken);
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;console.log('Decoded JWT:', decodedJWT);
&nbsp;&nbsp; &nbsp; &nbsp;} catch (err) {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;console.error('Token verification failed:', err);
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;throw new Error('Token verification failed');
&nbsp;&nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;}

&nbsp;&nbsp; &nbsp;// Create the onBehalfOf claim structure
&nbsp;&nbsp; &nbsp;const&nbsp;behalfOfClaim = decodedJWT ? {
&nbsp;&nbsp; &nbsp; &nbsp;sub: decodedJWT.sub,
&nbsp;&nbsp; &nbsp; &nbsp;username: decodedJWT.username,
&nbsp;&nbsp; &nbsp; &nbsp;groups: decodedJWT['cognito:groups'] || []
&nbsp;&nbsp; &nbsp;} : {};

&nbsp;&nbsp; &nbsp;// Customized token returned to client
&nbsp;&nbsp; &nbsp;event.response = {
&nbsp;&nbsp; &nbsp; &nbsp;"claimsAndScopeOverrideDetails": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"accessTokenGeneration": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"claimsToAddOrOverride": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"onBehalfOf": behalfOfClaim,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"callerApp": callerApp
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;};

&nbsp;&nbsp; &nbsp;return event;

&nbsp;&nbsp;} catch (error) {
&nbsp;&nbsp; &nbsp;console.error('Error in Lambda execution:', error);
&nbsp;&nbsp; &nbsp;throw error;
&nbsp;&nbsp;}
};</code></pre> 
</div> 
<p>Notice in the preceding Lambda function that two custom claims are being dynamically created within the <code style="color: #000000;">event.response: onBehalfOf</code> and <code style="color: #000000;">callerApp</code>. The <code style="color: #000000;">onBehalfOf</code>&nbsp;claim contains nested claims that were extracted from the human user’s access token. The <code style="color: #000000;">callerApp</code> is carried forward from the frontend application and provided alongside the user access token. It’s recommended for the <code style="color: #000000;">callerApp</code> value to also be verified against some custom logic to add additional layer of protection. The return AI agent’s access token would look like the following JWT.</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">
{&nbsp;&nbsp; &nbsp;
	"sub": "agent-identity-4e4c-example-7cede8e609a2",
	"onBehalfOf": {
		"sub": "user-identity-4e4c-example-7cede8e609a2",
		"username": "my-example-username",
		"groups": [
			"readaccess"&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;
				]&nbsp;&nbsp; &nbsp;
		},&nbsp;&nbsp; &nbsp;
		"iss": "https://cognito-idp..amazonaws.com/_example",
		"version": 2,
		"client_id": "1example23456789",
		"callerApp": "ExampleChatApplication",
		"token_use": "access",
		"scope": "crossDomainService123/read userData/read",
		"auth_time": 499192140,
		"exp": 1445444940,
		"iat": 499192140,
		"jti": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
}
</code></pre> 
</div> 
<h3>Cross-domain resource server authorization check</h3> 
<p>At this point, shown in Figure 4, the human user has successfully authenticated to the web application, the human user’s access token was sent as context to the backend, an AI agent obtained its own customized access token containing the human user context, and now the agent is ready to call an external cross-domain service.<br /> </p>
<div class="wp-caption alignnone" id="attachment_38945" style="width: 1440px;">
 <img alt="Figure 4: Cross-domain resource server performing fine-grained authorization with Amazon Verified Permissions" class="size-full wp-image-38945" height="876" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/06/16/Figure-4-3.png" width="1430" />
 <p class="wp-caption-text" id="caption-attachment-38945">Figure 4: Cross-domain resource server performing fine-grained authorization with Amazon Verified Permissions</p>
</div>
<br /> As shown in Figure 4, the cross-domain service is the resource server and therefore needs to perform an authorization check. For this example, we’ll keep things straightforward and make sure that three core things are verified:
<p></p> 
<ol> 
 <li>The AI agent’s OAuth access token is valid</li> 
 <li>The AI agent is authorized to access this service</li> 
 <li>The AI agent is authorized to interact with the user data</li> 
</ol> 
<p>Depending on your use case and requirements, you might also need to verify that the user’s consent has been obtained prior to the AI agent acting on their behalf. Ultimately, you want to verify that the AI agent can access a user’s data on their behalf and only for the purpose for which consent has been provided by the user.</p> 
<p>For the token verification, use the&nbsp;<a href="https://github.com/awslabs/aws-jwt-verify" rel="noopener noreferrer" target="_blank">aws-jwt-verify</a>&nbsp;library again. The following is a Node.js example to verify the AI agent’s access token.</p> 
<div class="hide-language"> 
 <pre><code class="lang-javascript">import { CognitoJwtVerifier } from "aws-jwt-verify";

// add custom logic to verify that AI agent is authorized to perform this action on behalf of the user

// Verifier that expects valid access tokens:
const verifier = CognitoJwtVerifier.create({
&nbsp;&nbsp;userPoolId: "&lt;user_pool_id&gt;", // user pool for AI agent authentication
&nbsp;&nbsp;tokenUse: "access",
&nbsp;&nbsp;clientId: "&lt;client_id&gt;",
});

try {
&nbsp;&nbsp;const payload = await verifier.verify(
&nbsp;&nbsp; &nbsp;"eyJraWQeyJhdF9oYXNoIjoidk..." //this will be the AI agent's access token
&nbsp;&nbsp;);
&nbsp;&nbsp;console.log("Token is valid. Payload:", payload);
} catch {
&nbsp;&nbsp;console.log("Token not valid!");
}</code></pre> 
</div> 
<h3>Fine-grained authorization with Verified Permissions</h3> 
<p>As a security best practice, the zero trust principle of enforcing fine-grained identity-based authorization should take place using Verified Permissions. The preceding Node.js code sample is a basic validation of the AI agents access token that can happen within the application logic. Instead of keeping authorization logic within the resource server, you can use Verified Permissions to offload the authorization policies to a managed service. The following is an example <a href="https://www.cedarpolicy.com/en/learn" rel="noopener noreferrer" target="_blank">Cedar policy</a> for this use case.</p> 
<div class="hide-language"> 
 <pre><code class="lang-cedar">permit(
&nbsp;&nbsp; &nbsp;principal == Agent::"agent-identity-4e4c-example-7cede8e609a2",
&nbsp;&nbsp; &nbsp;action == Action::"readOnly",
&nbsp;&nbsp; &nbsp;resource == Resource::"crossDomainService123::userData"
)
when {
&nbsp;&nbsp; &nbsp;resource.scope == Scope::"crossDomainService123/read" &amp;&amp;
&nbsp;&nbsp; &nbsp;resource.owner == User::" user-identity-4e4c-example-7cede8e609a2" &amp;&amp;
&nbsp;&nbsp; &nbsp;context.onBehalfOf.sub == " user-identity-4e4c-example-7cede8e609a2" &amp;&amp;
&nbsp;&nbsp; &nbsp;context.callerApp == "ExampleChatApplication"
};</code></pre> 
</div> 
<p>With the preceding Cedar policy example, you are permitting the AI agent to read <code style="color: #000000;">userData</code> from the <code style="color: #000000;">crossDomainService123</code> resource. This is only permitted when the AI agent’s access token contains the <code style="color: #000000;">crossDomainService/read</code>&nbsp;scope and when the resource owner and the <code style="color: #000000;">onBehalfOf</code>&nbsp;user (from the access token) are the same—the human user in this case. There’s also an additional when&nbsp;clause in the policy to make sure that this interaction initiated from <code style="color: #000000;">ExampleChatApplication</code>.</p> 
<p>The cross-domain resource server would use the AI agent’s access token and call the Verified Permissions&nbsp;<code style="color: #000000;">IsAuthorizedWithToken</code>&nbsp;API. To learn more, see <a href="https://aws.amazon.com/blogs/security/simplify-fine-grained-authorization-with-amazon-verified-permissions-and-amazon-cognito/" rel="noopener noreferrer" target="_blank">Simplify fine-grained authorization with Amazon Verified Permissions and Amazon Cognito</a>.</p> 
<p>The following is a Node.js example using the <code style="color: #000000;">IsAuthorizedWithToken</code>&nbsp;API from Verified Permissions using the <a href="https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/Package/-aws-sdk-client-verifiedpermissions/" rel="noopener noreferrer" target="_blank">AWS SDK for JavaScript v3</a>.</p> 
<div class="hide-language"> 
 <pre><code class="lang-javascript">import { VerifiedPermissionsClient, IsAuthorizedWithTokenCommand } from "@aws-sdk/client-verifiedpermissions";

const client = new VerifiedPermissionsClient({ region: "&lt;region&gt;" });

// Dynamically provided token 
const jwtToken = "eyJraWQiOiJrMWtleSIsInR..."; //AI agent's access token

async function checkAccess() {
&nbsp;&nbsp;const input = {
&nbsp;&nbsp; &nbsp;policyStoreId: "ps-abc123example", // your AVP policy store ID
&nbsp;&nbsp; &nbsp;accessToken: jwtToken,
&nbsp;&nbsp; &nbsp;action: {
&nbsp;&nbsp; &nbsp; &nbsp;actionType: "Action",
&nbsp;&nbsp; &nbsp; &nbsp;actionId: "readOnly"
&nbsp;&nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp;resource: {
&nbsp;&nbsp; &nbsp; &nbsp;entityType: "crossDomainService123",
&nbsp;&nbsp; &nbsp; &nbsp;entityId: "userData"
&nbsp;&nbsp; &nbsp;}
&nbsp;&nbsp;};

&nbsp;&nbsp;const command = new IsAuthorizedWithTokenCommand(input);

&nbsp;&nbsp;try {
&nbsp;&nbsp; &nbsp;const response = await client.send(command);
&nbsp;&nbsp; &nbsp;console.log("Authorization Decision:", response.decision);
&nbsp;&nbsp;} catch (err) {
&nbsp;&nbsp; &nbsp;console.error("Authorization error:", err);
&nbsp;&nbsp;}
}</code></pre> 
</div> 
<p>Based on the preceding examples of the AI agent’s access token (with user context), the Cedar policy, and the&nbsp;<code style="color: #000000;">IsAuthorizedWithToken</code>&nbsp;API call, the resource server would get an Allow decision for this action to take place. The following is an example of the authorization decision response.</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
&nbsp;&nbsp; &nbsp;"decision": "Allow",
&nbsp;&nbsp; &nbsp;"determiningPolicies": [{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"determiningPolicyId": "ps-abc123example"
&nbsp;&nbsp; &nbsp;}],
&nbsp;&nbsp; &nbsp;"errors": []
}</code></pre> 
</div> 
<p>Before this policy can be evaluated, you must <a href="https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/schema-edit.html" rel="noopener noreferrer" target="_blank">define a schema</a> that includes the relevant entity types (Agent, User, Resource, Scope, and so on), and create corresponding entities in your policy store that match the IDs used in the policy and request.</p> 
<p>Bringing it all together, the requested data from the AI agent, on behalf of the user, is returned from the cross-domain service to the AI agent. This additional data can now be used within the context of the AI agent workload. The entire solution can be used for a chat application, such as the one described in <a href="https://aws.amazon.com/blogs/machine-learning/protect-sensitive-data-in-rag-applications-with-amazon-bedrock/" rel="noopener noreferrer" target="_blank">Protect sensitive data in RAG applications with Amazon Bedrock</a>.</p> 
<h2>Conclusion</h2> 
<p>Amazon Cognito M2M access token customization and support for passing client metadata provides you the extensibility to solve complex use cases and enables emerging ones like AI agent identity and access management. For example, passing contextual client metadata and customizing access tokens at runtime can help software as a service (SaaS) and multi-tenant service providers scale to an unlimited number of resource servers, because these can be dynamically determined at runtime. As organizations increasingly explore the use of AI agents, having a secure, scalable identity management solution becomes crucial for maintaining control and accountability. By using these new features, you can build more secure and scalable solutions with Amazon Cognito to prepare for the future of autonomous AI agent use cases.</p> 
<p>Use the comments section to leave feedback about this post. If you have questions about this post, start a new thread on&nbsp;<a href="https://repost.aws/tags/TAkhAE7QaGSoKZwd6utGhGDA/amazon-cognito" rel="noopener noreferrer" target="_blank">Amazon Cognito re:Post</a> or <a href="https://console.aws.amazon.com/support/home" rel="noopener noreferrer" target="_blank">contact AWS Support</a>.</p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Abrom Douglas" class="aligncenter size-full wp-image-32242" height="157" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/06/16/abrom-headshot.jpg" width="113" /> 
  </div> 
  <h3 class="lb-h4">Abrom Douglas III</h3> 
  <p>Abrom is a Senior Solutions Architect within AWS Identity with nearly 20 years of software engineering and security experience, specializing in the identity and access management space. He loves speaking with customers about how identity and access management can provide secure outcomes that enable both business and technology initiatives. In his free time, he enjoys cheering for Arsenal FC, photography, travel, volunteering, and competing in duathlons.</p> 
  <p></p>
 </div> 
</footer>
