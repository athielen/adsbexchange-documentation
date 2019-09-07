# ADS-B Exchange Developer Documentation
Documentation for absd exchange for any incoming developers or current ones looking for a refresher

## Ingestion in it's Current State
<p align="center">
  <img src="https://raw.githubusercontent.com/athielen/adsbexchange-documentation/master/content/draw-io-diagram.png">
</p>

1. Feeders (which encompasses Raspberry pi's, IoT devices, or custom solutions) will recieve ADS-B and MLAT broadcasts through a combination of reciever and a proram like [dump1090]()

2. Feeders will send these boradcasts over a TCP connection to adsb-exchanges proxy (feed.adsbexchange.com:30005). Because they are sent to port 30005, this is usually a sign that the braodcasts will be sent over in [BEAST Mode-s format](https://mode-s.org/decode/)

```
### Example broadcast in BEAST Mode-s
*8da48e3a5831d5652ab9d932a61b;
*8daba1a0f80300060049b89cb788;
*8daba1a0990c950b509c042e09f3;
*8da3b4b5582971fa8645b5c71966;
*8daba1a023101331c38d205b04e5;
*5da3b4b53bb2ef;
*8da3b4b5582961fa6c45ca5a3d9a;
*5da3b4b53bb2cf;
*02a1861c8c4dff;

### Example broadcast of MLAT
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

3. The proxy will load balance the raw broadcasts from the feeders over several severs running several [feedmerge](https://github.com/adsbxchange/feedmerge) instances to create a single merged stream of data per server.

4. The single merge steam per server will be sent to a decoder on each server that will decode the hex to [AircraftJson format]()
```
# Example JSON response
{
  "acList": [
    {
      "Alt": 65079,
      "AltT": 1,
      "GAlt": 65200,
      "Icao": "A26BC2",
      "InHg": 30.041338,
      "Lat": 40.803589,
      "Long": -91.926636,
      "Mlat": false,
      "Sig": 68,
      "Spd": 6.4,
      "Trak": 141.3
    },
    {
      "Icao": "43BF96"
    },
    {
      "Icao": "3C1FBB"
    },
    {
      "Alt": 200,
      "AltT": 0,
      "Call": "CHX84",
      "CallSus": false,
      "GAlt": 200,
      "Gnd": false,
      "Icao": "3DDC73",
      "Sig": 188,
      "Spd": 3.2,
      "SpdTyp": 0,
      "Sqk": "",
      "Tisb": false,
      "Trak": 71.6,
      "TrkH": false,
      "Trt": 2,
      "Vsi": 640,
      "VsiT": 1
    }
  ]
}
```

5. Streams of AircraftJson from these broadcasts are then merged into one at another VRS server.

6. This stream of aggregated jsons are then sent to [tcp-relay-pub-vrs](https://github.com/adsbxchange/tcp-relay-pub-vrs) which goes from one to many tcp connections connecting to any consumers of ADSB Exchanges data.

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
