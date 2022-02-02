<h1>Keep It Clean Functional Specification Document</h1>
<p>:wave: Welcome to Keep It Clean! An SDK designed to help chatbots detect and block inappropriate content.</p>
<br>
<h2>Table of Contents</h2>
<ul>
    <li><a href="#introduction">Introduction</a>
        <ul>
            <li><a href="#problem-summary">Problem Summary</a></li>
            <li><a href="#solution-business-logic">Solution Business Logic</a></li>
            <li><a href="#stakeholders">Stakeholders</a></li>
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
    <li><a href="#further-considerations">Further Considerations</a></li>
    <li><a href="#discussion">Discussion</a></li>
</ul>
<br>
<h2 id="introduction">Introduction</h2>
<h3 id="problem-summary">Problem Summary</h3>
<p>SUMMARY CONTENT</p>
<h3 id="solution-business-logic">Solution Business Logic</h3>
<p>PROPOSED SOLUTION CONTENT</p>
<h3 id="stakeholders">Stakeholders</h3>
<p>STAKEHOLDERS CONTENT</p>
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
    <li>Client calls AWS API WebSocket Gateway endpoint. If client has not already connected to the WebSocket, their connection request is to be authenticated
        as outlined in the following steps (2-3). Else, their connection request can skip authentication, and will skip to <a href="#step-4">Step 4</a>.
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
                    <li>Client user connects to the WebSocket. 
                        This indicates that the policy has been evaluated to an authenticated request. The Connect Route Handler is then invoked.
                    </li>
                    <li>403 network error is returned to the client. This indicates that the policy has been evaluated to an unauthenticated request.</li>
                </ol>
            </li>
            <li>Connect Route Handler saves the WebSocket connection ID to the DynamoDB. 
                This will be used later to know which socket ID data the returned inappropriate content data should be sent to.
            </li>
        </ol>
        <p><b>NOTE:</b> Policies that evaluate to an authenticated request are to be cached, thus allowing an authorized user to skip the invokation of the                 Authorizer Handler for any subsequent requests (for a set period of time) made to the API Gateway. For our use case, caching time should be set to 30             minutes.<b>Users with cached policies that evaluate to an authenticated request are to have their request skip step 3&4 and 
          go to <a href="#step-4">step 4.</a></b>
        </p>
    </li>
</ol>
<p><b>References</b></p>
<p><a href="wss://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html">
    AWS Documentation: Using API Gateway Lambda authorizers
</a></p>
<p><a href="wss://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-lambda-auth.html">
    AWS Documentation: Creating a Lambda REQUEST authorizer function with the <em>Websocket</em> API
</a></p>
<br>
<h3 id="request-detection-data">Requesting Inappropriate Content Detection Data</h3>
<p id="step-4">Step 4:</p>

<ol type="a">
    <li>Invoke Detect Inappropriate Content Lambda Function. Once 4a)-5b) are completed, this service executes our business based on the parameters passed                 in the request. Our business logic can be seen detailed <a href="business-logic">here</a>.
    </li>
    <li>Request connection ID from DynamoDB.</li>
    <li>Return connection ID from DynamoDB. Detect Inappropriate Content Lambda Function checks if a connection is still open.</li>
</ol>
<p><b>References</b></p>
<p><a href="wss://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html">
    AWS Documentation: Using API Gateway Lambda authorizers
</a></p>
<p><a href="wss://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-lambda-auth.html">
    AWS Documentation: Creating a Lambda REQUEST authorizer function with the <em>Websocket</em> API
</a></p>

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
<p><b>References</b></p>
<p><a href="wss://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html">
    AWS Documentation: Using API Gateway Lambda authorizers
</a></p>
<p><a href="wss://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-lambda-auth.html">
    AWS Documentation: Creating a Lambda REQUEST authorizer function with the <em>Websocket</em> API
</a></p>
<br>
<h3 id="disconnection-from-websocket">Disconnnecting From The WebSocket</h3>
<p id="step-6">Step 6:</p>
<ol type="a">
    <li>Invoke Disconnect Route Handler.</li>
    <li>Query for connection ID of user who has disconnected from the WebSocket, and delete their ID.
        Thus closing the WebSocket for their associated ID.</li>
</ol>
<p><b>References</b></p>
<p><a href="wss://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html">
    AWS Documentation: Using API Gateway Lambda authorizers
</a></p>
<p><a href="wss://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-lambda-auth.html">
    AWS Documentation: Creating a Lambda REQUEST authorizer function with the <em>Websocket</em> API
</a></p>

<br>
<h2 id="api-documentation">API Documentation</h2></p>
<p><b>Base Server:</b> <code>wss://api.keepitclean.com`</code></p>
<p><b>Security:</b> An API key is a token that you provide when making API calls.</p>
<p>Include the token on opening of a WebSocket connection in a query parameter.</p>
<p>The query parameter is called<code>appid</code> Example: <code>?appid=123</code></p>

<br>
<h3>Connecting to:<b> <code>/spamdetection</code></b></h3>
<p><b>Opening the WebSocket Sample: Javascript</b></p>
<p><code>new Websocket('wss://api.keepitclean.com?appid=&lt;YOUR-API-KEY-HERE&gt;')</code></p>

<br>
<h3>Sending data to:<b> <code>/spamdetection</code></b></h3>
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
<p><b>WebSocket Response Object Parameters</b></p>
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
         <li>A score of 99 indicates a very high chance that a text contains inapropriate content</li>
      </ul>
    </td>
    <td>integer</td>
  </tr>
  <tr>
    <td>sanitizedText</td>
    <td>Returns text sanitized based on the inputted <code>alternativeText</code> or <code>alternativeWord</code> parameters</td>
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
  "score": "7",
  "sanitizedText": "&lt;This text has been censored as it has been deemed to contain inappropriate content&gt;"
}
```

<h3>Disconnecting from:<b> <code>/spamdetection</code></b></h3>
<b>Closing the WebSocket Sample: Javascript</b>
<p><code>WebSocket.close()</code></p>

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
  alternativeWord: "&gt;Explicit Word&lt;"
}
```

<h3><code>spam.connect()</code></h3>
<p>Opens a connection to the WebSocket</p>
<br>
<h3><code>spam.disconnect()</code></h3>
<p>Closes the connection to the WebSocket</p>
<br>
<h3><code>spam.score(options)</code></h3>
<p>Returns an integer score from 1-99 representing the likelihood a text contains inappropriate content.</p>
<br>
<h3><code>spam.alternativeText(options)</code></h3>
<p>If the calculated spam score exceeds the passed `threshold` value, returns an alternative text based on passed the `alternativeText` parameter. 
Else, the original text is returned.</p>

<br>
<h2 id="further-considerations">Further Considerations</h2>

<br>
<h2 id="discussion">Discussion</h2>
