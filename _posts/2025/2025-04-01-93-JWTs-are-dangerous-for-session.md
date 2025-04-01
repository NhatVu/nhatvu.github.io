---
layout: post
title:  "93. JWTs are dangerous for user session - Redis blog"
date:   2025-04-01 08:02:00 +0000
category: technical
---
- [1. Traditonal approach](#1-traditonal-approach)
- [2. JWT and its problems](#2-jwt-and-its-problems)
- [3. Why is JWT dangerous for user authentication?](#3-why-is-jwt-dangerous-for-user-authentication)
  - [3.1. Logout doesn’t really log you out!](#31-logout-doesnt-really-log-you-out)
  - [3.2. Blocking users doesn’t immediately block them.](#32-blocking-users-doesnt-immediately-block-them)
  - [3.3. Could have stale data](#33-could-have-stale-data)
  - [3.4. Length of the token](#34-length-of-the-token)
  - [3.5. Revoked token database](#35-revoked-token-database)
- [4. Where can I use it?](#4-where-can-i-use-it)
- [5. References](#5-references)

For more details, please refer to the original blog at [1]. This article only summaries the main idea of why JWT are dangerous for user session. 

### 1. Traditonal approach 

![alt text](/assets/images/2025/93-traditional-approach.png)

The main problems with the traditional approach is slow and center session database. Every request needs to talk with database that will slow down the overall response time. When multiple services interact with session database, it will become a bottleneck and single point of failure (I think)

There are two ways to solve this problem: 
- Option 1: Somehow eliminate database lookup for users completely (i.e. eliminate step four). 
- Option 2: Make the extra database lookup much faster so that the additional hop won’t matter. 

With Optional 2, we can use fast database like Redis. 

With Option 1, we will come with JWT solution. 

### 2. JWT and its problems 
JWT is self-contain token, stateless and don't need to connect to sever to verify. More details on JWT, you can find on internet. 

For common vulnerablities of JWT, check at [2]
- None hashing algorithm: when alg:"none" is valid
- Token Sidejacking: when a token has been intercepted/stolen by an attacker and they use it to gain access to the system using targeted user identity.
- No Built-In Token Revocation by the User: This problem is inherent to JWT because a token only becomes invalid when it expires. The user has no built-in feature to explicitly revoke the validity of a token. This means that if it is stolen, a user cannot revoke the token itself thereby blocking the attacker.
- Weak Token Secret: sign algorithm depends entirely on secret value. Hacker can crack the secret using tools such as John the Ripper or Hashcat.
- Token Information Disclosure: information in payload portion normally do base64 encode, not encrypted. Can easy be decode and expose information. 

### 3. Why is JWT dangerous for user authentication?

The biggest problem with JWT is the token revoke problem. Since it continues to work until it expires, the server has no easy way to revoke it.

Below are some use cases that’d make this dangerous.

#### 3.1. Logout doesn’t really log you out!
 Because JWT is self-contained and will continue to work until it expires. So if someone gets access to that token during that time, they can continue to access it until it expires.

#### 3.2. Blocking users doesn’t immediately block them.
Imagine you are a moderator of Twitter. And as a moderator, you want to quickly block someone from abusing the system. You can’t, again for the same reason. Even after you block, the user will continue to have access to the server until the token expires.

#### 3.3. Could have stale data
Imagine the user is an admin and got demoted to a regular user with fewer permissions. Again this won’t take effect immediately and the user will continue to be an admin until the token expires.

#### 3.4. Length of the token
In many complex real-world apps, you may need to store a ton of different information. And storing it in the JWT tokens could exceed the allowed URL length or cookie lengths causing problems. Also, you are now potentially sending a large volume of data on every request.

#### 3.5. Revoked token database 
One popular solution is to store a list of “revoked tokens” in a database and check it for every call. And if the token is part of that revoked list, then block the user from taking the next action. But then now you are making that extra call to the DB to check if the token is revoked and so deceives the purpose of JWT altogether. 

### 4. Where can I use it?
There are scenarios where you are doing server-to-server (or microservice-to-microservice) communication in the backend and one service could generate a JWT token to send it to a different service for authorization purposes. And other narrow places, such as reset password, where you can send a JWT token as a one-time short-lived token to verify the user’s email.

### 5. References 
1. [JSON Web Tokens (JWT) are Dangerous for User Sessions—Here’s a Solution](https://redis.io/blog/json-web-tokens-jwt-are-dangerous-for-user-sessions/)
2. [JSON Web Token Cheat Sheet for Java](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)



