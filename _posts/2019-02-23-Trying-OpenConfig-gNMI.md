---
layout: post
comments: true
title: "Trying Openconfig and gNMI interface"
categories: blog
descriptions: Trying Openconfig and gNMI interface using some python libraries
tags: 
  - openconfig
  - gnmi
  - python
  - arista
  - veos
date: 2019-02-23T10:39:55-04:01
---



## Background

More device vendors are supporting Openconfig and gNMI interface, but for some reason, it was not easy for me to find a working example of using it. 
Arista published their own library [https://github.com/aristanetworks/goarista](https://github.com/aristanetworks/goarista), which seems to be working fine, at least on gNMI part, but it is not easy to introduce a new language into existing monitoring/orchestration system just to enable a new "feature or interface". 


My preference is python based library, and after a while, i found 2 ready to use library and example that compliment each other. 

* The first one is gnxi from Google
    * [https://github.com/google/gnxi/tree/master/gnmi_cli_py](https://github.com/google/gnxi/tree/master/gnmi_cli_py)
    * This one support set and get operation, but i don't see any example of doing event subscription, at least from the python example. 

* The second one is pygnmi from Nokia
    * [https://github.com/nokia/pygnmi](https://github.com/nokia/pygnmi)
    * this one only provide example of doing event subscription. 


So, let's try both of them


## Install

* clone both repo

```
git clone https://github.com/nokia/pygnmi.git
git clone https://github.com/google/gnxi.git
```

* create and activate virtual environment

```
virtualenv venv --no-site-packages --no-download
. venv/bin/activate
```

* start with gnxi, install the dependencies (check their Readme for updated instruction)

```
cd gnxi/gnmi_cli_py/
pip install -r requirements.txt
```

* back to main folder

```
cd ../../
```

* now install dependencies for pygnmi (again, check their README for updated instruction)

```
cd pygnmi/
curl -O https://raw.githubusercontent.com/openconfig/gnmi/c5b444cd3ab8af669d0b8934f47a41ed6a985cdc/proto/gnmi/gnmi_pb2.py
```

    * Note: i don't know why they need special version of gnmi_pb2 module, but if i compared with the gnmi_pb2 provided by gnxi, there are some differences in term of subscription function. For now, let's stick with the instruction.




## Let's try on Arista vEOS


For this exercise, we will use Arista vEOS. Since this vEOS has tested gNMI interface that work with goarista library, then we will know if anything does not work, that means there is problem on the script or library side. 


* Enable gnmi on Arista

```
management api gnmi
   transport grpc GRPC
      port 3333
      vrf MGMT
!
```

* Try to get BGP config in openconfig format via gNMI using py_gnmicli.py from gnxi. We will use -n to disable TLS.

```
python gnxi/gnmi_cli_py/py_gnmicli.py -n -m get -t 192.168.97.11 -x / -user admin -pass admin -p 3333

<snip>
{
  "openconfig-network-instance:neighbor": [
    {
      "neighbor-address": "192.100.132.0",
      "route-reflector": {
        "state": {},
        "config": {}
      },

<snip>

      "apply-policy": {
        "state": {
          "default-import-policy": "REJECT_ROUTE",
          "default-export-policy": "REJECT_ROUTE"
        },
        "config": {
          "default-import-policy": "REJECT_ROUTE",
          "default-export-policy": "REJECT_ROUTE"
        }
      },
      "timers": {
        "state": {
          "connect-retry": "30.0",
          "minimum-advertisement-interval": "30.0"
        },
        "config": {
          "connect-retry": "30.0",
          "minimum-advertisement-interval": "30.0"
        }
      },
      "afi-safis": {
        "afi-safi": [
          {
            "add-paths": {
              "state": {
                "send": false
              },
              "config": {
                "send": false
              }
            },
            "state": {
              "afi-safi-name": "IPV4_UNICAST"
            },
            "apply-policy": {
              "state": {
                "default-import-policy": "REJECT_ROUTE",
                "default-export-policy": "REJECT_ROUTE"
              },
              "config": {
                "default-import-policy": "REJECT_ROUTE",
                "default-export-policy": "REJECT_ROUTE"
              }
            },
            "config": {
              "afi-safi-name": "IPV4_UNICAST"
            },
            "afi-safi-name": "IPV4_UNICAST",
            "graceful-restart": {
              "state": {},
              "config": {}
            }
          },
          {
            "add-paths": {
              "state": {
                "send": false
              },
              "config": {
                "send": false
              }
            },
            "state": {
              "afi-safi-name": "IPV6_UNICAST"
            },
            "apply-policy": {
              "state": {
                "default-import-policy": "REJECT_ROUTE",
                "default-export-policy": "REJECT_ROUTE"
              },
              "config": {
                "default-import-policy": "REJECT_ROUTE",
                "default-export-policy": "REJECT_ROUTE"
              }
            },
            "config": {
              "afi-safi-name": "IPV6_UNICAST"
            },
            "afi-safi-name": "IPV6_UNICAST",
            "graceful-restart": {
              "state": {},
              "config": {}
            }
          }
        ]
      },
      "ebgp-multihop": {
        "state": {
          "enabled": false,
          "multihop-ttl": 0
        },
        "config": {
          "enabled": false,
          "multihop-ttl": 0
        }
      },
      "use-multiple-paths": {
        "ebgp": {
          "state": {
            "allow-multiple-as": false
          },
          "config": {
            "allow-multiple-as": false
          }
        },
        "state": {
          "enabled": false
        },
        "config": {
          "enabled": false
        }
      },
      "config": {
        "send-community": "NONE",
        "neighbor-address": "192.100.132.0",
        "local-as": 0,
        "description": "",
        "route-flap-damping": false,
        "peer-group": "abcd",
        "peer-as": 4000857625,
        "enabled": true,
        "auth-password": ""
      },
      "transport": {
        "state": {
          "remote-address": "192.100.132.0",
          "mtu-discovery": false,
          "remote-port": 0
        },
        "config": {
          "mtu-discovery": true
        }
      }
    },
</snip>

```


* Try to subscribe the interface counter via gNMI_Subscribe.py from pygnmi

```
python pygnmi/gNMI_Subscribe.py  --server 192.168.97.11:3333 --username admin --password admin /interfaces/interface[name=*]/state/counters


<snip>

19/02/22 18:38:20,986 Update received
update {
  timestamp: 1550863399426914822
  update {
    path {
      elem {
        name: "interfaces"
      }
      elem {
        name: "interface"
        key {
          key: "name"
          value: "Ethernet1"
        }
      }
      elem {
        name: "state"
      }
      elem {
        name: "counters"
      }
      elem {
        name: "out-multicast-pkts"
      }
    }
    val {
      uint_val: 1522
    }
  }
}

19/02/22 18:38:20,987 Update received
update {
  timestamp: 1550863399426929905
  update {
    path {
      elem {
        name: "interfaces"
      }
      elem {
        name: "interface"
        key {
          key: "name"
          value: "Ethernet1"
        }
      }
      elem {
        name: "state"
      }
      elem {
        name: "counters"
      }
      elem {
        name: "out-octets"
      }
    }
    val {
      uint_val: 207773
    }
  }
}


</snip>
```



So, that's it. Now we can play around by changing the xpath to get different section of config or different telemetry counters. And, the next step is, based on this examples, to create a new script that follow our current workflow. 



