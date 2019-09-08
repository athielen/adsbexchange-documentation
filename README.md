# ADS-B Exchange Developer Documentation
Documentation for absd exchange for any incoming developers or current ones looking for a refresher

## Ingestion in it's Current State
<p align="center">
  <img src="https://raw.githubusercontent.com/athielen/adsbexchange-documentation/master/content/draw-io-diagram.png">
</p>

1. Feeders (which encompasses Raspberry pi's, IoT devices, or custom solutions) will recieve ADS-B and MLAT broadcasts through a combination of software defined radio and a software or hardware based decoder like [dump1090]

2. Feeders will send these broadcasts over a TCP connection to server located at feed.adsbexchange.com (beast binary).  Proxy server inspects incoming stream for data payload before making a decision on routing or dropping.  SBS basestation, CompressedVRS, AircraftList JSON, and other formats not beast binary are rerouted or dropped.  Reject or reroute any format not Beast binary.

Sample Beast Binary decoder written in Python.  

(https://github.com/junzis/pyModeS/blob/2e3ceed0b00cd4ad22e1d6576fd517fa4ca8e365/decoder.py)

```
### Example broadcast in Raw AVR Hex
*8da48e3a5831d5652ab9d932a61b;
*8daba1a0f80300060049b89cb788;
*8daba1a0990c950b509c042e09f3;
*8da3b4b5582971fa8645b5c71966;
*8daba1a023101331c38d205b04e5;
*5da3b4b53bb2ef;
*8da3b4b5582961fa6c45ca5a3d9a;
*5da3b4b53bb2cf;
*02a1861c8c4dff;

### Example broadcast of MLAT in Basestaion
MSG,3,1,1,C06080,1,2019/09/07,17:12:33.215,2019/09/07,17:12:35.700,,35997,,,36.4139,-121.3726,,,,,,
MSG,3,1,1,C06080,1,2019/09/07,17:12:35.283,2019/09/07,17:12:35.910,,35973,,,36.4169,-121.3752,,,,,,
MSG,3,1,1,AD7BDF,1,2019/09/07,17:12:34.486,2019/09/07,17:12:37.031,,31477,,,35.9813,-120.7624,,,,,,
```

```
### Same broadcasts using dump1090 decoding
*8da48e3a5831d5652ab9d932a61b;
CRC: d8d449
No. of bit errors fixed: 1
RSSI: -5.0 dBFS
Score: 900
Time: 1365342349.17us
DF:17 AA:A48E3A CA:5 ME:5831D5652AB9D9
 Extended Squitter Airborne position (barometric altitude) (11)
  ICAO Address:  A48E3A (Mode S / ADS-B)
  Air/Ground:    airborne
  Baro altitude: 8925 ft
  CPR type:      Airborne
  CPR odd flag:  odd
  CPR latitude:  44.84009 (45717)
  CPR longitude: -93.39819 (47577)
  CPR decoding:  global
  NIC:           8
  Rc:            0.186 km / 0.1 NM
  NIC-B:         0
  NACp:          8
  SIL:           2 (p <= 0.001%, unknown type)

*8daba1a0f80300060049b89cb788;
CRC: 093902
No. of bit errors fixed: 1
RSSI: -1.4 dBFS
Score: 900
Time: 1365402556.33us
DF:17 AA:ABA1A0 CA:5 ME:F80300060049B8
 Extended Squitter Aircraft operational status (airborne) (31/0)
  ICAO Address:  ABA1A0 (Mode S / ADS-B)
  Air/Ground:    airborne
  NIC-A:         0
  NIC-baro:      1
  NACp:          9
  GVA:           2
  SIL:           3 (p <= 0.00001%, per flight hour)
  SDA:           2
  Aircraft Operational Status:
    Version:            2
    Capability classes: ARV TS
    Operational modes:  SAF
    Heading ref dir:    True heading
```

3. The feed proxy balances the raw broadcasts from the feeders over multiple servers running several [feedmerge](https://github.com/adsbxchange/feedmerge) instances to create a single merged stream of data per server.


4. The single merge stream per server are sent to a decoder on each server that will decode the Beast binary to a single stream of data in beast and json.


5. Streams of AircraftJson from these load balanced decoders are then merged into one at VRS server.  (Original non decoder feed-merge would likely be more efficent at this eventually)


6. This stream of aggregated VRS jsons are then sent to [tcp-relay-pub-vrs](https://github.com/adsbxchange/tcp-relay-pub-vrs) which listens for incoming TCP connections and relays single a single merged stream of data to ADSBx data consumers.


## Project goal

GOAL OF PROJECT:

  A feedermerge instance will listen for incoming TCP connections from the proxy server on a local port and proceed to merge and decode those routed connections.  Connection will be Beast binary format. 
   
  The Mode-S Beast binary format is an escaped transparent binary protocol that always contains time and signal level information.  ADSBX feeders send Beast binary. We should preserve this format and thus preserve the timestamp and signal level of the incoming data points from the feeder.

  Sample Beast Binary decoder written in Python (https://github.com/junzis/pyModeS/blob/2e3ceed0b00cd4ad22e1d6576fd517fa4ca8e365/decoder.py)

Goal is to support at least 50 incoming streams per feedmerge thread/instance, or as many as performance will allow while outputting 2 decoded formats.  Ideal target on a 4Ghz CPU single thread would be 100-300 feeds.

### Input format

Beast binary from multiple incoming TCP connections.  Investigate if we can drop 'late' or 'stale' timestamps.

### Output Format 1 (primary format)

A VRS style AircraftJSON for ingest / relay to VRS consumers and other TCP style ingest consumers. Enthusiast oriented.

  Example live data stream can be found at pub-vrs.adsbexchange.com port 32030. Requires ADSBx firewall authorization. TCP stream that sends a burst of data (JSON object) every x seconds to connected consumers.  Target would be every second with max time delay of 3 seconds.

```
{
	"acList": [{
		"Sig": 176,
		"Icao": "A26BC2",
		"Alt": 62379,
		"GAlt": 62500,
		"AltT": 1,
		"Lat": 38.589198,
		"Long": -88.805725,
		"Mlat": false,
		"Spd": 3.6,
		"Trak": 146.3
	}, {
		"Sig": 185,
		"Icao": "4B5CCD"
	}, {
		"Icao": "4B5CD0"
	}, {
		"Sig": 0,
		"Icao": "4B5CCE"
	}]
}
```

### Optional Output Format 2 (for ADSBx internal use)
  
  Our own JSON in format of ADSBx REST API. Listener that serves incoming request for 'current status' of feed merge and list of tracked aircraft.

### Optional Output Format 3 (may prove too difficult to implement)
  
  A merged Beast binary stream.  We must investigate whether it is possible to substantially reduce the message count after merging streams without reducing fidelity of the data stream.  This would allow rebroadcast of a merged beast stream to consumers for use it other decoders.

  This would let us merge all the mergers into one Beast format stream, JSON, and API JSON output.

  It may be more productive to merge above JSON format(s).

  

## Getting started
TODO

### Running Locally using Docker
TODO

# References
[ABDSExchange.com](https://www.adsbexchange.com/)

[Understanding Mode-S and ADS-B data](https://mode-s.org/decode/)

[tcp-relay-pub-vrs](https://github.com/adsbxchange/tcp-relay-pub-vrs)

[feedmerge](https://github.com/adsbxchange/feedmerge)

[readsb - forked dump1090](https://github.com/Mictronics/readsb)

[topic library - feedmerge uses](https://github.com/tv42/topic)

[Virtual Rader Server (VRS)](http://www.virtualradarserver.co.uk/)

[Beast Binary Decoder](https://github.com/junzis/pyModeS/blob/2e3ceed0b00cd4ad22e1d6576fd517fa4ca8e365/decoder.py)

[golang ADSB/BEAST decoder](https://github.com/mtigas/simurgh)
