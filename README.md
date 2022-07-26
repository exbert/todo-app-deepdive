## PROGRAMING CHALLENGE

### [Description](#description)

### [Inventory](#inventory)

### [Logical Topology](#logical_topology)

### [Ansible Playbook](#ansible_playbook)

### [Challenges](#challenges)

### [Conclusion](#conclusion)

---

### **Description <a name="description"></a>**

With in this Readme, You will find the explanation of How Ansible - Docker - Azure elements are used to create a ansible managed container placed in Azure.

Brief process of this project is; Ansible Controller VM takes the Yml file and executes with ansible-playbook.

First server connects the Azure, creates the demanded App VM. Then registers its hostname and ip address to ansible/hosts file.

Later Ansible server connects to newly created App VM and installs Docker, Python3-pip, Docker SDK, Docker-compose and Git.

Then, App VM pulls the required app from github repository and modifies the app index file to writes app vm hostname and ip address.

Finally, App VM build the app with docker-compose and starts the app services.

Application can be accessed with this url:

[http://bsch-todo-app.germanywestcentral.cloudapp.azure.com:3000](http://bsch-todo-app.germanywestcentral.cloudapp.azure.com:3000)

---

### **Inventory <a name="inventory"></a>**

### **Logical Topology <a name="logical_topology"></a>**

![Logical Topology](https://github.com/exbert/todo-app-deepdive/blob/master/img/ansible_project_topology.png "Logical Topology")

### **Ansible Playbook <a name="ansible_playbook"></a>**

### **Challenges <a name="challenges"></a>**

### **Conclusion <a name="conclusion"></a>**
