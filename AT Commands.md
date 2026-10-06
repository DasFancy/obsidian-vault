Turn on the modem: `AT+CFUN=1`
Turn off the modem: `AT+CFUN=4`

Checking the SIM connection: `AT+CPIN?`
	Output: `+CEREG:` lines are **unsolicited registration updates** as the modem moves through attach/reselection states. For LTE/EPS registration, the key status values are: `2` = searching, `1` = registered on home network, `4` = unknown, and `5` = registered while roaming.
	In general: `+CEREG: <n>, <stat>, <tac>, <ci>, <AcT>, <cause_type>, <reject_cause>, <Active-Time>, <Periodic-TAU>`


------------
## Setting up an NTN connection
Flight mode, disables LTE for configuration
```
 AT+CFUN=4
```
 Which radio system to enable (GNSS on)?
```
 AT%XSYSTEMMODE=0,0,1,0,0
```
Activate GNSS mode
```
 AT+CFUN=31
```
Start GNSS (GPS) tracking. `AT#XGNSS=<start/stop>,<mode>,<interval>,<timeout>`
```
 AT#XGPS=1,0,0,0
```
take *30 seconds to several minutes* (Give it a try)
	out: 
		# XGPS: 1,1
		# XGPS: 51.019348,13.746376,228.210083,37.534512,1.446435,339.407349,"2026-09-21 08:31:26"
		# XGPS: 1,4
	#XGPS: 51.019452,13.746134,230.853577,27.797586,0.319848,0.000000,"2026-09-28 09:39:19"
Shut down GPS track
```
AT#XGPS=0
```
Turn off GPS
```
AT+CFUN=30
```
 Turn on NTN
```
AT%XSYSTEMMODE=0,0,0,0,1
```
```
AT%XBANDLOCK=2,,"23,255,256"
```
APN+GNSS location
```
AT+CGDCONT=0,"IP","internet.m2mportal.de"
```
```
AT%LOCATION=2,"51.019391","13.746151","193.633667",15,0
```
Allow notifications:
```
AT+CEREG=5
```
```
AT+CNEC=24
```
```
AT+CSCON=3
```
```
AT%MDMEV=2
```
```
AT%CESQ=1
```

 Then start NTN registration
```
AT+CFUN=1
```

As output, check for: +CEREG: 5,1 or  +CEREG: 5,5 (if roaming)
```
AT+CEREG?
```
and for: +CGATT: 1 with
```
AT+CGATT? 
```


------------
## Sending packets and echo

Once your nRF9151 is registered on NTN, the test can be just:

```
AT+CEREG?
```

Confirm that you are registered, then create a UDP socket:

```
AT#XSOCKET=1,2,0
```

You should get something similar to:

```
#XSOCKET: 0,2,17
OK
```

Then send exactly. For example:

```
SUP?
```


So, assuming your UDP echo server is at `203.0.113.10` port `5000`:

```
AT#XSENDTO="147.185.221.214",4707,"1,1"
```
147.185.221.216:35564
147.185.221.213:4707
147.185.221.214:4707
Expected:

```
#XSENDTO: 13
OK
```

Then receive the echo:

```
AT#XRECVFROM=0
```

Finally:

```
AT#XSOCKET=0
```


+CEREG: 5,"3AA9","001A2B49",14,,,"00000000","00000110"
+CEREG: 5,"0FBA","0027BBFA",14,,,"00000000","00000110"

```
AT#XRESET
```

CCID: **89882280000130797307** $\rightarrow$ nrf: 90
CCID: **89882280000173655651** $\rightarrow$ nrf: 18
CCID: **89882280000173655479** $\rightarrow$ nrf: 41

