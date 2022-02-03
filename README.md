<h1>Keep It Clean Functional Specification Document</h1>
<p>:wave: Welcome to Keep It Clean! An SDK designed to help chat applications detect and block inappropriate content.</p>
<br>
<h2>Table of Contents</h2>
<ul>
    <li><a href="#introduction">Introduction</a>
        <ul>
            <li><a href="#problem-summary">Problem Summary</a></li>
            <li><a href="#solution-business-logic">Business Logic</a></li>
        </ul>
    </li>
    <li><a href="#api-structure-diagram">API Structure Diagram</a></li>
    <li><a href="#api-build-guide">API Build Guide</a>
        <ul>
            <li><a href="#connecting-to-websocket">WebSocket Connection & Authentication</a></li>
            <li><a href="#request-detection-data">Requesting Inappropriate Content Detection Data</a></li>
            <li><a href="#return-detection-data">Determining and Returning Inappropriate Content Detection Data</a></li>
            <li><a href="#disconnection-from-websocket">Disconnecting From The WebSocket</a></li>
        </ul>
    </li>
    <li><a href="#api-documentation">API Documentation</a></li>
    <li><a href="#sdk-reference">SDK Reference</a></li>
    <li><a href="#future-considerations">Future Considerations</a>
        <ul>
            <li><a href="#training-our-model">Using Our Own Data To Help Train Our ML Model</a></li>
            <li><a href="#automating-providing-of-api-keys">Automating Providing Of API Keys</a></li>
            <li><a href="#format-of-sanitized-text">Format Of Sanitized Text</a></li>
        </ul>
    </li>
    <li><a href="#discussion">Discussion</a>
        <ul>
            <li><a href="#why-websockets">Why WebSockets over HTTP?</a></li>
            <li><a href="#why-serverless">Why Serverless?</a></li>
        </ul>
    </li>
    <li><a href="#reference">Reference</a></li>
</ul>
<br>
<h2 id="introduction">Introduction</h2>
<h3 id="problem-summary">Problem Summary</h3>

<p><b>From the client's perspective</b><p>
<p>As a stakeholder of a chat application, I would like to integrate an easy-to-use inappropriate content detection service, 
   allowing us to focus on improving our app, rather than detecting and blocking inappropriate content.
</p>

<p><b>From the client's user perspective</b><p>
<p>As a user of a chat application, I would like any required inappropriate content filter to be accurate and responsive.
</p>

<br>
<h3 id="solution-business-logic">Business Logic</h3>
<p>Clients may call the <code>contentCleaner.cleanedContentData(options)</code> as they see fit depending on their business requirements.
<p>
    To check a message for inappropriate content, the client would call the above function when a user receives a message, passing the message content to the         function.
</p>
<br>
<p><b>The following details the input parameters:</b></p>
<table>
  <tr>
    <th>Field</th>
    <th>Definition</th>
    <th>Required</th>
    <th>Type</th>
    <th>Default</th>
  </tr>
  <tr>
    <td>content</td>
    <td>Text that is to be checked for inappropriate context</td>
    <td>Yes</td>
    <td>string</td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>threshold</td>
    <td>Used when a sanitized version of the inappropriate text is to be returned. Determines the threshold score in which a text is considered inappropriate. Must be used with <code>alternativeText</code></td>
    <td>No</td>
    <td>integer</td>
    <td>40</td>
  </tr>
   <tr>
    <td>alternativeText</td>
    <td>Used when a sanitized version of the inappropriate text is to be returned. An alternative text to replace the original. Must be used with                          <code>threshold</code>
    </td>
    <td>No</td>
    <td>string</td>
    <td>&lt;This text has been censored as it has been deemed to contain inappropriate content&gt;</td>
  </tr>
</table>
<br>
<p><b>The following details the returned values:</b></p>
<table>
  <tr>
    <th>Field</th>
    <th>Definition</th>
    <th>Type</th>
  </tr>
  <tr>
    <td>score</td>
    <td>A score from 1-99 representing the likelihood a text contains inappropriate content.
       <ul>
         <li>A score of 1 indicates a very low chance that a text contains inappropriate content</li>
         <li>A score of 99 indicates a very high chance that a text contains inappropriate content</li>
      </ul>
    </td>
    <td>integer</td>
  </tr>
  <tr>
    <td>sanitizedText</td>
    <td>Returns text sanitized based on the inputted <code>alternativeText</code> parameter</td>
    <td>string</td>
  </tr>
</table>

<br>
<p><b>The following details the structure of the returned data based on the supplied parameters:</b></p>
 <img src="https://github.com/roverandrew/keepitclean/blob/main/business-logic-flowchart.jpg" width="800" height="600">
<br>
<h2 id="api-structure-diagram">API Structure Diagram</h2>
<p>Insert image here</p>
<br>
<h2 id="api-build-guide">API Build Guide</h2>
<h3 id="connecting-to-websocket">WebSocket Connection & Authentication</h3>
<p>Upon opening the given chat application, the client is to make a request to the API Gateway, and provide an authorization token, in our case, an API key. If the API key is valid, the user may make use of the inappropriate content detection features provided by our service. The client provides their users with an API key via a subscription to the <em>Keep It Clean</em> service.</p>

<p><em>Note:</em> The following steps map to those outlined in the <a href="api-structure-diagram">API Structure Diagram</a><p>
<br>
<p><b>Steps 1-3:</b></p>
<ol>
    <li>Client calls AWS API WebSocket Gateway endpoint. If client has not already connected to the WebSocket, their connection request is to be authenticated.           If they have already been authenticated, their connection request can skip authentication, and will proceed to <a href="#step-3-auth">Step 3: a) Option I.</a>
    </li>
    <li>Authorization
        <ol type="a">
            <li>Gateway invokes the AWS Authorizer Handler, passing the context of the request and the authorization token as parameters.</li>
            <li>Authorizer Handler queries the AWS DynamoDB for valid tokens that match the passed token.</li>
            <li>DynamoDB returns a valid token match to the Authorizer Handler, if it exists.</li>
            <li>Authorizer Handler returns an AWS policy stating how the request is to be dealt with, i.e. if it is to be authorized or not.</li>
        </ol>
    </li>
    <li>Connect to the WebSocket
        <ol type="a">
            <li>API Gateway makes a request based on the policy returned by the Authorizer Handler. Either:
                <ol type="I">
                    <li id="step-3-auth">Client user connects to the WebSocket. 
                        This indicates that the policy has been evaluated to an authenticated request. The Connect Route Handler is then invoked.
                    </li>
                    <li>403 network error is returned to the client. This indicates that the policy has been evaluated to an unauthenticated request.</li>
                </ol>
            </li>
            <li>Connect Route Handler saves the WebSocket connection ID to the DynamoDB. 
                This will be used later to know which socket ID data the returned inappropriate content data should be sent to.
            </li>
        </ol>
        <p><b>NOTE:</b> Policies that evaluate to an authenticated request are to be cached, thus allowing an authorized user to skip the invokation of the                 Authorizer Handler for any subsequent requests (for a set period of time) made to the API Gateway. For our use case, caching time should be set to 30             minutes. <b>Users with cached policies that evaluate to an authenticated request are to have their request skip step 3 & 4 and 
          go to <a href="#step-4"> step 4.</a></b>
        </p>
    </li>
</ol>

<br>
<h3 id="request-detection-data">Requesting Inappropriate Content Detection Data</h3>
<p id="step-4">Step 4:</p>

<ol type="a">
    <li>Invoke Detect Inappropriate Content Lambda Function. Once 4a)-5b) are completed, this service executes our business based on the parameters passed                 in the request. Our business logic can be seen detailed <a href="solution-business-logic">here</a>.
    </li>
    <li>Request connection ID from DynamoDB.</li>
    <li>Return connection ID from DynamoDB. Detect Inappropriate Content Lambda Function checks if a connection is still open.</li>
</ol>

<br>
<h3 id="return-detection-data">Determining and Returning Inappropriate Content Detection Data</h3>
<p id="step-5">Step 5:</p>

<ol type="a">
    <li>Request pre-trained ML model from S3.</li>
    <li>Return pre-trained ML model from S3.</li>
    <li>Our Lambda Function uses the pre-trained model to run our proprietary ML algorithm 
        to determine whether the supplied content is inappropriate or not. Based on whether the content was deemed inappropriate or not, our business                     logic determines the data that is to be sent back to the gateway. This data is then sent to the API Gateway.
    </li>
    <li>Data is then returned to the client</li>
</ol>

<br>
<h3 id="disconnection-from-websocket">Disconnnecting From The WebSocket</h3>
<p id="step-6">Step 6:</p>
<ol type="a">
    <li>Invoke Disconnect Route Handler.</li>
    <li>Query for connection ID of user who has disconnected from the WebSocket, and delete their ID.
        Thus closing the WebSocket for their associated ID.</li>
</ol>

<br>
<h2 id="api-documentation">API Documentation</h2></p>
<p><b>Base Server:</b> <code>wss://api.keepitclean.com</code></p>
<p><b>Security:</b> An API key is a token that you provide when making API calls.</p>
<p>Include the token on opening of a WebSocket connection in a query parameter.</p>
<p>The query parameter is called<code>appid</code> Example: <code>?appid=123</code></p>

<br>
<h3>Connecting to:<b> <code>/detect</code></b></h3>
<p><b>Opening the WebSocket Sample: Javascript</b></p>
<p><code>const ws = new Websocket('wss://api.keepitclean.com/detect?appid=&lt;YOUR-API-KEY-HERE&gt;')</code></p>

<br>
<h3>Sending data to:<b> <code>/detect</code></b></h3>
<p>Analyzes text for inappropriate content such as spam, foul language, harassment, and adult content.</p>
<h3>Web Socket Request</h3>
<p><b>Web Socket Request Object Parameters</b><p>
<table>
  <tr>
    <th>Field</th>
    <th>Definition</th>
    <th>Required</th>
    <th>Type</th>
    <th>Default</th>
  </tr>
  <tr>
    <td>content</td>
    <td>Text that is to be checked for inappropriate context</td>
    <td>Yes</td>
    <td>string</td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>threshold</td>
    <td>Used when a sanitized version of the inappropriate text is to be returned. Determines the threshold score in which a text is considered inappropriate. Must be used with <code>alternativeText</code></td>
    <td>No</td>
    <td>integer</td>
    <td>40</td>
  </tr>
   <tr>
    <td>alternativeText</td>
    <td>Used when a sanitized version of the inappropriate text is to be returned. An alternative text to replace the original. Must be used with                          <code>threshold</code>
    </td>
    <td>No</td>
    <td>string</td>
    <td>&lt;This text has been censored as it has been deemed to contain inappropriate content&gt;</td>
  </tr>
</table>

<h3>Response</h3>
<p><b>WebSocket JSON Response Object Parameters</b></p>
<table>
  <tr>
    <th>Field</th>
    <th>Definition</th>
    <th>Type</th>
  </tr>
  <tr>
    <td>score</td>
    <td>A score from 1-99 representing the likelihood a text contains inappropriate content.
       <ul>
         <li>A score of 1 indicates a very low chance that a text contains inappropriate content</li>
         <li>A score of 99 indicates a very high chance that a text contains inappropriate content</li>
      </ul>
    </td>
    <td>integer</td>
  </tr>
  <tr>
    <td>sanitizedText</td>
    <td>Returns text sanitized based on the inputted <code>alternativeText</code> parameter</td>
    <td>string</td>
  </tr>
</table>

<b>Request Sample 1: Javascript</b>
```
ws.send({ 
    content:"The quick brown fox jumps over the lazy dog, but does this text contain foul language?" 
})
```

**Response Example 1**
```
{
  "score": "20",
  "sanitizedText": "The quick brown fox jumps over the lazy dog, but does this text contain foul language?"
}
```

**Request Sample 2:  Javascript**
```
ws.send({
    content: "Shit. The quick brown fox jumps over the lazy dog, but does this text contain foul language?",
    threshold: 25,
})
```

**Response Example 2**
```
{
  "score": "70",
  "sanitizedText": "&lt;This text has been censored as it has been deemed to contain inappropriate content&gt;"
}
```

<h3>Disconnecting from:<b> <code>/detect</code></b></h3>
<b>Closing the WebSocket Sample: Javascript</b>
<p><code>ws.close()</code></p>

<br>
<h2 id="sdk-reference">SDK Reference</h2>

All request parameters are to be passed to the following functions via an object.<br>
Example 1:
```
{
  text: "The quick brown fox jumps over the lazy dog, but does this text contain foul language?"
}
```

Example 2:
```
{
  content: "Shit. The quick brown fox jumps over the lazy dog, but does this text contain foul language?".
  threshold: 5,
  alternativeWord: "<Explicit Word>"
}
```

<h3><code>contentCleaner.connect()</code></h3>
<p>Opens a connection to the WebSocket.</p>
<br>
<h3><code>contentCleaner.disconnect()</code></h3>
<p>Closes the connection to the WebSocket.</p>
<br>

<h3><code>contentCleaner.cleanedContentData(options)</code></h3>
<p>Parameters are to be passed in a Javascript object</p>
<p>Option keys:</p>
<p><b>content</b></p> : The content that is to bee analyzed for inappropriate content.

<h3><code>contentCleaner.score(options)</code></h3>
<p>Returns an integer score from 1-99 representing the likelihood a text contains inappropriate content.</p>
<br>
<h3><code>contentCleaner.alternativeText(options)</code></h3>
<p>
    If the calculated inappropriate content score exceeds the passed `threshold` value, 
    returns an alternative text based on passed the `alternativeText` parameter. Else, the original text is returned.
</p>

<br>
<h2 id="future-considerationss">Future Considerations</h2>
<h3 id="training-our-model">Using Our Own Data To Help Train Our ML Model</h3>
<p>
   Our current version of the Keep It Clean API relies on a pre-trained ML Model. A potential feature is thus building out the ability for client users to
   send content that was incorrectly labelled as inappropriate/not inappropriate. This data can then be used to further train and thus improve our model.
   However, this feature may only lead to noticeable detection improvements when dealing with incorrectly labelled data at a very large scale. 
   Consequently, this feature is not deemed to be a priority for the initial release, but it can be of value in the future, and should be kept in mind.
</p>
<h3 id="automating-providing-of-api-keys">Automating Providing Of API Keys</h3>
<p>
   With our current implementation, API keys are manually provided to our clients for use by their users. We believe this to be sufficient for an initial release,
   but acknowledge it is not a scaleable solution. In the future, automation of payment processing and consequently API key providing to our clients would be
   a useful feature to develop.
</p>
<h3 id="format-of-sanitized-text">Format Of Sanitized Text</h3>
<p>
    When an <code>alternativeText</code> parameter is passed to the request, clients may invoke the <code>contentCleaner.alternativeText</code> 
    function to receive an alternative text for inappropriate content. The following question may then arise:
    "why not provide the same text with the inappropriate content removed on a per-word basis?" 
    This feature was considered, but ultimately abandoned. The reasoning is as follows: Users may still infer any inappropriate language
    in a message chat even if it replaced with an alternative word. For example, for the following text: "That guy is a piece of &lt;Explicit&gt;", it can
    still easily be inferred what is being written. For this reason we believe implementing such a feature is simply not worth the added engineering cost,
    and instead choose to mark the entire message as inappropriate.
</p>
<br>
<h2 id="discussion">Discussion</h2>
<h3 id="why-websockets">Why WebSockets over HTTP?</h3>
<ol>
    <li><p><b>Lower latency compared to HTTP</b></p>
        <p>
           The decision to use WebSockets was taken due to the real-time nature of a chat application. 
           The single, persistent connection opened by a WebSocket allows for much lower latency between the client and the server. 
           Due to the instantaneous nature of chat applications, 
           we believe that a quick response time for the detecting and blocking of data to be extremely important to the user experience.
        </p>
    </li>
    <li><p><b>Added engineering overhead is worth the increase in user experience</b></p></li>
        <p>
           WebSockets require a bit more engineering cost in order to maintain the connections betweeen the clients. 
           But we do not believe it to be significant enough for the real-time benefits to not be worthwhile.
        </p>
    <li><p><b>Although browser support is less than HTTP, it is more than sufficient</b></p>
        <p>
            We do not believe WebSocket's relatively lower browser support compared to HTTP to be an issue.
            Nearly all browsers developed after 2011 support it, and nearly all people using a browser before 2011 will not be using a chat application anyway.
        </p>
    </li>
</ol>

<h3 id="why-serverless">Why Serverless?</h3>
<ol>
    <li><p><b>Automatic infrastructure maintenance</b></p>
        <p>
            Things like deployments, OS updates and patches are all done under the hood, 
            allowing developers to focus on code rather than the infastructure behind it.
        </p>
    </li>
    <li><p><b>Automatic scaling</b></p>
        <p>
            AWS Lambda automatically scales to support the rate of incoming requests.
        </p>
    </li>
    <li><p><b>Cheap</b></p>
        <p>
            AWS Lambda charges on a per-invocation basis. Since it only invokes code when it is needed, it ends up being very cost effective.
        </p>
    </li>
</ol>

<br>
<h2 id="reference">Reference</h2>
<p><a href="https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-create-empty-api.html">
    AWS Documentation: Create a WebSocket API in API Gateway
</a></p>
<p><a href="https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html">
    AWS Documentation: Using API Gateway Lambda authorizers
</a></p>
<p><a href="https://docs.aws.amazon.com/lambda/index.html">
    AWS Documentation: Lambda
</a></p>
<p><a href="https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-lambda-auth.html">
    AWS Documentation: Creating a Lambda REQUEST authorizer function with the <em>Websocket</em> API
</a></p>
<p><a href="https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-route-keys-connect-disconnect.html#apigateway-websocket-api-routes-about-connect">
    AWS Documentation: Managing connected users and client apps: $connect and $disconnect routes
</a></p>
<p><a href="https://docs.aws.amazon.com/dynamodb/">
    AWS Documentation: DynamoDB
</a></p>
<p><a href="https://docs.aws.amazon.com/s3/index.html">
    AWS Documentation: S3
</a></p>
