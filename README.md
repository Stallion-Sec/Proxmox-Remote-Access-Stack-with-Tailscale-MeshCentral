# Proxmox Remote-Access Stack with Tailscale + MeshCentral

**Quick summary:** I built a secure, travel-friendly remote access stack for my Proxmox environment.  
It uses **Tailscale** as a private network overlay, **MeshCentral** for browser remote desktop, and **SPICE / QEMU Guest Agent** where I needed a rich local console. This README documents the goals, architecture, required commands, step-by-step setup, verification steps, and troubleshooting tips.



                                                    ┌────────────────────────────┐
                                                    │     Windows Laptop         │
                                                    │   (Your main local device) │
                                                    │       • Tailscale          │
                                                    └──────────────┬─────────────┘
                                                                   │
                                                                   │  Encrypted Mesh (Tailnet)
                                                                   ▼
                                                   ┌────────────────────────────┐
                                                   │        Proxmox Host        │
                                                   │   • Runs Tailscale agent   │
                                                   │   • Provides SPICE console │
                                                   │   • Manages VMs            │
                                                   └──────────────┬─────────────┘
                                                                  │
                                                                  │
                                           ┌──────────────────────┼─────────────────────────────┐
                                           │                      │                             │
                                           ▼                      ▼                             ▼
                                   
                                   ┌────────────────┐   ┌───────────────────┐     ┌──────────────────────────┐
                                   │ Ubuntu Desktop │   │ Ubuntu Server     │     │ Other Proxmox VMs        │
                                   │ VM             │   │ (CLI only)        │     │ (Security labs, tools,   │
                                   │ • Docker infra │   │ • MeshCentral     │     │ experiments, etc.)       │
                                   │ • Mesh Agent   │   │ • Node.js runtime │     │ • Each running Tailscale │
                                   │ • Tailscale    │   │ • Portainer       │     └──────────────────────────┘
                                   │ • SPICE        │   │ • Tailscale       │
                                   └────────────────┘   └───────────────────┘



---

## 🚩 TL;DR (for readers who want the result)
- Remote into your Proxmox and VMs from anywhere using a stable `100.x.x.x` Tailscale address.  
- Control desktops from a browser using MeshCentral (agent installed on VMs).  
- Use SPICE + QEMU Guest Agent for high-quality console access and clipboard support when needed.  
- No public firewall ports required; MagicDNS from Tailscale gives friendly hostnames like `proxmox.YOUR-TAILNET.ts.net`.

---

## 📚 Why I built this
I wanted a reliable, secure way to operate my home lab while traveling not just SSH, but full desktop access (copy/paste, file transfer) without punching holes in my router or exposing public IPs.

---

## 🏗 Architecture (high level)


Your Laptop (Anywhere)
↕ (encrypted peer-to-peer)
Tailscale overlay network (100.x.x.x)
↕
Proxmox Host (Tailscale client) ──> VMs (MeshCentral agents + QEMU guest agent)
└─> SPICE console for local high-fidelity access


---

## 🔌 Components
- **Proxmox VE** — hypervisor running VMs.
- **Tailscale** — WireGuard-based overlay network for private connectivity & MagicDNS.
- **MeshCentral** — browser-based remote desktop / agent manager (installed on a VM).
- **QEMU Guest Agent** — improves guest-host integration (clipboard, clean shutdown).
- **SPICE + virt-viewer** — optional high-quality console for local/advanced use.

---

## ✅ Prerequisites
- Proxmox host with ssh access.
- Ubuntu server to host MeshCentral (or use an existing server).
- A Tailscale account (free tier is fine).
- Local machine (Windows/macOS/Linux) to open `.vv` SPICE files (virt-viewer / Remote Viewer).

---

## 🔧 Setup — step-by-step

> Each major section includes commands. For clarity: run commands on the Proxmox host (SSH) or inside the VM depending on the note.

---

### A — Proxmox: basic checks & enable QEMU guest agent
1. SSH into Proxmox:

       ssh root@<tailscale-or-local-ip>


Make sure QEMU guest agent is enabled in the VM config (Proxmox UI → VM → Options → QEMU Guest Agent = Yes) and installed inside the VM:

Inside Ubuntu VM:

    sudo apt update
    sudo apt install qemu-guest-agent -y
    sudo systemctl enable --now qemu-guest-agent
        

Why: Guest agent improves console features and helps SPICE clipboard behave.

### B — Install Tailscale

On Proxmox host (Debian/Proxmox)

    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up
# Follow the URL printed to authenticate (use your Tailscale account)


On any Ubuntu VM you want reachable (e.g., MeshCentral host):

    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up


Notes:

After ```tailscale up```, check the admin panel: https://login.tailscale.com/admin/machines

<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.pngtai.png" width="800">
</p>

MagicDNS (optional) can be enabled from the admin console, no paid domain required.

### C — MeshCentral (on a dedicated Ubuntu VM)

Create a directory (recommended place):

    sudo mkdir -p /opt/meshcentral
    cd /opt/meshcentral

<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.pngC.png" width="800">
</p>


Install Node.js + MeshCentral (example commands; use distro packages if preferred):

# install Node (if not present)
    curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
    sudo apt install -y nodejs

<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png1.png" width="800">
</p>

# install meshcentral globally
    sudo npm install -g meshcentral

<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png2.png" width="700">
</p>


Initialize by running:

    sudo node /usr/lib/node_modules/meshcentral/meshcentral.js

<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png%203.png" width="700">
</p>


Stop it after it creates meshcentral-data folders and config.json. Then create a systemd service:

<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png4.png" width="700">
</p>


#### Confirm and configure meshcentral.service by running:

       sudo nano /etc/systemd/system/meshcentral.service

--
    
    [Unit]
    Description=MeshCentral Server
    After=network.target
    
    [Service]
    Type=simple
    User=root
    WorkingDirectory=/opt/meshcentral
    ExecStart=/usr/bin/node /usr/lib/node_modules/meshcentral/meshcentral.js
    Restart=always
    
    [Install]
    WantedBy=multi-user.target


Enable and start:

    sudo systemctl daemon-reload
    sudo systemctl enable --now meshcentral
    sudo systemctl status meshcentral

<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png6.png" width="800">
</p>


If MeshCentral reports no config.json, run node once manually as above, let it generate files, then use ```systemd```

Add an admin user via MeshCentral web UI on first run. Access via:

#### https://(tailscale-ip for mesh-server)


<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png7.png" width="850">
</p>


(You’ll get self-signed certs by default; accept or use Let’s Encrypt if public domain used.)

### D — Add MeshCentral agent to VMs

From MeshCentral web UI → Add Device → copy the provided install script (it looks like wget https://'mesh-host'/meshagents?script=1 ...) and run it on the target VM. Example:

      (wget "https://<mesh-ip>/meshagents?script=1" --no-check-certificate -O ./meshinstall.sh || \
       wget "https://<mesh-ip>/meshagents?script=1" --no-proxy --no-check-certificate -O ./meshinstall.sh) \
       && chmod 755 ./meshinstall.sh && sudo -E ./meshinstall.sh https://<mesh-ip> '<AGENTKEY>'

<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png8.png" width="700">
</p>
<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png9.png" width="700">
</p>


Verify the agent shows Connected in MeshCentral. At this point, the meshcentral creats a device access to the VM through the agent

<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png10.png" width="45%" />
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png11.png" width="45%" />
</p>



#### VM is ready to be accessed remotely through Meshcentral Desktop

<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png12.png" width="700">
</p>


#### Meshcentral also gives direct and seemless access to the VM files

<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png13.png" width="700">
</p>



## Overview on completion

<p align="center">
  <img src="https://github.com/Stallion-Sec/Proxmox-Remote-Access-Stack-Tailscale-MeshCentral-SPICE/blob/images/image.png14.png" width="1000">
</p>




### E — SPICE setup (optional, for high-fidelity console)

#### Proxmox-side

* Shut down VM.

* Hardware → Display → Edit: set Graphic Card to VirtIO-GPU and Display to SPICE.

* Save and boot VM.

#### Local-side (Windows)

* Install Remote Viewer / virt-viewer MSI.

* From Proxmox console → choose SPICE → download pvespice.vv → open with Remote Viewer.

* Inside VM

* Force Xorg (if using GNOME) to prevent Wayland issues:

      sudo nano /etc/gdm3/custom.conf
      # set WaylandEnable=false
      sudo reboot


* Test clipboard copy/paste both directions.

## 🔍 Verification checklist (what I tested)

- ssh root@tailscale ip' works from remote networks.

- https://tailscale-ip:8006 (Proxmox UI) reachable via Tailscale.

- MeshCentral web UI reachable via Tailscale new ip.

- MeshCentral agents appear connected.

- SPICE opens via virt-viewer and clipboard works both ways.

- Tailscale shows Proxmox online even when local LAN IP changes.


## 📝 Tailscale discovery (Discovery Note)

While testing remote connectivity I discovered that accessing Proxmox via its Tailscale address (100.x.x.x) is more reliable than relying on the host’s local LAN IP when you are off-site. The Tailscale address remains stable regardless of local network changes, enabling consistent remote access.

## 🚨 Common troubleshooting (quick)

* Blank SPICE screen: ensure VM uses VirtIO-GPU + guest uses Xorg (disable Wayland).

* MeshCentral environment unreachable: check container/agent type selected (use Docker option for Docker environments).

* Port 8006 not responding: restart services on Proxmox:

      sudo systemctl restart pveproxy pvedaemon pvestatd
      ss -tulpn | grep 8006


* Tailscale connected but no SSH: ensure tailscale up ran successfully on host and UFW/firewall allows connections from Tailscale (usually not necessary).

MeshCentral agent cert warnings: use --no-check-certificate for internal testing or import the generated CA certificate.

## 🔐 Security notes

MeshCentral’s default uses self-signed certs; for production use get proper TLS (Let’s Encrypt) or use Tailscale’s identity/MagicDNS.

Keep MeshCentral and Node.js up to date; run as non-root if you can.

Don’t publish Tailscale IPs/screenshots publicly (I removed mine from screenshots). Use masked IPs in public posts.



---

Credits

Built and documented by Joshua; Home lab, Proxmox, MeshCentral, Tailscale.


---
