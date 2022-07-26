## PROGRAMING CHALLENGE

### [Description](#description)

### [Inventory](#inventory)

### [Logical Topology](#logical_topology)

### [Ansible Playbook Analysis](#ansible_playbook_analysis)

### [Challenges](#challenges)

### [Conclusion](#conclusion)

### **Description** <a name="description"></a>

---

With in this Readme, You will find the explanation of How Ansible - Docker - Azure elements are used to create a ansible managed container placed in Azure.

Brief process of this project is; Ansible Controller VM takes the Yml file and executes with ansible-playbook.

-   First server connects the Azure, creates the demanded App VM. Then registers its hostname and ip address to ansible/hosts file.

-   Later Ansible server connects to newly created App VM and installs Docker, Python3-pip, Docker SDK, Docker-compose and Git.

-   Then, App VM pulls the required app from github repository and modifies the app index file to writes app vm hostname and ip address.

-   Finally, App VM build the app with docker-compose and starts the app services.

Application can be accessed with this url:

[http://bsch-todo-app.germanywestcentral.cloudapp.azure.com:3000](http://bsch-todo-app.germanywestcentral.cloudapp.azure.com:3000)

### **Inventory** <a name="inventory"></a>

---

..

| Name            | Description           | Value | Image                  | Python |
| --------------- | --------------------- | ----- | ---------------------- | ------ |
| bsch-ansible01  | Ansible Controller VM | -     | Centos 8.5             | 3.9.13 |
| bsch-testhost0X | Created App VM        | -     | EuroLinux 9.0 (RedHat) | 3.9.10 |

..

**Application installed with the Playbook**

| Hostname        | Application        |
| --------------- | ------------------ |
| bsch-testhost0X | Docker Engine      |
|                 | Python3-Pip        |
|                 | Docker SDK         |
|                 | Docker-Compose Pip |
|                 | Docker-Compose     |
|                 | Git                |

### **Logical Topology** <a name="logical_topology"></a>

![Logical Topology](https://github.com/exbert/todo-app-deepdive/blob/master/img/ansible_project_topology.png "Logical Topology")

### **Ansible Playbook Analysis** <a name="ansible_playbook_analysis"></a>

-   Create Azure VM
    >Azure VM creation script starts
    -   Create Public IP address
        >Creating a dynamic public IP address.
        ```yaml
        - name: Create Public IP address
          azure_rm_publicipaddress:
            resource_group: "{{resourcegroup_name}}"
            allocation_method: Dynamic
            domain_name: "{{vm_tag}}-todo-app"
            name: "{{vm_name}}-pb-ip"
        ```
    -   Create Network Security Group that allows SSH - 3000
        >Creating NSG with allowing SSH - and tcp port 3000
        ```yaml
        - name: Create Network Security Group that allows SSH - 3000
          azure_rm_securitygroup:
            resource_group: "{{resourcegroup_name}}"
            name: "{{vm_name}}-sg"
            rules:
                - name: SSH
                  protocol: Tcp
                  destination_port_range: 22
                  access: Allow
                  priority: 1001
                  direction: Inbound
                - name: port_3000
                  protocol: Tcp
                  destination_port_range: 3000
                  access: Allow
                  priority: 1002
                  direction: Inbound
        ```
    -   Create Virtual Network Interface Card
        >Creating nic with the defined subnet, public ip and nsg
        ```yaml
        - name: Create Virtual Network Interface Card
          azure_rm_networkinterface:
            resource_group: "{{resourcegroup_name}}"
            name: "{{vm_name}}-nic"
            virtual_network: "{{virtual_network}}"
            subnet: "{{virtual_network_subnet}}"
            public_ip_name: "{{vm_name}}-pb-ip"
            security_group: "{{vm_name}}-sg"
        ```
    -   Create VM
        >Creating the Virtual Machine
        ```yaml
        - name: Create VM
          azure_rm_virtualmachine:
            resource_group: "{{resourcegroup_name}}"
            name: "{{vm_name}}"
            vm_size: Standard_DS1_v2
            admin_username: azureuser
            managed_disk_type: Standard_LRS
            ssh_password_enabled: false
            ssh_public_keys:
                - path: /home/azureuser/.ssh/authorized_keys
                  key_data: "{{lookup('file', '{{admin_pub_path_name}}') }}"
            network_interfaces: "{{vm_name}}-nic"
            image:
                offer: "eurolinux-9-0-free"
                publisher: eurolinuxspzoo1620639373013
                sku: "eurolinux-9-0-free"
                version: latest
            plan:
                name: "eurolinux-9-0-free"
                product: "eurolinux-9-0-free"
                publisher: "eurolinuxspzoo1620639373013"
        ```
    -   Get Public IP address
        > Gets the public IP address of VM because Public IP is selected Dynamic. And saves to a variable to be used to save to hosts file
        ```yaml
        - name: Get Public IP address
          azure_rm_publicipaddress_info:
            resource_group: "{{resourcegroup_name}}"
            name: "{{vm_name}}-pb-ip"
        register: output_ip_address
        ```
    -   Write / Check Hosts file for Dockerservers
        > Creates and checks the hosts file with for server group name
        ```yaml
        - name: Write / Check Hosts file for Dockerservers
          lineinfile:
            path: /etc/ansible/hosts
            line: "[dockerservers]"
            create: yes
        ```
    -   Entering Server info to hosts file
        > Writes the ip and hostname of the new created vm to ansible/hosts file
        ```yaml
        - name: Entering Server info to hosts file
          lineinfile:
            path: /etc/ansible/hosts
            insertafter: EOF
            line: "{{vm_name}} ansible_host={{(output_ip_address | json_query('publicipaddresses[].ip_address | [0]'))}}"
        ```
    -   Refresh inventory to ensure new hosts exist in hosts file
        > Initiate to recheck the ansible/hosts file
        ```yaml
        - name: Refresh inventory to ensure new hosts exist in hosts file
          meta: refresh_inventory
        ```
-   Docker Configuration and Creation
    > After VM creation now Ansible connects to hosts for Docker configuration ans application building.
    -   Creating a Docker Repository
        > Create a Docker Repository to pull the required docker packages
        ```yaml
        - name: Creating a Docker Repository
          yum_repository:
            description: Repo for Docker
            name: docker-ce
            baseurl: https://download.docker.com/linux/centos/9/x86_64/stable/
            gpgcheck: no
        ```
    -   Installing Docker
        ```yaml
        - name: Installing Docker
          remote_user: azureuser
          package:
            name: docker-ce
            state: present
        ```
    -   Starting Docker services
        ```yaml
        - name: Starting Docker services
          service:
            name: docker
            state: started
        ```
    -   Installing python3-pip
        ```yaml
        - name: Installing python3-pip
          package:
            name: python3-pip
            state: present
        ```
    -   Installing Docker SDK
        > Installing Docker SDK required for docker-compose to build the image of app
        ```yaml
        - name: Installing Docker SDK
          become: yes
          pip:
            name: docker
        ```
    -   Installing Docker-Compose Pip - Root
        ```yaml
        - name: Installing Docker-Compose Pip - Root
          become: yes
          pip:
            name: docker-compose
        ```
    -   Installing Docker-Compose Pip - User
        ```yaml
        - name: Installing Docker-Compose Pip - User
          pip:
            name: docker-compose
        ```
    -   Add user to Docker Group
        > For local user able to run Docker-Compose not as root
        ```yaml
        - name: Add user to Docker Group
          remote_user: azureuser
          user:
            name: "azureuser"
            group: "docker"
            append: yes
        ```
    -   Installing Docker-Compose
        ```yaml
        - name: Installing Docker-Compose
          remote_user: azureuser
          get_url:
            url: https://github.com/docker/compose/releases/download/v2.7.0/docker-compose-linux-x86_64
            dest: /usr/local/bin/docker-compose
            mode: "u+x,g+x"
        ```
    -   Give permission to Docker-Compose
        ```yaml
        - name: Give permission to Docker-Compose
          become: yes
          file:
            path: /usr/local/bin/docker-compose
            owner: azureuser
            group: docker
            mode: "1777"
        ```
    -   Installing Git
        ```yaml
        - name: Installing Git
          remote_user: azureuser
          yum:
            name: git
            state: present
        ```
    -   Making a directory on managed node
        > Create a directory for app to be stored in
        ```yaml
        - name: Making a directory on managed node
          remote_user: azureuser
          file:
            path: /home/azureuser/todo_app/app
            state: directory
        ```
    -   Pulling Todo App from Github
        ```yaml
        - name: Pulling Todo App from Github
          remote_user: azureuser
          git:
            repo: https://github.com/exbert/todo-app-project.git
            dest: /home/azureuser/todo_app/app
            version: master
        ```
    -   Find my public ip
        > Getting Public IP address to write on index.html
        ```yaml
        - name: Find my public ip
          uri:
            url: http://ifconfig.me/ip
            return_content: yes
        register: ip_response
        ```
    -   Updating Index.html with the Hostname and IP address
        ```yaml
        - name: Updating Index.html with the Hostname and IP address
          blockinfile:
            path: /home/azureuser/todo_app/app/src/static/index.html
            marker: "<!-- {mark} ANSIBLE MANAGED BLOCK -->"
            insertafter: "<body>"
            block: |
                <h3>Welcome to {{ inventory_hostname }}</h3>
                <h3>Public IP Address is :  {{ ip_response.content }}</h3>
        ```
    -   Starting Todo App from Docker-Compose
        ```yaml
        - name: Starting Todo App from Docker-Compose
          remote_user: azureuser
          community.docker.docker_compose:
            project_src: /home/azureuser/todo_app/app
        ```
    -   Setting Docker Service to start on reboot
        ```yaml
        - name: Setting Docker Service to start on reboot
          command: sudo systemctl enable docker.service
          become: yes
        ```
    -   Setting Containerd Service to start on reboot
        ```yaml
        - name: Setting Containerd Service to start on reboot
          command: sudo systemctl enable containerd.service
          become: yes
        ```

### **Challenges** <a name="challenges"></a>

### **Conclusion** <a name="conclusion"></a>
