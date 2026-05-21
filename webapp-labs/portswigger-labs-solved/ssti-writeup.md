# SSTI Writeup

**Author:** Ihtisham\
**Topic:** SSTI (Server-Side Template Injection)\
**Platform:** PortSwigger Web Security Academy\
**Tools:** Burp Suite Professional, Web Browser

***

## Table of Contents

1. [Lab 1 — Basic SSTI](ssti-writeup.md#lab-1--basic-ssti)
2. [Lab 2 — Basic SSTI (Code Context)](ssti-writeup.md#lab-2--basic-ssti-code-context)
3. [Lab 3 — Server-side template injection using documentation](ssti-writeup.md#lab-3--server-side-template-injection-using-documentation)
4. [Lab 4 — Server-side template injection in an unknown language with a documented exploit](ssti-writeup.md#lab-4--server-side-template-injection-in-an-unknown-language-with-a-documented-exploit)
5. [Lab 5 — Server-side template injection with information disclosure via user-supplied objects](ssti-writeup.md#lab-5--server-side-template-injection-with-information-disclosure-via-user-supplied-objects)
6. [Lab 6 — Server-side template injection in a sandboxed environment](ssti-writeup.md#lab-6--server-side-template-injection-in-a-sandboxed-environment)
7. [Lab 7 — Server-side template injection with a custom exploit](ssti-writeup.md#lab-7--server-side-template-injection-with-a-custom-exploit)

***

## Before You Start

### What is SSTI?

Server-Side Template Injection (SSTI) occurs when user input is unsafely embedded directly into a template engine (like Jinja, Twig, FreeMarker, ERB) before the template is rendered on the server.

Instead of just rendering text, the template engine processes the input as code. This allows an attacker to inject template directives that execute arbitrary code (RCE) on the server, read sensitive files, or disclose internal variables.

**How to find it:** We typically inject mathematical expressions like `<%= 7*7 %>` or `{{7*7}}`. If the server evaluates the math and returns `49`, we know it's executing our template syntax.

***

## Lab 1 — Basic SSTI

**Goal:** Delete the `morale.txt` file from Carlos's home directory.

### Steps

**Step 1:** Browse the application. We see a parameter in the URL that reflects our input onto the page.

![Parameter reflecting on the page](<../../.gitbook/assets/Unknown image (165)>)

**Step 2:** We test for SSTI by passing `<%= 7*7 %>`. The application evaluates it and returns `49`. This syntax (`<%= ... %>`) belongs to **ERB (Ruby)** templates.

**Step 3:** To delete the file, we inject a Ruby payload that executes a system command:

```ruby
<%= system("rm /home/carlos/morale.txt") %>
```

![Successfully executing the payload](<../../.gitbook/assets/Unknown image (166)>)

The command executes on the server, deleting the file and solving the lab.

***

## Lab 2 — Basic SSTI (Code Context)

**Goal:** Delete the `morale.txt` file from Carlos's home directory.

### Steps

**Step 1:** Log in with `wiener:peter` and leave a comment on a blog post. Intercept the request.

![Intercepting comment request](<../../.gitbook/assets/Unknown image (167)>)

**Step 2:** The `blog-post-author` display uses the `user.name` variable. By generating an error with invalid syntax, the error message reveals that the application is using the **Tornado (Python)** template engine.

**Step 3:** We need to execute arbitrary Python code. We can import the `os` module and use `os.system` within the Tornado template syntax:

```python
user.name}}{% import os %}{{os.system('rm /home/carlos/morale.txt')
```

![Executing Tornado payload](<../../.gitbook/assets/Unknown image (168)>)

When the template renders the comment, the Python command executes, deleting the file.

***

## Lab 3 — Server-side template injection using documentation

**Goal:** Identify the engine and delete the `morale.txt` file from Carlos's home directory.

### Steps

**Step 1:** Log in and edit a product template. There is an "Edit Template" feature where we can inject our code.

![Edit template feature](<../../.gitbook/assets/Unknown image (169)>)

**Step 2:** By replacing values with intentionally broken syntax, we generate an error message that explicitly states it is using **FreeMarker (Java)**.

**Step 3:** We look up the FreeMarker documentation for executing shell commands. We inject the FreeMarker execution payload:

```html
<#assign ex="freemarker.template.utility.Execute"?new()> ${ ex("rm /home/carlos/morale.txt") }
```

![Executing FreeMarker payload](<../../.gitbook/assets/Unknown image (170)>)

The command deletes the file and solves the lab.

***

## Lab 4 — Server-side template injection in an unknown language with a documented exploit

**Goal:** Identify the engine and delete the `morale.txt` file from Carlos's home directory.

### Steps

**Step 1:** The `message` parameter in the URL is reflecting our input.

![Parameter reflecting on the page](<../../.gitbook/assets/Unknown image (171)>)

**Step 2:** We cause an error by sending invalid template syntax. The error reveals it's running a **Java**-based application.

![Java error message](<../../.gitbook/assets/Unknown image (172)>)

**Step 3:** Further testing reveals it's using the **Handlebars** template engine. We find a known Handlebars RCE payload online.

![Handlebars RCE payload](<../../.gitbook/assets/Unknown image (173)>)

**Step 4:** We modify the payload to execute our `rm` command instead of `whoami`.

```javascript
{{#with "s" as |string|}}
  {{#with "e"}}
    {{#with split as |conslist|}}
      ...
      rm /home/carlos/morale.txt
      ...
```

**Step 5:** We URL-encode the payload and inject it into the `message` parameter.

![URL encoding the payload](<../../.gitbook/assets/Unknown image (174)>)

The payload executes, deleting the file.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (175)>)

***

## Lab 5 — Server-side template injection with information disclosure via user-supplied objects

**Goal:** Steal the `SECRET_KEY` from the Django settings.

### Steps

**Step 1:** Log in as `content-manager:C0nt3ntM4n4g3r`. There is a feature to edit product templates.

![Edit template feature](<../../.gitbook/assets/Unknown image (176)>)

**Step 2:** By injecting invalid syntax, we get an error revealing the engine is **Django (Python)**.

![Django error message](<../../.gitbook/assets/Unknown image (177)>)

**Step 3:** Instead of trying to get Remote Code Execution (RCE), the lab asks us to disclose a secret variable. In Django, the `settings` object is often accessible in the template context.

![Django documentation about settings](<../../.gitbook/assets/Unknown image (178)>)

**Step 4:** We inject `{{settings.SECRET_KEY}}` into the template.

![Injecting the payload](<../../.gitbook/assets/Unknown image (179)>)

When the product page renders, it leaks the secret key on the screen. We submit it to solve the lab.

***

## Lab 6 — Server-side template injection in a sandboxed environment

**Goal:** Break out of the FreeMarker sandbox and read `my_password.txt`.

### Steps

**Step 1:** Log in to the application. The product preview feature reflects user input.

![Product preview feature](<../../.gitbook/assets/Unknown image (180)>)

**Step 2:** We generate an error to confirm the engine is **FreeMarker**. However, the standard `freemarker.template.utility.Execute` payload from Lab 3 is blocked because this environment is strictly sandboxed.

![Sandbox error message](<../../.gitbook/assets/Unknown image (181)>)

**Step 3:** We need a known sandbox escape for FreeMarker. We use a complex payload that exploits the `ProtectionDomain.classLoader` to bypass the sandbox restrictions and execute commands.

```html
<#assign classloader=Product.class.protectionDomain.classLoader>
<#assign owc=classloader.loadClass("freemarker.template.ObjectWrapper")>
<#assign dwf=owc.getField("DEFAULT_WRAPPER").get(null)>
<#assign ec=classloader.loadClass("freemarker.template.utility.Execute")>
${dwf.newInstance(ec,null)("cat /home/carlos/my_password.txt")}
```

![Executing the sandbox bypass payload](<../../.gitbook/assets/Unknown image (182)>)

**Step 4:** The payload successfully escapes the sandbox and prints the contents of `my_password.txt`.

![Retrieving the password](<../../.gitbook/assets/Unknown image (183)>)

Submit the password to solve the lab.

***

## Lab 7 — Server-side template injection with a custom exploit

**Goal:** Create a custom exploit to delete `/.ssh/id_rsa`.

### Steps

**Step 1:** Log in. The application has an avatar upload and a blog post feature.

![Application features](<../../.gitbook/assets/Unknown image (184)>)

**Step 2:** Intercepting requests reveals we can edit our avatar and display name. By injecting into the display name and generating errors, we discover the `user` object is accessible in the template.

![Discovering the user object](<../../.gitbook/assets/Unknown image (185)>) ![Error when submitting the user object](<../../.gitbook/assets/Unknown image (186)>)

**Step 3:** We explore the `user` object's methods. We find `user.setAvatar()`.

![Calling user.setAvatar()](<../../.gitbook/assets/Unknown image (187)>)

**Step 4:** The error messages tell us `setAvatar()` requires arguments (filename and MIME type).

![Error: requires more arguments](<../../.gitbook/assets/Unknown image (188)>) ![Error: requires valid MIME type](<../../.gitbook/assets/Unknown image (189)>)

**Step 5:** We provide the correct arguments: `user.setAvatar("filename", "image/jpeg")`.

![Successfully calling setAvatar](<../../.gitbook/assets/Unknown image (190)>)

**Step 6:** When we load our avatar image, the server processes the file path we provided. This proves we can control the avatar file path through the template injection!

![Loading the avatar image](<../../.gitbook/assets/Unknown image (191)>) ![Controlling the file path](<../../.gitbook/assets/Unknown image (192)>)

**Step 7:** We try to set the avatar to `/.ssh/id_rsa` to read it, but the file is empty or unreadable as an image.

![Reading id\_rsa fails](<../../.gitbook/assets/Unknown image (193)>) ![Empty image response](<../../.gitbook/assets/Unknown image (194)>)

**Step 8:** The lab goal is to _delete_ it, not read it. Earlier, we noticed a `gdprDelete()` method in the application source code.

![Discovering gdprDelete()](<../../.gitbook/assets/Unknown image (195)>) ![Source code of gdprDelete()](<../../.gitbook/assets/Unknown image (196)>)

**Step 9:** The `gdprDelete()` function deletes the user and their associated avatar file. We set our avatar path to `/.ssh/id_rsa` using `user.setAvatar()`, and then we invoke `user.gdprDelete()` in the template!

![Executing the custom delete exploit](<../../.gitbook/assets/Unknown image (197)>)

The application deletes the user account along with its "avatar" (which is actually the `id_rsa` file). The lab is solved.

***

## How to Prevent SSTI

1. **Use Logic-less Templates:** Use template engines like Mustache that strictly separate logic from presentation and don't allow arbitrary code execution.
2. **Never Concatenate Input:** Never concatenate user input directly into template strings. Always pass user input as data variables to the template context.
3. **Sandboxing:** If you must allow users to edit templates, use an engine with a secure, battle-tested sandbox mode that completely disables the ability to run system commands or load arbitrary classes.

***

> **Disclaimer:** This writeup is for educational purposes only. All labs were done on PortSwigger's intentionally vulnerable platform. Don't try this on real systems without permission.
