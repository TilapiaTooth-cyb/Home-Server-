# Home-Server-
I didn't record much of what I did before because my tamal(google it) size brain didn't think about this as a good project to keep a diary of :)

## Installation
I did the basic:
- Got the Proxmox ISO on a Ventoy USB and installed the hypervisor twice (I forgot my password the first time, I know, dumb)
- Realized that I didn't have an Ethernet cable, dammit.
- Used my phone with USB tethering but the gateway wasn't gatwaying. So I checked /var/lib/dhcp/dhclient.leases to see the gateway IP available for the tethered interface, did some magic (`ip route replace` command) to swap the routes and wallah, it worked.
- apt update :)
- Installed wpa_supplicant to check if the wireless adapter could see my home network and add the credentials to connect.
- IP wasn't IProtocoling, because the installer had set everything up for the Ethernet port that doesn't exist. Turns out you can't just bridge Wi-Fi like you would a cable, so:
  - The Wi-Fi card (`wlp2s0`) got the real IP on my network (static, because Proxmox gets cranky when its IP changes).
  - `vmbr0` became a lonely internal bridge (`10.10.10.0/24`) with no physical port, and the laptop NATs everything out through the Wi-Fi.
  - Every container lives behind that NAT, so if I want to reach one from my LAN I have to forward ports by hand with iptables. Fun!

## Things that broke (a lot)
- **I could ping the router and 8.8.8.8 from Proxmox, but not from my PC to Proxmox.** Spent a long time blaming client isolation, ARP, firewalls... Then it just started working. I have no idea why. Moving on. (lmao)
- **After a reboot the Wi-Fi was DOWN.** The only reason it worked before is that I had started wpa_supplicant by hand. Nothing was starting it at boot. Fixed it by running it as a systemd service for that interface (`wpa_supplicant@wlp2s0`). Lesson: if it only works because you typed it, it doesn't really work.
- **Gateway confusion.** I kept trying to ping the gateway I typed during installation (192.168.88.1) over the phone tether. It was never going to answer, because that network was on a dead Ethernet port. The only gateway that mattered was whatever the phone handed out.

## Things I set up
- **Closed-lid server:** told systemd-logind to ignore the lid (`HandleLidSwitch=ignore`) so I can close the laptop without it taking a nap.
- **Storage:** checked the LVM and it says 93% "assigned", which sounded scary but it's only allocated, not used. The 16 GB left free is on purpose, LVM needs headroom for thin pool metadata.
- **Pi-hole in an LXC container** (ID 100, `10.10.10.2`) so my whole network can block ads at the DNS level:
  - Forwarded DNS (port 53) and the web UI (8080) from the laptop's Wi-Fi IP into the container.
  - Had to set Pi-hole to listen on all origins, since LAN devices look like foreigners after the NAT.
  - Tested with `nslookup` (it answered, I felt like a genius).
  - Changed the DNS in the router's DHCP settings to point at the Pi-hole, reconnected my Linux PC and it picked it up.
  - My laptop ended up on `192.168.88.5`, not the `.20` I was typing for hours. Wrong IP, great confidence.
- **Single point of failure alert:** now my whole network's DNS depends on a laptop on Wi-Fi. If it dies, the internet "dies". Making a note to keep a fallback plan.

<img width="1149" height="630" alt="image" src="https://github.com/user-attachments/assets/b6414ce5-fc8c-447c-bf22-7a51aef4cc0d" />

## Infrastructure as Code (in progress) [Pray so I don't leak the Terraform token :D]
I'm trying to do things properly now: no more clicking and typing commands I forget, everything in code and in Git.
- Made a dedicated `terraform@pve` user and API token in Proxmox (not root, in case I leak it).
- Set up a Terraform project with the `bpg/proxmox` provider, with a `.gitignore` so the token and the state file never touch GitHub.
- First error: "the API token must be in the format USER@REALM!TOKENID=UUID". My token had a blank space at the end, fixed (peak plataforming) :D
- Fixed it, ran `terraform apply`, and it found my node (`home`). A tiny victory, but it means Terraform and Proxmox are talking.
- Next up: a Terraform-built LXC container with Docker inside (not on the host itself, because Docker messes with iptables and I have a lot of hand-written iptables rules I'd like to keep).
- Learned I need an SSH key first (I didn't have one on any machine, oops).

## Roadmap
- [x] Docker LXC container created with Terraform
<img width="1090" height="458" alt="image" src="https://github.com/user-attachments/assets/77c1ccf9-fc8f-4a16-a43a-ba691e993f09" />

- [ ] Ansible to configure it (install Docker, harden SSH, updates)
- [ ] Put everything in Git with a proper README and diagram
- [ ] Import the hand-made Pi-hole container into Terraform
- [ ] Monitoring (Uptime Kuma, then Grafana/Prometheus)
- [ ] Reverse proxy and local names instead of IP:port
- [ ] Tailscale for remote access
- [ ] Buy a USB Ethernet adapter and a cable (finally)

## Lessons learned
- Read the error message, then read it again.
- A placeholder is not a value.
- Write things down while you do them, not afterwards.
