<h1>Keep It Clean</h1>
<p>:wave: Welcome to Keep It Clean! An SDK designed to help chatbots detect and block inappropriate content.</p>

<h3>Table of Contents</h3>
<ul>
    <li><a href="#api-structure">API Structure</a></li>
    <li><a href="#api-build-guide">API Build Guide</a>
        <ul>
            <li><a href="#api-gateway">API Gateway</a></li>
        </ul>
    </li>
    <li><a href="#api-documentation">API Documentation</a></li>
    <li><a href="#sdk-reference">SDK Reference</a></li>
</ul>

<br><br>
<h2 id="api-structure">API Structure</h2>
<p>Insert image here</p>

<br><br>
<h2 id="api-build-guide">API Build Guide</h2>
<h3 id="api-gateway">API Gateway</h3>
<p><b>Authentication</b><p>
Upon opening a chatbot a user must make a request to the API Gateway, and provide an authorization token, in our case, an API key. If the API key is valid, the user may make use of the inappropriate content detection features provided by our service. The client provides their users with an API key via a subscription to the <em>Keep It Clean</em> service.

<p><b>Steps:</b></p>
<ol>
    <li>Client user calls AWS API WebSocket Gateway endpoint. If client user has already connected to the WebSocket, skip to <a href="#step-4">Step 4</a></li>
    <li>Authorization
        <ol type="a">
            <li>Gateway invokes the Authorizer AWS Lambda handler, passing the context of the request and the authorization token as parameters</li>
            <li>Authorizer Handler queries the AWS DynamoDB for valid tokens that match the passed token</li>
            <li>DynamoDB returns a valid token match to the Authorizer Handler, if it exists</li>
            <li>Authorizer Handler returns a policy stating how the request is to be dealt with, i.e. if it is to be authorized or not.</li>
        </ol>
    </li>
    <li>Connect to the WebSocket
        <ol type="a">
            <li>API Gateway makes a request based on the policy returned by the Authorizer Handler.
                <ol type="upper-roman">
                    <li>Connect to the WebSocket. Policy has been evaluated to an authenticated request. Connect Route Handler is invoked.</li>
                    <li>Return a 403 network error to the client user. Policy has been evaluated to an unauthenticated request.</li>
                </ol>
            </li>
            <li>Connect Route Handler saves the client ID to the DynamoDB for data persistence. 
                Thus indicating a WebSocket is to opened indefinitely for the given Client ID
            </li>
        </ol>
        <p><b>NOTE:</b> policies that evaluate to an authenticated request are to be cached, thus allowing an authorized user to skip the invokation of the                 Authorizer Handler for any subsequent requests (for a set period of time) made to the API Gateway. For our use case, caching time should be sent to 30             minutes. <b>Users with cached policies that evaluate to an authenticated request are to skip step 3 and go to <a href="#step-4">step 4.</a></b>
        </p>
    </li>
    <li id="step-4"> Request inappropriate content detection data
        <ol type="a">
            <li>Invoke Spam Detection Route Handler</li>
            <li>Request user ID from DynamoDB</li>
            <li>Return user ID</li>
            <li>Machine learning step next?</li>
        </ol>
    </li>
</ol>

<p><b>Useful Resources</b></p>
<p><a href="https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html">
    AWS Documentation: Using API Gateway Lambda authorizers
</a></p>
<p><a href="https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-lambda-auth.html">
    AWS Documentation: Creating a Lambda REQUEST authorizer function with the <em>Websocket</em> API
</a></p>

<h2 id="api-documentation">API Documentation</h2></p>
<p><b>Base Server:</b>  <code>https://api.keepitclean.com`</code></p>
<p><b>Security:</b> An API key is a token that you provide when making API calls.</p>
<p>Include the token in a query parameter called <code>appid</code> Example: <code>?appid=123</code></p>

<br>

<h3> POST <b><code>/spamdetection</code></b></h3>
Analyzes text for inappropriate content such as spam, foul language, harassment, and adult content.
<h3>Request</h3>
<p><b>Request Body Parameters</b><p>
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
    <td>Used when a sanitized version of the inappropriate text is to be returned. Determines the threshold score in which a text is considered inappropriate. Must be used with <code>alternativeText</code> or <code>alternativeWord</code></td>
    <td>No</td>
    <td>integer</td>
    <td>5</td>
  </tr>
   <tr>
    <td>alternativeText</td>
    <td>Used when a sanitized version of the inappropriate text is to be returned. An alternative text to replace the original. Must be used with <code>threshold</code></td>
    <td>No</td>
    <td>string</td>
    <td>&lt;This text has been censored as it has been deemed to contain inappropriate content&gt;</td>
  </tr>
   <tr>
    <td>alternativeWord</td>
    <td>Used when a sanitized version of the inappropriate text is to be returned. An alternative word to replace all explicit words found in the original text. Must be used with <code>threshold</code></td>
    <td>No</td>
    <td>string</td>
    <td>&lt;explicit content&gt;</td>
  </tr>
</table>

<h3>Response</h3>
<p><b>Response Body Parameters</b></p>
<table>
  <tr>
    <th>Field</th>
    <th>Definition</th>
    <th>Type</th>
  </tr>
  <tr>
    <td>score</td>
    <td>A score from 1-9 representing the likelihood a text contains inappropriate content.
       <ul>
         <li>A score of 1 indicates a very low chance that a text contains inappropriate content</li>
         <li>A score of 9 indicates a very high chance that a text contains inapropriate content</li>
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

<b>Request Sample 1:  Shell / cURL</b>
```
curl --request POST \
  --url 'http://localhost:3000/spamdetection?appid=' \
  --header 'Content-Type: application/json' \
  --data '{
    "content": "The quick brown fox jumps over the lazy dog, but does this text contain foul language?"
  }'
```

**Response Example 1**
```
{
  "score": 2,
  "sanitizedText": "The quick brown fox jumps over the lazy dog, but does this text contain foul language?"
}
```

**Request Sample 2:  Shell / cURL**
```
curl --request POST \
  --url 'http://localhost:3000/spamdetection?appid=' \
  --header 'Content-Type: application/json' \
  --data '{
    "content": "Shit. The quick brown fox jumps over the lazy dog, but does this text contain foul language?"
    "threshold": 5
    "alternativeWord": "<Explicit Word>"
    
  }'
```

**Response Example 2**
```
{
  "score": 2,
  "sanitizedText": "<Explicit Word>. The quick brown fox jumps over the lazy dog, but does this text contain foul language?"
}
```

<br>

<h3>POST<b><code>/spamdetection/error</code></b></h3>
<p>Report text that was erroneously labelled.</p>
<h3>Request</h3>
<p><b>Request Body Parameters</b><p>
<table>
  <tr>
    <th>Field</th>
    <th>Definition</th>
    <th>Required</th>
    <th>Type</th>
    <th>Default</th>
  </tr>
  <tr>
    <b><td>shouldBeInappropriate</td></b>
    <td>Report false positive / false negatives. If the text was labelled as <em><b>not</b></em> inappropriate, but it <em><b>should be</b></em> considered inappropriate, pass <code>true</code> </td>
    <td>Yes</td>
    <td>boolean</td>
    <td>N/A</td>
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
    <td>Used when a sanitized version of the inappropriate text is to be returned. Determines the threshold score in which a text is considered inappropriate. Must be used with <code>alternativeText</code> or <code>alternativeWord</code></td>
    <td>No</td>
    <td>integer</td>
    <td>5</td>
  </tr>
   <tr>
    <td>alternativeText</td>
    <td>Used when a sanitized version of the inappropriate text is to be returned. An alternative text to replace the original. Must be used with <code>threshold</code></td>
    <td>No</td>
    <td>string</td>
    <td>&lt;This text has been censored as it has been deemed to contain inappropriate content&gt;</td>
  </tr>
   <tr>
    <td>alternativeWord</td>
    <td>Used when a sanitized version of the inappropriate text is to be returned. An alternative word to replace all explicit words found in the original text. Must be used with <code>threshold</code></td>
    <td>No</td>
    <td>string</td>
    <td>&lt;explicit content&gt;</td>
  </tr>
</table>

<br>
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
  content: "Shit. The quick brown fox jumps over the lazy dog, but does this text contain foul language?"
  threshold: 5
  alternativeWord: "<Explicit Word>"
}
```

### ```spam.score(options)```
Returns an integer score from 1-9 representing the likelihood a text contains inappropriate content.
```spam.score(options)```

### ```spam.alternativeText(options)```
If the calculated spam score exceeds the passed `threshold` value, returns an alternative text based on passed the `alternativeText` parameter. 
Else, the original text is returned.


### ```spam.alternativeWord(options)```
If the calculated spam score exceeds the passed `threshold` value, returns the original text with every detected foul word replaced by the passed `alternativeWord` parameter. Else, the original text is returned.

### ```spam.reportError(options)```
Report a potential false positive and/or false negative. Pass true if a message should be inappropriate, else pass false.


