+++
title = "Announcing the Meraki Operator"
+++

# Meraki Operator

The Proof of Viability (PoV) for the Meraki Operator is nearly finished.
The final step is to build and publish the image so that it can run
within the cluster itself as it is currently only tested running
external to the cluster.  It has a limited feature set, but includes
the following:

* Create a Meraki Network
* Create an RF Profile
  * Set valid channels for dynamic channel adjustments
  * Set channel width for 5Ghz
  * Set minimum and maximum transmit power for transmit power modifications
  * Enable/disable client balancing
* Create an SSID
  * Configure as Open or PSK authentication
  * Set a PSK
  * Set a networking mode
    * NAT
    * Bridge
    * Layer 3 Roaming
  * Set default VLAN ID
* Adopt an MR Access Point
  * Set static 2.4Ghz and/or 5Ghz channels
  * Set a static power target
  * Set device tags
* Configure global settings
  * Upgrade policy
    * Minimize client downtime
    * Minimize upgrade time
  * Mesh networking
  * Analytics collection
* Support for multiple organizations and networks

As an example, the following manifests will:

* Create a network, RF Profile, and SSID
* Adopt an Access Point and select an RF Profile
* Define the upgrade strategy for APs to minimize client impact

```yaml
---
apiVersion: meraki.network-operator.io/v1alpha1
kind: Network
metadata:
  name: yyz-corporate
spec:
  organization: Example Organization
  name: YYZ Corporate
  statusPage:
    local: enabled
    remote: disabled
---
apiVersion: meraki.network-operator.io/v1alpha1
kind: RFProfile
metadata:
  name: YYZ Default RF Profile
spec:
  organization: Example Organization
  network: YYZ Corporate
  band:
    selectionType: ap
    steering: true
  clientBalancing: true
  fiveGhz:
    channel:
      width: 20
---
apiVersion: meraki.network-operator.io/v1alpha1
kind: SSID
metadata:
  name: YYZ Corporate Employee SSID
spec:
  organization: Example Organization
  network: YYZ Corporate
  authentication:
    mode: psk
    psk: ExamplePSK123
---
apiVersion: meraki.network-operator.io/v1alpha1
kind: AccessPoint
metadata:
  name: ap1-yyz
spec:
  organization: Example Organization
  network: YYZ Corporate
  physicalAddress: |
    123 Example St
    Columbia, SC
    USA
  radios:
    24ghz:
      channel: 6
  rfProfile:  YYZ Default RF Profile
  serialNumber: ABCD-1234-WXYZ
  tags:
  - yyz
  - corp
---
apiVersion: meraki.network-operator.io/v1alpha1
kind: GlobalWirelessSettings
metadata:
  name: yyz-corporate-wifi-settings
spec:
  upgradeSetting: minimizeClientDowntime
```

On its own, this can be accomplished with Ansible (and, in fact, the
PoV uses Ansible to do all of this!) or with a custom Python or Go
project -- really, anything that leverages the Meraki API could do
everything that the Meraki Operator does.  However, the value comes
from leveraging the Kubernetes ecosystem.  The primary driver for
this project was to:

* Provide an API frontend on which a company may already be standardized
* Provide an API frontent on which traditional developers could manage their networks
* Reduce the need to create custom frameworks, libraries, codebases, Ansible roles/playbooks, etc.
* Remove the need for managing authentication, authorization, auditing/logging, role-based access control, etc.

The above items are handled by leveraging native Kubernetes features
and the Operator pattern.  However, being able to leverage the wider
Kubernetes ecosystem is another massive advantage.

## Integrating OPA Gatekeeper

Compliance is a huge advantage of the Kubernetes ecosystem.  While some
things are provided by Kubernetes itself (as mentioned above), some more
complex requirements are not.  For example, consider the following
requirements for your wireless network:

* You must not use DFS channels
* The only SSID that can use "Open" authentication must be named `Guest WiFi`
* Your RADIUS servers must be from a list of servers

The first is an example of a local regulatory requirement; the second is
an example of a mandate from your Security team; the third is an
internal engineering standards requirement.  All of these can be
enforced via manual efforts and audits, peer review, etc.  However, when
leveraging the Meraki Operator with OPA Gatekeeper, these sorts of
requirements can be enforced separate from your codebase, without
requiring peer review, and with confidence that the state of your WiFi
network is complying with regulatory, security, and architectural
requirements!  Additionally, areas where peer review might miss a
violation (such as a typo in an IP address for a RADIUS server, such as
writing `192.168.31.150` instead of `192.168.13.150`) are now enforced
by code for free!  This provides benefits to the persona writing the
change, the persona reviewing the change, and the person who initially
defined the standard!

To show an example of this, we'll examine the most simple of the above
three examples: `The only SSID that can use "Open" authentication must be named "Guest WiFi"`.
Normally, we would need peer review and audits to validate this.
However, if we leverage OPA Gatekeeper to enforce this, we can be
confident that our standards are enforced without requiring additional
cognitive workload.

First, install OPA Gatekeeper:

```bash
$ kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.7/deploy/gatekeeper.yaml
```

Once this is done, wait a few minutes for images to pull and pods to
start.  Next, create the `ConstraintTemplate`.  This will create a
parameterized `Constraint` Custom Resource Definition that we'll use to
define the name of the SSID that is valid.

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: openssidrequirements
spec:
  crd:
    spec:
      names:
        kind: OpenSSIDRequirements
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          properties:
            ssid_name:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package openssidrequirements

        violation[{"msg": msg}] {
          provided := input.review.object.metadata.name
          required := input.parameters.ssid_name
          auth_mode := input.review.object.spec.authentication.mode
          auth_mode == "open"
          provided != required
          msg := sprintf("the ssid name must be: %v", [required])
        }
```

The above template checks to see if we've defined an SSID with an
authentication mode of `open`.  If it is, then it checks to see if the
SSID name that was provided by the object matches the one defined by the
`Constraint`.  It's important to note that a `ConstraintTemplate`
doesn't do anything on its own.  To actually enforce our desired policy,
we need to create the `Constraint` itself:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: OpenSSIDRequirements
metadata:
  name: open-ssid-name-must-be-guest
spec:
  match:
    kinds:
      - apiGroups: ["meraki.network-operator.io"]
        kinds: ["SSIDs", "SSID"]
  parameters:
    ssid_name: "Guest WiFi"
```

The above `Constraint` is of a type `OpenSSIDRequirements`, which
matches the `.spec.crd.spec.names.kind` value from our
`ConstraintTemplate` above.  It operates only on the `SSIDs` and `SSID`
objects from the `meraki.network-operator.io` API.  Since these objects
are where we create SSIDs and define authentication modes, these are the
only objects we need to worry about.  The `.spec.parameters` are passed
to the associated `ConstraintTemplate`, so in this case, our
`Constraint` says that the SSID name must be `Guest WiFi` if the
authentication mode is `open`.  To simplify the steps above, our logic
is now:

```python
if apiGroup == "meraki.network-operator.io" and kind in ["SSIDs", "SSID"]:
    if object["spec"]["authentication"]["mode"] == "open":
        if object["spec"]["name"] != "Guest WiFi":
            raise ValueError("When using Open authentication, the SSID name must be 'Guest WiFi'")
```

> The above is written in Python, but is for illustrative purposes only.
> The code above does not exist anywhere.

Now, if we try to create a new SSID, we'll get an error.  Given the
following manifest:

```yaml
apiVersion: meraki.network-operator.io/v1alpha1
kind: SSID
metadata:
  name: invalid-wifi-name-open
spec:
  organization: Example Organization
  network: YYZ Corporate
  name: Corporate WiFi
  authentication:
    mode: open
```

When we try to apply it, we get an error with an appropriate message:

```bash
$ kubectl apply -f invalid-wifi-name-open.yml
Error from server ([open-ssid-name-must-be-guest] the ssid name must be: Guest WiFi): error when creating "fail.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [open-ssid-name-must-be-guest] the ssid name must be: Guest WiFi
```

We can verify that the `Constraint` picked this up:

```bash
$ kubectl describe openssidrequirements | sed -n '/Status/,$p' 
Status:
  Audit Timestamp:  2022-01-16T21:27:33Z
  By Pod:
    Constraint UID:       1746ddbe-7715-425c-8924-825c1607af2d
    Enforced:             true
    Id:                   gatekeeper-audit-59d4b6fd4c-tw6sk
    Observed Generation:  3
    Operations:
      audit
      status
    Constraint UID:       1746ddbe-7715-425c-8924-825c1607af2d
    Enforced:             true
    Id:                   gatekeeper-controller-manager-66f474f785-g9r66
    Observed Generation:  3
    Operations:
      mutation-webhook
      webhook
    Constraint UID:       1746ddbe-7715-425c-8924-825c1607af2d
    Enforced:             true
    Id:                   gatekeeper-controller-manager-66f474f785-ntkkh
    Observed Generation:  3
    Operations:
      mutation-webhook
      webhook
    Constraint UID:       1746ddbe-7715-425c-8924-825c1607af2d
    Enforced:             true
    Id:                   gatekeeper-controller-manager-66f474f785-qkxsv
    Observed Generation:  3
    Operations:
      mutation-webhook
      webhook
  Total Violations:  1
  Violations:
    Enforcement Action:  deny
    Kind:                SSID
    Message:             the ssid name must be: Guest WiFi
    Name:                invalid-wifi-name-open
    Namespace:           default
Events:                  <none>```
```

This enforces our standards using a decoupled approach.  Different teams
can own the policies, implementations, etc.  We leverage technologies
that are common to multiple teams, so our Security team can write a
policy leveraging exactly the same toolset as they do for enforcing
policies with developers deploying to Kubernetes, such as enforcing
Read-Only Filesystems.  This provides collaboration and low-friction
cross-team interactions.  It removes requirements to remember every
tiny detail when implementing and reviewing: if something doesn't
conform to a standard, it will be rejected with a meaningful message,
so you can simply write a change and adjust as necessary.

Hopefully this helps to show the advantage of managing a Meraki
infrastructure (or any network configuration!) via the Kubernetes
Operator pattern.
