# Docker OpenVPN Setup on AWS EC2

This repository contains the setup for running OpenVPN in a Docker container on an AWS EC2 instance. This guide helps in setting up a secure VPN that can be used to access internal resources like SSH on your AWS servers.

## Prerequisites
- AWS EC2 instance (This guide uses Amazon Linux 2)
- Docker installed on the EC2 instance
- Basic knowledge of SSH and networking

## Setup Instructions

### Step 1: Install Docker on AWS EC2

1. **Update the package list and install Docker**:
    ```sh
    sudo yum update -y
    sudo amazon-linux-extras install docker -y
    sudo systemctl start docker
    sudo systemctl enable docker
    sudo usermod -aG docker ec2-user
    ```

2. **Log out and log back in** (or use `newgrp docker`) to apply the group change.

3. **Verify Docker installation**:
    ```sh
    docker --version
    ```

### Step 2: Pull the OpenVPN Docker Image

1. **Pull the OpenVPN Docker image**:
    ```sh
    docker pull kylemanna/openvpn
    ```

### Step 3: OpenVPN Configuration

1. **Create a directory to store OpenVPN configuration**:
    ```sh
    mkdir -p ~/openvpn-data
    ```

2. **Initialize OpenVPN configuration**:
    Replace `<PUBLIC_IP>` with your EC2's public IP address.
    ```sh
    docker run -v ~/openvpn-data:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://<PUBLIC_IP>
    ```

3. **Initialize the Public Key Infrastructure (PKI)** and set up the CA:
    ```sh
    docker run -v ~/openvpn-data:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
    ```

    You'll be prompted to create a CA key passphrase. Choose a strong passphrase and remember it.

### Step 4: Start the OpenVPN Server

1. **Run the OpenVPN server in a detached mode**:
    ```sh
    docker run -v ~/openvpn-data:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn
    ```

2. **Check the status of the running container**:
    ```sh
    docker ps
    ```

### Step 5: Generate Client Configuration

1. **Generate a client certificate**:
    ```sh
    docker run -v ~/openvpn-data:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full client1 nopass
    ```

2. **Export the client configuration to an `.ovpn` file**:
    ```sh
    docker run -v ~/openvpn-data:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient client1 > client1.ovpn
    ```

3. **Transfer the `.ovpn` file to your local machine** using SCP:
    ```sh
    scp -i /path/to/key.pem ec2-user@<PUBLIC_IP>:/home/ec2-user/client1.ovpn /path/to/local/machine
    ```

### Step 6: Set Up OpenVPN Client

1. **Install OpenVPN Client** on your machine (PC, Mac, or Mobile).
2. **Import the `.ovpn` file** you transferred from the server.
3. **Connect** to the VPN.

### Step 7: Secure SSH Access via VPN

1. **Close SSH access (Port 22) to the public** and restrict it to the VPN.
    - Modify the security group in AWS to close **port 22** from `0.0.0.0/0` and only allow access through the VPN.
2. **Connect to the server using SSH** via the VPN:
    ```sh
    ssh -i /path/to/key.pem ec2-user@<PRIVATE_IP>
    ```

    Use the **private IP** of your EC2 instance, which is accessible only through the VPN.

### Troubleshooting

#### IP Forwarding
Ensure IP forwarding is enabled on your EC2 instance for OpenVPN to route traffic correctly:
```sh
sudo sysctl -w net.ipv4.ip_forward=1
```

#### Make it permanent by adding it to /etc/sysctl.conf:
```sh
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
```
This will ensure that IP forwarding remains active even after the instance is rebooted.


#### Checking Docker Logs
To check the logs of your OpenVPN Docker container:
```sh
docker logs <CONTAINER_ID>
```

#### Common Issues
1. **Connection refused when trying SSH**: Make sure port 22 is only accessible through the VPN by modifying the AWS Security Group settings.
2. **Can't connect to VPN**: Ensure that port 1194/UDP is open in the AWS Security Group.






