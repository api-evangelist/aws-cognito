---
title: "How to implement relationship-based access control with Amazon Verified Permissions and Amazon Neptune"
url: "https://aws.amazon.com/blogs/security/how-to-implement-relationship-based-access-control-with-amazon-verified-permissions-and-amazon-neptune/"
date: "Mon, 30 Sep 2024 18:00:17 +0000"
author: "Henry Ho"
feed_url: "https://aws.amazon.com/blogs/security/tag/amazon-cognito/feed/"
---
<p>Externalized authorization for custom applications is a security approach where access control decisions are managed outside of the application logic. Instead of embedding authorization rules within the application’s code, these rules are defined as policies, which are evaluated by a separate system to make an authorization decision. This separation enhances an application’s security posture by aligning with Zero Trust principles of continual real-time authorization, simplifies the management of security policies, and enables consistent policy enforcement across multiple applications.&nbsp;<a href="https://aws.amazon.com/verified-permissions/" rel="noopener" target="_blank">Amazon Verified Permissions</a>&nbsp;is a scalable permissions management and fine-grained authorization service that you can use to externalize application authorization.</p> 
<p>Two common access control models that you might consider when implementing your authorization system are&nbsp;<a href="https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-api-access-authorization/access-control-types.html#rbac" rel="noopener" target="_blank">role-based access control (RBAC)</a>&nbsp;and&nbsp;<a href="https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-api-access-authorization/access-control-types.html#abac" rel="noopener" target="_blank">attribute-based access control (ABAC)</a>. RBAC grants permissions to users based on their assigned roles within an organization, simplifying the management of access by grouping permissions into roles that correspond to job functions. ABAC grants permissions based on a set of attributes associated with users, resources, and the context, allowing for more fine-grained and dynamic authorization decisions. However, as systems become more complex and have more interconnected data—especially in environments like social networks, collaborative environments, and multi-tenant applications—the limitations of RBAC and ABAC become apparent. These models often fail to effectively capture the relationships between entities. Relationship-based access control (ReBAC) offers a more nuanced approach by using the relationships between users and resources to make decisions about permitted actions, thus addressing scenarios more efficiently than other models.</p> 
<p>In this blog post, we show you how to implement ReBAC using Verified Permissions and&nbsp;<a href="https://aws.amazon.com/neptune/" rel="noopener" target="_blank">Amazon Neptune</a>, a managed, serverless graph database on AWS.</p> 
<h2>What is relationship-based access control?</h2> 
<p>The core principle of ReBAC is that authorization decisions are based on the relationships between the principal requesting access and the resource being accessed. These relationships can be of several types—ownership, collaboration, or membership relationships—that form hierarchical structures. Examples of ReBAC can be found in multiple domains, including social media sites, project management tools, and content management systems. For example, in a social media application, ReBAC can be used to control who can view, comment, or share a post based on the relationships between the poster, their connections, and the content itself.</p> 
<p>Conceptually, roles are types of relationships, and relationships are subsets of attributes.</p> 
<h2>Benefits of ReBAC</h2> 
<p>In some types of applications, relationships change dynamically. For example, in a collaborative or social media application, relationships such as <em>contributor</em> or <em>co-owner</em> are continually being established between individual users and resources. Compared to traditional access control models, ReBAC offers the following benefits in these use cases.</p> 
<ul> 
 <li><strong>Fine-grained access control</strong> – ReBAC grants access at the level of an individual resource based on a user’s relationship with that resource. For example, a user can update individual photo albums with which they have a contributor relationship.</li> 
 <li><strong>Scalability and adaptability</strong> – Relationships can change dynamically. Access permissions are updated automatically when a relationship changes. For example, when the contributor relationship is removed, the user no longer has access.</li> 
 <li><strong>Support for hierarchies</strong> – ReBAC can handle hierarchical relationships. For example, the contributor relationship can be inherited down through an album hierarchy, permitting the user to update photo albums that are members of the album with which they have the relationship.</li> 
</ul> 
<h2>Common relationship models in ReBAC</h2> 
<p>Here are some common relationship models, also shown in Figure 1, for consideration when building the application and its authorization system:</p> 
<ul> 
 <li><strong>Resource ownership</strong> – Permissions to access or manipulate a resource are granted based on whether a user <em>owns</em> that resource. For example, you can delete a GitHub repository if you are the owner of the repository.</li> 
 <li><strong>Resource hierarchies</strong> – Permissions to access or manipulate a resource are granted based on the permissions that a principal has for the parent resource. For example, a GitHub repository contributor can close issues that belong to that repository.</li> 
 <li><strong>User hierarchies</strong> – These are similar to <a href="https://aws.amazon.com/iam" rel="noopener" target="_blank">AWS Identity and Access Management (IAM)</a> user groups. Principals that belong to a group will have the permissions granted to that group.</li> 
</ul> 
<p style="line-height: 1.25em;"></p>
<div class="wp-caption aligncenter" id="attachment_35898" style="width: 790px;">
 <img alt="Figure 1: Common relationship models in ReBAC" class="size-full wp-image-35898" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/09/26/img1-9.png" style="border: 1px solid #bebebe;" width="780" />
 <p class="wp-caption-text" id="caption-attachment-35898">Figure 1: Common relationship models in ReBAC</p>
</div>
<p></p> 
<p>In a relationship model, direct relationships represent clear, explicit links between users and resources, such as an employee owns their expense reports or a file is a member of a folder. These connections are straightforward and simply definable.</p> 
<p>However, relationship models often extend beyond these direct links to include hierarchical structures. These create indirect relationships that are more complex in nature. For example, team managers might have access to all expense reports filed by their subordinates, even though they don’t directly own these reports. Similarly, folder owners might have access to all files within their subfolders, regardless of who created those files.</p> 
<p>These indirect relationships are derived from a series of direct relationships. They form a relationship chain that, while not explicitly defined, is implied by the hierarchical structure. Because of their complexity and potential for far-reaching implications, these indirect relationships require careful consideration when designing an authorization system.</p> 
<p>In this blog post, we focus on the implementation of the relationship models that use resource ownership and resource hierarchies, and relationship hierarchies in these models.</p> 
<h2>Example scenario</h2> 
<p>Consider a video application that allows users to manage and share videos of their pets. Alice and Bob are individual users within the environment and so they only have access permissions to their own directory or videos. Because Alice and Bob directly own their resources, they have direct <code style="color: #000000;">OWNER</code> relationships to these resources, represented as solid lines in Figure 2. <code style="color: #000000;">aliceCatVideo.mp4</code> is a video resource stored in the <code style="color: #000000;">aliceVideoDirectory</code> directory. There is a <code style="color: #000000;">MemberOf</code> relationship between these resources.</p> 
<div class="wp-caption aligncenter" id="attachment_35899" style="width: 790px;">
 <img alt="Figure 2: Alice has direct relationship to resources that she has direct ownership" class="size-full wp-image-35899" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/09/26/img2-8.png" style="border: 1px solid #bebebe;" width="780" />
 <p class="wp-caption-text" id="caption-attachment-35899">Figure 2: Alice has direct relationship to resources that she has direct ownership</p>
</div> 
<p>Charlie has direct <code style="color: #000000;">OWNER</code> relationship to the root directory <code style="color: #000000;">petVideosDirectory</code>. Because <code style="color: #000000;">aliceVideoDirectory</code> is a subdirectory of <code style="color: #000000;">petVideosDirectory</code>, Charlie inherits an <code style="color: #000000;">OWNER</code> relationship to <code style="color: #000000;">aliceVideoDirectory</code> and the video resource <code style="color: #000000;">aliceCatVideo.mp4</code> inside. This indirect <code style="color: #000000;">OWNER</code> relationship is inherited through the <code style="color: #000000;">MemberOf</code> relationship between resources and is represented as dotted lines in Figure 3.</p> 
<div class="wp-caption aligncenter" id="attachment_35900" style="width: 790px;">
 <img alt="Figure 3: Charlie has indirect relationship to resources that inherited from the MemberOf relationship" class="size-full wp-image-35900" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/09/26/img3-6.png" style="border: 1px solid #bebebe;" width="780" />
 <p class="wp-caption-text" id="caption-attachment-35900">Figure 3: Charlie has indirect relationship to resources that inherited from the MemberOf relationship</p>
</div> 
<p>When implementing access control for this scenario, both RBAC and ABAC offer distinct approaches. In RBAC, you might define roles such as&nbsp;<code style="color: #000000;">OWNER</code>&nbsp;and&nbsp;<code style="color: #000000;">VIEWER</code>, and grant Charlie full access to each resource through the&nbsp;<code style="color: #000000;">OWNER</code>&nbsp;role. While initially straightforward, this method can become inflexible as the application grows, potentially leading to role proliferation. For example, you might want to have separate roles to manage different resources (such as photos or videos) for each type of pet (such as cats or dogs). In ABAC, you might assign attributes such as <code style="color: #000000;">OWNER</code>&nbsp;and&nbsp;<code style="color: #000000;">VIEWER</code> and grant each user permissions to resources with specific attributes. This approach offers more flexibility, but fine-grained control can be more complex to set up and manage. As the application’s hierarchy becomes more intricate, both models face challenges in maintaining scalability while maintaining proper access control.</p> 
<p>ReBAC addresses these limitations by implementing an access control model that uses direct and indirect relationships between principals and resources. In the example scenario, when Charlie requests access to the video resource <code style="color: #000000;">aliceCatVideo.mp4</code>, the application traverses the relationship graph in Neptune to retrieve the inherited <code style="color: #000000;">OWNER</code> relationship through the <code style="color: #000000;">MemberOf</code> relationship and make the authorization decision.</p> 
<h2>Overview of a ReBAC application</h2> 
<p>In this solution, relationship data is stored in Neptune. Prior to requesting an authorization decision from Verified Permissions, the application runs a Neptune query that traverses the relationship graph to retrieve the set of principals that have a specific relationship with the resource. The application then constructs an authorization request for Verified Permissions, using the results of this query to populate the entity data in the request.</p> 
<p>In the Cedar schema, the resource has an attribute—named for the relationship—that contains the set of principals that have that relationship with the resource. In our sample application, entities of type <code style="color: #000000;">Video</code> have an attribute called <code style="color: #000000;">OWNER</code>, which contains the set of users that have an owner relationship, directly or indirectly, with a video. Each potential relationship is represented by a distinct resource attribute and requires a dedicated query to fetch the set of principals that have that relationship.</p> 
<p>See the&nbsp;<a href="https://github.com/aws-samples/implementing-relationship-based-access-control-with-amazon-verified-permissions-and-amazon-neptune" rel="noopener" target="_blank">GitHub repository</a>&nbsp;for the step-by-step walkthrough. In this post, we focus on the key concepts of the solution.</p> 
<h3>Architecture</h3> 
<div class="wp-caption aligncenter" id="attachment_35901" style="width: 790px;">
 <img alt="Figure 4: Solution architecture" class="size-full wp-image-35901" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/09/26/img4-5.png" style="border: 1px solid #bebebe;" width="780" />
 <p class="wp-caption-text" id="caption-attachment-35901">Figure 4: Solution architecture</p>
</div> 
<p>The solution architecture, as shown in Figure 4, includes the following:</p> 
<ol> 
 <li>The user authenticates with <a href="https://aws.amazon.com/cognito" rel="noopener" target="_blank">Amazon Cognito</a> and obtains an access token and an ID token.</li> 
 <li>The user accesses the application through <a href="https://aws.amazon.com/api-gateway" rel="noopener" target="_blank">Amazon API Gateway</a> with the provided token.</li> 
 <li>An application <a href="https://aws.amazon.com/lambda" rel="noopener" target="_blank">AWS Lambda</a> function traverses the relationship graph in Neptune and returns the set of principals that have a specific relationship with the resource.</li> 
 <li>The application Lambda function constructs the requests by putting relationship data in the entities field and passes the requests to Verified Permissions. Verified Permissions acts as the policy decision point (PDP) and evaluates the Cedar policies to arrive at an authorization decision.</li> 
 <li>The application Lambda function acts as the policy enforcement point (PEP) to enforce the authorization decision returned by Verified Permissions by allowing or denying access to the API.</li> 
</ol> 
<h3>Data modelling and queries in Neptune</h3> 
<p>Relationships between entities are created and stored in Neptune as a property graph. A property graph is a set of vertices and edges with respective properties (key-value pairs). The vertices represent entities such as User, Directory, and Video in our example, and the edges represent directional relationships between vertices. Each edge has a label that denotes the type of relationship.</p> 
<p>Neptune supports multiple graph query languages, including Gremlin, openCypher, and SPARQL, to access a graph. In this solution, we use Gremlin as the graph query language. For more information about Gremlin, see the documentation from&nbsp;<a href="https://tinkerpop.apache.org/gremlin.html" rel="noopener" target="_blank">Apache TinkerPop</a>. You can use&nbsp;<a href="https://docs.aws.amazon.com/neptune/latest/userguide/graph-notebooks.html" rel="noopener" target="_blank">Neptune graph notebooks</a>&nbsp;to work with a Neptune graph.</p> 
<p>You can visualize the relationship graph (Figure 5) using the following query. We use <code style="color: #000000;">elementMap()</code> to include attributes to represent a vertex or an edge.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text"># Visualizing the relationship graph and extracting the attributes of each vertex and edge
%%gremlin -p v,oute,inv
g.V().outE().inV().path().by(elementMap('name','directoryId','videoId','ownerName','ownerId','userId','isPublic').order().by(keys))</code></pre> 
</div> 
<div class="wp-caption aligncenter" id="attachment_35902" style="width: 790px;">
 <img alt="Figure 5: Relationship graph in Neptune" class="size-full wp-image-35902" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/09/26/img5-3.png" style="border: 1px solid #bebebe;" width="780" />
 <p class="wp-caption-text" id="caption-attachment-35902">Figure 5: Relationship graph in Neptune</p>
</div> 
<p>The following code snippet shows how to add a vertex for <code style="color: #000000;">entity</code> and an edge for <code style="color: #000000;">relationship</code> in a relationship graph. Static attributes such as <code style="color: #000000;">ownerId</code>, <code style="color: #000000;">ownerName</code>, and <code style="color: #000000;">isPublic</code> are defined as properties of a vertex. In our example, we will define two relationships—<code style="color: #000000;">MEMBEROF</code> and <code style="color: #000000;">OWNER</code>—to denote the direct relationships between resources-to-resources and resources-to-users respectively.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text"># Adding video vertices (eg. aliceCatVideo_vertex)
g.addV('video').property('name', 'aliceCatVideo.mp4').property('videoId', aliceCatVideo_id).property('ownerId', alice_id).property('ownerName', 'alice').property('isPublic', False)

# Adding relationship edges
g.V(aliceCatVideo_vertex).addE('MEMBEROF').to(aliceVideosDir_vertex)
g.V(alice_vertex).addE('OWNER').to(aliceCatVideo_vertex)</code></pre> 
</div> 
<p>It’s a best practice to assign universally unique identifiers (UUIDs) for all principal and resource identifiers. Another best practice is to not include personally identifying, confidential, or sensitive information as part of the unique identifier for your principals or resources.</p> 
<p>To traverse the relationship graph to obtain the owner vertex of a resource vertex, you can use the following query. This query returns the vertex that has a direct <code style="color: #000000;">OWNER</code> relationship to the resource vertex <code style="color: #000000;">aliceCatVideo.mp4</code>.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text"># Retrieve the direct owner of a specific video
g.V().hasLabel('video').has('name', 'aliceCatVideo.mp4').in('OWNER').values(‘name’)</code></pre> 
</div> 
<p>You can use the following query to discover inherited <code style="color: #000000;">OWNER</code> relationships through <code style="color: #000000;">MemberOf</code> relationships between resources. The query traverses the relationship graph starting from a video vertex and return the <code style="color: #000000;">OWNER</code> vertex of each resource vertex along the path to the root directory <code style="color: #000000;">petVideosDirectory</code>. It outputs the set of owners after deduplication. This query discovers the inherited <code style="color: #000000;">OWNER</code> in the file system hierarchy and includes them in the entities list of authorization requests.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text"># Retrieve the direct and transitive owners of a specific video
g.V().hasLabel('video').has('videoId',video_id).union(in('OWNER'),repeat(out('MEMBEROF')).until(has('name', 'petVideosDirectory')).in('OWNER')).dedup().values('userId').toList()</code></pre> 
</div> 
<h3>Cedar policy design</h3> 
<p>Verified Permissions uses the&nbsp;<a href="https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/what-is-avp.html#avp-cedar" rel="noopener" target="_blank">Cedar policy language</a>&nbsp;to define fine-grained permissions. The default decision for an authorization response is <code style="color: #000000;">DENY</code>. The first policy permits a principal to perform actions in the action group <code style="color: #000000;">OwnerActions</code> on resources in <code style="color: #000000;">petVideosDirectory</code> only when the same principal is included in the set of resource owners.</p> 
<pre><code class="lang-cedar">// Resource owner and related persons can access the resources
permit (
	principal,
	action in [PetVideosApp::Action::"OwnerActions"],
	resource in PetVideosApp::Directory::&lt;petVideosDirectory_UUID&gt; ) 
when { 
	resource has owner &amp;&amp; 
	principal in resource.owner };</code></pre> 
<p>The second policy is an ABAC policy that permits a principal to perform actions in the action group <code style="color: #000000;">PublicActions</code> on resources in <code style="color: #000000;">petVideosDirectory</code> only when the resource has the static attribute <code style="color: #000000;">isPublic</code> and its value is <code style="color: #000000;">true</code>.</p> 
<pre><code class="lang-cedar">// Allow public access to the resources
permit (
	principal,
	action in [PetVideosApp::Action::"PublicActions"], 
	resource in PetVideosApp::Directory::&lt;petVideosDirectory_UUID&gt; ) 
when { 
	resource has isPublic &amp;&amp;
	resource.isPublic == true };</code></pre> 
<p>Implementing ReBAC using this Cedar design pattern in conjunction with a relationship graph requires the careful construction of queries. Verified Permissions will validate that the Cedar policies are correct, based on the Cedar schema, but cannot validate that the Neptune queries correctly traverse the graph to return the correct set of principals with the referenced relationship.</p> 
<p>When designing your policies and queries, take account of the following guidelines.</p> 
<ul> 
 <li>Each Cedar policy governs the behaviors of a specific relationship, in this case <code style="color: #000000;">OWNER</code>. Use a distinct Cedar policy for each relationship in your use cases.</li> 
 <li>Define action groups for each relationship in your use cases.</li> 
 <li>Each new relationship referenced in a Cedar policy requires its own query, and the application needs to run this query if the relationship is relevant to the authorization request. Policy writers must collaborate closely with the application developer to help ensure that the application fetches all data that’s relevant to the authorization request.</li> 
 <li>Indirect relationships can be hard to intuit and prone to errors. The example here of an <code style="color: #000000;">OWNER</code> relationship inherited through the <code style="color: #000000;">MEMBEROF</code> relationship is relatively intuitive. However, we recommend avoiding policies that rely on indirect relationships that are derived from multiple different types of direct relationship.</li> 
 <li>Indirect relationships can be over-permissive when there is no permission boundary defined. In our example, the boundary for inherited relationship is defined at the root level of the directory (<code style="color: #000000;">petVideosDirectory</code>). Follow the least privilege principle to limit inherited relationship within a clearly defined permission boundary.</li> 
 <li>Use <code style="color: #000000;">MEMBEROF</code> to denote the parent relationship in your graph to align with Cedar policy terminology. However, remember that Verified Permissions cannot auto-discover the Neptune graph, so your queries will still need to be designed to traverse it correctly.</li> 
</ul> 
<h3>Authorization request to Verified Permissions</h3> 
<p>The following example shows the structure of an authorization request made to Verified Permissions. In the example, Amazon Cognito is used as the identity source of the Verified Permissions policy store. Cognito user ID claims are mapped to the user entity <code style="color: #000000;">PetVideosApp::User</code>. Tokens issued by Cognito are mapped to a principal ID in the format <code style="color: #000000;">&lt;user pool ID&gt;|&lt;sub&gt;</code> by Verified Permissions.</p> 
<p>The following request was made for action <code style="color: #000000;">ViewVideo</code> to the video resource entity with UUID <code style="color: #000000;">878c101a-ca0e-4733-904d-af3f252abf50</code> (the video ID of <code style="color: #000000;">aliceCatVideo.mp4</code>) using the ID token of <code style="color: #000000;">alice</code>. The user IDs for <code style="color: #000000;">alice</code> and <code style="color: #000000;">charlie</code> were returned after traversing the relationship graph in Neptune to fetch users with the <code style="color: #000000;">OWNER</code> relationship and include these in the owner attribute in the entities field. The entities field is an array of attributes that Verified Permissions can examine when evaluating the policies. The resource hierarchy of this video resource was shown by including the parent directories (<code style="color: #000000;">petVideosDirectory</code> and <code style="color: #000000;">aliceVideosDirectory</code>) as the parent entities in the authorization request.</p> 
<p>With reference to the Cedar policy <code style="color: #000000;">&lt;Resource owner and related persons can access the resources&gt;</code>, the following authorization request returns an <code style="color: #000000;">ALLOW</code> decision.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">{
    "policyStoreId": "HhuNNuHBJJYJd4MfEhAZzD",
    "identityToken": [ID Token Redacted],
    "action": {
        "actionType": "PetVideosApp::Action",
        "actionId": "ViewVideo"
    },
    "resource": {
        "entityType": "PetVideosApp::Video",
        "entityId": "878c101a-ca0e-4733-904d-af3f252abf50"
    },
    "entities": {
        "entityList": [
            {
                "identifier": {
                    "entityType": "PetVideosApp::Video",
                    "entityId": "878c101a-ca0e-4733-904d-af3f252abf50"
                },
                "attributes": {
                    "owner": {
                        "set": [
                            {
                                "entityIdentifier": {
                                    "entityType": "PetVideosApp::User",
                                    "entityId": "ap-southeast-2_K9khoza7q|696e7428-e021-708d-7996-2d322fcf4b29"
                                }
                            },
                            {
                                "entityIdentifier": {
                                    "entityType": "PetVideosApp::User",
                                    "entityId": "ap-southeast-2_K9khoza7q|f91eb468-2001-7080-b860-eff8e20c333c"
                                }
                            }
                        ]
                    },
                    "isPublic": {
                        "boolean": false
                    }
                },
                "parents": [
                    {
                        "entityType": "PetVideosApp::Directory",
                        "entityId": "8e46133a-18da-47dc-bb7c-5e8640f45043"
                    },
                    {
                        "entityType": "PetVideosApp::Directory",
                        "entityId": "5e732639-692b-4fb0-8b69-d305926144fe"
                    }
                ]
            }
        ]
    }
}</code></pre> 
</div> 
<h3>Combining ReBAC policies with ABAC policies</h3> 
<p>ReBAC policies are a great fit when you want to create access based on a relationship between the principal and the resource. However, there can be cases where an ABAC policy is a more intuitive expression of a business rule. For example, in the sample application, you might want to grant all principals permission to view any public resource.</p> 
<p>With ReBAC, you would need to create a vertex <code style="color: #000000;">public</code> in the relationship graph, create <code style="color: #000000;">MEMBEROF</code> relationships between all public resources and this vertex, and then create a <code style="color: #000000;">VIEWER</code> relationship between all principals and the vertex <code style="color: #000000;">public</code>.</p> 
<p>With Cedar, you can create a policy store that is a mix of ReBAC and ABAC policies, enabling you to express this access rule with a single ABAC policy that allows public access to resources, as described in the section Cedar Policy Design. This policy grants broad access on resources with the attribute <code style="color: #000000;">isPublic</code> set to <code style="color: #000000;">true</code>.</p> 
<p>You can use the following Gremlin query to modify the static property <code style="color: #000000;">isPublic</code> of the video resource vertex <code style="color: #000000;">bobDogVideo.mp4</code> to <code style="color: #000000;">true</code>.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text"># Set the property "isPublic" to "true" for a specific video
g.V().hasLabel('video').has('name','bobDogVideo.mp4').property(single,'isPublic',true)</code></pre> 
</div> 
<p>You can verify the value of property <code style="color: #000000;">isPublic</code> of <code style="color: #000000;">bobDogVideo.mp4</code> with the following Gremlin query.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text"># Verify the value of property "isPublic" of a specific video
g.V().hasLabel('video').has('name','bobDogVideo.mp4').values('isPublic')</code></pre> 
</div> 
<p>The following authorization request is made to Verified Permissions using the principal <code style="color: #000000;">alice</code> after you have set the <code style="color: #000000;">isPublic</code> property of the video resource <code style="color: #000000;">bobDogVideo.mp4</code>. In the entities field, there is the attribute <code style="color: #000000;">isPublic</code> with <code style="color: #000000;">true</code> as the value.</p> 
<p>With reference to the Cedar policy <code style="color: #000000;">&lt;Allow public access to the resources&gt;</code>, the following authorization request returns <code style="color: #000000;">ALLOW</code>.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">{
    "policyStoreId": "HhuNNuHBJJYJd4MfEhAZzD",
    "identityToken": [ID Token Redacted], ,
    "action": {
        "actionType": "PetVideosApp::Action",
        "actionId": "ViewVideo"
    },
    "resource": {
        "entityType": "PetVideosApp::Video",
        "entityId": "8646429e-dca1-4229-aa26-9afcf75f053b"
    },
    "entities": {
        "entityList": [
            {
                "identifier": {
                    "entityType": "PetVideosApp::Video",
                    "entityId": "8646429e-dca1-4229-aa26-9afcf75f053b"
                },
                "attributes": {
                    "owner": {
                        "set": [
                            {
                                "entityIdentifier": {
                                    "entityType": "PetVideosApp::User",
                                    "entityId": "ap-southeast-2_K9khoza7q|b99ee448-f081-7078-5343-826a680f781f"
                                }
                            },
                            {
                                "entityIdentifier": {
                                    "entityType": "PetVideosApp::User",
                                    "entityId": "ap-southeast-2_K9khoza7q|f91eb468-2001-7080-b860-eff8e20c333c"
                                }
                            }
                        ]
                    },
                    "isPublic": {
                        "boolean": true
                    }
                },
                "parents": [
                    {
                        "entityType": "PetVideosApp::Directory",
                        "entityId": "b1551923-838e-43dc-946c-9fc63a85f445"
                    },
                    {
                        "entityType": "PetVideosApp::Directory",
                        "entityId": "5e732639-692b-4fb0-8b69-d305926144fe"
                    }
                ]
            }
        ]
    }
}</code></pre> 
</div> 
<h2>Conclusion</h2> 
<p>In this post, we showed you what ReBAC is and its benefits and demonstrated the implementation of ReBAC using Amazon Verified Permissions and Amazon Neptune. We also reviewed Cedar policy design patterns and considerations, in addition to the authorization request structure for a ReBAC application. You also saw how to combine ReBAC policies with ABAC policies.</p> 
<p>To learn more about this solution and the source code, visit the <a href="https://github.com/aws-samples/implementing-relationship-based-access-control-with-amazon-verified-permissions-and-amazon-neptune" rel="noopener" target="_blank">GitHub repository</a>. For more information, see&nbsp;<a href="https://docs.cedarpolicy.com/" rel="noopener" target="_blank">Cedar Policies</a>,&nbsp;<a href="https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/what-is-avp.html" rel="noopener" target="_blank">Amazon Verified Permissions</a>, and&nbsp;<a href="https://docs.aws.amazon.com/neptune/latest/userguide/intro.html" rel="noopener" target="_blank">Amazon Neptune</a>.</p> 
<p>&nbsp;<br />If you have feedback about this post, submit comments in the<strong> Comments</strong> section below. If you have questions about this post, <a href="https://console.aws.amazon.com/support/home" rel="noopener noreferrer" target="_blank">contact AWS Support</a>.<br />&nbsp;</p> 
<footer> 
 <div class="blog-author-box">
  <img alt="Henry Ho" class="alignleft size-full" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/09/26/kinhang.jpg" style="margin-left: 12px; margin-right: 18px; margin-top: 12px; margin-bottom: 6px; width: 93.750px; height: 125px;" />
  <span class="lb-h4" style="line-height: 2.1em; padding-top: 12px; margin-top: 24px;">Henry Ho</span>
  <br />Henry is a Senior Solutions Architect at AWS, dedicated to serving enterprise customers in Hong Kong. He specializes in cybersecurity and works with customers from different segments to establish secure landing zones on AWS, elevate their cloud security postures, and advocate cloud security.
 </div> 
 <div class="blog-author-box">
  <img alt="Christine Chan " class="alignleft size-full" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/09/26/yuenchan.jpg" style="margin-left: 12px; margin-right: 18px; margin-top: 12px; margin-bottom: 6px; width: 93.750px; height: 125px;" />
  <span class="lb-h4" style="line-height: 2.1em; padding-top: 12px; margin-top: 24px;">Christine Chan</span>
  <br />Christine is an Enterprise Support Technical Account Manager (TAM) based in Hong Kong. She focuses on serving large customers from different industries, using her expertise to provide guidance and technical support. She assists in delivering scalable, resilient, and cost-effective solutions. Apart from work, she also enjoys doing sports.
 </div> 
</footer>
