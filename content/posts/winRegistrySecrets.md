+++
title = "Win-Registry-Secrets"
date = "2025-03-20T13:50:08-07:00"
author = "Ceald"
authorTwitter = "" #do not include @
cover = ""
tags = ["windows", "new", "go"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
+++
## Intro
The windows registry is a system database that contains keys and values. Some things in the registry include; Windows credentials, cached passwords, usernames, and other credentials. In windows a group of keys is called a "hive" the hives that are the cool ones are; SAM, System, and Security.

## SYSTEM Hive

The most important registry hive is the "System" hive, in the key: `CurrentControlSet\Control\Lsa` there are the necessary components to craft the boot key which will be used to decrypt the rest of the registry database to get things like hashes for users. Here's some example code from "Go-Go-Gadget-Katz" for getting the boot key:
```go
key, err := registry.OpenKey(registry.LOCAL_MACHINE, 
		`SYSTEM\CurrentControlSet\Control\Lsa`, registry.READ|registry.QUERY_VALUE)
	if err != nil {
		return nil, fmt.Errorf("error opening LSA key: %v", err)
	}
	defer key.Close()

	// Read the values from the registry
	names := []string{"JD", "Skew1", "GBG", "Data"}
	for _, name := range names {
		subKey, err := registry.OpenKey(key, name, registry.READ|registry.QUERY_VALUE)
		if err != nil {
			return nil, fmt.Errorf("error opening subkey %s: %v", name, err)
		}
		defer subKey.Close()

		// Pre-allocate a large enough buffer for the class
		const maxClassSize = 1024 // Larger buffer size
		classBytes := make([]uint16, maxClassSize)
		classLen := uint32(maxClassSize)
		
		// Get the class string with all other parameters
		var subKeyCount, maxSubKeyLen, maxClassLen uint32
		var valueCount, maxValueNameLen, maxValueLen uint32
		var secDescLen uint32
		var lastWriteTime windows.Filetime

		err = windows.RegQueryInfoKey(
			windows.Handle(subKey),
			&classBytes[0],
			&classLen,
			nil,
			&subKeyCount,
			&maxSubKeyLen,
			&maxClassLen,
			&valueCount,
			&maxValueNameLen,
			&maxValueLen,
			&secDescLen,
			&lastWriteTime,
		)
```
In the code you are looping through the registry keys to get the boot key then you would decode it from UTF16 and hex to then unscramble the boot key.


to be continued......