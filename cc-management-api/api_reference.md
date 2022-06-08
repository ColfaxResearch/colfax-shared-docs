### JWT Guide
----

The user REST API is protected through [JWT authentication](https://jwt.io/).
The HTTPs requests must have the Bearer token in the authorization header.
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```
You can construct the header manually, but it is recommended to use a library if available.


For the header part of the JWT, there is one requirement; we only accept HS256 algorithm.
So a typical header will be:
```
{
  "alg": "HS256",
  "typ": "JWT"
}
```
Trying to use other algorithms will result in an 400 response.


For the payload, the only required item is iss.
This should be set to the issuer name.
So the minimum required payload is.
```
{
  "iss": "issuername"
}
```
Here `issuename` is the issuer name assigned by the API admins.

Additionally, there are two more optional payload items that are supported.
* exp: unix time specifying when the token expires. This makes it difficult for any attackers tamper with the request.
* nonce: a one-time passcode. This helps to prevent repeat attacks. 

So an example payload will be:
```
{
  "iss": "issuername",
  "exp": 1619200888, 
  "nonce": "df42b780-cb97-4a75-9a60-ac53db66307c"
}
```

#### Python example

Here is an example with Python. Please create a file `secret.txt` with the provided secret in the same directory.

```
########
# Dependencies:
#  pip3 install pyjwt requests
#
########

import requests
import jwt
import time
import json

urlbase = "https://apidomain.com"
apibase = urlbase + "/apibase"

class JWTAuth(requests.auth.AuthBase):
  def __init__(self):
    # Create this file with the provided secret
    with open('secret.txt') as f:
      self.secret = f.read().strip()
  def __call__(self, r):
    # Adding nonce. This is recommended but optional.
    r_nonce = requests.get('{}/nonce'.format(urlbase))
    payload = {'iss': iss,              # name of the issuer.
              'exp': time.time()+10,    # Optional: expiration time.
              'nonce': r_nonce.text}    # Optional: nonce.
    encoded_jwt = jwt.encode(payload, self.secret, algorithm='HS256') # only HS256 supported
    
    # Setting HTTP header for JWT Authorization
    r.headers['Authorization'] = "Bearer "+encoded_jwt.decode('utf-8')
    # In some versions of pyjwt the above may cause an TypeError. If so, try:
    #  r.headers['Authorization'] = "Bearer "+encoded_jwt
    
    return r

# provided issuer name
iss = "issuername"

r = requests.get('{}/user'.format(apibase), auth=JWTAuth())
print(r.status_code)
print(r.text)

```


### User API 

All of the following API endpoints are relative to the base API path, which is typically looks like:
```
https://apidomain.com/usermgmt
```
The exact path depends on the project. Contact admin to get the correct base API path.


#### Endpoints

<details>
 <summary><code>GET /user/</code> - Get all users, or an user with a specific name or EID</summary>

* *****Optional query args*****
  * `username` - Username of the user to get. Note: if `eid` is set, this is ignored.
  
  * `eid` - EID of the user to get. Note: multiple linux accounts will be returned if the user has multiple.

  * `include_expired` - Whether to return expired accounts. By default set to none.
    
  * `limit` - Integer limit of the number of entries returned.
  
  * `offset` - Offset used in conjunction with limit.



* *****Success Response:*****

Note: `pagination` is only present if `limit` is set

```
200 OK
{ 
  "status" : "success", 
  "data": {
    "user_info": [
      {
        "Account_RID": 1,
        "CurrentStatus": "PENDING ACTIVATION",
        "EndDate": null,
        "RID": 1,
        "StartDate": 1639704528,
        "eid": "0123456789abcdef",
        "username": "u1"
      },
      {
        "Account_RID": 1,
        "CurrentStatus": "PENDING ACTIVATION",
        "EndDate": null,
        "RID": 2,
        "StartDate": 1641852105,
        "eid": "0123456789abcdef",
        "username": "u2"
      }
    ],
    "pagination": {
      "limit": 2,
      "offset": null,
      "total": 10
    }
  }
}
```
</details>



<details>
 <summary><code>POST /user</code> - Add a new user</summary>

* *****Form Params*****

  * **required parameters**

    * `eid` - External ID. This must be an unique identifier.

    * `start_date` - Unix timestamp for when to begin access. Set it to current time to make it immediate.

  * **optional parameters**

    * `username` - Username for the user. If not provided, it will be assigned the default "u<UserRID>" value.
  
    * `duration` - Duration of the account. If not provided, the account will not expire.
  
    * `end_date` - The exact end_date. If this and duration are both set, this takes prescedence. If set to none, account will not expire. 

  * **additional parameters**
    
    <b>ALL</b> additional parameters will be stored for record keeping, but will not have any effect. 


* *****Success Response:*****
```
200 OK
{
  "data": {
    "Account_RID": 1,
    "CurrentStatus": "PENDING ACTIVATION",
    "EndDate": null,
    "LinuxUserStatus_RID": 1,
    "StartDate": 1642637135,
    "username": "u1"
  },
  "status": "success"
}
```
</details>


<details>
 <summary><code>GET /user/:user_rid</code> - Get a specific user.</summary>

* *****URL Params*****

  * `user_rid` - The RID of the user.

* *****Optional query args*****

  * `full` - Get full information in the database, as opposed to just the minimum. Defaults to false.

* *****Success Response:*****

Note: `info` is only available if `full=True`

```
200 OK
{
  "status": "success",
  "data": {
    "Account_RID": 1,
    "CurrentStatus": "PENDING ACTIVATION",
    "EndDate": null,
    "RID": 2,
    "StartDate": 1641852105,
    "eid": "0123456789abcdef",
    "info": {
      "company": "mycompany",
      "email": "myemail@email.com",
      "fname": "Jane",
      "lname": "Doe"
    },
    "username": "u2"
  }
}
```
</details>


<details>
 <summary><code>PUT /user/:user_rid</code> - Update user</summary>

* *****URL Params*****

  * `user_rid` - The record ID of the user.

* *****Form Params*****

  * **optional parameters**

    * `extension` - Number of days to extend the user's access. Ignored if the user does not have an expiration date.

* *****Success Response:*****
```
200 OK
{ 
  "status" : "success", 
  "data" : null
}
```
</details>


<details>
 <summary><code>DELETE /user/:user_rid</code> - Terminate user access.</summary>

* *****URL Params*****

  * `user_rid` - The record ID of the user.

* *****Form Params*****

  * **optional parameters**

    * `deactivation_reason` - Reason for deactivation, recorded as a note. Default empty. If set, deactivation_issuer is automatically recorded.



* *****Success Response:*****
```
200 OK
{ 
  "status" : "success", 
  "data" : null
}
```
</details>

<details>
 <summary><code>GET /user/:user_rid/groups</code> - Get all groups of an user.</summary>

* *****URL Params*****

  * `user_rid` - The RID of the user.

* *****Success Response:*****
```
200 OK
{
  "data": {
    "groups": ["group1, "group2"]
  },
  "status": "success"
}
```
</details>


<details>
 <summary><code>GET /user/:user_rid/ssh_key</code> - Get the public SSHKeys of an user.</summary>

* *****URL Params*****

  * `user_rid` - The RID of the user.

* *****Success Response:*****
```
200 OK
{
  "data": {
    "all_keys": [
      {
        "RID": 1,
        "SSHKey": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDSPaqycEZEDeR45wDjYHMyB5I9fYVjtGC4ZuzRocO3wwYfVYeXZgGA+jbeTrUEhl30kDuWhr7+m9OBikYN0rtYit1VT6enw1kAHQFXsYDL9JTEdnUeS/UiA5WMjqIA+LZHQt7GsxQfPnsvUVhvR0eutE8fNd3jqogipDsVSlhW/DI75wUlr591uu+vNYxfrJ+fB5sShmUJIvBAzWXDcYBEDX726CZEtg9s8vllE5X88fWYilSFF60RbEisSgyKJ7sDdsEhGy56MrU1iItA2zJCMEJJfINCUwtkltljfqF9akBz2IXxSg/CoHKUCqkNwaCcuAnTFUYicL2LjD4V1WYnJZir5GHCak0XsObMGvq+dc36Y15iN4MhNKk2MBE5UoNP/veC6AVXq9XKcROJFBpK6NO9KV6jsNbsw8mGzfZnAu64b2B4Evybup0dUmmTeSvqDp97yrx+z9wlexaFVAFLbNAE6YrpbMvpyQpQbgL+Ks74DVV7L9O6IupwVpn/p2s="
      }
    ]
  },
  "status": "success"
}
```
</details>


<details>
 <summary><code>POST /user/:user_rid/ssh_key</code> - Add public SSHKeys for an user.</summary>

* *****URL Params*****

  * `user_rid` - The RID of the user.

* *****Form Params*****

  * `SSH` - SSHkey of the user. Depending on the implementation, there may be limitation on what is accepted as valid.

* *****Success Response:*****
```
200 OK
{
  "data": null,
  "status": "success"
}
```
</details>


<details>
 <summary><code>DELETE /user/:user_rid/ssh_key/:key_rid</code> - Remove a specific public SSHKey for an user.</summary>

* *****URL Params*****

  * `user_rid` - The RID of the user.
  * `key_rid` - The RID of the SSH key.

* *****Success Response:*****
```
200 OK
{
  "data": null,
  "status": "success"
}
```
</details>


### Group API 


All of the following API endpoints are relative to the base API path, which is typically looks like:
```
https://apidomain.com/usermgmt
```
The exact path depends on the project. Contact admin to get the correct base API path.


#### Endpoints

<details>
 <summary><code>GET /groups</code> - Get all available groups</summary>

* *****Success Response:*****
```
200 OK
{
  "data": {
    "all_groups": [
      {
        "Guid": 1001,
        "Name": "mygroup1"
      },
      {
        "Guid": 1002,
        "Name": "mygroup2"
      }
    ]
  },
  "status": "success"
}
```
</details>

<details>
 <summary><code>GET /groups/:groupname</code> - Get all users of a group</summary>

* *****URL Params*****

  * `groupname` - Name of the group to add the user to.

* *****Success Response:*****
```
200 OK
{ 
  "status" : "success", 
  "data" : null
}
```
</details>

  
<details>
 <summary><code>POST /groups/:groupname/user</code> - Add an user to a group</summary>

* *****URL Params*****

  * `groupname` - Name of the group to add the user to.

* *****Required Form Params*****

  *  `username` - Username for the user. 

* *****Success Response:*****
```
200 OK
{ 
  "status" : "success", 
  "data" : null
}
```
</details>

<details>
 <summary><code>DELETE /groups/:groupname/user/:username</code> - Add an user to a group</summary>

* *****URL Params*****

  * `groupname` - Name of the group to add the user to.
  * `username` - Name of the group to add the user to.

* *****Required Form Params*****

  *  `username` - Username for the user. 

* *****Success Response:*****
```
200 OK
{ 
  "status" : "success", 
  "data" : null
}
```
</details>

### Account API 

All of the following API endpoints are relative to the base API path, which is typically looks like:
```
https://apidomain.com/usermgmt
```
The exact path depends on the project. Contact admin to get the correct base API path.

#### Endpoints

<details>
 <summary><code>GET /account/</code> - Get all accounts, or an account with a specific EID</summary>

* *****Optional query args*****

  * `eid` - EID of the account to get.

* *****Success Response:*****
```
200 OK
{ 
  "status" : "success", 
  "data": {
    1: {"eid": "0123456789abcdef"},
    2: {"eid": "myeidnumber"}
  }
}
```
</details>



<details>
 <summary><code>GET /account/:account_rid</code> - Get a specific account.</summary>

* *****URL Params*****

  * `account_rid` - The RID of the account.

* *****Success Response:*****

```
200 OK
{
  "status": "success"
  "data": {
    "eid": "0123456789abcdef",
    "info": {
      "company": "mycompany",
      "email": "myemail@email.com",
      "fname": "Jane",
      "lname": "Doe"
    },
    "Account_RID": 1
  }
}
```
</details>


<details>
 <summary><code>PUT /account/:account_rid/info</code> - Update account information</summary>

* *****URL Params*****

  * `account_rid` - The RID of the account.

* *****Form Params*****

  * **optional parameters**

    * All parameters (excluding <code>eid</code>) are used to update or add information. 

* *****Success Response:*****
```
200 OK
{ 
  "status" : "success", 
  "data" : null
}
```
</details>
  
<details>
 <summary><code>DELETE /account/:account_rid/info/<urlsafe_key></code> - Delete account information</summary>

* *****URL Params*****

  * `account_rid` - The RID of the account.

  * `urlsafe_key` - Key to delete. Must be URL quoted string (e.g. " " to "+").

* *****Success Response:*****
```
200 OK
{ 
  "status" : "success", 
  "data" : null
}
```
</details>

<details>
 <summary><code>GET /account/:account_rid/users</code> - Get all users of a specific account.</summary>

* *****URL Params*****

  * `account_rid` - The RID of the account.

* *****Success Response:*****

```
200 OK
{
  "status": "success"
  "data": { 
    "all_users": [ 
      {"RID":1,"username":"u1", "CurrentStatus": "PENDING ACTIVATION"},
      {"RID":2,"username":"u2", "CurrentStatus": "PENDING ACTIVATION"}
    ]
  }
}
```
</details>




