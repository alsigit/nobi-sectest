# nobi-sectest
nobi security technical test by sigit setiawan
all tools used during this test listed in tools.txt in this repo

A. Find Vulnerability
Tasks:
- Security analysis on our API : https://testing.addityar.xyz/
Test Points:
- Find the vulnerability on our API
- Can provide security solutions from vulnerabilities found

**Answer**
I found so many vulnerabilities in this endpoint API:

**1. Leaked endpoint with no Authentication.**
   API can lack authentication mechanisms altogether. Assume API endpoints will only be accessed by authorized applications and will not be discovered by anyone else. But It will depend on business process of the application, sometimes this could be a part of application flow process that required free access for anyone.

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

**2. Laravel Debug Mode Enabled**
It is dangerous if developer enabling debug mode on production server or public server, Attacker can see some sensitive information like path, route, or etc.

Step to reproduce:
1. open your browser, go to https://testing.addityar.xyz/api/v1/user/add
<img src="https://github.com/alsigit/nobi-sectest/blob/main/debug_mode.png" width="700"/>

By analize some information in debug mode, I try to re-enumerate / fuzzing directory and successfully found this endpoint:
https://testing.addityar.xyz/api/v1/ib/member
https://testing.addityar.xyz/api/v1/ib/withdraw
https://testing.addityar.xyz/api/v1/ib/topup
https://testing.addityar.xyz/api/v1/ib/updateTotalBalance
https://testing.addityar.xyz/api/v1/ib/listNAB



