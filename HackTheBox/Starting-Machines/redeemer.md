# Redeemer — HackTheBox Starting Machine
**Date:** 2025-05-18
**Difficulty:** Very Easy
**Category:** Redis / Enumeration
**Status:** ✅ Completed

---

## Summary
Redeemer is a starting machine focused on Redis enumeration. First we identify a non-standard open port via full port scanning, connect to an unauthenticated Redis instance, and extract the flag from the in-memory database.

---

## Enumeration

### Ping
First I ping the victim machine, testing if I have connection to it:
```
ping <target_ip>
```

### Nmap — Initial Scan
Next nmap, this will allow us to see open ports and see what we can target:
```
nmap -sV <target_ip>
```
No ports showed up with a standard nmap -sV command — the port must be higher than the 1000 max.

### Nmap — Full Port Scan
We run nmap with -p-, this tests all ports. However this is really slow, so we use -T5 which runs really fast:
```
nmap -p- -T5 -sV <target_ip>
```
After nmap, we found port **6379** which is a Redis port.

> ⚠️ **OPSEC Note:** -T5 will probably get logged on a SIEM due to how fast we're pinging. In a real engagement use -T2 or -T3 to stay quieter.

---

## Foothold

### Redis
After googling we learn that Redis is an in-memory database — this means we will be able to access it on the target machine since it is stored in memory.

I found the Redis CLI utility docs [here](https://redis.io/docs/latest/develop/tools/cli/) and used Ctrl-F to search for the port flag. Found the command:
```
redis-cli -h <target_ip> -p 6379
```
After running this, we gained access to the database.

---

## Exploitation

### Server Info
I then googled Redis commands and found the server info using INFO:
```
INFO
```
Reference: [Redis Commands Doc](https://redis.io/docs/latest/commands/)

### Select the Database
HTB has us go to index 0, so we run:
```
SELECT 0
```
This brings us into DB 0.

### Check the Keyspace
We run INFO again — at the very bottom there is a keyspace section showing `db0:keys=4`. This tells us there are 4 keys in the database.
```
INFO keyspace
```

### List All Keys
After googling how to return all keys, I found it [here](https://stackoverflow.com/questions/5252099/redis-command-to-get-all-available-keys):
```
KEYS *
```
We find one key called `flag`.

### Get the Flag
After some tedious googling, I found how to return a string from a key using GET:
```
GET flag
```
**Result:** [FLAG OBTAINED] ✅

---

## Key Takeaways
- Default nmap only scans the top 1000 ports — always run -p- on HTB machines
- -T5 is fast but loud and would get flagged on a monitored network
- Redis with no authentication = immediate access, no exploit needed
- Always check keyspace before dumping keys so you know what you're working with

---

## Tools Used
| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service enumeration |
| `redis-cli` | Redis database interaction |

---

## References
- [Redis CLI Documentation](https://redis.io/docs/latest/develop/tools/cli/)
- [Redis Commands Reference](https://redis.io/docs/latest/commands/)
- [Redis KEYS command](https://stackoverflow.com/questions/5252099/redis-command-to-get-all-available-keys)
