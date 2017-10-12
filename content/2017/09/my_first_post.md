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
