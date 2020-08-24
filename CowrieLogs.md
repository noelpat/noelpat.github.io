# Introduction

After taking a look at some cowrie json logs myself, I wanted to create and share another way to analyze cowrie json logs with a Python script at the command line.

First, what is cowrie?
> Cowrie is a medium to high interaction SSH and Telnet honeypot designed to log brute force attacks and the shell interaction performed by the attacker. In medium interaction mode (shell) it emulates a UNIX system in Python, in high interaction mode (proxy) it functions as an SSH and telnet proxy to observe attacker behavior to another system.
> Source: https://github.com/cowrie/cowrie

# Analysis

This Python script will filter out commands from the cowrie json logs by the individual session IDs and IP addresses. The output will be in the following format:  
Count : timestamp, session ID, source IP, command(s)

See the Python 2 script below:
```Python
"""
-Python2 script for iterating through a json log file from a cowrie honeypot.
-Script will filter by session id and command inputs from attacker IP addresses.
-Be aware that when creating the script there could multiple connections from the same IP address.
-You can use the "session" attribute to seperate/distinguish between different connections.
"""

import sys
import urllib
from urllib import urlopen
import json

if len(sys.argv) != 2:
    print("Missing argument.")
    print("Usage: python logScrape.py cowrie.json.log")
    exit()

inFile = sys.argv[1]
data = []
with open(inFile) as f:
    for line in f:
        data.append(json.loads(line))

def get_commands(full_list):
    count_dict = {}
    current = 0

    for item in full_list:
        if item['eventid'] == "cowrie.command.input" or item['eventid'] == "cowrie.command.failed":
            # make list of comamnds
            count_dict[current] = item['timestamp'], item['session'], item['src_ip'], item['input']
            current += 1

    return count_dict

# Get a list of all of the commands and session ids
command_list = get_commands(data)

print("\nFiltering commands from individual IP addresses and session ids...\n")
for key in command_list:
    print(key, command_list[key])
```

See an example of my output screenshot below:
![Example Output](https://i.imgur.com/qSq42x1.png)

If you want to send the output to a text file simply add "> output.txt" to the end of your command at your terminal:
```
python logScrape.py cowrie.json.log > output.txt
```

Relevant reference for more example code: https://zeroaptitude.com/zerodetail/analyzing-cowrie-honeypot-results/
