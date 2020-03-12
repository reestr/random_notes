# Remote access CRC running on a dedicated host

## Objective

The following are the technical steps required to deploy CRC to a system and configure remote access.  A goal of this approach is to retain as much as what CRC provides and configures on behalf of the user, and only port-forward the required TCP ports 6443, 80, 443 from the Host network interface to the CRC configured NAT network and VM.  In order to access CRC from other systems on the local network after port-forwarding has been configured, the DNS record api.crc.testing and wildcard domain *.apps-crc.testing must be configured on a DNS server used by the clients on the network.  If no DNS server is available, a client based DNS server (such as locally installed dnsmasq) could instead be used.

These steps assume the reader is deploying CRC to a compatible linux host, this procedure has been carried out on Fedora Server 31 running libvirt, the same steps will likely also work on Fedora Workstation or RHEL/CentOS 7.

### Procedural overview
1. Install CRC
2. Configure Port Forwarding
3. Create systemd unit for automatic start/stop [optional]
4. DNS configuration for clients

## 1. Initial installation of CRC

- Follow the [installation steps](https://code-ready.github.io/crc/#installation_gsg) to deploy CRC as you normally would.  It is important to proceed through the installation steps to completion, providing a pull secret and ensuring CRC will start, stop and function as you would expect.
- If you have additional spare capacity on your system, you can set the resources for CRC to make use of them.  The following examples setting 32GB of RAM and 6 vCPU.  Per the output you will need to delete and redeploy CRC after applying configuration changes:
```
$ crc config set cpus 6
Changes to configuration property 'cpus' are only applied when a new CRC instance is created.
If you already have a CRC instance, then for this configuration change to take effect, delete the CRC instance with 'crc delete' and start a new one with 'crc start'.
$ crc config set memory 32768
Changes to configuration property 'memory' are only applied when a new CRC instance is created.
If you already have a CRC instance, then for this configuration change to take effect, delete the CRC instance with 'crc delete' and start a new one with 'crc start'.
$ crc config get cpus
cpus : 6
$ crc config get memory
memory : 32768
```
- Start CRC with `crc start` and authenticate using `oc` as advised on the console, verify CRC is running:
```
$ oc get nodes
NAME                 STATUS   ROLES           AGE   VERSION
crc-w6th5-master-0   Ready    master,worker   27d   v1.16.2
```

## 2. Configure Port Forwarding
To remotely access the CRC VM from other hosts on the local network, the firewall on the CRC host needs to be opened to permit TCP ports 6443, 443, 80.  In addtion to this, port forward rules need to be configured with iptables to forward traffic from the host network to the NAT bridge configured by the CRC installer.  

### Open Fireall ports on the host
- Verify the network zones which have been set up on the Host system.  In the following snippet, eno1 is the physical interface used by the host, virbr0 is the default network configured by libvirt (we can ignore that), and crc is setup by the CRC installer.  Make a note of the zone used by the physical interface, if it is using a zone different to 'FedoraServer' you will need to substitute the following step:
```
$ sudo firewall-cmd --get-active-zones
libvirt
  interfaces: crc virbr0
FedoraServer
  interfaces: eno1
```
- Apply the following firewall rules, reload firewalld and verify they were successfully added.  If you are using a different firewall zone for your physical adaptor, replace `FedoraServer` with the name of that zone:
```
$ sudo firewall-cmd --permanent --zone=FedoraServer --add-port=6443/tcp
$ sudo firewall-cmd --permanent --zone=FedoraServer --add-port=443/tcp
$ sudo firewall-cmd --permanent --zone=FedoraServer --add-port=80/tcp
$ sudo firewall-cmd --reload
$ sudo firewall-cmd --list-all --zone=FedoraServer
FedoraServer
  target: default
  icmp-block-inversion: no
  interfaces:
  sources:
  services: cockpit dhcpv6-client ssh
  ports: 6443/tcp 443/tcp 80/tcp
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```
### Apply forward rules
The simplest way to manage port forward rules is to use qemu hook scripts, these will directly add or remote iptables rules on the host corresponding to libvirt VM start/stop events.  The libvirt project outlines some background to this on the [upstream documentation](https://wiki.libvirt.org/page/Networking#Forwarding_Incoming_Connections), we'll install a [convienience script](https://github.com/saschpe/libvirt-hook-qemu) and manage the ports we want to forward from a configuration file.  The steps will walkthough installation of the hooks scripts and the configuration required for CRC
 - Clone and install the hook script:
```
$ git clone https://github.com/saschpe/libvirt-hook-qemu.git
$ cd libvirt-hook-qemu
$ sudo make install
```
 - Open an editor and paste the following configuration into `/etc/libvirt/hooks/hooks.json`.  This configuration is for the crc virtual machine connected via the crc bridge (interface).  The private IP is configured by the CRC installer and in most cases be **192.168.130.11**.  The TCP ports we want to forward are 6443. 443 and 80.
```
{
   "crc": {
        "interface": "crc",        
        "private_ip": "192.168.130.11",
        "port_map": {
            "tcp": [6443,443,80]
        }
    }
}
```
- Once the above configuation is added, restart libvirt.  Assuming CRC is still running on the system, the new entries in iptables will show per the following output:
```
$ sudo systemctl restart libvirtd
$ sudo iptables -L -n -t nat | grep crc
DNAT-crc   all  --  0.0.0.0/0            192.168.178.3
DNAT-crc   all  --  0.0.0.0/0            192.168.178.3
SNAT-crc   all  --  192.168.130.11       192.168.130.11
Chain DNAT-crc (2 references)
Chain SNAT-crc (1 references)
```
- To verify the firewall port is open on the Host, and packets are being forward as expected to the CRC VM, use a tool such as `nmap` on any other client which is on the local network.  The following example uses 192.168.178.3 which is the IP address of the CRC Host system:
```
~ sudo nmap -sT 192.168.178.3 -p 6443,443,80
Starting Nmap 7.80 ( https://nmap.org ) at 2020-03-12 22:26 CET
Nmap scan report for mothra.mynetwork.local (192.168.178.3)
Host is up (0.0039s latency).

PORT     STATE SERVICE
80/tcp   open  http
443/tcp  open  https
6443/tcp open  sun-sr-https
MAC Address: XX:XX:XX:XX:XX:XX

Nmap done: 1 IP address (1 host up) scanned in 0.07 seconds
```

## 3. Create systemd unit for automatic start/stop [optional]

This step is only required if you wish for CRC to start and stop with host startup and shutdown.  We want to avoid configuring auto-start on the VM because the `crc` binary runs a number of checks, including certificate expiry when it starts and stops the crc VM.  To maintain the supported use of CRC with the crc binary, we'll create a systemd unit file for crc.  Libvirt also needs to be configured to auto-start with the Host.

Once the systemd unit is configured, CRC can then be managed by systemd.  We want to prevent systemd automatically restarting on failure, if there is any issue starting CRC from the `crc` command we may need to manually intervene and investigate, we don't want systemd to continually attempt to restart CRC.

- Ensure Libvirt starts with the Host startup
`$ sudo systemctl enable libvirtd`
- Create a systemd unit file called `/etc/systemd/system/crc.service` and paste the following into it, replace `<USERNAME>` with your userid on the host for which you run CRC:
```
[Unit]
Description=crc
After=libvirtd.service

[Service]
RemainAfterExit=yes
User=<USERNAME>
Type=forking
ExecStart=/usr/local/bin/crc start
ExecStop=/usr/local/bin/crc stop
TimeoutStartSec=900
TimeoutStopSec=300
Restart=no

[Install]
WantedBy=multi-user.target
```
- Install the systemd unit file, if CRC is running, stop it the tranditional way (`crc stop`), and start it using the systemd unit file.  Finally enable the unit file so it will autostart when the Host is turned on.  Note: Starting CRC may take up to 5 minutes, progress can be viewed using journald:
```
$ sudo systemctl daemon-reload
$ crc stop
$ sudo systemctl start crc
$ sudo systemctl status crc
● crc.service - crc
   Loaded: loaded (/etc/systemd/system/crc.service; enabled; vendor preset: disabled)
   Active: active (exited) since Thu 2020-03-12 20:49:32 CET; 2h 37min ago
  Process: 1106 ExecStart=/usr/local/bin/crc start (code=exited, status=0/SUCCESS)
      CPU: 3.952s

Mar 12 20:46:32 mothra.mynetwork.local crc[1106]: level=info msg="Starting OpenShift cluster ... [waiting 3m]"
Mar 12 20:49:32 mothra.mynetwork.local crc[1106]: level=info
Mar 12 20:49:32 mothra.mynetwork.local crc[1106]: level=info msg="To access the cluster, first set up your environment by following 'crc oc-env' instructions"
Mar 12 20:49:32 mothra.mynetwork.local crc[1106]: level=info msg="Then you can access it by running 'oc login -u developer -p developer https://api.crc.testing:6443'"
Mar 12 20:49:32 mothra.mynetwork.local crc[1106]: level=info msg="To login as an admin, run 'oc login -u kubeadmin -p XXXXX-XXXXX-XXXXX-XXXXX https://api.crc.testing:6443'"
Mar 12 20:49:32 mothra.mynetwork.local crc[1106]: level=info
Mar 12 20:49:32 mothra.mynetwork.local crc[1106]: level=info msg="You can now run 'crc console' and use these credentials to access the OpenShift web console"
Mar 12 20:49:32 mothra.mynetwork.local crc[1106]: Started the OpenShift cluster
Mar 12 20:49:32 mothra.mynetwork.local crc[1106]: level=warning msg="The cluster might report a degraded or error state. This is expected since several operators have been disabled to lower the resource usage. For more information, please >
Mar 12 20:49:32 mothra.mynetwork.local systemd[1]: Started crc.
```

## 4. DNS configuration for clients

In order for clients on the network to access CRC and the applications it hosts, we need to configure the DNS record api.crc.testing and wildcard domain *.apps-crc.testing on a network based DNS server used by the clients, or dnsmasq (or equivalent) running locally on the client machines.

- Assuming a DNS server is available to you, configure the DNS entries pointing to the IP address for the CRC Host system.  For dnsmasq the entries would look like the following for the IP **192.168.178.3**:
```
address=/api.crc.testing/192.168.178.3
address=/.apps-crc.testing/192.168.178.3
```

If no DNS Server is available, dnsmasq can be installed on each Mac or Linux based client and configured with the above records.  Unfortunaly, due to the need for a wildcard entry, the Hosts file cannot be used on systems.
 - See [this guide](https://hybridhacker.com/wildcard-dns-with-dnsmasq-on-mac-os-x.html) for MacOS
 - See [this guide](https://fedoramagazine.org/using-the-networkmanagers-dnsmasq-plugin/) for Fedora

Once DNS is configured, accessing CRC is as simple as installing the oc or kubectl onto your clients, and logging in using the guidance provided from `crc start` (or `journalctl -n 10 -u crc`) on the CRC host machine.

## Other approaches to remote access CRC

An alternative approach to using port forwrding to allow remote access would be to install and configure HAProxy on the CRC host system.  The following guide steps through that process:
- [Using HAProxy to access CRC](https://gist.github.com/tmckayus/8e843f90c44ac841d0673434c7de0c6a)
