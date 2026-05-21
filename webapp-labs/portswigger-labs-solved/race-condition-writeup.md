# Race Condition Writeup

**Author:** Ihtisham\
**Topic:** Race Conditions (Business Logic Flaws)\
**Platform:** PortSwigger Web Security Academy\
**Tools:** Burp Suite Professional, Web Browser

***

## Table of Contents

1. [Lab 1 — Limit overrun](race-condition-writeup.md#lab-1--limit-overrun)
2. [Lab 2 — Bypassing rate limits via race conditions](race-condition-writeup.md#lab-2--bypassing-rate-limits-via-race-conditions)
3. [Lab 3 — Multi-endpoint race conditions](race-condition-writeup.md#lab-3--multi-endpoint-race-conditions)
4. [Lab 4 — Single-endpoint race conditions](race-condition-writeup.md#lab-4--single-endpoint-race-conditions)
5. [Lab 5 — Partial construction race conditions](race-condition-writeup.md#lab-5--partial-construction-race-conditions)
6. [Lab 6 — Time-sensitive vulnerabilities](race-condition-writeup.md#lab-6--time-sensitive-vulnerabilities)

***

## Before You Start

### What is a Race Condition?

A race condition happens when a website tries to do multiple things at the same time but gets confused because the steps overlap.

Imagine a website where you can apply a 10% discount code. The server does this:

1. Check if the code is valid.
2. Apply the discount.
3. Mark the code as "used".

If you send 20 requests at the exact same millisecond, the server might check all 20 requests at step 1 before it has a chance to mark the code as "used" at step 3. The result? You get a 200% discount.

### How we test this

In Burp Suite, you can send requests in **parallel**. You put multiple requests into a Tab Group in the Repeater tool, and use the "Send group (parallel)" option. This fires all the requests at the exact same millisecond.

***

## Lab 1 — Limit overrun

**Goal:** Use a race condition to apply a discount code multiple times and buy the leather jacket.

### Steps

**Step 1:** Log in and add the Leather Jacket to your cart.

![Adding item to cart](<../../.gitbook/assets/Unknown image (111)>)

**Step 2:** Apply the `PROMO20` discount code. The server applies it, but if you try to apply it again, it says "Coupon already applied".

**Step 3:** Intercept the request that applies the discount code. Send it to Burp Repeater.

![Intercepting discount request](<../../.gitbook/assets/Unknown image (112)>)

**Step 4:** In Repeater, duplicate the request many times. Group them all into a Tab Group and send them in parallel.

![Grouping requests to send in parallel](<../../.gitbook/assets/Unknown image (113)>)

The server processes them all at once. Several of the requests succeed because the server hasn't updated the "already applied" state yet.

**Step 5:** Check your cart. The discount has been applied multiple times, bringing the price down to an affordable level!

![Cart showing multiple discounts applied](<../../.gitbook/assets/Unknown image (114)>)

**Step 6:** Checkout and buy the jacket to solve the lab.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (115)>)

***

## Lab 2 — Bypassing rate limits via race conditions

**Goal:** Brute-force Carlos's password to log in and delete him. The site normally locks you out after 3 failed attempts.

### Steps

**Step 1:** The login page normally locks you out if you guess the password wrong 3 times.

**Step 2:** Intercept a login request and send it to Burp Intruder.

![Intruder showing login request](<../../.gitbook/assets/Unknown image (116)>)

**Step 3:** Change the username to `carlos`. We have a list of 30 possible passwords. If we test them normally, we will get locked out. But if we send all 30 requests at the exact same time, the server checks them all before it has a chance to trigger the lockout!

Create a resource pool with 30 concurrent connections to send them all at once.

![Intruder resource pool settings](<../../.gitbook/assets/Unknown image (117)>)

**Step 4:** Look for a 302 response (a successful login). You will find one!

**Step 5:** Use that password to log in as Carlos, go to the Admin panel, and delete the user.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (118)>)

***

## Lab 3 — Multi-endpoint race conditions

**Goal:** Buy the expensive leather jacket by confusing the cart and checkout endpoints.

### Steps

**Step 1:** In this lab, we have an expensive item (ID 1) and a cheap item (ID 2).

**Step 2:** Add the cheap item (ID 2) to your cart and go to checkout. Capture both the "Add to cart" request and the "Checkout" request in Burp.

![Capturing requests](<../../.gitbook/assets/Unknown image (119)>)

**Step 3:** Send both requests to Repeater. Change the `productId` in the "Add to cart" request to `1` (the expensive jacket).

![Changing ID to 1](<../../.gitbook/assets/Unknown image (120)>)

**Step 4:** Group the "Add to cart" (for ID 1) and the "Checkout" request together in a Tab Group in Repeater.

![Grouping the two different requests](<../../.gitbook/assets/Unknown image (121)>)

**Step 5:** Send them in parallel.

![Sending the group in parallel](<../../.gitbook/assets/Unknown image (122)>)

What happens is the server validates your cart total (which is cheap because it only had ID 2), but right before it finishes the checkout, your parallel request adds ID 1 to the cart. It checks out the new cart using the old, validated cheap total.

**Step 6:** You successfully buy the expensive jacket for the price of the cheap one.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (123)>)

***

## Lab 4 — Single-endpoint race conditions

**Goal:** Claim the `carlos` admin email address by racing the email update function.

### Steps

**Step 1:** The site lets you update your email. You enter a new email, it sends a confirmation link, you click it, and your email changes.

![Email update form](<../../.gitbook/assets/Unknown image (124)>)

**Step 2:** Intercept the email update request. Group multiple identical requests to change the email.

In some requests, set the email to your own controlled email. In others, set it to the admin email `carlos@ginandjuice.shop`.

![Repeater showing parallel email updates](<../../.gitbook/assets/Unknown image (125)>)

**Step 3:** Send them in parallel. The server gets confused and sends a confirmation link to YOUR email, but internally associates it with the `carlos` email due to the race condition.

**Step 4:** Click the confirmation link you received.

![Clicking confirmation link](<../../.gitbook/assets/Unknown image (126)>)

**Step 5:** Your account is now linked to the admin email. Go to the admin panel and delete Carlos.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (127)>)

***

## Lab 5 — Partial construction race conditions

**Goal:** Exploit how the site generates password reset tokens to get a token for Carlos.

### Steps

**Step 1:** When you request a password reset, the server generates a token.

If you don't send a session cookie, the server takes slightly longer to generate a new session. We can use this timing difference to race two requests.

![Request with no cookie](<../../.gitbook/assets/Unknown image (128)>)

**Step 2:** The server creates a new CSRF token and session.

![Response with new tokens](<../../.gitbook/assets/Unknown image (129)>)

**Step 3:** We create two password reset requests. One for our user, one for `carlos`. We send them at exactly the same time.

![Two reset requests in parallel](<../../.gitbook/assets/Unknown image (130)>)

**Step 4:** Because they hit the server at the exact same moment, the server's partial construction logic fails. It generates ONE reset token and assigns it to both users!

![Token generated for both users](<../../.gitbook/assets/Unknown image (131)>)

**Step 5:** Check your email to get the reset link (which contains the shared token).

![Getting reset link from email](<../../.gitbook/assets/Unknown image (132)>)

**Step 6:** Use that same token, but change the username to `carlos` in the password reset submission. You can now change Carlos's password!

![Changing Carlos's password](<../../.gitbook/assets/Unknown image (133)>)

**Step 7:** Log in as Carlos, access the admin panel, and delete the user to solve the lab.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (134)>)

***

## Lab 6 — Time-sensitive vulnerabilities

**Goal:** Bypass email verification by confirming the email before the server generates the required token.

### Steps

**Step 1:** We want to register an account with `@ginandjuicee` (an email we don't own). Normally, we can't confirm it because we don't have the token.

![Registering with the target email](<../../.gitbook/assets/Unknown image (135)>)

**Step 2:** Look at the backend code. It checks for a token. If we don't provide a token, the value is `null`. We can target this `null` state during a specific race window before the real token is attached to the user account.

![Backend code snippet](<../../.gitbook/assets/Unknown image (136)>)

**Step 3:** The confirmation endpoint is `/confirm?token[]=`. By leaving it empty, we pass a null/empty token.

![Confirmation request with empty token](<../../.gitbook/assets/Unknown image (137)>)

**Step 4:** Send the registration request to Burp Intruder.

![Intruder registration request](<../../.gitbook/assets/Unknown image (138)>)

**Step 5:** We use a Python script in Burp's Turbo Intruder. The script sends 1 registration request, and then immediately sends 50 confirmation requests (with the empty token) in parallel.

The goal is to hit the confirmation endpoint at the exact millisecond between when the user is created and when their actual token is generated, catching it while the expected token is still `null`.

![Turbo Intruder script](<../../.gitbook/assets/Unknown image (139)>)

**Step 6:** Look at the responses in Turbo Intruder. We will eventually see a 200 OK response on the confirmation, meaning we successfully confirmed the account without the real token.

![Successful confirmation response](<../../.gitbook/assets/Unknown image (140)>)

**Step 7:** Log in with the account we just registered and verified.

![Logging in with registered account](<../../.gitbook/assets/Unknown image (141)>)

**Step 8:** Access the admin panel and delete Carlos. Lab solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (142)>)

***

## How to Prevent Race Conditions

1. **Database Locking:** Use pessimistic or optimistic locking in your database so that only one request can modify a row at a time.
2. **Atomic Operations:** Make sure operations that check a state and then update it happen as a single, uninterrupted transaction.
3. **Session Consistency:** Don't rely on multi-step state changes that share data across different threads or processes.

***

> **Disclaimer:** This writeup is for educational purposes only. All labs were done on PortSwigger's intentionally vulnerable platform. Don't try this on real systems without permission.
