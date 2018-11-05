+++
author = "Pat Coughlin"
author_url = "https://www.linkedin.com/in/patcoughlin/"
categories = ["Pat Coughlin", "Encore","End User Compute"]
date = "2018-10-16"
description = "Why consider a virtual desktop solution"
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "V D Why?"
type = "post"
 
+++
 
*Virtual Desktops are a topic that runs hot and cold through the enterprise space.  Many enterprises begin pilots and projects in the space before building a clearly defined plan, and these are the environments that need additional remediation down the road.  Before we start in with architecture, getting a firm answer to “Why do we need them?” is critical to a successfully scope, cost, and validate the design.  This month we will focus on the security use case which is one of the ‘easy wins’ for virtual end user compute.*

# Use Case #1 – Security

Presenting windows applications across security boundary’s (think PCI or HIPPA) can present many challenges.  Typically, multiple communication paths and multiple servers are involved with delivering these multi-tier applications to the user’s endpoint.  If we must open the firewall rules to much, this can cause the user’s endpoints to become part of that data protection scope.  The security controls in a protected space degrade the user experience and can inspire them to create dangerous workarounds.  Alternatively, we can place a tightly controlled virtual desktop inside the PCI domain and allow the user to access this system from their “out of scope” endpoint.  The major VDI vendors can provide access via a SSL gateway so only one firewall rule (ALLOW 443 TCP) to a single ip address is all that is required to grant targeted access into the security domain.  Also, additional policy controls like preventing file copies and printing are standard features the VDI product suites and can complement existing controls in those spaces. 
 
For most security use cases, I recommend non-persistent desktops.  These desktops are literally destroyed after the user completes their activities and rebuild to the pristine standard “gold” image.  This can prevent the dreaded “persistent threats” from taking root in your environment.  The constrained virtual environment allows to you to created tasked focused restrictions like email and web browsing prevention inside the secured environment and allow more relaxed policies on the user’s endpoint.  Non-persistent images allow software updates and patches to be applied to a single curated instance of the OS and then, once validate a simple reboot places the updated image into production. 

VDI deployments can simplify audit for secure environments.  Every access to the environment is automatically logged with the user’s information including the start and end time of the session.  The policy engines in the environment create visible record of the security posture of the space.  In some highly restricted spaces the ability to record the entire session including mouse clicks is of interest and can be enabled within the products. One vendor even can watermark the screen with user identifying information if the “picture of the screen” issue is a serious concern.
If providing seamless simple access to hardened environments with specific security requirement is in scope for your project, a targeted VDI deployment should be on your radar.

-Pat Coughlin