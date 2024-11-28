<!-- README.md -->

# WireGuard Setup Guide for Arch Linux

This guide provides a streamlined, step-by-step process to set up a secure WireGuard VPN on Arch Linux. It ensures proper configuration of public and private keys to avoid common issues related to authentication and traffic routing.

[Русская версия](README-ru.md)
## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Key Generation](#key-generation)
4. [Server Configuration](#server-configuration)
5. [Client Configuration](#client-configuration)
6. [Firewall and Routing](#firewall-and-routing)
7. [Starting WireGuard](#starting-wireguard)
8. [Verification](#verification)
9. [Troubleshooting](#troubleshooting)

## Prerequisites

- **Arch Linux** installed on both server and client machines.
- **Root** or **sudo** privileges on both machines.
- **Public IP** address for the server.

## Installation

### On Server and Client

1. **Update the system:**

    ```bash
    sudo pacman -Syu
    ```

2. **Install WireGuard:**

    ```bash
    sudo pacman -S wireguard-tools
    ```

3. **Install Nano Editor (Optional but Recommended):**

    Nano is a user-friendly text editor that simplifies editing configuration files.

    ```bash
    sudo pacman -S nano
    ```

## Key Generation

### On Server

1. **Navigate to WireGuard directory:**

    ```bash
    sudo mkdir -p /etc/wireguard
    cd /etc/wireguard
    ```

2. **Generate server keys:**

    ```bash
    umask 077
    wg genkey | tee server_privatekey | wg pubkey > server_publickey
    ```

    - `server_privatekey`: Server's private key.
    - `server_publickey`: Server's public key.

### On Client

1. **Generate client keys:**

    ```bash
    wg genkey | tee client_privatekey | wg pubkey > client_publickey
    ```

    - `client_privatekey`: Client's private key.
    - `client_publickey`: Client's public key.

## Server Configuration

1. **Create/Edit WireGuard configuration:**

    ```bash
    sudo nano /etc/wireguard/wg0.conf
    ```

2. **Add the following configuration:**

    ```ini
    [Interface]
    Address = 10.0.0.1/24
    ListenPort = 51820
    PrivateKey = <server_privatekey>

    # Enable IP forwarding and NAT
    PostUp = sysctl -w net.ipv4.ip_forward=1
    PostUp = iptables -t nat -A POSTROUTING -o <external_interface> -j MASQUERADE
    PostUp = iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
    PostUp = iptables -A FORWARD -s 10.0.0.0/24 -j ACCEPT
    PostDown = sysctl -w net.ipv4.ip_forward=0
    PostDown = iptables -t nat -D POSTROUTING -o <external_interface> -j MASQUERADE
    PostDown = iptables -D FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
    PostDown = iptables -D FORWARD -s 10.0.0.0/24 -j ACCEPT

    [Peer]
    PublicKey = <client_publickey>
    AllowedIPs = 10.0.0.2/32
    ```

    - Replace `<server_privatekey>` with the contents of `server_privatekey`.
    - Replace `<external_interface>` with your server's external network interface (e.g., `ens1`, `eth0`).
    - Replace `<client_publickey>` with the client's public key.

3. **Save and exit** (`Ctrl + O`, `Enter`, `Ctrl + X`).

## Client Configuration

1. **Create/Edit WireGuard configuration:**

    ```bash
    sudo nano /etc/wireguard/wg0.conf
    ```

    *On Windows, use the WireGuard application to add a new tunnel and input the configuration.*

2. **Add the following configuration:**

    ```ini
    [Interface]
    PrivateKey = <client_privatekey>
    Address = 10.0.0.2/24
    DNS = 8.8.8.8

    [Peer]
    PublicKey = <server_publickey>
    Endpoint = <server_public_ip>:51820
    AllowedIPs = 0.0.0.0/0, ::/0
    PersistentKeepalive = 25
    ```

    - Replace `<client_privatekey>` with the contents of `client_privatekey`.
    - Replace `<server_publickey>` with the server's public key.
    - Replace `<server_public_ip>` with your server's public IP address.

3. **Save and exit** (`Ctrl + O`, `Enter`, `Ctrl + X`).

## Firewall and Routing

### On Server

1. **Configure iptables rules:**

    ```bash
    sudo iptables -t nat -A POSTROUTING -o <external_interface> -j MASQUERADE
    sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
    sudo iptables -A FORWARD -s 10.0.0.0/24 -j ACCEPT
    ```

2. **Save iptables rules for persistence:**

    ```bash
    sudo iptables-save | sudo tee /etc/iptables/iptables.rules
    sudo systemctl enable iptables
    sudo systemctl start iptables
    ```

3. **Enable IP forwarding:**

    ```bash
    echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/99-sysctl.conf
    sudo sysctl -p /etc/sysctl.d/99-sysctl.conf
    ```

## Starting WireGuard

### On Server and Client

1. **Start and enable WireGuard:**

    ```bash
    sudo systemctl start wg-quick@wg0
    sudo systemctl enable wg-quick@wg0
    ```

## Verification

1. **Check WireGuard status:**

    ```bash
    sudo wg show
    ```

    - Ensure `wg0` is active with peers listed.

2. **Test Connectivity:**

    - **Ping Server from Client:**

        ```bash
        ping 10.0.0.1
        ```

    - **Ping External IP from Client:**

        ```bash
        ping 8.8.8.8
        ```

    - **Test DNS Resolution:**

        ```bash
        nslookup google.com
        ```

    - **Access Websites:**
      
      Open a web browser and navigate to any website (e.g., [https://www.google.com](https://www.google.com)).

## Troubleshooting

- **Incorrect Key Pairing:**
  
  - Ensure the server's `[Peer]` has the **client's public key**.
  - Ensure the client's `[Peer]` has the **server's public key**.

- **Firewall Rules:**
  
  - Verify iptables rules:

    ```bash
    sudo iptables -L -v
    sudo iptables -t nat -L -v
    ```

- **IP Forwarding:**
  
  - Confirm IP forwarding is enabled:

    ```bash
    sysctl net.ipv4.ip_forward
    ```

    Should return `net.ipv4.ip_forward = 1`.

- **Logs Review:**
  
  - Check WireGuard logs on the server:

    ```bash
    sudo journalctl -u wg-quick@wg0
    ```

- **Port Accessibility:**
  
  - Ensure UDP port `51820` is open and listening:

    ```bash
    sudo ss -ulnp | grep 51820
    ```

- **DNS Issues:**
  
  - If DNS resolution fails, try different DNS servers (e.g., `1.1.1.1`, `8.8.4.4`).

## Common Issues and Solutions

### Cause: Misconfigured Public Keys

**Issue:** Client was using the server's private key as the peer's public key, preventing proper authentication.

**Solution:**
- Ensure the client's `[Peer] PublicKey` is set to the **server's public key**.
- Ensure the server's `[Peer] PublicKey` is set to the **client's public key**.

### Cause: Duplicate iptables Rules

**Issue:** Multiple identical `MASQUERADE` rules caused routing conflicts.

**Solution:**
- Remove duplicate iptables rules and retain only one `MASQUERADE` rule.
  
    ```bash
    sudo iptables -t nat -F POSTROUTING
    sudo iptables -t nat -A POSTROUTING -o <external_interface> -j MASQUERADE
    ```

### Cause: Disabled IP Forwarding

**Issue:** IP forwarding was not enabled, blocking traffic routing through VPN.

**Solution:**
- Enable IP forwarding permanently.
  
    ```bash
    echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/99-sysctl.conf
    sudo sysctl -p /etc/sysctl.d/99-sysctl.conf
    ```

## Conclusion

Proper configuration of public and private keys, along with correct firewall and routing settings, is crucial for a functional WireGuard VPN on Arch Linux. By following this guide, you can set up WireGuard securely and efficiently, minimizing potential issues related to authentication and traffic routing.

For further assistance, refer to the [WireGuard Documentation](https://www.wireguard.com/#documentation) or seek help from the Arch Linux community.
