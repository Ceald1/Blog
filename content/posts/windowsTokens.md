+++
title = "Windows-Tokens"
date = "2025-03-20T14:17:56-07:00"
author = "Ceald"
authorTwitter = "" #do not include @
cover = ""
tags = ["", ""]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
+++

## Intro

In windows things like access permissions are handled with tokens. Think of them as API tokens, they have certain permissions that you can do like impersonate other users or enable debugging on windows. The things that will be looked at in this post is how you can use windows tokens to get lower permissions than the Administrator user.

## Stealing Tokens Via Windows Processes

Getting right to the point. In windows processes also have access tokens and certain processes like winlogon you can duplicate the token and use it to run commands as another user. This is done by first opening the process to get the token like so:
```go

func findSystemProcess() (*windows.Handle, error) {
	h, err := windows.CreateToolhelp32Snapshot(windows.TH32CS_SNAPPROCESS, 0) // take snapshots of processes.
	if err != nil {
		return nil, fmt.Errorf("CreateToolhelp32Snapshot failed: %v", err)
	}
	defer windows.CloseHandle(h) // close it when the function closes
	var pe windows.ProcessEntry32 // create the process entry struct.
	pe.Size = uint32(unsafe.Sizeof(pe))
	if err = windows.Process32First(h, &pe); err != nil {
		return nil, fmt.Errorf("Process32First failed: %v", err)
	}
	systemProcesses := []string{"lsass.exe", "winlogon.exe", "services.exe"} // system processes with low level token access.
	for {
		name := windows.UTF16ToString(pe.ExeFile[:])
		for _, procName := range systemProcesses {
			if name == procName {
				handle, err := windows.OpenProcess(windows.PROCESS_QUERY_INFORMATION, false, pe.ProcessID) // open the process and get the handle
				if err != nil {
					return nil, fmt.Errorf("OpenProcess failed for %s: %v", procName, err)
				}
				return &handle, nil // return no error, break out of the loop and return the handle
			}
		}
		err = windows.Process32Next(h, &pe) // next process
		if err != nil {
			if err == syscall.ERROR_NO_MORE_FILES {
				break
			}
			return nil, fmt.Errorf("Process32Next failed: %v", err)
		}
	}
	return nil, fmt.Errorf("No suitable system process found")
}
```
In the "systemProcesses" array there are windows process that are ran by the system user which is a user that has even more access than the Administrator user. The only user that has even more privileges than the system user is the "TrustedInstaller" user which can delete everything in windows.


## Token Attacks

There's 5 different ways to attack windows access tokens.

1. Token Impersonation:
    * Token Duplication using "DuplicateTokenEx" (used in go-go-gadget-katz)
    * Tokens obtained by this method are used with ImpersonateLoggedOnUser to impersonate users
2. Create Process With token
    * Creating a new process using an existing token

to be continued....