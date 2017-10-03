+++
author = "Mike Flora"
author_url = "https://github.com/MikeFlora"
categories = ["Mike Flora", "Encore"]
date = "2017=10-02"
description = "Introduction the Encore DevOps Blog"
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "My First Blog Post"
type = "post"

+++
# Something about ServiceNow
 
## Execute Shell Commands on MID Server
 
The ServiceNow MID Server has a built-in feature that allows functionality to be extended to any program/application that can be ran, via a shell command, on the MID server.
If data is to be returned to the ServiceNow instance, the program merely needs to write to the `stdout`.

The data that is written to `stdout` is captured and returned to the ServiceNow instance in the response via the ECC Queue.

Therefore, any program/application can merely write a JSON formatted string to `stdout` and the data can be made available to the ServiceNow instance.

This is a nice, quick way to do a simple integration.

 Example:
   A Python script can be written to query something, say an SQL database, and return the results to the ServiceNow instance.

-Mike

-----
# Notes while following the instructions:

Used to clone repo (Windows):
```
Had to use the following to clone the repo.
git clone https://github.com/EncoreTechnologies/blog.git

The following did not work for me:
git clone git@github.com:EncoreTechnologies/blog.git

Cloning into 'blog'...
The authenticity of host 'github.com (192.30.253.112)' can't be established.
RSA key fingerprint is SHA256:nThbg6kXUpJWGl7E1IGOCspRomTxdCARLviKw6E5SY8.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'github.com,192.30.253.112' (RSA) to the list of known hosts.
Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

```
hugo new blog/2017/09/my-first-post.md
content\blog\2017\09\my-first-post.md created
```

The "my-first-post.md" file had the following content.
```
+++
author = ""
categories = []
description = ""
linktitle = ""
featured = ""
featuredpath = ""
featuredalt = ""

+++

*** This was from the template in "blog\archetypes\default.md"
*** Not from the "blog.md" file

*** Manually copied the "blog.md" to "my_first_post.md"
```

Tried to push updates...
```
git push origin post/my-first-post
remote: Permission to EncoreTechnologies/blog.git denied to MikeFlora.
fatal: unable to access 'https://github.com/EncoreTechnologies/blog.git/': The requested URL returned error: 403
```