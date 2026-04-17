# Troubleshooting Log

Connectivity issues hit during deployment and how they were worked through.

---

## Issue 1: One-Way Ping Between VM and Main PC

**Symptom:**

After the VM was up I ran basic ping tests to verify connectivity between the main PC and the VM. Pinging the VM from the PC worked fine. Pinging the PC from the VM got no response.

**Initial suspicion:**

First thought was a routing or firewall rule issue on the router. The VM was on a separate VLAN from the main PC, so inter-VLAN traffic has to pass through the router and be explicitly permitted by firewall policy. Checked the firewall rules on the router and they looked correct in both directions.

Ran the ping from the VM again. Still nothing.

**Root cause:**

The host firewall on the main PC was blocking inbound ICMP echo requests. When the VM sent a ping to the PC, the PC was receiving it but the host firewall was silently dropping it before any reply went out. From the VM's perspective it looked identical to the traffic never arriving at all.

This is default behavior on many operating systems. Host firewalls block inbound ICMP on most network profiles out of the box, which makes it easy to miss when your attention is focused on the network infrastructure.

**Fix:**

Enabled the inbound ICMP echo request rule in the host firewall, scoped to the private network profile only. After that, ping worked in both directions and confirmed basic connectivity before moving on to testing the actual service port.

**Takeaway:**

When troubleshooting a connectivity issue, check both ends of the connection, not just the infrastructure in the middle. The router rules were correct the whole time. The problem was the endpoint firewall on the destination machine.

---

## Issue 2: Verifying the Port Forward Was Working

**Symptom:**

After configuring the port forward on the router, I needed to confirm the service was actually reachable from outside the network before considering it live.

**What I checked:**

Testing port forwards from inside your own network does not work reliably because NAT hairpinning behavior varies by router. The correct way to test is from a device that is genuinely outside the network.

Used an external port checking tool to verify the forwarded port was open and responding. Confirmed the service process was listening on the correct port inside the VM before running the external test.

**Port forward configuration:**

- Protocol scoped to what the service required
- External port: single service port only
- Internal destination: VM on the server VLAN
- No other ports forwarded

Keeping the forward scoped to exactly one port on exactly one internal destination is the right approach. There is no reason to forward a port range or leave the destination open to the whole subnet.

**Takeaway:**

Always verify port forwards from outside the network. Testing from inside gives false confidence if hairpin NAT is not configured. Also assign a fixed IP address to any VM that has a port forward pointing at it, otherwise the forward breaks the next time the address changes.

---

## General Notes

Both issues resolved quickly once the right layer of the stack was being examined. The pattern that worked:

1. Check the infrastructure first -- router rules, VLAN config, port forwards
2. If that looks correct, check the endpoints -- firewall on both the source and destination
3. Verify from outside the network, not from another internal device

Connectivity problems that look like routing issues are sometimes endpoint firewall issues and vice versa. Working through the layers systematically is faster than guessing.
