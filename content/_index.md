# Network Operators for Kubernetes

This project will be the home for using the Kubernetes Operator pattern
for managing (primarily) on-premises network infrastructure and/or VMs.

## What is it?

`network-operator.io` is a domain registered for creating Kubernetes
Operators with a valid, owned domain.

The aim is to support multiple Operators for different things, such as:

* Meraki Deployments
* Streaming Telemetry Deployments
* Configuration of abstracted elements across multiple vendors, such as:
  * NTP
  * Syslog
  * ACLs

This operators would be generally opinionated while also allowing
vendor-specific extensions/options.

### Proposed Operators: Meraki

An example of using a proposed Meraki Operator to define wireless
networks might be the following:

```yaml
# manifests/network.yaml
apiVersion: meraki.network-operator.io/v1alpha1
kind: Network
metadata:
  name: network-sample
spec:
  organization: example-org
  statusPage:
    local: enabled
    remote: enabled
---
# manifests/rfprofile.yaml
apiVersion: meraki.network-operator.io/v1alpha1
kind: RFProfile
metadata:
  name: rfprofile-sample
spec:
  network: network-sample
  band:
    selectionType: ap
    clientBalancing: yes
    steering: yes
  5ghz:
    power:
      minimum: 8
      maximum: 10
    channel:
      width: 20
      autoChannels:
        - 56
        - 6
  24ghz:
    power:
      minimum: 8
      maximum: 10
    channel:
      width: 20
      autoChannels:
        - 1
        - 6
        - 11
---
# manifests/ssid.yaml
apiVersion: meraki.network-operator.io/v1alpha1
kind: SSID
metadata:
  name: example-ssid
spec:
  network: network-sample
  ipAssignmentMode: nat
  vlans:
    tagging: yes
    default: 10
  apAvailabilityTags:
    - site1
    - prod
  bandSelection: steering
  auth:
    mode: psk
    psk: examplepsk123
  encryption: wpa
  broadcast: yes
  lanIsolation: no
---
# manifests/globalwirelesssettings.yaml
apiVersion: meraki.network-operator.io/v1alpha1
kind: GlobalWirelessSettings
metadata:
  name: globalwirelesssettings-sample
spec:
  network: network-sample
  upgradeSetting: minimizeClientDowntime
  ledLights: on
  locationAnalytics: no
  mesh: no
---
# manifests/ap1.yaml
apiVersion: meraki.network-operator.io/v1alpha1
kind: AccessPoint
metadata:
  name: accesspoint1-sample
spec:
  network: network-sample
  serialNumber: abc123
  physicalAddress: |
    123 Example St
    Somewhere, USA
  tags:
    - prod
    - site1
  rfProfile: rfprofile-sample
  radios:
    5ghz:
      channel: 60
      power: 12
    24ghz:
      channel: 6
      power: 11
---
# manifests/ap2.yaml
apiVersion: meraki.network-operator.io/v1alpha1
kind: AccessPoint
metadata:
  name: accesspoint2-sample
spec:
  network: network-sample
  serialNumber: 123abc
  physicalAddress: |
    123 Example St
    Somewhere, Canada
  tags:
    - prod
    - site1
  rfProfile: rfprofile-sample
```

The above could be easily templated with Helm and/or customized with Kustomize.

### Proposed Operators: Abstracted

The Abstracted Network Operator would allow you to define generic configurations that apply to all (or most) devices, such as NTP or Syslog.  An example might be the following for configuring NTP on all devices:

```yaml
---
# manifests/device-a.yaml
apiVersion: abstracted.network-operator.io/v1alpha1
kind: Device
metadata:
  name: RouterA
spec:
  serialNumber: xyz123
  physicalAddress: |
    123 Example St
    Somerewhere, Canada
  os:
    name: nxos
    version: 7.0.3
  tags:
    - prod
    - site3
---
# manifests/device-b.yaml
apiVersion: abstracted.network-operator.io/v1alpha1
kind: Device
metadata:
  name: RouterB
spec:
  serialNumber: 123xyz
  physicalAddress: |
    123 Example St
    Somerewhere, UK
  os:
    name: eos
  tags:
    - prod
    - site2
---
# manifests/ntp.yaml
apiVersion: abstracted.network-operator.io/v1alpha1
kind: NTP
metadata:
  name: site-a
spec:
  servers:
    - name: 0.pool.ntp.org
      preferred: yes
    - name: 1.pool.ntp.org
  vrf: management
  tags:
    - site3
```

The above Custom Resources would add two routers to device inventory, each with different tags.  Then, it would create an NTP configuration, but only for devices in the inventory with the `site3` tag (so, only `RouterA` in this example).  Possibly extensions to his Custom Resource Definition might including being able to use external inventory sources, such as NetBox, with an additional parameter.  An example of that might be:

```yaml
---
# manifests/netbox-inventory.yaml
apiVersion: abstracted.network-operator.io/v1alpha1
kind: InventorySource
metadata:
  name: netbox
spec:
  server: netbox.example.com
---
# manifests/ntp.yaml
apiVersion: abstracted.network-operator.io/v1alpha1
kind: NTP
metadata:
  name: site-a
spec:
  inventory: netbox
  servers:
    - name: 0.pool.ntp.org
      preferred: yes
    - name: 1.pool.ntp.org
  tags:
    - site4
```
