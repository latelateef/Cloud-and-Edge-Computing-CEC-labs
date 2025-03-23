# Lab 1: Private Cloud & VM Operations with OpenStack

This lab demonstrates the setup of a private cloud using OpenStack (via MicroStack) and the operation of virtual machines (VMs) to deploy applications. It includes two parts:

- **Part A**: Launch a VM, install a calculator, and make it accessible remotely.
- **Part B**: Deploy a web server, create a REST API endpoint, and build a web form to interact with the API.

## Prerequisites
- Ubuntu/Debian system (physical or VM)
- `MicroStack` installed and initialized

## Installation

### 1. Install and Initialize MicroStack
```bash
sudo snap install microstack --beta --devmode
sudo microstack init --auto --control
```

### 2. Access OpenStack Dashboard  
Retrieve the Keystone password:  
```bash
sudo snap get microstack config.credentials.keystone-password
```  
Access the OpenStack dashboard at using username as `admin` and `retrieved password`:  
```
https://10.20.20.1
```

## Troubleshooting & Maintenance

### 1. Check MicroStack Services Status
To verify the status of all MicroStack services:
```bash
sudo systemctl status snap.microstack.*
```

### 2. List MicroStack Services
To view all MicroStack-related services and their status:
```bash
sudo snap services microstack
```

### 3. Restart MicroStack
To restart all MicroStack services:
```bash
sudo snap restart microstack
```

### 4. Remove MicroStack
To completely remove MicroStack and its configurations:
```bash
sudo snap remove --purge microstack
```

## Creating a VM Instance

### 1. Generate SSH Keys  
On your local machine (Linux/macOS), run:  
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/microstack_key
```
- This creates:
  - **Public key:** `~/.ssh/microstack_key.pub`
  - **Private key:** `~/.ssh/microstack_key` (Keep this secure!)

### 2. Download Ubuntu Jammy Cloud Image  
If you haven't already, download the Ubuntu 22.04 (Jammy) cloud image:  
```bash
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
```

### 3. Modify the Image with SSH Setup  
Run the following command to customize the image:  
```bash
sudo virt-customize -a jammy-server-cloudimg-amd64.img \
  --run-command "mkdir -p /root/.ssh /home/ubuntu/.ssh" \
  --run-command "echo '' > /root/.ssh/authorized_keys" \
  --run-command "echo '' > /home/ubuntu/.ssh/authorized_keys" \
  --run-command "echo '$(cat ~/.ssh/microstack_key.pub)' >> /root/.ssh/authorized_keys" \
  --run-command "echo '$(cat ~/.ssh/microstack_key.pub)' >> /home/ubuntu/.ssh/authorized_keys" \
  --run-command "chmod 600 /root/.ssh/authorized_keys /home/ubuntu/.ssh/authorized_keys" \
  --run-command "sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config" \
  --run-command "systemctl restart ssh"
```

### 4. Upload the Customized Image to OpenStack
```bash
sudo microstack.openstack image create \
  --file jammy-server-cloudimg-amd64.img \
  --disk-format qcow2 --container-format bare \
  --public ubuntu
```
Verify the upload:
```bash
sudo microstack.openstack image list
```
You should see **ubuntu** in the output.

### 5. Create a Router
```bash
sudo microstack.openstack router create my-router
```

### 6. Verify or Create the Network
Check if a private network exists:
```bash
sudo microstack.openstack network list
```
If not, create it:
```bash
sudo microstack.openstack network create private
```
Also, create a subnet (if needed):
```bash
sudo microstack.openstack subnet create --network private --subnet-range <your_ipv4_add in 192.168.1.0/24 format> private
```

### 7. Connect the Internal Network to the Router
```bash
sudo microstack.openstack router add subnet my-router private
```

### 8. Set an External Gateway for the Router
Connect your router to an external network for external connectivity:
```bash
sudo microstack.openstack router set my-router --external-gateway external
```
The `external` network is usually preconfigured in MicroStack.

### 9. Verify the Setup
```bash
sudo microstack.openstack router show my-router
```

### 10. Add the Key to OpenStack
Upload the public key to OpenStack:
```bash
sudo microstack.openstack keypair create --public-key ~/.ssh/microstack_key.pub my-key
```
Verify the key is added:
```bash
sudo microstack.openstack keypair list
```

---

## **Part A: Calculator on a Remote-Accessible VM**

### **1. Create a New Instance**
Launch a VM using the uploaded image:
```bash
sudo microstack.openstack server create \
  --image ubuntu \
  --flavor m1.small \
  --network private \
  --key-name my-key \
  my-instance
```

### **2. Allocate a Floating IP**
Allocate a floating IP from the external network:
```bash
sudo microstack.openstack floating ip create external
```
This will return a floating IP address, e.g., `10.20.30.40`.

### **3. Associate the Floating IP with the Instance**
```bash
sudo microstack.openstack server add floating ip my-instance 10.20.30.40
```
Verify the Floating IP assignment:
```bash
sudo microstack.openstack server show my-instance
```
Ensure the floating IP is listed under `addresses`.

### **4. Connect to the VM**
Use SSH to connect to the instance:
```bash
ssh -i ~/.ssh/microstack_key ubuntu@10.20.30.40
```
If you face login issues, try:
```bash
ssh -i ~/.ssh/microstack_key root@10.20.30.40
```

### **5. Install a Command-Line Calculator**
```bash
sudo apt update && sudo apt install bc -y
```
---

### **6. Using the `bc` Calculator on the Remote VM**  
Once connected to the VM via SSH, simply run:  
```bash
bc -l
```
This opens the **bc** calculator in interactive mode. You can now perform calculations, e.g.:  
```bash
5 + 10
scale=2; 10/3   # Sets decimal precision to 2
sqrt(25)        # Square root of 25
```
To exit, type:  
```bash
quit
```
---
## **Part B: Web Server & REST API on Separate VMs**

### **1. Launch VMs for Web Server and REST API Server**
#### **Web Server VM**
```bash
sudo microstack.openstack server create \
  --image ubuntu \
  --flavor m1.small \
  --network private \
  --key-name my-key \
  webserver-vm
```

#### **REST API Server VM**
```bash
sudo microstack.openstack server create \
  --image ubuntu \
  --flavor m1.small \
  --network private \
  --key-name my-key \
  restserver-vm
```

### **2. Allocate and Assign Floating IPs**
#### **Web Server VM**
```bash
sudo microstack.openstack floating ip create external
```
```bash
sudo microstack.openstack server add floating ip webserver-vm <webserver-ip>
```

#### **REST API Server VM**
```bash
sudo microstack.openstack floating ip create external
```
```bash
sudo microstack.openstack server add floating ip restserver-vm <restserver-ip>
```

#### **REST API Server VM**
```bash
ssh -i ~/.ssh/microstack_key ubuntu@<restserver-ip>
```
---


## **Part C: Setting Up REST API Server**

### **1. Install Flask**
```bash
sudo apt install python3-flask -y
```

### **2. Create Flask REST API**
Save the following file as `echo.py`:
```python
from flask import Flask, request
app = Flask(__name__)

@app.route('/echo', methods=['GET'])
def echo():
    return request.args.get('msg', '')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### **3. Security Groups Are Blocking Port 5000**
Check if your instance’s security group allows port 5000:
```bash
sudo microstack.openstack security group rule list default
```
If TCP (port 5000) is missing, allow it:
```bash
sudo microstack.openstack security group rule create --protocol tcp --dst-port 5000 default
```

### **4. Start the Flask App**
```bash
python3 echo.py
```

### **5. Test the Echo Service**
```bash
curl http://localhost:5000/echo?msg=Hello
# Output: Hello
```

---
## **Part D: Setting Up Web Server**

### **1. Connect to the Web Server VM**
```bash
ssh -i ~/.ssh/microstack_key ubuntu@<webserver-ip>
```

### **2. Install Apache**
```bash
sudo apt install apache2 -y
sudo ufw allow 'Apache'
sudo systemctl start apache2 && sudo systemctl enable apache2
```

### **3. Create Web Form**
Save the following file as `index.html`:
```html
<!DOCTYPE html>
<html>
<body>
  <h1>Enter your Message:</h1>
  <!-- Point the form to Flask VM's IP:5000/echo -->
  <form action="http://<flask-vm-ip>:5000/echo" method="GET">
    <input type="text" name="msg" placeholder="Enter message">
    <input type="submit" value="Echo">
  </form>
</body>
</html>
```
Move `index.html` to `/var/www/html/index.html`:
```bash
mv index.html /var/www/html/index.html
```

### **4. Security Groups Are Blocking Port 80**
Check if your instance’s security group allows port 80:
```bash
sudo microstack.openstack security group rule list default
```
If TCP (port 80) is missing, allow it:
```bash
sudo microstack.openstack security group rule create --protocol tcp --dst-port 80 default
```

### **5. Testing:**

- Visit the url `http://<webserver-vm_floating_ip>/index.html` , you should see the form.
- Try sending a message and it should return the response.

---
