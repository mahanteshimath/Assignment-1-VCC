# Assignment-1-VCC

## Assignment Objective:
Create and configure multiple Virtual Machines (VMs) using VirtualBox, establish a network between them, and deploy a microservice-based application across the connected VMs.

## Deliverables:

1. **Document Report:**
    - Step-by-Step Instructions for Implementation:
        - Installation of VirtualBox and creation of multiple VMs.
        - Configuration of network settings to connect the VMs.
        - Deployment of a simple microservice application (e.g., a RESTful API or a Node.js-based service) across the VMs.

2. **Architecture Design:**
    - Diagram showing the connection of VMs and their roles in hosting the microservice application.
    - Detailed steps

## 2. Phase 1: Environment Setup

### 2.1 Download Prerequisites

Ensure you have the following software downloaded:

- **Oracle VirtualBox**: for Windoows https://www.virtualbox.org/wiki/Downloads
- **Ubuntu ISO**: Desktop ISO image (Version 24.04 LTS is recommended) https://ubuntu.com/download/desktop

### 2.2 Create Virtual Machines

You must create three separate Virtual Machines:

- **MH-VM1**: Application Server 1
- **MH-VM2**: Application Server 2
- **MH-VM3**: Load Balancer

**Steps for each VM:**

1. Open VirtualBox and click the **New** button.
2. **Name**: Enter a unique name (e.g., "Microservice MH-VM1").
3. **ISO Image**: Select the downloaded Ubuntu ISO file.
4. **Hardware**: Allocate at least 2GB of RAM and 2 CPUs.
5. **Hard Disk**: The default 25GB is sufficient.
6. **Finish**: Click Finish to create the VM.
7. **Install OS**: Start the VM and follow the on-screen prompts to install Ubuntu.
8. **Repeat**: Perform these steps again to create VM2 and VM3.

## 3. Phase 2: Networking & Basic Configuration

### 3.1 Verify Connectivity

1. Open the terminal in MH-VM1 and MH-VM2.




2. Find the IP address of each machine using the following command:

```bash
ip a
```

# MH-VM1 IP Address 
![alt text](image-1.png)

# MH-VM2 IP Address 
![alt text](image.png)
3. Note down the IP addresses (e.g., 10.0.2.4 for VM1 and 10.0.2.5 for VM2).

### 3.2 Test Connection

 VMs can communicate with each other, I tried ping command from MH-VM2 to MH-VM1:

```bash
ping 10.0.2.15
```
![alt text](image-2.png)

If you see bytes being transferred, the connection is successful.

### 3.2.1 Fix Same IP Address Issue (If VMs Have Same IP)

If both VMs show the same IP address (e.g., 10.0.2.15), you need to configure a Host-Only Network to allow proper communication between VMs.

**Steps to Configure Host-Only Network:**

1. **Create a Host-Only Network in VirtualBox:**
   - Open VirtualBox Manager
   - Go to **Tools** → **Network Manager**
   - Select **Host-only Networks** tab
   - Click **Create** to add a new Host-Only Network
   - Note the network details (e.g., 192.168.56.0/24)
   - Ensure DHCP Server is enabled or configure manually

2. **Configure Network Adapter for Each VM:**
   - Shut down both VMs
   - Right-click **MH-VM1** → **Settings** → **Network**
   - **Adapter 1**: Change "Attached to" to **Host-Only Adapter**
   - Select the Host-Only network you created
   - **Adapter 2** (Optional): Set to **NAT** for internet access
   - Click OK
   - Repeat the same steps for **MH-VM2**

3. **Assign Static IP Addresses Inside Each VM:**
   
   Boot each VM and open terminal, then configure static IPs:

   **For MH-VM1:**
   ```bash
   sudo nano /etc/netplan/01-netcfg.yaml
   ```
   
   Add the following configuration:
   ```yaml
   network:
     version: 2
     ethernets:
       enp0s3:
         dhcp4: no
         addresses: [192.168.56.10/24]
         nameservers:
           addresses: [8.8.8.8, 8.8.4.4]
   ```

   **For MH-VM2:**
   ```bash
   sudo nano /etc/netplan/01-netcfg.yaml
   ```
   
   Add the following configuration:
   ```yaml
   network:
     version: 2
     ethernets:
       enp0s3:
         dhcp4: no
         addresses: [192.168.56.11/24]
         nameservers:
           addresses: [8.8.8.8, 8.8.4.4]
   ```

4. **Apply the Network Configuration:**
   ```bash
   sudo netplan apply
   ```

5. **Verify the New IP Addresses:**
   ```bash
   ip a
   ```
   
   MH-VM1 should show: 192.168.56.10  
   MH-VM2 should show: 192.168.56.11

6. **Test Connectivity Between VMs:**
   
   From MH-VM1, ping MH-VM2:
   ```bash
   ping 192.168.56.11
   ```
   
   From MH-VM2, ping MH-VM1:
   ```bash
   ping 192.168.56.10
   ```

   ![alt text](image-3.png)

**Note**: Update all IP addresses in subsequent steps to use the new static IPs (192.168.56.10 and 192.168.56.11).

### 3.3 Install Dependencies (VM1 & VM2)

Update the package manager and install Node.js and npm on both application VMs (VM1 and VM2):

```bash
sudo apt update
sudo apt install -y nodejs npm
```

## 4. Phase 3: Microservice Development (VM1)

### 4.1 Initialize Project

Create a new directory for your service:

```bash
mkdir new-service && cd new-service
```

Initialize the Node.js project:

```bash
npm init -y
```

Install the Express framework:

```bash
npm install express
```

### 4.2 Create Application Code

Create a file named `index.js` using a text editor (like nano or vi) and paste the following code:

```javascript
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
    res.json({ message: 'Hello from Ubuntu VM' });
});

app.listen(port, '0.0.0.0', () => {
    console.log(`Server running on port ${port}`);
});
```

### 4.3 Test Locally

Start the server:

```bash
node index.js
```

Open a terminal in VM2 and test the connection:

```bash
curl http://<VM1_IP>:3000
```

You should receive a JSON response: `{"message": "Hello from Ubuntu VM"}`.

## 5. Phase 4: Containerization with Docker

### 5.1 Install Docker (VM1 & VM2)

Run the following commands on both VM1 and VM2 to install and enable Docker:

```bash
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

### 5.2 Create Dockerfile (VM1)

Inside the `new-service` directory on VM1, create a file named `Dockerfile` with the following content:

```dockerfile
FROM node:18
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

### 5.3 Build & Run Image (VM1)

Build the Docker image:

```bash
sudo docker build -t new-service .
```

Run the container in detached mode:

```bash
sudo docker run -d -p 3000:3000 new-service
```

Verify it is running:

```bash
sudo docker ps
```

## 6. Phase 5: Deployment via Docker Hub

### 6.1 Push Image (VM1)

Log in to Docker Hub (create an account at hub.docker.com if you haven't already):

```bash
sudo docker login
```

Tag your image:

```bash
sudo docker tag new-service <your_dockerhub_username>/new-service:latest
```

Push the image to the registry:

```bash
sudo docker push <your_dockerhub_username>/new-service:latest
```

### 6.2 Pull & Run (VM2)

Switch to VM2 and deploy the image:

Pull the image from Docker Hub:

```bash
sudo docker pull <your_dockerhub_username>/new-service:latest
```

Run the container:

```bash
sudo docker run -d -p 3000:3000 <your_dockerhub_username>/new-service:latest
```

## 7. Phase 6: Load Balancer Setup (VM3)

### 7.1 Install Nginx

On the third VM (VM3), install Nginx:

```bash
sudo apt update
sudo apt install -y nginx
```

### 7.2 Configure Load Balancing

Edit the Nginx configuration file (typically `/etc/nginx/nginx.conf` or `/etc/nginx/sites-available/default`). Add or modify the `http` block to include the upstream configuration:

```nginx
http {
    upstream backend_cluster {
        # Replace with your actual IPs
        server 10.0.2.4:3000; 
        server 10.0.2.5:3000;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://backend_cluster;
        }
    }
}
```

### 7.3 Restart Nginx

Validate the configuration and restart the service:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

## 8. Final Verification

To verify that the load balancer is working and distributing traffic between VM1 and VM2:

1. Find the IP address of VM3 (Load Balancer).
2. Run the following loop command from your host machine or another terminal:

```bash
while true; do curl http://<VM3_IP>; echo; sleep 1; done
```

**Result**: Nginx will distribute these requests between VM1 and VM2 (often using a Round Robin algorithm by default).
