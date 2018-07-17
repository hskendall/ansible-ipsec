# Host-to-host transport-mode IPSec

Ansible role to enable IPSec encryption between Ansible-managed nodes with minimal performance
overhead. This role is especially suitable for protecting communications between farms of
cloud servers and can effectively replace the need for the complexity of configuring TLS for
each service running on the servers.

## Inventory

Create group `ipsec` and add all hosts that should be IPSec connected:

    [ipsec]
    test1
    test2

This role will always create IPSec configuration for full `ipsec` group on each host, regardless
of current play scope limitation. This is to ensure that scope-limited runs don't leave some
hosts with IPSec configuration and their counterparts without one, which will cause issues
with the `require` policy.

## Firewall

For IPSec to work the following ports and protocols must be opened:

* `500/udp` IKE (`iptables -A INPUT -p udp --dport 500`)
* `esp` the ESP protocol (`iptables -A INPUT -p esp`)

These ports should be only opened to the other IPSec peers, there's no need to open them
publicly.

## Configuration

Master IPSec secret, used as seed to securely generate unique pre-shared key for each host pair.
This remains the same across all Ansible managed hosts. Use `ansible-vault` for secure storage.

    ipsec_secret: '088d7633c620f24... generate your own with openssl rand -hex 60'

Should IPSec work in fail-close or fail-open mode? 
* `use` - fail open: if IPSec cannot be established, the traffic will flow unencrypted.
* `require` - fail-close: not traffic will be allowed without IPSec.

    ipsec_policy: 'use'

Keying method.
* `ike` is the preferred keying mode with IKE daemon managing keys and refreshing them at proper
   intervals, suitable for long-term production environments
* `setkey` uses day-dependent static keys which is **insecure** in long term but may be suitable for
  development environments with frequent Ansible builds that will replace the keys; the IKE daemon
  is not used, everything happens on the kernel network stack

    ipsec_mode: 'ike'

Never require IPSec for SSH. The assumption is that SSH provides trusted channel on its own and 
it allows remote access to IPSec-enabled servers even if something goes wrong with IPSec channels.
        
    ipsec_open_ssh: yes

Never require IPSec for ICMP protocol. This allows network troubleshooting messages such as ping
or port unreachable still work between IPSec-enabled hosts.

    ipsec_open_icmp: yes

Create IPSec forwarding policies in addition to input and output policies. This is normally only
needed if you use Docker and other such solutions sending traffic through virtual network interfaces
that IPSec will consider forwarded traffic. Not needed for regulard host-to-host traffic and 
disabled by default.

    ipsec_forward: no

## Key generation

Keys for both `ike` and `setkey` mode are derived from the `ipsec_secret` and a number of other
paramters to ensure they are unique for each host pair. For `ike` mode keys are stored in
the `/etc/racoon/psk.txt` file and they are long term keys generated using the following Jinja2
syntax:

```
(hostname, inventory_hostname, ipsec_secret) | sort | join | hash('sha256')
```

The `setkey` mode uses manual keying so keys need to be generated for each direction, and for
ESP (encryption) and MAC (authentication) separately. In addition, non-secret connection identifiers
(SPI and IPC) need to be generated for each direction. Playbook run date is included into the key
input to cycle keys on daily basis (assuming you run Ansible daily). A static key identifier (ESP,
MAC, SPI, IPC) is also mixed into the key to ensure each key is different.

```
run_date=(template_run_date['year'], template_run_date['month'], template_run_date['day'])
( run_date, host1 , host2 , ipsec_secret , "ESP" )   | sort | join | hash('sha256') | truncate(32,end='')
( run_date, host1 , host2 , ipsec_secret , "MAC" )   | sort | join | hash('sha256') | truncate(64,end='')
( run_date, host1 , host2 , ipsec_secret , "SPI" )   | sort | join | hash('sha256') | truncate(6,end='')
( run_date, host1 , host2 , ipsec_secret , "IPC" )   | sort | join | hash('sha256') | truncate(6,end='')
```
