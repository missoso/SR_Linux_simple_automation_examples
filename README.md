# SR Linux - Simple automation examples

The goal of this repository is to illustrate with very simple examples the multiple ways available to perform automation tasks, and how to interact with both the Nokia SR Linux YANG model and the OpenConfig YANG model.

Three main scenarios:

1 - GNMI for both SR Linux YANG model and the OpenConfig YANG model

2 - JSON-RPC mgmt interface for the SR Linux YANG model

3 - Netconf for the OpenConfig YANG model

The topology itself is a very simple setup of one leaf and two spines (copied from https://github.com/missoso/srl-evpn-l2 but trimmed down to only 3 nodes), the relevant part here is "how to perform simple automation tasks on a per box basis", so the rest of the configuration and protocol state is simply to be ignored.

---

## 🛠️ Lab Setup

The lab is deployed with the [containerlab](https://containerlab.dev) project, where [`srl-simple-automation.clab.yml`](https://github.com/missoso/SR_Linux_simple_automation_examples/blob/main/srl-simple-automation.clab.yml) file declaratively describes the lab topology.

**Deploy the LAB**

```bash
# change into the cloned directory and execute
sudo containerlab deploy --reconfigure
```

```bash
╭────────┬───────────────────────────────┬─────────┬────────────────╮
│  Name  │           Kind/Image          │  State  │ IPv4/6 Address │
├────────┼───────────────────────────────┼─────────┼────────────────┤
│ leaf2  │ nokia_srlinux                 │ running │ 172.80.80.12   │
│        │ ghcr.io/nokia/srlinux:24.10.1 │         │ N/A            │
├────────┼───────────────────────────────┼─────────┼────────────────┤
│ spine1 │ nokia_srlinux                 │ running │ 172.80.80.21   │
│        │ ghcr.io/nokia/srlinux:24.10.1 │         │ N/A            │
├────────┼───────────────────────────────┼─────────┼────────────────┤
│ spine2 │ nokia_srlinux                 │ running │ 172.80.80.22   │
│        │ ghcr.io/nokia/srlinux:24.10.1 │         │ N/A            │
╰────────┴───────────────────────────────┴─────────┴────────────────╯
```


**Access credentials**

```bash
admin/NokiaSrl1!
```

**To remove the lab**
```bash
# change into the cloned directory and execute
sudo containerlab destroy  --cleanup
```

**Enable OpenConfig in the SR Linux device**

We will be using leaf2 only

```bash
--{ running }--[  ]--
A:leaf2# enter candidate

--{ candidate shared default }--[  ]--
A:leaf2# system management openconfig admin-state enable

--{ * candidate shared default }--[  ]--
A:leaf2# commit now
All changes have been committed. Leaving candidate mode.
```

**Enable json-rpc-server (if not using containerlab)**

A node deployed by containerlab will have JSON-RPC enabled, however, when using a phsyical node this needs to be enabled

```bash
--{ running }--[  ]--
A:srl# info /system json-rpc-server
    system {
        json-rpc-server {
            admin-state enable
            network-instance mgmt {
                http {
                    admin-state enable
                }
                https {
                    admin-state enable
                    tls-profile clab-profile
                }
            }
        }
    }
```





# YANG paths

Depending on the model used (SR Linux or OpenConfig) the paths to apply the SET and GET RPC commands will change

**SR Linux / YANG**

To check the SR Linux YANG paths: https://yang.srlinux.dev/ or alternatively use the SR Linux cli, example for interface:

```bash
--{ running }--[  ]--
A:leaf2# info  interface ethernet-1/1  | as json
{
  "interface": [
    {
      "name": "ethernet-1/1",
      "admin-state": "enable",
      "subinterface": [
        {
          "index": 0,
          "type": "bridged",
          "admin-state": "enable"
        }
      ]
    }
  ]
}
```

**OpenConfig / YANG paths**

Same logic for OpenConfig, https://www.openconfig.net/ or in the SR Linux CLI:

```bash
--{ + running }--[  ]--
A:leaf2# enter oc

--{ + oc running }--[  ]--
A:leaf2# info /interfaces interface ethernet-1/1 | as json
{
  "interfaces": {
    "interface": [
      {
        "name": "ethernet-1/1",
        "config": {
          "name": "ethernet-1/1",
          "type": "ethernetCsmacd",
          "enabled": true
        },
        "subinterfaces": {
          "subinterface": [
            {
              "index": 0,
              "config": {
                "index": 0,
                "enabled": true
              }
            }
          ]
        }
      }
    ]
  }
}
```


# GNMI / gNMIc

Using the gNMIc tool https://gnmic.openconfig.net/install/ to run GNMI commands from the local host

SR Linux uses port 57400, plus using the --skip-verify flag to skip the TLS configuration

**gNMIc: CLI (SRL YANG)**

SRl YANG path: /interface[name=ethernet-1/1]/description

SET command

```bash
$ gnmic -a 172.80.80.12:57400 -u admin -p 'NokiaSrl1!' --skip-verify set   --update-path /interface[name=ethernet-1/1]/description   --update-value “gnmic-srl-yang”
{
  "source": "172.80.80.12:57400",
  "timestamp": 1770632495402244221,
  "time": "2026-02-09T10:21:35.402244221Z",
  "results": [
    {
      "operation": "UPDATE",
      "path": "interface[name=ethernet-1/1]/description"
    }
  ]
}
```


GET command

```bash
gnmic -a 172.80.80.12:57400 -u admin -p 'NokiaSrl1!' --skip-verify get --path /interface[name=ethernet-1/1]/description --encoding JSON_IETF
[
  {
    "source": "172.80.80.12:57400",
    "timestamp": 1770632560324575761,
    "time": "2026-02-09T10:22:40.324575761Z",
    "updates": [
      {
        "Path": "srl_nokia-interfaces:interface[name=ethernet-1/1]/description",
        "values": {
          "srl_nokia-interfaces:interface/description": "“gnmic-srl-yang”"
        }
      }
    ]
  }
]
```

**gNMIc: JSON input file (SRL YANG)**

SRl YANG path: /interface[name=ethernet-1/1]/description

**One update**
input file = gnmi-srl-via-json.json

```bash
cat gnmi-srl-via-json.json
{
  "updates": [
      {
          "path": "/interface[name=ethernet-1/1]",
          "value": {
              "description": “gnmi_srl_via_json_file”
           },
           "encoding": "json_ietf"
      }
  ]
 }
```

Apply it (the flag to specify the file may change between gNMIc releases, in v0.43 it is --request-file)

```bash
gnmic set \
  -a 172.80.80.12:57400 \
  -u admin \
  -p 'NokiaSrl1!' \
  --skip-verify \
  --encoding json_ietf \
  --request-file gnmi-srl-via-json.json
{
  "source": "172.80.80.12:57400",
  "timestamp": 1770632890879410353,
  "time": "2026-02-09T10:28:10.879410353Z",
  "results": [
    {
      "operation": "UPDATE",
      "path": "interface[name=ethernet-1/1]"
    }
  ]
}
```
**Multiple updates on the same path**

Input file = gnmi-srl-via-json-two-updates-one-path.json

```bash
cat gnmi-srl-via-json-two-updates-one-path.json
{
  "updates": [
      {
          "path": "/interface[name=ethernet-1/1]",
          "value": {
              "description": "disable",
              "admin-state": "disable"
           },
           "encoding": "json_ietf"
      },
     
  ]
 }
```

```bash
gnmic set   -a 172.80.80.12:57400   -u admin   -p 'NokiaSrl1!'   --skip-verify   --encoding json_ietf   --request-file gnmi-srl-via-json-two-updates-one-path.json
{
  "source": "172.80.80.12:57400",
  "timestamp": 1770633802056124100,
  "time": "2026-02-09T10:43:22.0561241Z",
  "results": [
    {
      "operation": "UPDATE",
      "path": "interface[name=ethernet-1/1]"
    }
  ]
}
```


**Multiple updates on different paths**

Input file = gnmi-srl-via-json-two-updates-two-paths.json

The *trick* is that the update list needs to be split into two different objects, otherwise only the last *path* defined in the file is used

```bash
cat gnmi-srl-via-json-two-updates-two-paths.json
{
  "updates": [
      {
          "path": "/interface[name=ethernet-1/1]",
          "value": {
              "description": "123"
           },
           "encoding": "json_ietf"
      },
      {
          "path": "/interface[name=ethernet-1/2]",
          "value": {
              "description": "321"
           },
           "encoding": "json_ietf"
      }
  ]
 }
```

```bash
gnmic set   -a 172.80.80.12:57400   -u admin   -p 'NokiaSrl1!'   --skip-verify   --encoding json_ietf   --request-file gnmi-srl-via-json-two-updates-two-paths.json
{
  "source": "172.80.80.12:57400",
  "timestamp": 1770634152375439321,
  "time": "2026-02-09T10:49:12.375439321Z",
  "results": [
    {
      "operation": "UPDATE",
      "path": "interface[name=ethernet-1/1]"
    },
    {
      "operation": "UPDATE",
      "path": "interface[name=ethernet-1/2]"
    }
  ]
}
```



**gNMIc CLI - OpenConfig YANG**

OpenConfig YANG path: openconfig:/interfaces/interface[name=ethernet-1/1]/config/description

SET command

```bash
gnmic -a 172.80.80.12:57400 -u admin -p 'NokiaSrl1!' --skip-verify set   --update-path openconfig:/interfaces/interface[name=ethernet-1/1]/config/description   --update-value “OC”
{
  "source": "172.80.80.12:57400",
  "timestamp": 1770634610918624927,
  "time": "2026-02-09T10:56:50.918624927Z",
  "results": [
    {
      "operation": "UPDATE",
      "path": "openconfig:/interfaces/interface[name=ethernet-1/1]/config/description"
    }
  ]
}
```


GET command

```bash
gnmic -a 172.80.80.12:57400 -u admin -p 'NokiaSrl1!' --skip-verify get --path openconfig:/interfaces/interface[name=ethernet-1/1]/config/description --encoding JSON_IETF
[
  {
    "source": "172.80.80.12:57400",
    "timestamp": 1770634643308081032,
    "time": "2026-02-09T10:57:23.308081032Z",
    "updates": [
      {
        "Path": "openconfig:/interfaces/interface[name=ethernet-1/1]/config/description",
        "values": {
          "interfaces/interface/config/description": "“OC”"
        }
      }
    ]
  }
]
```

**gNMIc input file - OpenConfig YANG**

Input file: gnmi-oc.json

```bash
cat gnmi-oc.json 
{
  "updates": [
      {
          "path": "openconfig:/interfaces/interface[name=ethernet-1/1]/config",
          "value": {
              "description": "oc 123"
           },
           "encoding": "json_ietf"
      },
      {
          "path": "openconfig:/interfaces/interface[name=ethernet-1/2]/config",
          "value": {
              "description": "oc 321"
           },
           "encoding": "json_ietf"
      }
  ]
}

```

```bash
gnmic -a 172.80.80.12:57400 -u admin -p 'NokiaSrl1!' --skip-verify set --request-file gnmi-oc.json
{
  "source": "172.80.80.12:57400",
  "timestamp": 1770634998502562583,
  "time": "2026-02-09T11:03:18.502562583Z",
  "results": [
    {
      "operation": "UPDATE",
      "path": "openconfig:/interfaces/interface[name=ethernet-1/1]/config"
    },
    {
      "operation": "UPDATE",
      "path": "openconfig:/interfaces/interface[name=ethernet-1/2]/config"
    }
  ]
}
```

# JSON-RPC

Fully covered in https://learn.srlinux.dev/tutorials/programmability/json-rpc/basics/ so just a very quick examples in this repository

Model: SRL YANG

SET interface description

```bash
curl -s 'http://admin:NokiaSrl1!@leaf2/jsonrpc' -d @- <<EOF | jq
{
    "jsonrpc": "2.0",
    "id": 0,
    "method": "set",
    "params": {
        "commands": [
            {
                "action": "update",
                "path": "/interface[name=ethernet-1/1]/description:set-via-json-rpc"
            }
        ]
    }
}
EOF
```

```bash
{
  "result": [
    {}
  ],
  "id": 0,
  "jsonrpc": "2.0"
}
```

GET interface description (and also system version to illustrate how to run multiple commands)

```bash
curl -s 'http://admin:NokiaSrl1!@leaf2/jsonrpc' -d @- <<EOF | jq
{
    "jsonrpc": "2.0",
    "id": 0,
    "method": "get",
    "params": {
        "commands": [
            {
                "path": "/system/information/version",
                "datastore": "state"
            },
            {
                "path": "/interface[name=ethernet-1/1]/description",
                "datastore": "state"
            }
        ]
    }
}
EOF
```


```bash
{
  "result": [
    "v24.10.1-492-gf8858c5836",
    "set-via-json-rpc"
  ],
  "id": 0,
  "jsonrpc": "2.0"
}
```

# Netconf

Using netconf-console2 https://pypi.org/project/netconf-console2/

Model: OpenConfig YANG

SR Linux supports applying changes to both the candidate and the running configuration, in this example we will apply to the candidate and then issue the commit command. 

XML target parameter:
```bash
[...]
  <target>
    <candidate/>
  </target>
[...]
```

Complete file with the configuration changes to be applied

```bash
cat if-desc-oc-netconf.xml
<edit-config xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <target>
    <candidate/>
  </target>
  <config>
    <interfaces xmlns="http://openconfig.net/yang/interfaces">
      <interface>
        <name>ethernet-1/1</name>
        <config>
          <name>ethernet-1/1</name>
          <description>netconf-1-1</description>
        </config>
      </interface>
      <interface>
        <name>ethernet-1/2</name>
        <config>
          <name>ethernet-1/2</name>
          <description>netconf-1-2</description>
        </config>
      </interface>
    </interfaces>
  </config>
</edit-config>
```

File to issue the commit

```bash
cat commit-oc-netconf.xml 
<commit xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"/>
```

Apply the changes
```bash
netconf-console2 \
  --host leaf2 \
  --port 830 \
  --user admin \
  --password NokiaSrl1! \
  --rpc if-desc-oc-netconf.xml
```

```bash
<?xml version='1.0' encoding='UTF-8'?>
<rpc-reply xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:cb72fa5a-94c1-4ce6-b140-e198cb1b2b61">
    <ok/>
</rpc-reply>
```


Issue the commit
```bash
netconf-console2 \
  --host leaf2 \
  --port 830 \
  --user admin \
  --password NokiaSrl1! \
  --rpc commit-oc-netconf.xml
```

```bash
<?xml version='1.0' encoding='UTF-8'?>
<rpc-reply xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:81f5034f-90d4-46e1-9011-45625ebea492">
    <ok/>
</rpc-reply>
```

