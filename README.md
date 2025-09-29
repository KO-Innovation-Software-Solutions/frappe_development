<div align="center">
    <h2>Cohenix Codespaces and Local Development for Frappe</h2>

</div>

This repository is maintained by Cohenix, a subsidiary of the Group Elephant Fund and part of the EPI-USE group of companies. Our solution builds on the Frappe.io framework, integrating custom modules tailored to client needs. For inquiries, terms of use or contributions, please contact the Cohenix development team at christiaan.swart@epiuse.com

## Introduction

Cohenix codespaces allow quick and easy setup of the development environment in the cloud or on local VS Code instance, a ready out of the box development environment.

## Frappe Repositories

- cohenix_erp
- cohenix_hr
- cohenix_crm
- cohenix_learning
- cohenix_payment
- cohenix_helpdesk

## Installation

### Install Local Development Environment
#### Prerequisites:
- Docker Desktop
- VS Code (Plugins: Dev Containers, Container Tools, Docker, Database Client,Database Client JDBC)
#### Instructions:

- Step 1: Create a folder where you will store the files and cd into that folder.
- Step 2: git clone https://github.com/EPIUSECX/frappe_cohenix_development.git
- Step 3: Open VS Code where you have isntalled all of the plugins.
- Step 4: Open the folder that was pulled into the folder that you created during step 1
- Step 5: Vs Code will prompt you to "Reopen in Dev Container" click on that:
- Step 6: The initial docker image will be pulled and installed in Docker Desktop:
- Step 7: The script will then be run and you will be able to view the progress in the "development-bench" Logs folder:




### Install codespace plugin on VS Code

- Open VS Code
- Click on “Extensions” on the left hand menu bar
- Type in the search “codespace”
- Select “GITHub Codespaces” and install

![COHENIX_CODESPACES](images/codespaces_01.png)

### Running codespaces on VS Code

- Open VS Code
- Click on “Remote Explorer” on the left hand menu bar
- Click on + icon to “Create New Codespace”

![COHENIX_CODESPACES](images/codespaces_02.png)

- Type “frappe_cohenix” in top search bar and all codespace you have access too will appear.

![COHENIX_CODESPACES](images/codespaces_03.png)

- Select “frappe_cohenix_development” and then select the branch

![COHENIX_CODESPACES](images/codespaces_04.png)

- Select the virtual environment

![COHENIX_CODESPACES](images/codespaces_05.png)

- Wait for the codespace to spin up, the first time you spin up the codespace may take longer.

![COHENIX_CODESPACES](images/codespaces_06.png)

- Your codespace has been created locally and remotely, you are ready to work!
- cd development-bench
- Now you can run bench commands.

### How to view Logs

- Once the workspace start and you are able to view a folder structure in VS Code Explorer.
- Navigate to the “development-bench\logs” folder.
- Select and open the “bench.log” file.

![COHENIX_CODESPACES](images/codespaces_08.png)

- Another more detailed log to check is.
- In VS Code, press ctrl + shift + p

![COHENIX_CODESPACES](images/codespaces_09.png)

- Select and click “Codespaces: View Creation Log”

![COHENIX_CODESPACES](images/codespaces_10.png)
