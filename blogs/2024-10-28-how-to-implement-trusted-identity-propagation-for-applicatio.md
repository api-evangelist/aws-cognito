---
title: "How to implement trusted identity propagation for applications protected by Amazon Cognito"
url: "https://aws.amazon.com/blogs/security/how-to-implement-trusted-identity-propagation-for-applications-protected-by-amazon-cognito/"
date: "Mon, 28 Oct 2024 17:43:24 +0000"
author: "Joseph de Clerck"
feed_url: "https://aws.amazon.com/blogs/security/tag/amazon-cognito/feed/"
---
<p><a href="https://aws.amazon.com" rel="noopener" target="_blank">Amazon Web Services (AWS)</a> recently released <a href="https://aws.amazon.com/iam/identity-center/" rel="noopener" target="_blank">AWS IAM Identity Center</a> <a href="https://docs.aws.amazon.com/singlesignon/latest/userguide/trustedidentitypropagation.html" rel="noopener" target="_blank">trusted</a> identity propagation to create identity-enhanced IAM role sessions when requesting access to AWS services as well as to <a href="https://docs.aws.amazon.com/singlesignon/latest/userguide/using-apps-with-trusted-token-issuer.html" rel="noopener" target="_blank">trusted</a> token issuers. These two features can help customers build custom applications on top of AWS, which requires fine-grained access to data analytics-focused AWS services such as <a href="https://aws.amazon.com/q/business/" rel="noopener" target="_blank">Amazon Q Business</a>, <a href="https://aws.amazon.com/athena" rel="noopener" target="_blank">Amazon Athena</a>, and <a href="https://aws.amazon.com/lake-formation/" rel="noopener" target="_blank">AWS Lake Formation</a>, and <a href="https://aws.amazon.com/s3/access-grants/" rel="noopener" target="_blank">Amazon S3 Access Grants</a>. You can use AWS services compatible with trusted identity propagation to grant access to users and groups belonging to IAM Identity Center instead of solely relying on <a href="https://aws.amazon.com/iam/" rel="noopener" target="_blank">AWS Identity and Access Management (IAM)</a> role permissions. With a trusted token issuer, you can propagate identities that you have authenticated in your custom application to the underlying AWS services. In the case of an Amazon Q Business application, you can create a different web experience or integrate an Amazon Q Business application as an assistant into an existing web application to help your workforce.</p> 
<p>These two features rely on the OAuth 2.0 protocol to exchange user information. For the identity to be consumable by AWS services, your custom application’s identity provider needs to be able to issue OAuth 2.0 tokens for your users.</p> 
<p><a href="https://aws.amazon.com/blogs/storage/how-to-develop-a-user-facing-data-application-with-iam-identity-center-and-s3-access-grants/" rel="noopener" target="_blank">This blog post</a> from November 2023 covers how to interconnect with an OAuth 2.0 compatible identity provider such as Microsoft Entra ID, Okta, or PingFederate.</p> 
<p>In this post, I show you how to use an <a href="https://aws.amazon.com/cognito/" rel="noopener" target="_blank">Amazon Cognito</a> user pool as a trusted token issuer for IAM Identity Center. You will also learn how to use IAM Identity Center as a <a href="https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-identity-federation.html" rel="noopener" target="_blank">federated identity provider</a> for a Cognito user pool to provide a seamless authentication flow for your IAM Identity Center custom applications. Note that this content doesn’t cover building a custom application for Amazon Q Business. If needed, you can find more details in <a href="https://aws.amazon.com/blogs/machine-learning/build-a-custom-ui-for-amazon-q-business/" rel="noopener" target="_blank">Build a custom UI for Amazon Q Business</a>.</p> 
<h2 id="iam-identity-center-concepts">IAM Identity Center concepts</h2> 
<p>IAM Identity Center is the recommended service for managing your workforce’s access to AWS applications. It supports multiple identity sources, such as an internal directory, external Active Directory, or a SAML-compliant identity provider (IdP) with optional <a href="https://scim.cloud/" rel="noopener" target="_blank">SCIM</a> integration.</p> 
<p>With <a href="https://docs.aws.amazon.com/singlesignon/latest/userguide/trustedidentitypropagation.html" rel="noopener" target="_blank">trusted identity propagation</a>, a user can sign in to an application, and that application can pass the user’s identity context when creating an identity-enhanced AWS session to access data in AWS services. Because access is now tied to the user’s identity in IAM Identity Center, AWS services can rely on both the IAM role permissions to authorize access as well as the user’s granted scopes and group memberships.</p> 
<p><a href="https://docs.aws.amazon.com/singlesignon/latest/userguide/using-apps-with-trusted-token-issuer.html" rel="noopener" target="_blank">Trusted token issuers</a> are OAuth 2.0 authorization servers that create signed tokens and enable you to use trusted identity propagation with applications that authenticate outside of AWS. With trusted token issuers, you can authorize these applications to make requests on behalf of their users to access AWS managed applications. The trusted token issuers feature is completely independent from the authentication feature of IAM Identity Center and doesn’t need to be the same identity provider as is used for authenticating into IAM Identity Center.</p> 
<p>When performing a token exchange, the token must contain an attribute that maps to an <em>existing</em> user in IAM Identity Center, such as an email address or external ID. A token can be exchanged only once.</p> 
<p>On the other side, an Amazon Cognito user pool is a user directory and an OAuth 2.0 compliant identity provider (IdP). From the perspective of your application, a Cognito user pool is an OpenID Connect (OIDC) IdP. Your application users can either sign in directly through a user pool, or they can federate through a third-party IdP. When you federate Cognito to a SAML IdP, or OIDC IdPs, your user pool acts as a bridge between multiple identity providers and your application.</p> 
<h2 id="overview-of-solution">Overview of solution</h2> 
<p>The solution architecture includes the following elements and steps and is depicted in Figure 1.</p> 
<ol type="1"> 
 <li><strong>The custom application</strong>: The custom application provides access to the Amazon Q Business application through APIs. Users are authenticated using Amazon Cognito as an OAuth 2.0 IdP.</li> 
 <li><strong>Amazon Q Business</strong>: The Amazon Q Business application requires identity-enhanced AWS credentials issued by <a href="https://aws.amazon.com/sts" rel="noopener" target="_blank">AWS Security Token Service (AWS STS)</a> to authorize requests from the custom application.</li> 
 <li><strong>AWS STS</strong>: STS issues identity-enhanced AWS credentials to the custom application through the <code style="color: #000000;">setContext</code> and <code style="color: #000000;">AssumeRole</code> API calls. <code style="color: #000000;">SetContext</code> requires the user’s identity context to be passed from a JSON web token (JWT) issued by IAM Identity Center.</li> 
 <li><strong>IAM Identity Center</strong>: To issue a JWT, IAM Identity Center requires the custom application to perform a token exchange operation from a trusted IAM role and a trusted token issuer (Cognito).</li> 
 <li><strong>Amazon Cognito user pool</strong>: The user pool authenticates users into the custom application. The user pool uses SAML federation to delegate authentication to Identity Center. Users are automatically created in the user pool when the federated authentication is successful. The user pool returns a JWT to the custom application.</li> 
 <li><strong>SAML-based customer managed application</strong> (when IAM Identity Center is acting as a SAML identity provider): By using the SAML customer managed application in IAM Identity Center, you can delegate the authentication from Cognito to IAM Identity Center. One benefit of using IAM Identity Center is to help guarantee that the user exists in IAM Identity Center before authenticating to Cognito, as long as IAM Identity Center is the only way to authenticate to the client application. User existence is a requirement to perform the token exchange operation.</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_36178" style="width: 1251px;">
 <img alt="Figure 1: Solution architecture" class="size-full wp-image-36178" height="692" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/10/23/Implement-trusted-identity-propagation1.png" width="1241" />
 <p class="wp-caption-text" id="caption-attachment-36178">Figure 1: Solution architecture</p>
</div> 
<h2 id="walkthrough">Walkthrough</h2> 
<p>The focus of this post is steps 3–6 of the architecture, which follow a three-step approach.</p> 
<ol type="1"> 
 <li>Creation and initial configuration of the Amazon Cognito user pool and domain</li> 
 <li>Configuration of the OAuth integration for trusted identity propagation</li> 
 <li>Configuration of the SAML federation trust between IAM Identity Center and Cognito</li> 
</ol> 
<h3 id="prerequisites">Prerequisites</h3> 
<p>For this walkthrough, you need the following prerequisites:</p> 
<ul> 
 <li>An <a href="https://signin.aws.amazon.com/signin?redirect_uri=https%3A%2F%2Fportal.aws.amazon.com%2Fbilling%2Fsignup%2Fresume&amp;client_id=signup" rel="noopener" target="_blank">AWS account</a></li> 
 <li>An organization instance of <a href="https://docs.aws.amazon.com/singlesignon/latest/userguide/get-set-up-for-idc.html" rel="noopener" target="_blank">IAM Identity Center</a></li> 
 <li>An Amazon Q Business application <a href="https://docs.aws.amazon.com/amazonq/latest/qbusiness-ug/setting-up.html" rel="noopener" target="_blank">setup</a> (optional)</li> 
 <li>Administrative access to the AWS account</li> 
 <li><a href="https://aws.amazon.com/cli" rel="noopener" target="_blank">AWS Command Line Interface (AWS CLI)</a></li> 
</ul> 
<h3 id="step-1-create-the-cognito-user-pool-the-user-pool-domain-and-the-user-pool-client">Step 1: Create the Cognito user pool, the user pool domain and the user pool client</h3> 
<p>The following bash script sets up the Amazon Cognito user pool, user pool domain, and user pool client and outputs the issuer URL and audience that you need to set up IAM Identity Center.</p> 
<blockquote>
 <p><strong>Note</strong>: The Cognito user pool domain prefix must be unique across all AWS accounts for a given AWS Region. Replace <code style="color: #ff0000;"><em>&lt;demo-tti&gt;</em></code> with a unique prefix for your user pool domain.</p>
</blockquote> 
<div class="hide-language"> 
 <pre><code class="lang-text">#!/bin/bash
export AWS_PAGER="" # Disable sending response to less
export USER_POOL_NAME=BlogTrustedTokenIssuer
export COGNITO_DOMAIN_PREFIX=<span style="color: #ff0000;"><em>&lt;demo-tti&gt;</em></span> # Must be unique
# Create the user pool
USER_POOL_ID=$(aws cognito-idp create-user-pool \
  --pool-name ${USER_POOL_NAME} \
  --alias-attributes email \
  --schema Name=email,Required=true,Mutable=true,AttributeDataType=String \
  --query "UserPool.Id" \
  --admin-create-user-config AllowAdminCreateUserOnly=True \
  --output text)
# Create the user pool domain
aws cognito-idp create-user-pool-domain \
  --domain ${COGNITO_DOMAIN_PREFIX} \
  --user-pool-id ${USER_POOL_ID}
# Create the user pool client
AUDIENCE=$(aws cognito-idp create-user-pool-client \
  --user-pool-id ${USER_POOL_ID} \
  --client-name TTI \
  --explicit-auth-flows ALLOW_REFRESH_TOKEN_AUTH ALLOW_USER_SRP_AUTH \
  --allowed-o-auth-flows-user-pool-client \
  --allowed-o-auth-scopes openid email profile \
  --allowed-o-auth-flows code \
  --callback-urls "http://localhost:8080" \
  --query "UserPoolClient.ClientId" \
  --output text )
ISSUER_URL="https://cognito-idp.${AWS_REGION}.amazonaws.com/${USER_POOL_ID}"
</code></pre> 
</div> 
<h3 id="step-2-create-the-oauth-integration-for-trusted-identity-propagation">Step 2: Create the OAuth integration for trusted identity propagation</h3> 
<p>To create the OAuth integration, you need to set up a trusted token issuer and configure the OAuth customer managed application.</p> 
<h4 id="configure-a-trusted-token-issuer">Configure a trusted token issuer</h4> 
<p>Start by configuring IAM Identity Center to trust tokens issued by the Amazon Cognito user pool.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">INSTANCE_ARN=$(aws sso-admin list-instances --output text --query "Instances[*].InstanceArn")
TRUSTED_TOKEN_ISSUER_ARN=$(aws sso-admin create-trusted-token-issuer \
  --name cognito \
  --instance-arn $INSTANCE_ARN \
  --trusted-token-issuer-configuration "OidcJwtConfiguration={IssuerUrl=$ISSUER_URL,ClaimAttributePath=email,IdentityStoreAttributePath=emails.value,JwksRetrievalOption=OPEN_ID_DISCOVERY}" \
  --trusted-token-issuer-type OIDC_JWT\
  --output text \
  --query "TrustedTokenIssuerArn"
  )
</code></pre> 
</div> 
<h4 id="configure-oauth-customer-managed-application">Configure OAuth customer managed application</h4> 
<p>Create the OAuth customer managed application, which will allow your AWS account to exchange tokens issued for the Cognito user pool client.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text"># Create the OAuth customer managed application
OAUTH_APPLICATION_ARN=$(aws sso-admin create-application \
    --instance-arn $INSTANCE_ARN \
    --application-provider-arn "arn:aws:sso::aws:applicationProvider/custom" \
    --name DemoApplication \
    --output text \
    --query "ApplicationArn")

# Disable using explicit assignment for user access to this application
aws sso-admin put-application-assignment-configuration \
    --application-arn $OAUTH_APPLICATION_ARN \
    --no-assignment-required
# Allow token exchange process for tokens issuer by the trusted token issuer
cat &lt;&lt; EOF &gt; /tmp/grant.json
{
  "JwtBearer": {
    "AuthorizedTokenIssuers": [
      {
        "TrustedTokenIssuerArn": "$TRUSTED_TOKEN_ISSUER_ARN",
        "AuthorizedAudiences": ["$AUDIENCE"]
      }
    ]
  }
}
EOF
aws sso-admin put-application-grant \
    --application-arn $OAUTH_APPLICATION_ARN \
    --grant-type "urn:ietf:params:oauth:grant-type:jwt-bearer" \
    --grant file:///tmp/grant.json
# Allow use of this application for Q Business applications
for scope in qbusiness:messages:access qbusiness:messages:read_write qbusiness:conversations:access qbusiness:conversations:read_write qbusiness:qapps:access; do
    aws sso-admin put-application-access-scope \
        --application-arn $OAUTH_APPLICATION_ARN \
        --scope $scope
done
# Allow this AWS Account Id to invoke the API to exchange token (CreateTokenWithIAM)
AWS_ACCOUNTID=$(aws sts get-caller-identity --output text --query "Account")
cat &lt;&lt; EOF &gt; /tmp/authentication-method.json
{
  "Iam": {
    "ActorPolicy": {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": { 
            "AWS": "${AWS_ACCOUNTID}" 
          },
          "Action": "sso-oauth:CreateTokenWithIAM",
          "Resource": "$OAUTH_APPLICATION_ARN",
        }
      ]
    }
  }
}
EOF
aws sso-admin put-application-authentication-method \
  --application-arn $OAUTH_APPLICATION_ARN \
  --authentication-method file:///tmp/authentication-method.json \
  --authentication-method-type IAM
</code></pre> 
</div> 
<h3 id="step-3-create-the-saml-federation-trust-between-iam-identity-center-and-cognito">Step 3: Create the SAML federation trust between IAM Identity Center and Cognito</h3> 
<p>The SAML integration between IAM Identity Center and Amazon Cognito is useful when your source of identity is IAM Identity Center. In this scenario, SAML integration helps ensure that users will authenticate with IAM Identity Center credentials before being authenticated to your Cognito user pool. When using federated identities, the Cognito user pool will automatically create user profiles, so you don’t need to maintain the user directory separately.</p> 
<h4 id="configure-iam-identity-center">Configure IAM Identity Center</h4> 
<ol type="1"> 
 <li>Sign in to the AWS Management Console and navigate to IAM Identity Center.</li> 
 <li>Choose <strong>Applications</strong> from the navigation pane.</li> 
 <li>Choose <strong>Add application</strong>.</li> 
 <li>Select <strong>I have an application I want to set up</strong>, select <strong>SAML 2.0</strong>, and then choose <strong>Next</strong>.</li> 
 <li>For <strong>Display name</strong>, enter <code style="color: #000000;">DemoSAMLApplication</code>.</li> 
 <li>Copy the IAM Identity Center SAML metadata file URL for later use.</li> 
 <li>For <strong>Application properties</strong>, leave both fields blank.</li> 
 <li>For <strong>Application ACS URL,</strong> enter <code style="color: #000000;">https://<span style="color: #ff0000;"><em>&lt;CognitoUserPoolDomain&gt;</em></span>.auth.<span style="color: #ff0000;"><em>&lt;AWS_REGION&gt;</em></span>.amazoncognito.com/saml2/idpresonse</code>. <p> Replace <code style="color: #ff0000;"><em>&lt;CognitoUserPoolDomain&gt;</em></code> with the domain you chose in Step 1 and <code style="color: #ff0000;"><em>&lt;AWS_REGION&gt;</em></code> with the Region in which you created the Cognito user pool.</p></li> 
 <li>For <strong>Application SAML audience</strong>, enter <code style="color: #000000;">urn:amazon:cognito:sp:<span style="color: #ff0000;"><em>&lt;CognitoUserPoolId&gt;</em></span></code>. <p> Replace <code style="color: #ff0000;"><em>&lt;CognitoUserPoolId&gt;</em></code> with the ID of the Cognito user pool you created in Step 1.</p></li> 
 <li>Choose <strong>Submit</strong>.</li> 
</ol> 
<h4 id="configure-mapping-attributes">Configure mapping attributes</h4> 
<ol type="1"> 
 <li>Choose <strong>Actions</strong> and select <strong>Edit attribute mappings</strong>.</li> 
 <li>Enter <code style="color: #000000;">${user:email}</code> for the field <strong>Maps to this string value or user attribute in IAM Identity Center</strong>.</li> 
 <li>Select <strong>Persistent</strong> for <strong>Format</strong>.</li> 
 <li>Choose <strong>Save changes</strong>.</li> 
</ol> 
<h4 id="configure-cognito-user-pool">Configure Cognito user pool</h4> 
<ol type="1"> 
 <li>Navigate to the Amazon Cognito console and choose <strong>User pools</strong> from the navigation pane.</li> 
 <li>Select the user pool created in Step 1.</li> 
 <li>Choose the <strong>Sign-in experience</strong> tab.</li> 
 <li>Under <strong>Federated identity provider sign-in</strong>, choose <strong>Add identity provider</strong>.</li> 
 <li>Select <strong>SAML</strong>.</li> 
 <li>Under <strong>Provider name</strong>, enter <code style="color: #000000;">IAMIdentityCenter</code>.</li> 
 <li>Under <strong>Metadata document source</strong>, select <strong>Enter metadata document endpoint URL</strong> and paste the URL copied from step 6 of <strong>Configure IAM Identity Center</strong></li> 
 <li>Under <strong>SAML attribute</strong>, enter <code style="color: #000000;">Subject</code>.</li> 
 <li>Choose <strong>Add Identity Provider</strong>.</li> 
</ol> 
<h4 id="configure-app-integration-to-use-iam-identity-center">Configure app integration to use IAM Identity Center</h4> 
<ol type="1"> 
 <li>Choose the <strong>App integration</strong> tab.</li> 
 <li>Under <strong>App clients and analytics</strong>, choose <strong>TTI</strong>.</li> 
 <li>Under <strong>Hosted UI</strong>, choose <strong>Edit</strong>.</li> 
 <li>For <strong>Identity providers</strong>, select <strong>IAMIdentityCenter</strong>.</li> 
 <li>Choose <strong>Save changes</strong>.</li> 
</ol> 
<h3 id="architecture-diagram">Architecture diagram</h3> 
<p>Figure 2 shows the authentication flow from the user connecting to the web application up to the chat interaction with Amazon Q Business APIs.</p> 
<blockquote>
 <p><strong>Note</strong>: The AWS resources can be in the same Region, but it’s not required for Amazon Cognito and IAM Identity Center.</p>
</blockquote> 
<ol type="1"> 
 <li>The application redirects the user to Amazon Cognito for authentication.</li> 
 <li>Cognito redirects the user to IAM Identity Center for authentication.</li> 
 <li>Cognito parses the SAML assertion from IAM Identity Center.</li> 
 <li>Cognito returns a JWT to the application.</li> 
 <li>The application exchanges the token with IAM Identity Center.</li> 
 <li>The application assumes an IAM role and sets the context using the IAM Identity Center token.</li> 
 <li>The application invokes the Amazon Q Business APIs with the context-aware STS session.</li> 
</ol> 
<div class="wp-caption aligncenter" id="attachment_36179" style="width: 950px;">
 <img alt="Figure 2: Authentication flow" class="size-full wp-image-36179" height="610" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/10/23/Implement-trusted-identity-propagation2.png" width="940" />
 <p class="wp-caption-text" id="caption-attachment-36179">Figure 2: Authentication flow</p>
</div> 
<h3 id="clean-up">Clean up</h3> 
<p>To avoid future charges to your AWS account, delete the resources you created in this walkthrough. The resources include:</p> 
<ul> 
 <li>The Amazon Cognito user pool (deleting this will also delete sub resources such as the user pool client)</li> 
 <li>The SAML application in IAM Identity Center</li> 
 <li>The OAuth application in IAM Identity Center</li> 
 <li>The trusted token issuer configuration in IAM Identity Center</li> 
</ul> 
<h2 id="conclusion">Conclusion</h2> 
<p>In this post, we demonstrated how to implement trusted identity propagation for applications that are protected by Amazon Cognito. We also showed you how to authenticate Cognito users with IAM Identity Center to help ensure that users are authenticating using the correct mechanisms and policies and to reduce the operational burden of managing the Cognito directory by automatically provisioning users as they sign in.</p> 
<p>Using Amazon Cognito as a trusted token issuer is useful when your application is already secured with a user pool, and you want to implement data functionalities such as Amazon Q Business chat capabilities or secure access to S3 buckets using S3 Access Grants.</p> 
<p>If your users are authenticating with different identity providers, the solution in this post can reduce the work needed for identity integration by enabling you to add multiple identity providers to a single user pool. By using this solution, you will need to configure the trusted token issuer in IAM Identity Center only for Amazon Cognito and not for every token provider.</p> 
<p>This walkthrough doesn’t include a demo web application because I wanted to dive into the integration of IAM Identity Center and Amazon Cognito. I recommend reading <a href="https://aws.amazon.com/blogs/machine-learning/build-a-custom-ui-for-amazon-q-business/" rel="noopener" target="_blank">Build a custom UI for Amazon Q Business</a>, which shows you how to implement a custom user interface for an Amazon Q Business application using Amazon Cognito for user authentication.</p> 
<p>Because trusted identity propagation is becoming more prevalent within AWS services, I recommend the following blog posts to learn more about using it with various services.</p> 
<ul> 
 <li><a href="https://aws.amazon.com/blogs/storage/how-to-develop-a-user-facing-data-application-with-iam-identity-center-and-s3-access-grants/" rel="noopener" target="_blank">How to develop a user-facing data application with IAM Identity Center and S3 Access Grants</a> (in two parts)</li> 
 <li><a href="https://aws.amazon.com/blogs/security/simplify-workforce-identity-management-using-iam-identity-center-and-trusted-token-issuers/" rel="noopener" target="_blank">Simplify workforce identity management using IAM Identity Center and trusted token issuers</a></li> 
 <li><a href="https://aws.amazon.com/blogs/big-data/simplify-access-management-with-amazon-redshift-and-aws-lake-formation-for-users-in-an-external-identity-provider/" rel="noopener" target="_blank">Simplify access management with Amazon Redshift and AWS Lake Formation for users in an External Identity Provider</a></li> 
 <li><a href="https://aws.amazon.com/blogs/aws/aws-analytics-services-streamline-user-access-to-data-permissions-setting-and-auditing/" rel="noopener" target="_blank">AWS analytics services streamline user access to data, permissions setting, and auditing</a></li> 
 <li><a href="https://aws.amazon.com/blogs/security/access-aws-services-programmatically-using-trusted-identity-propagation/" rel="noopener" target="_blank">Access AWS services programmatically using trusted identity propagation</a></li> 
</ul> 
<p>If you have feedback about this post, submit comments in the <strong>Comments</strong> section below. </p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Joseph de Clerck " class="aligncenter size-full wp-image-36177" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/10/23/Joseph-de-Clerck.jpeg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Joseph de Clerck</h3> 
  <p>Joseph is a senior Cloud Infrastructure Architect at AWS. He uses his expertise to help enterprises solve their business challenges by effectively using AWS services. His broad understanding of cloud technologies enables him to devise tailored solutions on topics such as analytics, security, infrastructure, and automation.</p> 
  <p></p>
 </div> 
</footer>
