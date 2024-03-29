# nobi-sectest
# nobi security technical test by sigit setiawan
# all tools used during this test listed in tools.txt this repo
# Each severity based on CVSS calculator https://www.first.org/cvss/calculator/3.1
# I will put severity in each finding, then you can make a correction if any false positive or mistaken

**A. Find Vulnerability**

**Tasks:** 
- Security analysis on our API : https://testing.addityar.xyz/
Test Points:
- Find the vulnerability on our API
- Can provide security solutions from vulnerabilities found

**Answer**

I found so many vulnerabilities in this endpoint API:

**1. LEAKED API ENDPOINT WITH NO AUTHENTICATION.**

   API can lack authentication mechanisms altogether. Assume API endpoints will only be accessed by authorized applications and will not be discovered by anyone else. But It will depend on business process of the application, sometimes this could be a part of application flow process that required free access for anyone.
   
<img src="https://github.com/alsigit/nobi-sectest/blob/main/cvssscore1.png" width="500"/>

Step to reproduce:
1. Try to enumerate directory from https://testing.addityar.xyz/
2. by fuzzing this endpoint, I found one of endpoint with returned HTTP code 405 method not allowed
<img src="https://github.com/alsigit/nobi-sectest/blob/main/fuzzing.png" width="700"/>
https://testing.addityar.xyz/api/v1/user/add

3. using postman, I add some users with change from HTTP method GET to POST
<img src="https://github.com/alsigit/nobi-sectest/blob/main/postman1.png" width="700"/>
It will shown response below:

````
{
    "name": [
        "The name field is required."
    ]
}
````
we need to add key "name" in the payload to add user like this:

<img src="https://github.com/alsigit/nobi-sectest/blob/main/postman2.png" width="700"/>

and yeah, user successfully added !!

Impact :
since there is no authentication for this endpoint, an attacker / anyone can add many users but the severity can be flagged as false positive depending on business process.

**recommendation:**
- endpoint API ideally need authentication, from business perspective usually user with low privilige can not add new user before they are logged in and have session or token to authenticate.
- to prevent fuzzing directory, developer must not use common path name like /api/v2/xxx because this words already listed on many wordlists or attacker can brute / fuzzing this path (but it doesn't need if API have authentication)

**2. LARAVEL DEBUG MODE ENABLED**

It is dangerous if developer enabling debug mode on production server or public server, Attacker can see some sensitive information like path, route, or etc.

<img src="https://github.com/alsigit/nobi-sectest/blob/main/cvssscore2.png" width="500"/>

Step to reproduce:
1. open your browser, go to https://testing.addityar.xyz/api/v1/user/add
<img src="https://github.com/alsigit/nobi-sectest/blob/main/debug_mode.png" width="700"/>
we can see that web directory located at **/home/ubuntu/nobi-test/**

By analize some information in debug mode, I try to re-enumerate / fuzzing directory and successfully found this endpoint:
https://testing.addityar.xyz/api/v1/ib/member

https://testing.addityar.xyz/api/v1/ib/withdraw
<img src="https://github.com/alsigit/nobi-sectest/blob/main/withdraw.png" width="700"/>

https://testing.addityar.xyz/api/v1/ib/topup

<img src="https://github.com/alsigit/nobi-sectest/blob/main/topup.png" width="700"/>

https://testing.addityar.xyz/api/v1/ib/updateTotalBalance
<img src="https://github.com/alsigit/nobi-sectest/blob/main/updatebalance.png" width="700"/>

https://testing.addityar.xyz/api/v1/ib/listNAB

**regarding vulnerability at point 1 before, I can confirm all endpoint API lack of authentication, and I will update this severity as CRITICAL, since it will impact:**

1. Financial impact (fraud, etc), since anyone can do financial activity without any authentication and no authorization implemented
2. Data privacy issue**

**recommendation**:
- Always disable debug mode on production environment or public application.
- Implement best security practice for programming, user_id must not integer sequentially without any hashes.
- Implement auth/session management for endpoint API
- Do not put any PII in the response body whitout any hashing

**3. IDOR (Insecure Direct Object Reference)**

IDOR usually followed by excessive data exposure, which means anyone get access to something which is not allowed or don’t have that privilege to do that action on that web application.

<img src="https://github.com/alsigit/nobi-sectest/blob/main/cvssscore3.png" width="500"/>

Step to reproduce:
1. Using cURL, burpsuite or directly access via browser we can go to https://testing.addityar.xyz/api/v1/member
   <img src="https://github.com/alsigit/nobi-sectest/blob/main/member.png" width="700"/>
   
2. as we can see, it shown vulnerable endpoint https://testing.addityar.xyz/api/v1/member?page={IDOR_HERE}
  <img src="https://github.com/alsigit/nobi-sectest/blob/main/IDOR.png" width="700"/>
  
  by changing parameter page, anyone can see others user sensitive informations like id, name, balance and unit. This informations ideally should not shown to others user.

Impact : 
- Leaked PII (Personal Identifiable Information) possibility 
- Data breach

**recommendation:** Implement session management, PII should not shown to any users instead his/her own information. we can use authorization by implement JWT, or any session token.

**4. LACK OF RESOUCE & NO RATE LIMIT**

Rate-limiting prevent users overwhelming API with requests, limiting denial of service threats. But in this endpoint API test, I can confirm all endpoint does not have any rate limit for http requests.

<img src="https://github.com/alsigit/nobi-sectest/blob/main/cvssscore4.png" width="500"/>

Step to reproduce:
1. I give some sample for topup endpoint
2. Using burpsuite, try to access https://testing.addityar.xyz/api/v1/ib/topup with POST method and Content-Type: application/json
3. Intercept, request and sent it to intruder.
4. set payload type as sniper with sequentially payload like this
   
   ````
   {"user_id":"2", "amount_rupiah":"$10000$"}
   ````
 5. set payload position to anything you want.
 
  <img src="https://github.com/alsigit/nobi-sectest/blob/main/noratelimit.png" width="700"/>
 7. All request got 200 code, which means all success and no rate limit, and balance will increasing.
 
 <img src="https://github.com/alsigit/nobi-sectest/blob/main/result_balance.png" width="700"/>


Impact : 
- Potentially allow attackers to launch Denial of Service (DoS) attacks.
- Another issue, if you have any authentication endpoint, this issue will impact for brute force attack.
- This issue also have an impact to financial loss, if you have any notification service such as SMS gateway, because it will consume all costs for notification.

**recommendation:**
The appropriate rate and resource limit for each functionality usually always different. 
For instance, the rate limit for authentication endpoints should be much lower to prevent brute-forcing and password guessing attacks. 
The first thing you can do is to determine what is “normal usage” for that particular functionality. Then, block users whose request resources at a much higher rate than usual.
- If number of requests has exceeded the rate limits, it will throw 429 Too Many Requests
- After receiving a 429 response, your API should pause for a second from sending additional requests. 

**5. SUBDOMAIN TAKEOVER**

An attacker can hijack your subdomain, takeover the webpage for any malicious action.
This issue founded, since some subdomain with wordpress CMS doesn't yet finished installation.

<img src="https://github.com/alsigit/nobi-sectest/blob/main/cvssscore5.png" width="500"/>

Step to reproduce:
1. Try to enumerate all subdomain under addityar.xyz , we can use tools like subdomainfinder, raccoon, virustotal etc.
  <img src="https://github.com/alsigit/nobi-sectest/blob/main/subdomain.png" width="700"/>
  
2. The domain adit.addityar.xyz is vulnerable to takeover by attacker, just follow the step for wordpress installation.
  <img src="https://github.com/alsigit/nobi-sectest/blob/main/WP_takeover.png" width="700"/>
  
Impact : An attacker can takeover this subdomain for malicious action like pishing or etc, this will make an impact for brand reputation.

PS : I will not takeover this subdomain, since this domain used for test and I believe so many tester needs to check during NOBI recruitment test.

**recommendation**
1. for wordpress site, don't forget to finish your installation setup.
2. to avoid any subdomain takeover, you must periodically control all A record or CNAME record in the DNS Manager.
   <img src="https://github.com/alsigit/nobi-sectest/blob/main/WP_takeover.png" width="500"/>
   
**OTHER FINDING**

I update this submission from any standpoint pentest:

a. LARAVEL version are vulnerable to RCE (CVE-2021-3129) , recommended to use latest update version and as mentioned, do not enable debug mode.

b. Nginx Web server version is 1.18.0 (CVE-2020-12440) which is vulnerable to HTTP request smuggling, allows attacker to cache poisoning, credential hijacking, or security bypass. Since nginx offically closed this issue as informative, I will not put this as a concern.

**B. Test Cases**

**As Security Engineer you must secure your infrastructure and API services, if your API
always get attack from attacker ( ddos , brute force, etc ). How should you handle the
attack?. Can you give advice on how to design the best security for infrastructure
security and API services.**

**Answer**

Before design best practice for infrastructure, I also always see from business perspective for example :
- What goals, company vision and mision?
- How much costs/budget alocated to infrastructure?
- Where is the infratructure set up ? On premise, hybrid or Cloud?
All points above need to defined at first and will be a part for policy and governance, to make sure infratructure cost are not over kill, but also meets best practice for security.

Since this test running on AWS environment, I will answer it from cloud environment perspective.
First, we need to separate security based on OSI layer. for API, I will separate Application layer (Layer 7) and network layer (Layer 3)

**+++ OSI Layer 7+++***

As mentioned earlier, at application layer protection we need to do some risk mapping and mitigation:
- Conduct security assessment for application periodically.
  This security assessment must be based on OWASP top API (https://owasp.org/www-project-api-security/)
  for example:
  a. to prevent Dos and brute force attack, we need to implement rate limiting and sometimes we need to check Cross Origin Resouce Sharing (CORS).
  b. Code analysis, developer must use best practice coding reference.
     For example if using Codeigniter or Laravel framework, always use query builder do not use raw query without any escape method when sent or get data 
     from databases.
     Avoid Server Side Request Forgery (SSRF), but I am sure AWS has been patch for AWS SSRF issue.
  c. Implement secure hashing and cryptography, make sure mobile application have obfuscation method before go live.
  d. Conduct vulnerability assessment for all assets (OS server, web server, or any library/3rd application used.
     To conduct vulnerability asset, we can use some tools like Nesus (from tenable) or Nexpose (from Rapid7).
     Since this test env on AWS, I will prefer to use **Amazon Inspector** to scan continually all AWS asset.
     But if you don't have any enough budget, you can manually check AWS environment using opensource tools like **pacu**
     ==== pacu image ===
     <img src="https://github.com/alsigit/nobi-sectest/blob/main/pacu.png" width="600"/>
     
     === sample pacu modules ===
     pacu contains many modules like metasploit framework but designed for AWS environment.
     <img src="https://github.com/alsigit/nobi-sectest/blob/main/pacu_modules.png" width="500"/>
     
  e. Periodically check IAM users policy on AWS.
  f. Never use root user for AWS daily actifity , but give each users depending on their role.
     For example: - developer only access to EC2 for development, or S3 for development.
  g. Setup MFA for AWS root account
  h. Periodically rolling/rotate AWS access key and secret key.
  i. Using AWS Cognito for authentication (if needed)
  
**+++ OSI Layer 3+++**

For network layer, we must implement:
- AWS firewall and activate WAF with advanced rules (do not use basic rules, since so many cases it can bypassed)
- Separate between network application route and databases route.
  Application may facing to public internet, but databases only accept connection from internal network or specific application IP.
  But sometimes, we need to open internet access to download some patch/update. This can solved by setting IGW, NAT and route properly.
- Using Cloudfront for edge caching
- Combine Route 53 with AWS ELB/ALB for make sure high availability performance
- Implement Cloud shield for protecting DoS attack.

**+++ Create Policies and governance++**

1. governance must be separate with Risk management
2. Setup policy for organization (it can be adopt PCI DSS for financial industry)
   
Below sample AWS infrastruture recommendation for prevent attack and securing API:

<img src="https://github.com/alsigit/nobi-sectest/blob/main/architecture.png" width="700"/>

**C. Security checklist**

**You are asked to assess a new mobile application (android and ios) , this mobile application has
backend API as well resided in cloud. What are the checklist / to-dos to ensure security
measurement is enough for the app go live**

**answer**

I will tested all environment including cloud infratructure security, API and mobile application security test.
For Mobile application. Since almost every mobile app talks to a backend service, and those services are prone to the same types of web application attacks. 
below my checklist:

**1. Architecture, design and threat modelling**.  

     first, we must identify all application component in mobile app, and make sure ecurity controls are never enforced only on the client side, but on        backend endpoints. we must put security as part of DevOps (DevsecOps), define threat modelling to identify potential upcoming threat, and sometimes        we must design for policy, and make sure comply with law.
     
**2. Data storage and privacy**

     This will contains some checklist such as is there any storage for PII or sensitive information? Is there any criptography mechanism? 
     For storage, we must ensure there are no sensitive data stored on application log, No sensitive data stored in memory for a long time, no sensitive        data/information stored locally. If sensitive data stored locally, make sure we have some encryption for all data. Also we need to make sure is there      any data shared to 3rd application, etc.
     
**3. Cryptography**

     Application must have modern cryptography not primitive mechanism. Cryptography mechanism also must not deprecated for security purpose (0day with        critical CVE widely exposed). And make sure no key hardcoded in application.
     
**4. Authentication, Authorization and session management**

     This related with my finding on this test.
     - Authorization must be enforced at backend endpoint
     - Implement session management. If your mobile app using stateful session, API endpoint can use random generated session to authenticate client              requests without sending credentials data.
     - Endpoint terminated session when user log out.
     - Token must have expiration time
     - Endpoint must have rate limit
     - etc
     
 **5. Network communication**
 
    For example, data must be encrypted using TLS on the network layer. Implement SSL pinning with Java Native Interface (JNI), and make sure certifate
    signed by trusted CA.
    
 **6. Mobile Platform Interaction**
 
   - Make sure all input are sanitized, never trust user input.
   - Testing for possible deserialization
   - Make sure webview only rendering javascript within app package.
 
 **7. Code Quality and setting**
 
   - Always check for dependency confusion, some application use private package that vulnerable to takeover the dependency and will have impact to RCE.
   - Built setting must be in release mode, do not use debug mode.
   - Check for 3rd library, is there any CVE related.
   - How app catches and handles exceptions.
   - etc
 
 **8. Resilience (e.g from tampering and reverse engineering)**
 
   - Implement obfuscation
   - All executable files and libraries app are encrypted on the file level
   - Check for app detect and response to reverse engineering tools
   - etc

**9. Business logic**

   The other test that usually I do, I will check business process/logic. We can not trust only to technical test, because based on my experience I have
   found many business logic vulnerability that will make financial loss or fraud mechanism.
   
   
Thank you and Have a nice day !!

 
 
     
     
     
     

    
   




