# Directory-Room-Writeup
Directory Room TryHackMe Writeup

Disclaimer: This room proved to be really hard, especially after I dived down into the pcap file and understood what the questions wanted from me (I didn't have experience with kerberos and its traffic in the raw and I had to search google and in the end even read the other people's writeups (props to this particularly - https://medium.com/@seenujaat68/directory-tryhackme-85d3a3d12588) for directions and hints, the room doesn't have hints at all).

-What ports did the threat actor initially find open? Format: from lowest to highest, separated by a comma.

After downloading the pcap file (84 Mb, omg):

<img width="2861" height="831" alt="image" src="https://github.com/user-attachments/assets/f262db6a-6219-420c-804f-6c22aba733ec" />

I browsed it for a while and at the very start we see lots of requests 10.0.2.74 (the attacker presumably) to 10.0.2.75, this looks like a typical
port scan. I figured that for the first question the easy way is to filter like this: 
ip.src==10.0.2.75 && tcp.flags==0x12, 
filter for requests from victim and with SYN|ACK flags (indicating the port is open):

<img width="2685" height="851" alt="image" src="https://github.com/user-attachments/assets/461db04b-31f5-4ee9-8d4e-fa97ea8e2358" />

And we pretty much can answer the first question - we need the ports in ascending order: 53,80,88,135,139,389,445,464,593,636,3268,3269,5357

-The threat actor found four valid usernames, but only one username allowed the attacker to achieve a foothold on the server. What was the username? Format: Domain.TLD\username

I checked statistics and protocols and found kerberos (After scrolling the pcap a little you also see multiple NTLMSSP messages hinting at Windows authentication protocols):

<img width="653" height="326" alt="image" src="https://github.com/user-attachments/assets/4968c7ce-ea95-4abd-8811-56ade6785dd1" />

I then used chatgpt to look for filters:

<img width="1750" height="458" alt="image" src="https://github.com/user-attachments/assets/12cb4232-7dee-467c-aa19-bade19bf0b0e" />

I checked the docs and there's not errcode (only kerberos.error_code) so I just filtered for kerberos.msg_type == 11 (Successfull initial logon):

<img width="1532" height="186" alt="image" src="https://github.com/user-attachments/assets/e540edfe-b269-464b-996c-5d59499cbfbc" />

Only 2 packets and after checking packet details we see the answer: directory.thm\larry.doe

-The threat actor captured a hash from the user in question 2. What are the last 30 characters of that hash?

We need to expand the kerberos field and in the enc-part we find the cipher field which is what we're looking for: 55616532b664cd0b50cda8d4ba469f

Why did is answer in the second packet, not first one (they both send the encrypted cipher) - turns out that is because the latter packet is etype 23
which is crackable and used for AS-REP roasting, while etype 18 is not crackable (AES256), so the attacker captured an easy to crack short hash.

-What is the user's password?

We need to decrypt this etype 23 hash, I went to hashcat docs (https://hashcat.net/wiki/doku.php?id=example_hashes):

<img width="1826" height="370" alt="image" src="https://github.com/user-attachments/assets/2fdd0e51-fd25-467a-92c9-7bfb6e1f4068" />

We have the structure that we need to make and after adjustments the hash becomes: $krb5asrep$23$larry.doe@DIRECTORY.THM:f8716efbaa984508ddde606756441480$805ab8be8cfb018a282718f7c040cd43924c6f9afeb6171230bbd3dccc79294dcf2f877a44c1a0981aadb7bb7a9510dd52d8dda4039ef4dcb444f18c9902be1623035e10aebf16ce4bdf5f7064f480e67e96ec2eb32bad95c5a1247bd7a241273fe80e281f4e6a99926f7969fcf803190c7096b947a33407f8578d4c0fb8b52d2aa8d0405a44b72bd21e014563cb71e82aee0e12538d0d440c930b98abf766e18ddc99a964e6e812ecf8dc8994a912a02074d40e5e6906915c1d216653d45df88636b51656f2c37de2020a2fd86ee7ecf6f0afe3f509fd31144e1573f9587155616532b664cd0b50cda8d4ba469f

Run command hashcat -a 0 -m 18200 hash.txt rockyou.txt

<img width="985" height="441" alt="image" src="https://github.com/user-attachments/assets/e8d7fd4d-8933-49b7-b8cf-108a20706874" />


After only about 30 seconds, hash is cracked and the answer is Password1!

-What were the second and third commands that the threat actor executed on the system? Format: command1,command2

I thought initially that since we have the NTLMSSP password we can input it in Wireshark (Edit-Prefs-Protocols) and then filter for ntlmssp but Wireshark doesn't do that, so we need to run a script to decrypt the actual WinRM commands sent.


