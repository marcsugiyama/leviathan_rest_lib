# leviathan_rest_lib
`leviathan_rest_lib` provides a RESTish interface to the leviathan library 
functions.

# API - REST Resources
URI | Method | Body | Description
--- | ------ | ---- | -----------
/cpool | POST | JSON file | upload CPool JSON file
/cin | POST | JSON file | upload JSON file
/cin/prepare | POST | list of Cin Ids | prepare CINs
/cin/destroy | POST | list of Cin Ids | undo CINs
/cen | POST | JSON file | upload JSON file
/cen/CenId | PUT | none | create CEN
/cen/CenId | GET | none | get CEN structure
/cen/CenId | DELETE | none | remove CEN
/cen/prepare | POST | list of Cen Ids | prepare CENs
/cen/destroy | POST | list of Cen Ids | undo CENs
/host/HostId/ContainerId/CenId | PUT | none | add Container to Cen
/host/HostId/ContainerId/CenId | DELETE | none | remove Container from Cen
/host/HostId/ContainerId | GET | none | get Container structure
/host/HostId/ContainerId | PUT | none | add Container to host
/host/HostId/ContainerId | DELETE | none | remove Container from host

## CEN Structure
```
{
    "cenID": CenId,
    "contIDs": [ContId1, ContId2, ...],
    "wiring_type": WiringType
}
```
Where `contIDs` is the list of containers in the CEN.  `WiringType` is
`"bridge"`, `"wire"`, or `null`.

## Example CEN file
```
{"cenList":
 [{
     "cenID" : "cen1",
     "containerIDs" : [ "c1","c2","c13","c14"]
  },
  {
      "cenID":"cen2",
      "containerIDs":["c4","c5","c6","c7"]
  },
  {
      "cenID":"cen3",
      "containerIDs":["c15","c16","c9","c10","c11"]
  },
  {
      "cenID":"cen4",
      "containerIDs":["c11","c12"]
  },
  {
      "cenID":"cen5",
      "containerIDs":["c2","c3"]
  }]
}
```

## Examples

### Prepare CENs
Prepares a list of CENS on a host

This creates all artifacts like network namespaces, bridges, interfaces, etc 
```
curl -d '["cen1","cen2","cen3","cen4","cen5"]' http://<location>:8080/cen/prepare
```
(HTTP POST of a JSON list of the CENs to prepare).


### Destroy CENs
Destroys a list of CENS on a host

This destroy all artifacts like network namespaces, bridges, interfaces, etc 
```
curl -d '["cen1","cen2","cen3","cen4","cen5"]' http://<location>:8080/cen/destroy
```
(HTTP POST of a JSON list of the CENs to destroy).


### Add Container to Cen
```
curl -v -X PUT -H "content-type: application/json" http://localhost:8080/host/host1/container1/cen1
```

### Remove Container from Cen
```
curl -v -X DELETE -H "content-type: application/json" http://localhost:8080/host/host1/container1/cen1
```

### Upload CEN file
Uploads and compiles wiring for CENs described in a CEN file.

```
curl -d @/tmp/cen.json http://<location>:8080/cen
```
(HTTP POST of the CEN file).


## CPools

### CPool Structure

```
{
 "cpoolID": <CPoolId>,
 "type": <ContainerType>,
 "start_with": <Number of ContainerType to run>,
 "cins":[{"cinID":<CinID>,"default_route":<optional "true"|"false">},...]
 }
```

### Example CPool JSON File
```
{"cpoolList":
 [
     {"cpoolID":"pool1",
      "type":"ubuntu:14.04",
      "start_with": 3,
      "cins":[{"cinID":"cen1","default_route":"true"},
	      {"cinID":"cen2"}]},
     {"cpoolID":"pool2",
      "type":"ubuntu:14.04",
      "start_with": 3,
      "cins":[{"cinID":"cen2"}]}
 ]
}
```

# API - websocket

URI | Purpose
--- | -------
/leviathan/monitor | monitor containers

## /leviathan/monitor protocol

The `/leviathan/monitor` websocket supports a bidirectional protocol.
The commands and responses are encoded in JSON. This is an ansynchronous
channel in that the client and server may send messages at anytime.

### Create monitor
The client sends a request to create a new monitor or modify an existing monitor:
```
{
    "type":"start",
    "sequence":"Sequence",
    "monitor":"CIN",
    "monitorID":"MonitorID"
    "parameters":{
        "cinID":"Cin1",
        "container_tag":"Tag"
    }
}
```
Where `Sequence` is a token used by the client to match responses to requests.
`container_tag` is optional. `MonitorID` is optional and is the monitor identifier returned when the client creates a monitor. If the client includes the `MonitorID` in the request, the server replaces the monitor with that identifier with this monitor.

The server responds with with the monitor identifier and the initial state of the CIN:
```
{
    "type":"response",
    "sequence":"Sequence",
    "response":{
        "monitorID":"MonitorID"
        "state":[
            {
                "containerID":"ContainerID",
                "ipaddress":"IpAddress",
                "tag":"Tag"
            }, ...
        ]
    }
```
Where `Sequence` is the taken from the request, `MonitorID` uniquely identifies the created monitor. `state` is the current state of the monitored CIN, listing the container identifiers, their ip addresses and tags.

If the server cannot process the request, it returns an error response:
```
{
    "type":"error",
    "sequence":"Sequence",
    "response":{
        "message":"Message"
    }
```
Where `Message` is a message describing the reason for the error.

After creating a monitor, the server sends messages when the containers in CIN change. If `Tag` is provided in the request, then only containers with that tag are reported,

The messages are:
```
{
    "type":"event",
    "monitorID":"MonitorID",
    "event":"create",
    "message":[
        {
            "containerID":"ContainerID",
            "ipaddress":"IpAddress",
            "tag":"Tag"
        }, ...
    ]
}
```
Where `MonitorID` is the monitor identifier returned when the client created the monitor. `ContainerID` is the container identifier. `IpAddress` is the IP Address assigned to the container. `Tag` is the tag associated with the container.

Limitations of the current implementation:
1. one monitor per websocket connection
2. monitor only reports that containers are created
