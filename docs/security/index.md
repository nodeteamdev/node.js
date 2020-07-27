# Security

## Injection

### Server Side JS Injection

When `eval()`, `setTimeout()`, `setInterval()`, `Function()` are used to process user provided inputs, it can be exploited by an attacker to inject and execute malicious JavaScript code on server.

Web applications using the JavaScript `eval()` function to parse the incoming data without any type of input validation are vulnerable to this attack. An attacker can inject arbitrary JavaScript code to be executed on the server. Similarly `setTimeout()`, and `setInterval()` functions can take code in string format as a first argument causing same issues as `eval()`.

This vulnerability can be very critical and damaging by allowing attacker to send various types of commands.

An effective denial-of-service attack can be executed simply by sending the commands below to `eval()` function:

```javascript
while(1)
```

This input will cause the target server's event loop to use 100% of its processor time and unable to process any other incoming requests until process is restarted.

An alternative DoS attack would be to simply exit or kill the running process:

```javascript
process.exit() or process.kill(process.pid)
```

Another potential goal of an attacker might be to read the contents of files from the server. For example, following two commands list the contents of the current directory and parent directory respectively:

```javascript
res.end(require('fs').readdirSync('.').toString())
```

```javascript
res.end(require('fs').readdirSync('..').toString())
```

Once file names are obtained, an attacker can issue the command below to view the actual contents of a file:

```javascript
res.end(require('fs').readFileSync(filename))
```

An attacker can further exploit this vulnerability by writing and executing harmful binary files using `fs` and `child_process` modules.

**How Do I Prevent It?**

To prevent server-side js injection attacks:

- Validate user inputs on server side before processing
- Do not use `eval()` function to parse user inputs. Avoid using other commands with similar effect, such as `setTimeOut()`,  `setInterval()`, and `Function()`.
- For parsing JSON input, instead of using `eval()`, use a safer alternative such as `JSON.parse()`. For type conversions use type related `parseXXX()` methods.
- Include "use strict"at the beginning of a function, which enables strict mode within the enclosing function scope.

**Source Code Example**
In `routes/contributions.js`, the `handleContributionsUpdate()` function insecurely uses `eval()` to convert user supplied contribution amounts to integer.

```javascript
    // Insecure use of eval() to parse inputs
    var preTax = eval(req.body.preTax);
    var afterTax = eval(req.body.afterTax);
    var roth = eval(req.body.roth);
```

This makes application vulnerable to SSJS attack. It can fixed simply by using `parseInt()` instead.

```javascript
    //Fix for A1 -1 SSJS Injection attacks - uses alternate method to eval
    var preTax = parseInt(req.body.preTax);
    var afterTax = parseInt(req.body.afterTax);
    var roth = parseInt(req.body.roth);
```

In addition, all functions begin with use strictpragma.

### SQL and NoSQL Injection

SQL and NoSQL injections enable an attacker to inject code into the query that would be executed by the database. These flaws are introduced when software developers create dynamic database queries that include user supplied input.

Both SQL and NoSQL databases are vulnerable to injection attack. Here is an example of equivalent attack in both cases, where attacker manages to retrieve admin user's record without knowing password:

**1. SQL Injection**

Lets consider an example SQL statement used to authenticate the user with username and password

```sql
SELECT * FROM accounts WHERE username = '$username' AND password = '$password'
```

If this statement is not prepared or properly handled when constructed, an attacker may be able to supply `admin' --` in the username field to access the admin user's account bypassing the condition that checks for the password. The resultant SQL query would looks like:

```sql
SELECT * FROM accounts WHERE username = 'admin' -- AND password = ''
```

**2. NoSQL Injection**

The equivalent of above query for NoSQL MongoDB database is:

```javascript
db.accounts.find({username: username, password: password});
```

While here we are no longer dealing with query language, an attacker can still achieve the same results as SQL injection by supplying JSON input object as below:

```json
{
    "username": "admin",
    "password": {$gt: ""}
}
```

In MongoDB, `$gt` selects those documents where the value of the field is greater than (i.e. >) the specified value. Thus above statement compares password in database with empty string for greatness, which returns true.

The same results can be achieved using other comparison operator such as `$ne`.

The demo application is vulnerable to the NoSQL Injection. For example, on the Allocations page, running a search with a malicious input `1'; return 1 == '1` retrieves allocations for all the users in the database.

**SSJS Attack Mechanics**

Server-side JavaScript Injection (SSJS) is an attack where JavaScript code is injected and executed in a server component. MongoDB specifically, is vulnerable to this attack when queries are run without proper sanitization.

`$where` operator
MongoDB's `$where` operator performs JavaScript expression evaluation on the MongoDB server. If the user is able to inject direct code into such queries then such an attack can take place

Lets consider an example query:

```javascript
db.allocationsCollection.find({ $where: "this.userId == '" + parsedUserId + "' && " + "this.stocks > " + "'" + threshold + "'" });
```

The code will match all documents which have a `userId` field as specified by `parsedUserId` and a `stocks` field as specified by `threshold`. The problem is that these parameters are not validated, filtered, or sanitised, and vulnerable to SSJS Injection.

**NoSQL SSJS Injection**

An attacker can send the following input for the `threshold` field in the requests query, which will create a valid JavaScript expression and satisfy the `$where` query as well, resulting in a DoS attack on the MongoDB server:

```curl
http://localhost:4000/allocations/2?threshold=5';while(true){};'
```

You can also just drop the following into the Stocks Threshold input box:

```
';while(true){};'
```

**How Do I Prevent It?**

- Here are some measures to prevent SQL / NoSQL injection attacks, or minimize impact if it happens:
- Prepared Statements: For SQL calls, use prepared statements instead of building dynamic queries using string concatenation.
- Input Validation: Validate inputs to detect malicious values. For NoSQL databases, also validate input types against expected types
- Least Privilege: To minimize the potential damage of a successful injection attack, do not assign DBA or admin type access rights to your application accounts. Similarly minimize the privileges of the operating system account that the database process runs under.

### Log Injection

Log injection vulnerabilities enable an attacker to forge and tamper with an application's logs.

An attacker may craft a malicious request that may deliberately fail, which the application will log, and when attacker's user input is unsanitized, the payload is sent as-is to the logging facility. Vulnerabilities may vary depending on the logging facility:

**1. Log Forging (CRLF)**
Lets consider an example where an application logs a failed attempt to login to the system. A very common example for this is as follows:

```javascript
var userName = req.body.userName;
console.log('Error: attempt to login with invalid user: ', userName);
```

When user input is unsanitized and the output mechanism is an ordinary terminal stdout facility then the application will be vulnerable to CRLF injection, where an attacker can create a malicious payload as follows:

```curl
curl http://localhost:4000/login -X POST --data 'userName=vyva%0aError: alex moldovan failed $1,000,000 transaction&password=Admin_123&_csrf='
```

Where the `userName` parameter is encoding in the request the LF symbol which will result in a new line to begin. Resulting log output will look as follows:

```
Error: attempt to login with invalid user:  vyva
Error: alex moldovan failed $1,000,000 transaction
```

**2. Log Injection Escalation**

An attacker may craft malicious input in hope of an escalated attack where the target isn't the logs themselves, but rather the actual logging system. For example, if an application has a back-office web app that manages viewing and tracking the logs, then an attacker may send an XSS payload into the log, which may not result in log forging on the log itself, but when viewed by a system administrator on the log viewing web app then it may compromise it and result in XSS injection that if the logs app is vulnerable.

**How Do I Prevent It?**

As always when dealing with user input:

- Do not allow user input into logs
- Encode to proper context, or sanitize user input

```javascript
// Step 1: Require a module that supports encoding
var ESAPI = require('node-esapi');
// - Step 2: Encode the user input that will be logged in the correct context
// following are a few examples:
console.log('Error: attempt to login with invalid user: %s', ESAPI.encoder().encodeForHTML(userName));
console.log('Error: attempt to login with invalid user: %s', ESAPI.encoder().encodeForJavaScript(userName));
console.log('Error: attempt to login with invalid user: %s', ESAPI.encoder().encodeForURL(userName));
```


## Broken Authentication and Session Management

In this attack, an attacker (who can be anonymous external attacker, a user with own account who may attempt to steal data from accounts, or an insider wanting to disguise his or her actions) uses leaks or flaws in the authentication or session management functions to impersonate other users. Application functions related to authentication and session management are often not implemented correctly, allowing attackers to compromise passwords, keys, or session tokens, or to exploit other implementation flaws to assume other users’ identities.

Developers frequently build custom authentication and session management schemes, but building these correctly is hard. As a result, these custom schemes frequently have flaws in areas such as logout, password management, timeouts, remember me, secret question, account update, etc. Finding such flaws can sometimes be difficult, as each implementation is unique.

### 1. Session Management

Session management is a critical piece of application security. It is broader risk, and requires developers take care of protecting session id, user credential secure storage, session duration, and protecting critical session data in transit.

**Attack Mechanics**

Scenario #1: Application timeouts aren't set properly. User uses a public computer to access site. Instead of selecting “logout” the user simply closes the browser tab and walks away. Attacker uses the same browser an hour later, and that browser is still authenticated.

Scenario #2: Attacker acts as a man-in-middle and acquires user's session id from network traffic. Then uses this authenticated session id to connect to application without needing to enter user name and password.

Scenario #3: Insider or external attacker gains access to the system's password database. User passwords are not properly hashed, exposing every users' password to the attacker.

**How Do I Prevent It?**

Session management related security issues can be prevented by taking these measures:

- User authentication credentials should be protected when stored using hashing or encryption.
- Session IDs should not be exposed in the URL (e.g., URL rewriting).
- Session IDs should timeout. User sessions or authentication tokens should get properly invalidated during logout.
- Session IDs should be recreated after successful login.
- Passwords, session IDs, and other credentials should not be sent over unencrypted connections.

### 2 Password Guessing Attacks

Implementing a robust minimum password criteria (minimum length and complexity) can make it difficult for attacker to guess password.

**Attack Mechanics**

The attacker can exploit this vulnerability by brute force password guessing, more likely using tools that generate random passwords.

**How Do I Prevent It?**

Password length

Minimum passwords length should be at least eight (8) characters long. Combining this length with complexity makes a password difficult to guess and/or brute force.

Password complexity

Password characters should be a combination of alphanumeric characters. Alphanumeric characters consist of letters, numbers, punctuation marks, mathematical and other conventional symbols.

Username/Password Enumeration

Authentication failure responses should not indicate which part of the authentication data was incorrect. For example, instead of "Invalid username" or "Invalid password", just use "Invalid username and/or password" for both. Error responses must be truly identical in both display and source code

Additional Measures

For additional protection against brute forcing, enforce account disabling after an established number of invalid login attempts (e.g., five attempts is common). The account must be disabled for a period of time sufficient to discourage brute force guessing of credentials, but not so long as to allow for a denial-of-service attack to be performed.
Only send non-temporary passwords over an encrypted connection or as encrypted data, such as in an encrypted email. Temporary passwords associated with email resets may be an exception. Enforce the changing of temporary passwords on the next use. Temporary passwords and links should have a short expiration time.

## Cross-Site Scripting (XSS)

XSS flaws occur whenever an application takes untrusted data and sends it to a web browser without proper validation or escaping. XSS allows attackers to execute scripts in the victims' browser, which can access any cookies, session tokens, or other sensitive information retained by the browser, or redirect user to malicious sites.

**Attack mechanic**

There are two types of XSS flaws:

Reflected XSS: The malicious data is echoed back by the server in an immediate response to an HTTP request from the victim.
Stored XSS: The malicious data is stored on the server or on browser (using HTML5 local storage, for example), and later gets embedded in HTML page provided to the victim.
Each of reflected and stored XSS can occur on the server or on the client (which is also known as DOM based XSS), depending on when the malicious data gets injected in HTML markup.

**How Do I Prevent It?**

- Input validation and sanitization: Input validation and data sanitization are the first line of defense against untrusted data. Apply white list validation wherever possible.

- Output encoding for correct context: When a browser is rendering HTML and any other associated content like CSS, javascript etc., it follows different rendering rules for each context. Hence Context-sensitive output encoding is absolutely critical for mitigating risk of XSS.

- HTTPOnly cookie flag: Preventing all XSS flaws in an application is hard. To help mitigate the impact of an XSS flaw on your site, set the HTTPOnly flag on session cookie and any custom cookies that are not required to be accessed by JavaScript.

- Implement Content Security Policy (CSP): CSP is a browser side mechanism which allows creating whitelists for client side resources used by the web application, e.g. JavaScript, CSS, images, etc. CSP via special HTTP header instructs the browser to only execute or render resources from those sources. For example, the CSP header below allows content only from example site's own domain (mydomain.com) and all its sub domains.

```
Content-Security-Policy: default-src 'self' *.mydomain.com
```

- Apply encoding on both client and server side: It is essential to apply encoding on both client and server side to mitigate DOM based XSS attack, in which untrusted data never leaves the browser.

## Insecure Direct Object References

A direct object reference occurs when a developer exposes a reference to an internal implementation object, such as a file, directory, or database key. Without an access control check or other protection, attackers can manipulate these references to access unauthorized data.

**Attack mechanic**

If an applications uses the actual name or key of an object when generating web pages, and doesn't verify if the user is authorized for the target object, this can result in an insecure direct object reference flaw. An attacker can exploit such flaws by manipulating parameter values. Unless object references are unpredictable, it is easy for an attacker to access all available data of that type.

For example, the insure demo application uses userid as part of the url to access the allocations (/allocations/{id}). An attacker can manipulate id value and access other user's allocation information.

**How Do I Prevent It?**

- Check access: Each use of a direct object reference from an untrusted source must include an access control check to ensure the user is authorized for the requested object.
- Use per user or session indirect object references: Instead of exposing actual database keys as part of the access links, use temporary per-user indirect reference. For example, instead of using the resource’s database key, a drop down list of six resources authorized for the current user could use the numbers 1 to 6 or unique random numbers to indicate which value the user selected. The application has to map the per-user indirect reference back to the actual database key on the server.
- Testing and code analysis: Testers can easily manipulate parameter values to detect such flaws. In addition, code analysis can quickly show whether authorization is properly verified.

## Security Misconfiguration

This vulnerability allows an attacker to accesses default accounts, unused pages, unpatched flaws, unprotected files and directories, etc. to gain unauthorized access to or knowledge of the system.

Security misconfiguration can happen at any level of an application stack, including the platform, web server, application server, database, framework, and custom code.

Developers and system administrators need to work together to ensure that the entire stack is configured properly.

This vulnerability encompasses a broad category of attacks, but here are some ways attacker can exploit it:

**Attack mechanic**

1. If application server is configured to run as root, an attacker can run malicious scripts (by exploiting eval family functions) or start new child processes on server
2. Read, write, delete files on file system. Create and run binary files
3. If the server is misconfigured to leak internal implementation details via cookie names or HTTP response headers, then attacker can use this information towards building site's risk profile and finding vulnerabilities
4. If request body size is not limited, an attacker can upload large size of input payload, causing server to run out of memory, or make processor and event loop busy.

**How Do I Prevent It?**

Here are some node.js and express specific configuration measures:

- Use latest stable version of node.js and express (or other web framework you are using). Keep a watch on published vulnerabilities of these. The vulnerabilities for node.js and express.js can be found [here](http://expressjs.com/en/advanced/security-updates.html), respectively.
- Do not run application with root privileges. It may seem necessary to run as root user to access privileged ports such as 80. However, this can achieved either by starting server as root and then downgrading the non-privileged user after listening on port 80 is established, or using a separate proxy, or using port mapping.
- Review default in HTTP Response headers to prevent internal implementation disclosure.
- Use generic session cookie names
- Limit HTTP Request Body size by setting sensible size limits on each content type specific middleware ( `urlencoded`, `json`, `multipart`) instead of using aggregate `limit` middleware. Include only required middleware. For example if application doesn't need to support file uploads, do not include multipart middleware.
- If using multipart middleware, have a strategy to clean up temporary files generated by it. These files are not garbage collected by default, and an attacker can fill disk with such temporary files
- Vet npm packages used by the application
- Lock versions of all npm packages used, for example using [shrinkwarp](https://docs.npmjs.com/cli/shrinkwrap), to have full control over when to install a new version of the package.
- Set security specific HTTP headers

## Sensitive Data Exposure

This vulnerability allows an attacker to access sensitive data such as credit cards, tax IDs, authentication credentials, etc to conduct credit card fraud, identity theft, or other crimes. Losing such data can cause severe business impact and damage to the reputation. Sensitive data deserves extra protection such as encryption at rest or in transit, as well as special precautions when exchanged with the browser.

**Attack Mechanics**

If a site doesn’t use SSL/TLS for all authenticated pages, an attacker can monitor network traffic (such as on open wireless network), and steals user's session cookie. Attacker can then replay this cookie and hijacks the user's session, accessing the user's private data.

If an attacker gets access the application database, he or she can steal the sensitive information not encrypted, or encrypted with weak encryption algorithm

**How Do I Prevent It?**

- Use Secure HTTPS network protocol
- Encrypt all sensitive data at rest and in transit
- Don’t store sensitive data unnecessarily. Discard it as soon as possible.
- Ensure strong standard algorithms and strong keys are used, and proper key management is in place.
- Disable autocomplete on forms collecting sensitive data and disable caching for pages that contain sensitive data.

**Source Code Example**

1.The insecure demo application uses HTTP connection to communicate with server. A secure HTTPS sever can be set using https module. This would need a private key and certificate. Here are source code examples from `/server.js`

```javascript
// Load keys for establishing secure HTTPS connection
var fs = require("fs");
var https = require("https");
var path = require("path");
var httpsOptions = {
    key: fs.readFileSync(path.resolve(__dirname, "./app/cert/key.pem")),
    cert: fs.readFileSync(path.resolve(__dirname, "./app/cert/cert.pem"))
};
```

2. Start secure HTTPS sever

```javascript
// Start secure HTTPS server
https.createServer(httpsOptions, app).listen(config.port, function() {
    console.log("Express https server listening on port " + config.port);
});
```
3.The insecure demo application stores users personal sensitive information in plain text. To fix it, The data/profile-dao.jscan be modified to use crypto module to encrypt and decrypt sensitive information as below:

```javascript
// Include crypto module
var crypto = require("crypto");

//Set keys config object
var config = {
    cryptoKey: "a_secure_key_for_crypto_here",
    cryptoAlgo: "aes256", // or other secure encryption algo here
    iv: ""
};

// Helper method create initialization vector
// By default the initialization vector is not secure enough, so we create our own
var createIV = function() {
    // create a random salt for the PBKDF2 function - 16 bytes is the minimum length according to NIST
    var salt = crypto.randomBytes(16);
    return crypto.pbkdf2Sync(config.cryptoKey, salt, 100000, 512, "sha512");
};

// Helper methods to encryt / decrypt
var encrypt = function(toEncrypt) {
    config.iv = createIV();
    var cipher = crypto.createCipheriv(config.cryptoAlgo, config.cryptoKey, config.iv);
    return cipher.update(toEncrypt, "utf8", "hex") + cipher.final("hex");
};

var decrypt = function(toDecrypt) {
    var decipher = crypto.createDecipheriv(config.cryptoAlgo, config.cryptoKey, config.iv);
    return decipher.update(toDecrypt, "hex", "utf8") + decipher.final("utf8");
};

// Encrypt values before saving in database
user.ssn = encrypt(ssn);
user.dob = encrypt(dob);

// Decrypt values to show on view
user.ssn = decrypt(user.ssn);
user.dob = decrypt(user.dob);
```