# MikroTikApplePolicyRoutingProtonVPN
## Step 1: Extract WireGuard Configuration from ProtonVPN
Before configuring the router, you need your ProtonVPN connection details:
1. Log in to your account at [account.protonvpn.com](https://account.protonvpn.com).
2. Navigate to **Downloads** from the left menu and select **WireGuard configuration**.
3. Create a new configuration:
   * **Name:** Choose a name (e.g., MikroTik).
   * **Platform:** Select **Router**.
   * Choose your desired Server (e.g., NL or US).
4. Click **Create**, then download or view the file contents.
5. Save the following details for the next steps:
   * `Private Key`
   * `Public Key`
   * `Endpoint IP`
   * `Endpoint Port` (usually 51820)
   * `Address` (the internal IP, e.g., `10.2.0.2/24`).

---

## Step 2: Create and Configure WireGuard in MikroTik
Now, let's add the WireGuard interface and connect it to the ProtonVPN server.

**1. Create the Interface:**
Replace `YOUR_PRIVATE_KEY` with the private key you obtained.

```

```text
File generated successfully.

```routeros
/interface wireguard
add comment="WireGuard_Proton Interface" listen-port=51820 mtu=1420 name=WireGuard_Proton private-key="YOUR_PRIVATE_KEY"

```

**2. Assign IP Address to Interface:**
(Ensure you use the IP provided in your Proton config. Based on your script, we assume `10.2.0.2/24`).

```routeros
/ip address
add address=10.2.0.2/24 comment=ProtonVPN interface=WireGuard_Proton network=10.2.0.0

```

**3. Add the Server (Peer):**
Replace `YOUR_PUBLIC_KEY` and `ENDPOINT_IP` with your details.

```routeros
/interface wireguard peers
add allowed-address=0.0.0.0/0 client-address=10.2.0.2/24 endpoint-address=ENDPOINT_IP endpoint-port=51820 interface=WireGuard_Proton name=Proton_Server persistent-keepalive=25s public-key="YOUR_PUBLIC_KEY"

```

---

## Step 3: Create the Address List for Apple Traffic

Based on your script, we have extracted only the Apple-related addresses and grouped them under the name `VPN_Apps`.

```routeros
/ip firewall address-list
add address=apple.com list=VPN_Apps
add address=icloud.com list=VPN_Apps
add address=itunes.apple.com list=VPN_Apps
add address=mzstatic.com list=VPN_Apps
add address=apple-cloudkit.com list=VPN_Apps
add address=apple-dns.net list=VPN_Apps
add address=courier.push.apple.com list=VPN_Apps
add address=app-storeos.apple.com list=VPN_Apps
add address=swcdn.apple.com comment="Apple Updates" list=VPN_Apps
add address=icloud-content.com comment="iCloud Content" list=VPN_Apps
add address=gsa.apple.com comment="Apple Auth" list=VPN_Apps
add address=captive.apple.com comment="Apple WiFi Check" list=VPN_Apps
add address=iosapps.itunes.apple.com comment="App Store iOS" list=VPN_Apps
add address=me.com comment="Apple ME" list=VPN_Apps

```

---

## Step 4: Configure Routing and NAT

To ensure the routed internet traffic works correctly over the VPN, we must configure NAT and a new routing table.

**1. Add a New Routing Table:**

```routeros
/routing table
add comment=ProtonVPN fib name=to_proton

```

**2. Add a Default Route for the VPN:**

```routeros
/ip route
add distance=1 dst-address=0.0.0.0/0 gateway=WireGuard_Proton routing-table=to_proton

```

**3. Configure NAT for WireGuard:**
This is required for local devices to access the internet through the VPN interface.

```routeros
/ip firewall nat
add action=masquerade chain=srcnat comment="ProtonNAT" out-interface=WireGuard_Proton

```

---

## Step 5: Setup Mangle Rules for Traffic Routing

Finally, we will capture any LAN (ether5) traffic destined for the `VPN_Apps` list, mark it, and route it through `WireGuard_Proton`.

**1. Mark the Connection:**

```routeros
/ip firewall mangle
add action=mark-connection chain=prerouting comment="Mark LAN_Conn To VPN for Apple" connection-mark=no-mark connection-state=new dst-address-list=VPN_Apps dst-address-type=!local in-interface=ether5 new-connection-mark=VPN_conn passthrough=yes

```

**2. Route the Marked Connection:**

```routeros
/ip firewall mangle
add action=mark-routing chain=prerouting comment="Route Apple VPN_Apps to Proton" connection-mark=VPN_conn in-interface=ether5 new-routing-mark=to_proton passthrough=no

```

**3. Adjust MSS Clamping (Crucial for WireGuard):**
To prevent slow loading times or stalled downloads over the VPN:

```routeros
/ip firewall mangle
add action=change-mss chain=forward comment="MSS Clamping for WireGuard" new-mss=clamp-to-pmtu out-interface=WireGuard_Proton protocol=tcp tcp-flags=syn

```

---

**Final Note:** Once you copy and paste these commands into the `New Terminal` in WinBox, all connections to the specified Apple services will be securely routed through your ProtonVPN tunnel, while the rest of your network traffic continues through your standard ISP connection.
