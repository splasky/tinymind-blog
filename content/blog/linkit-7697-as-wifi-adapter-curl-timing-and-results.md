---
title: LinkIt 7697 as wifi adapter. Curl timing and results
date: 2026-07-17T08:20:18.463Z
---

# LinkIt 7697 `linkit0` iPerf3 and curl timing results

Test time: 2026-07-17 10:59 CST (UTC+08:00)

## Test setup

- Interface: `linkit0`
- Client IPv4: `192.168.31.236/24`
- Gateway: `192.168.31.1`
- Client binding: both `--bind-dev linkit0` and `-B 192.168.31.236`
- Candidate source: [Public iPerf3 Server List](https://www.iperf3serverlist.net/)
- Test Platform: LinkIt 7697(SeedStudio v1.0) devboard, Ubuntu 24.04 LTS

The client was explicitly bound to `linkit0`; these probes did not use the
laptop's `wlp1s0` source address.

## Successful iPerf3 throughput reference: Tokyo

Taiwan public iPerf3 endpoints were unavailable during this session, so the
following completed throughput numbers are from the working Tokyo endpoint
`89.187.160.1:5201`. These runs were also bound to `linkit0`.

UDP upload, host to WiFi:

| Requested rate | Receiver bitrate | Loss | Jitter | Notes |
| --- | ---: | ---: | ---: | --- |
| 20 Kbit/s | 19.9 Kbit/s | 0/98 | 4.188 ms | Stable |
| 50 Kbit/s | 49.4 Kbit/s | 0/244 | 0.163 ms | Stable |
| 100 Kbit/s | 99.1 Kbit/s | 0/489 | 0.221 ms | Stable |
| 250 Kbit/s, `-l512` | 212 Kbit/s | 0/611 | 11.794 ms | Completed, drained for 11.79 s |
| 500 Kbit/s | 253 Kbit/s | 0/1220 | 5.835 ms | Completed, drained for 19.72 s |

UDP reverse, WiFi to host:

| Requested rate | Receiver bitrate | Loss | Jitter | Notes |
| --- | ---: | ---: | ---: | --- |
| 20 Kbit/s | 20.3 Kbit/s | 0/99 | 0.625 ms | Stable |
| 50 Kbit/s | 9.26 Kbit/s | 1/180 | N/A | Stalled before completing normally |
| 100 Kbit/s | N/A | N/A | N/A | No effective test result |

TCP:

| Direction | Command shape | Remote receiver result | Notes |
| --- | --- | ---: | --- |
| Upload | default, 5 s | 88.1 Kbit/s | Client sent a burst of 419 Kbits, 1 retransmit, drained for 8.82 s |
| Upload | `-t 15 -w4K -M256` | 48.8 Kbit/s | Client sender accounting was 0 because ACKs arrived late |
| Reverse | 5 to 10 s | 0 app goodput | Remote sent about 39 to 43 KiB, but the client did not receive usable TCP payload in-window |

Interpretation: the current LinkIt 7697 port can move useful traffic, but the
stop-and-wait UART framing and the 4-frame firmware queue dominate throughput.
The sustainable direction today is UDP upload at roughly 200 to 250 Kbit/s.
Reverse UDP is reliable around 20 Kbit/s. TCP works for small transfers, but is
very sensitive to ACK delay and reverse-path buffering.


## Candidate 1: SG.GS Taipei

- Host: `tpe.speedtest.sggs.network`
- Resolved IPv4: `124.6.34.238`
- Listed location: Taipei, Taiwan

Latency through `linkit0`:

```text
5 packets transmitted, 5 received, 0% packet loss
rtt min/avg/max/mdev = 75.590/76.541/78.613/1.084 ms
```

## Candidate 2: Psychz Taipei Looking Glass

- IPv4: `103.59.110.170`
- Published location: Taipei, Taiwan

Latency through `linkit0`:

```text
3 packets transmitted, 3 received, 0% packet loss
rtt min/avg/max/mdev = 20.877/21.535/21.896/0.466 ms
```

## curl HTTP/HTTPS timing via linkit0

These tests used `curl --interface linkit0 -4` so the traffic was bound to the
LinkIt 7697 adapter. 

Clean HTTP request without following redirects:

```sh
curl --interface linkit0 -4 --max-time 60 -sS -o /dev/null \
  -w '...' http://splasky.github.io
```

Result:

```text
url=http://splasky.github.io/
http_code=301
remote_ip=185.199.108.153
num_redirects=0
time_namelookup=0.000867
time_connect=0.090997
time_appconnect=0.000000
time_starttransfer=0.205249
time_total=0.211548
size_download=162
speed_download=765
exitcode=0
```

HTTP request with `-L` enabled:

```text
url=https://splasky.github.io/
http_code=301
remote_ip=185.199.110.153
num_redirects=1
time_namelookup=0.002159
time_connect=0.099434
time_appconnect=0.000000
time_starttransfer=0.451511
time_redirect=0.457958
time_total=120.296259
exitcode=28
errormsg=Connection timed out after 119839 milliseconds
```

This confirms that HTTP connected and received the redirect quickly. The
subsequent HTTPS leg to GitHub Pages did not complete before curl's 120 second
limit in this run.

Direct HTTPS to `https://splasky.github.io`:

```text
url=https://splasky.github.io/
http_code=000
remote_ip=
num_redirects=0
time_namelookup=0.006276
time_connect=0.000000
time_appconnect=0.000000
time_starttransfer=0.000000
time_total=180.002933
exitcode=28
errormsg=Connection timed out after 180002 milliseconds
```

Direct HTTPS to the page referenced by the HTML meta refresh. The latest retest allowed curl to wait for 300 seconds:

```text
url=https://tinymind-alpha.vercel.app/splasky/blog
http_code=000
remote_ip=
num_redirects=0
time_namelookup=0.018073
time_connect=0.000000
time_appconnect=0.000000
time_starttransfer=0.000000
time_total=300.001973
size_download=0
speed_download=0
exitcode=28
errormsg=Connection timed out after 300001 milliseconds
```

Successful HTTPS baseline to a small Cloudflare-hosted endpoint:

```text
url=https://example.com/
http_code=200
remote_ip=172.66.147.243
num_redirects=0
time_namelookup=0.181365
time_connect=0.382486
time_appconnect=5.511462
time_starttransfer=5.740928
time_total=5.757614
size_download=559
speed_download=97
exitcode=0
```

`curl -L` follows HTTP `3xx` redirects with a `Location` header. The successful
HTML body shown earlier from `https://splasky.github.io` contains a browser
`meta refresh`, not an HTTP redirect, so curl does not follow that second hop
automatically.

## Retest after daemon default MTU 576

Test time: 2026-07-17 after rebuilding `linkit7697-net` with
`LINKIT_TAP_MTU 576`.

Runtime state:

```text
linkit0: mtu 576
IPv4: 192.168.31.236/24
Route to 1.1.1.1: via 192.168.31.1 dev linkit0 src 192.168.31.236
```

The rebuilt daemon created `linkit0` with MTU 576 automatically, so the MTU
workaround is now applied by the host daemon instead of manual `ip link set`.

### HTTPS retest

Single-stream Google HTTPS after daemon restart:

```text
URL: https://www.google.com
HTTP: 200
remote_ip: 142.251.157.119
time_namelookup: 0.020842 s
time_connect: 0.041587 s
time_appconnect: 0.285460 s
time_starttransfer: 0.427610 s
time_total: 2.514560 s
```

Sequential Vercel HTTPS retest:

```text
URL: https://tinymind-alpha.vercel.app/splasky/blog
success: 3/3
failure: 0/3
attempt 1: connect 0.024645 s, TLS 0.302415 s, first byte 0.743265 s, total 2.825054 s
attempt 2: connect 0.022375 s, TLS 0.305078 s, first byte 0.627691 s, total 2.161347 s
attempt 3: connect 0.022306 s, TLS 0.352918 s, first byte 0.691815 s, total 2.240396 s
```

A parallel Google+Vercel HTTPS stress run still failed 6/6 with 20 second
connection-phase timeouts. This means MTU 576 fixes single-stream HTTPS, but
concurrent HTTPS streams can still overload the current WiFi-to-host RX path.

### Taiwan endpoint availability

The SG.GS Taipei iPerf3 endpoint remained unavailable:

```text
iperf3 -c tpe.speedtest.sggs.network -p 5201
result: Connection refused
```

This matches the previous test and is still an endpoint availability issue, not
a `linkit0` connectivity issue.

### Tokyo iPerf3 retest

Server: `89.187.160.1:5201`

UDP upload, host to WiFi:

| Requested rate | Receiver bitrate | Loss | Jitter | Comparison |
| --- | ---: | ---: | ---: | --- |
| 100 Kbit/s | 98.1 Kbit/s | 0/120 | 0.328 ms | Same as prior 99.1 Kbit/s |
| 250 Kbit/s, `-l512` | 243 Kbit/s | 0/306 | 2.702 ms | Better than prior 212 Kbit/s and much shorter drain |
| 500 Kbit/s, `-l512` | 250 Kbit/s | 0/611 | 6.340 ms | Similar ceiling to prior 253 Kbit/s, still drains to 10.02 s |

UDP reverse, WiFi to host:

| Requested rate | Receiver bitrate | Loss | Jitter | Comparison |
| --- | ---: | ---: | ---: | --- |
| 20 Kbit/s | 21.0 Kbit/s | 0/25 | 1.032 ms | Same as prior reliable 20 Kbit/s class |
| 50 Kbit/s | 51.1 Kbit/s | 0/61 | 1.399 ms | Major improvement over prior 9.26 Kbit/s stalled run |

TCP:

| Direction | Result | Comparison |
| --- | --- | --- |
| Upload, 5 s | remote receiver 55.7 Kbit/s, 1 retransmit, drained to 8.73 s | Lower than prior 88.1 Kbit/s, still affected by ACK/flush timing |
| Reverse, 5 s | client receiver 0 bytes, remote sender 130 KBytes at 199 Kbit/s, 146 retransmits | Still not usable for reverse TCP app data |

### Retest conclusion

The MTU 576 host-daemon default materially changes the result:

- Single-stream HTTPS now works reliably enough for Google and the Vercel blog.
- UDP upload at 250 Kbit/s improved from 212 Kbit/s to 243 Kbit/s.
- Reverse UDP at 50 Kbit/s improved from a stalled 9.26 Kbit/s run to a clean
  51.1 Kbit/s with 0% loss.
- The practical UDP upload ceiling is still about 250 Kbit/s.
- Reverse TCP is still broken for useful application payload.
- Concurrent HTTPS still overloads the current RX path, so MTU 576 is a
  workaround, not a complete transport fix.

Thanks for [pico-usb-wifi](https://gitlab.com/baiyibai/pico-usb-wifi) project.