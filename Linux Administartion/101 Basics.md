Computer can be in many forms like mobile, Laptop, Washing machine, Server, AC ,Desktop
---
##### So what does it mean to be a computer it should have the following:
- RAM
- ROM
- CPU
- OS
- Processor
- **it should be a ip enabled device**
---
##### Servers use linux over windows or macos because:
- secure -SSH
- No graphics
  -  which leads to Low power , low bandwidth , low RAM ,fast
- heavy performance
- free
- runs for years(unlike windows which hangs)
- Anti virus not required
- no hardware and software tightly coupled
- opensource

> in general terms we call linux with different terms like BOX or NODE or SERVER all mean the same
---
#### CLIENT SERVER ARCHITECTURE
- if we use mobile to access the laptop or control laptop then mobile is client and laptop is server
![example 1](../Pictures/Linux%20Administration/clientserver1.png)
- if we use laptop to access the mobile or control mobile then laptop is client and mobile is server
![example 2](../Pictures/Linux%20Administration/clientserver2.png)
- if i access facebook.com then
![example 3](../Pictures/Linux%20Administration/clientserver3.png)
---
#### AUTHENTICATION MECHANISMS
- What you remember *(like password or security questions etc..)*
- What you have *(RSA tokens, OTP, Authentications)*
- What you are *(Fingerprints, Retina, Palm)*
---
#### PUBLIC KEY AND PRIVATE KEY MECHANISMS
- SSH keys creation we use `ssh-keygen -f filename` this give two files filename and filename.pub
- filename.pub has public key its safe to share it but the private key is secret and should not be shared
---
#### FIREWALL
- this controls the incoming traffic and outgoing traffic
- the incoming traffic will be checked definetely
- the outgoing traffic is not checked in general but will be checked in some special cases like NASA DRDO etc ...


