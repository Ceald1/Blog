+++
title = "Delta2"
date = "2024-11-10T16:56:20-07:00"
author = "Ceald"
authorTwitter = "" #do not include @
cover = ""
tags = ["delta2", "new", "cybersecurity", "projects", "python"]
keywords = ["projects", "delta2", "python"]
description = ""
showFullContent = false
readingTime = false
hideComments = false
+++

## Background
    Delta2 is an API that is a Swiss army knife for active directory. I wrote this project because I was sick of having to set up java and remembering how to use bloodhound python for doing active directory boxes on HackTheBox, so I wrote my own. Delta2's primary use is automation like automate kerberosing after mapping the domain or getting shortest paths to admin users and exploiting the necessary users to get to admin users.

    Delta2 was my first successful full-stack application and took many months to build. It uses Memgraph which is an alternative to Neo4j as the graphing database and FastAPI for the API, and it's containerized with docker for easy deployment.


## Process of building
### Plan
    My plan was to build some kind of script or API that will allow me to move through a domain efficiently. There were a few ways I could've done that like using bloodhound and neo4j to find paths in the graph, but that'd require me to read documentation and code that I didn't write. My second option was to "reinvent the wheel" which is what I did. Unfortunately I thought it'd be the easiest way which it probably wasn't, but I learned more than if I were to just make it based off of bloodhound.

### The Beginning
    After I kind of had a plan I started building it. I built an outline for the API with various routes I wanted like Kerberoasting, getting TGTs, AS-REP roasting, and a collection script. Testing was a bit of a pain because I would use boxes from HackTheBox, and sometimes I forgot to take out test code and I would have to take the repository down, remove the test code then put it back up to avoid spoilers for boxes. 

### Collection Script Works
    One big issue with building the collection script was supporting Kerberos and reading nTSecurityDescriptors. nTSecurityDescriptors describe what objects and permissions a user has over another for example: Bob has WriteAll on Administrator. I first realized the issue with Kerberos when I couldn't get ldap3 and impacket to work together like how they work in bloodhound python so I switched to BloodyAD which is an Active Directory privilege escalation framework. So I reverse engineered BloodyAD to figure out how to use it because there's no documentation for how to use its python modules and got it kind of working. I say kind of because now I had to learn how to read nTSecurityDescriptors. You can find how I learned the syntax for nTSecurityDescriptors here:
https://itconnect.uw.edu/tools-services-support/it-systems-infrastructure/msinf/other-help/understanding-sddl-syntax/

    Writing the collection script is what took the longest time because I also had to reverse engineer how bloodhound worked a little and do a bunch of debugging to figure out what information and how to commit it to the graph as well as how to connect nodes.


Post will be updated soon.
