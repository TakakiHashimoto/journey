## 1: check the spec first
- **hostnamectl**
```
Static hostname: takakihashimoto-IdeaPad-Slim-3-14ABR8  
      Icon name: computer-laptop  
        Chassis: laptop -  
     Machine ID: 54f6275e237f4c1ab949e78fc5f3768f  
        Boot ID: d8ac741dd2784fffaccc0e3e14c27e3f  
Operating System: Ubuntu 24.04.4 LTS                      
         Kernel: Linux 7.0.0-34-generic  
   Architecture: x86-64  
Hardware Vendor: Lenovo  
 Hardware Model: IdeaPad Slim 3 14ABR8  
Firmware Version: KYCN31WW  
  Firmware Date: Mon 2024-03-18  
   Firmware Age: 2y 6month 1w 2d
```

- **uname -a** : You can know information about system like kernel version etc
```
Linux takakihashimoto-IdeaPad-Slim-3-14ABR8 7.0.0-34-generic #34~24.04.1-Ubuntu SMP PREEMPT_DYNAMIC Fri Sep  4 15:38:29  
UTC 2 x86_64 x86_64 x86_64 GNU/Linux
```
- **uname -r** : kernel release
```
7.0.0-34-generic
```

- /proc is a pseudo-filesystem: 
    "You’re trying to read something that looks like a file. I’ll use that familiar file interface, but instead of reading bytes from disk, I’ll generate the bytes from my current internal state."
  => Normal file = sitting on SSD and everytime it is spread into the system 
   => /proc/meminfo = a program that read system info and displays it

- **cat /proc/stat** : cpu info
- **ip route** : default route
```
❯ ip route  
default via 10.128.128.128 dev wlp2s0 proto dhcp src 10.175.243.181 metric 600    
10.0.0.0/8 dev wlp2s0 proto kernel scope link src 10.175.243.181 metric 600    
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown    
172.18.0.0/16 dev br-64cfc714b61c proto kernel scope link src 172.18.0.1
```

- **cat /proc/net/dev** : per-network-interface traffic counters maintained by the Linux kernel.
```
❯ cat /proc/net/dev  
Inter-|   Receive                                                |  Transmit  
face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed  
   lo:  104332    1070    0    0    0     0          0         0   104332    1070    0    0    0     0       0          0  
wlp2s0: 78377618   62185    0    0    0     0          0         0 13077822   11885    0    0    0     0       0          0  
docker0:       0       0    0    0    0     0          0         0        0       0    0   33    0     0       0          0  
br-64cfc714b61c:    1204      43    0    0    0     0          0         0    10874      73    0    5    0     0       0          0  
veth161c098:     126       3    0    0    0     0          0         0    18351     160    0    0    0     0       0          0  
vethfc11ad7:   84471     166    0    0    0     0          0         0   208488     327    0    0    0     0       0          0  
vethcd06bc2:     126       3    0    0    0     0          0         0    18247     158    0    0    0     0       0          0  
veth3679dbc:     126       3    0    0    0     0          0         0    18622     156    0    0    0     0       0          0  
vethb161ca9:   21936     281    0    0    0     0          0         0    46397     338    0    0    0     0       0          0  
veth22989f1: 1357428   18365    0    0    0     0          0         0   957086    9683    0    0    0     0       0          0  
veth02d0dde:     126       3    0    0    0     0          0         0    18496     153    0    0    0     0       0          0  
veth5f74910: 8744152   70191    0    0    0     0          0         0  8868415   46669    0    0    0     0       0          0  
veth0fcc258:  276757     590    0    0    0     0          0         0   201501     460    0    0    0     0       0          0  
veth9dc495b: 9745406   55926    0    0    0     0          0         0 10055542   88941    0    0    0     0       0          0  
vethde067e7:   12854     113    0    0    0     0          0         0    30450     218    0    0    0     0       0          0
```


## get uptime

```python
from pathlib import Path
UPTIME_PATH = Path("/proc/uptime")
def format_uptime(seconds:float) -> str:
    minutes = int(seconds) // 60
    remaining_seconds = int(seconds) % 60
    hours = minutes // 60
    real_minute = minutes % 60
    return f"{hours}h {real_minute}m {remaining_seconds}s"

def get_uptime_seconds() -> float:
    raw = UPTIME_PATH.read_text().strip()
    uptime_text, _ = raw.split()
    return float(uptime_text)

uptime = get_uptime_seconds()
print("Uptime: ", format_uptime(uptime))
```

## memory usage
**this is how memory works inside linux** : [[How Memory works in linux]] 

## **rx_bytes and tx_bytes

rx_bytes and tx_bytes are how much network data a linux network interface has received or transmitted

## **CPU STATS**

- user: time spent executing normal userspace code (chrome, python, note etc)
- nice: userspace execution for process running with an adjusted 'nice' property
- system: CPU time executing kernel code
- idle: CPU time where the processor had no runnable work and was idle
- iowait: time accounted while waiting while I/O was outstanding
- irq: time servicing software interrupts

## **SPECIAL TERMINAL COMMANDS**
```
ENTER_ALT_SCREEN = "\x1b[?1049h"

EXIT_ALT_SCREEN = "\x1b[?1049l"

HIDE_CURSOR = "\x1b[?25l"

SHOW_CURSOR = "\x1b[?25h"

HOME = "\x1b[H"

CLEAR = "\x1b[2J"
```

## **DYNAMIC WIDTH**
**f"{text:<10}"
=> it means that left align the text and fill it with the rest of the widht so that total becomes 10 width

## **PACKAGING THE PROGRAM**

- **pyproject.toml** : 
```
[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[project]
name = "linux-hud"
version = "0.1.0"
description = "A lightweight Linux machine and network terminal HUD"
requires-python = ">=3.9"

[tool.setuptools.packages.find]
where = ["src"]
```

[build-system] 
= When somebody wants to build/install this project, use setuptools as the software responsible for turning the source tree into an installable Python distribution.

- **create a development virtual environment** :
  I don't want us installing random development packages into your system Python.
  ```python3 -m venv .venv```

- **main.py**
  inside main.py, there is a main() function and at the bottom of it is if \__name__ == "\__main__"
```
 python -m linux_hud.app
```
=> 
