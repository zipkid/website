+++
title = "Choria AAA"
toc = true
weight = 220
+++

Choria features a full suite of Authentication, Authorization and Auditing capabilities.

  * Authentication - _who_ you are, derived from a certificate
  * Authorization - _what_ you may do on any given node, keyed to your certificate-based identity
  * Auditing - _log_ of what you did, showing your certificate based identity and all requests

Earlier we made a certificate called _rip.mcollective_ which is used to establish your identity as _choria=rip.mcollective_ which will be used throughout in the AAA system.

## Authorization

Choria sets up the popular [Action Policy](https://github.com/puppetlabs/mcollective-actionpolicy-auth) based authorization and does so in a _default deny_ mode which means by default, no-one can make any requests.

Some plugins may elect to ship authorization rules that allow certain read only actions by default, like the _mco puppet status_ command, but you can change or override all of this.

### Site Policies

You can allow your own users only certain access, previously when configuring your first user, we did this already via _Hiera_:

```yaml
mcollective::site_policies:
  - action: "allow"
    callers: "choria=rip.mcollective"
    actions: "*"
    facts: "*"
    classes: "*"
```

You'll note this is an array so you can have many policies, site policies are applied to **all agents**.

### Agent specific policies

This will allow a specific certificate to only _block_ ip addresses on my firewall but nothing else:

```yaml
mcollective_agent_iptables::policies:
  - action: "allow"
    callers: "choria=typhon.mcollective"
    actions: "block"
    facts: "*"
    classes: "*"
```

For full details see the [Action Policy](https://github.com/puppetlabs/mcollective-actionpolicy-auth) docs.

### Per plugin default override

As mentioned by default all actions are denied across all agents, you can change a specific agent to default allow via _Hiera_:

```yaml
mcollective_agent_puppet::policy_default: allow
```

### Site wide default policy

By default all actions are denied, if like in a Lab environment you want to simplify things and allow all actions across all agents, you can set this in _Hiera_:

{{% notice warning %}}
Enabling this will allow anyone with a signed mcollective cert to perform any action on any node, please consider carefully before changing this setting.
{{% /notice %}}

```yaml
mcollective::policy_default: allow
```

## Authentication
### Custom certificate names

Authentication is done via the certname embedded in the certificate, certificates must be signed by the Puppet CA.

By default the only certificates that will be accepted from clients are those matching the pattern _/\.mcollective$/_, if you have some special needs you can adjust this via _Hiera_:

```yaml
mcollective_choria::config:
  security.certname_whitelist: "bob, jill, //\.mcollective$//"
```

And you can request custom certificate names on the CLI:

```bash
$ mco choria request_cert --certname bob
```

### Revoking access

Public certificates are distributed automatically but will never be removed.  To remove them you have to manually arrange for the files to be deleted from all nodes, perhaps using Puppet, before a new one can be distributed.  These live in <i>/etc/puppetlabs/puppet/choria_security/public_certs</i>.

### Privileged certificates

Unless specifically requested you should never use certificates matching the pattern _/\.privileged\.mcollective$/_, this is an advanced feature that is reserved for a future REST server where Authentication is delegated to a trusted piece of software.

## Auditing

Auditing is configured to write to a log file _/var/log/puppetlabs/mcollective-audit.log_ by default, you should set up rotation if desired (it's not done by the module), its contents looks like:

```bash
[2016-12-13 08:32:34 UTC] reqid=30d706be63e555db8c073ec17a23af44: reqtime=1481617954 caller=choria=rip.mcollective@dev3.example.net agent=rpcutil action=ping data={:process_results=>true}
[2016-12-13 08:32:43 UTC] reqid=e0c60ad2f58d52699e6524039decc257: reqtime=1481617963 caller=choria=rip.mcollective@dev3.example.net agent=puppet action=status data={:process_results=>true}
[2016-12-13 13:15:09 UTC] reqid=1235e001c9b15414b748ab26607e1063: reqtime=1481634909 caller=choria=rip.mcollective@dev3.example.net agent=puppet action=status data={:process_results=>true}
[2016-12-13 13:15:35 UTC] reqid=cf95bc7621ff55a8a197e3f2e394406e: reqtime=1481634935 caller=choria=rip.mcollective@dev3.example.net agent=puppet action=status data={:process_results=>true}
[2016-12-13 13:15:43 UTC] reqid=18192c7f260c5788a33b60ce4f01771c: reqtime=1481634943 caller=choria=rip.mcollective@dev3.example.net agent=puppet action=status data={:process_results=>true}
```

The above is the native MCollective logging format, it's a bit old and predates things like logstash being popular.  There's a new Choria plugin that you can enable via Hiera:

```yaml
mcollective::server_config:
  rpcauditprovider: "choria"
  plugin.rpcaudit.logfile: "/var/log/puppetlabs/choria-audit.log"
```

The format this will log in can be seen below, works well with ES and things like [./jq](https://stedolan.github.io/jq/).

```json
{"timestamp":"2017-01-07T07:13:30.174912+0000","request_id":"a1dbec0d79845644a5761e5969ec8ff2","request_time":1483773209,"caller":"choria=rip.mcollective","sender":"dev4.example.net","agent":"package","action":"status","data":{"package":"puppet-agent","version":null,"process_results":true}}
{"timestamp":"2017-01-07T07:41:36.173828+0000","request_id":"bbf1d1172abe5ea2a5a73ffcb436c5d1","request_time":1483774895,"caller":"choria=rip.mcollective","sender":"dev4.example.net","agent":"puppet","action":"disable","data":{"process_results":true}}
{"timestamp":"2017-01-07T07:42:47.278562+0000","request_id":"8f77f6f35e2358b0b697ad9b69f7f528","request_time":1483774967,"caller":"choria=rip.mcollective","sender":"dev4.example.net","agent":"puppet","action":"status","data":{"process_results":true}}
```
