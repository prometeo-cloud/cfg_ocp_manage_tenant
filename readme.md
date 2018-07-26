# cfg_ocp_manage_tenant

<a name="toc"></a>
## Table of Contents 
- [Configuration Description](#description)
- [Configuration Behaviour](#behaviour)
- [Configuration Variables](#variables)
- [Using this configuration](#using)
    - [How to add project(s) to a tenant space](#create-tenant)
    - [How to create a service account token](#create-token)

<a name="description"></a>
## Configuration Description [[up](#toc)]

A playbook using [ocp_manage_tenant](https://github.com/prometeo-cloud/ocp_manage_tenant_with_oc) and [ocp_manage_tenant_ldap](https://github.com/prometeo-cloud/ocp_manage_tenant_ldap) to manage OpenShift tenants.  

A **tenant** is a logical entity that comprises a collection of OCP projects (or Kubernetes Namespaces). 
A tenant provides the structure on which to build a Continuous Delivery (CD) pipeline.  

For example, a tenant can be made up of the following OCP projects (environments):  
- **DEV**: 
    - where the automation server is deployed to control the deployment of artefacts across the CD pipeline.
    - where the application source code is integrated, one or more Docker images are produced using Source to Image (S2I) technology in OCP.
    - where automated tests are executed.
    
- **TEST**: 
    - where manual functional testing activities are carried out. 
    
- **DEMO**:
    - where product owners can explore the features of the application before it progresses any further in the CD   pipeline.

Application promotion across environments can be achieved using image tagging.


<a name="behaviour"></a>
## Configuration Behaviour [[up](#toc)]

**Feature:** Add project(s) (to tenant space)
- **Given** the name of the tenant is known.
- **Given** OCP users exists in the Directory Server (LDAP).
- **Given** the name of one or more OCP projects (environments) is specified.
- **When** the create_projects playbook is executed.
- **Then** a number of OCP projects are created with a common base name and a different environment suffix.
- **Then** a group for each project / user role combination is created in the LDAP server. 

**Feature:** Delete Project(s) (from tenant space)
- **Given** the name of the tenant is known.
- **Given** the name of one or more OCP projects (environments) is specified.
- **When** the delete_projects playbook is executed.
- **Then** the requested OCP projects are deleted from OCP.
- **Then** the relevant ldap groups are deleted from the LDAP server.

**Feature:** Update Tenant Resource Quota
- **Given** the name of the tenant is known.
- **Given** the name of one or more OCP projects (environments) is specified.
- **Given** CPU and Memory values are specified for each named project.
- **When** the update_quota playbook is executed
- **Then** the CPU and Memory quota constraints on each named project are updated.

<a name="variables"></a>
## Configuration Variables [[up](#toc)]

A list of the external variables used by the role.

| Variable  | Description  | Example  | 
|---|---|---|
| **ocp_uri**  | The URI of the OCP server, including port.  |  https://127.0.0.1:8443 |
| **ocp_token**  | The token used to authenticate with the OCP server. It should be for a service account with cluster-admin role.  | "account-token-89si3" |
| **ldap_server_uri**  | The URI of the LDAP server used by OCP to authenticate users. | ldap://ldap_hostname:389/cn=users,cn=accounts, dc=eu-west-2,dc=compute,dc=internal?uid  |
| **local_admin_dn**  | The local admin username for the LDAP server. |  |
| **local_admin_pwd** | The local admin password for the LDAP server. |  |
| **group_dn** | The OU under which groupos will be created. | OU=Groups,OU=<<sub_OU_name>>,OU=<<main_OU_name>>,DC=<<domain>>,DC=com |
| **tenant_name**  | The name of the tenant to be created. This name is used to name the projects in the tenant space.  | MyApp  |
| **projects**  | A list of projects making up the Tenant. | See usage section - create tenant. |

<a name="using"></a>
## Using this configuration [[up](#toc)]

How to invoke the role from a playbook:

<a name="create-tenant"></a>
### How to add project to a tenant space

```yaml
- name: Creates Tenant
  host: localhost
  tasks:
      - name: Adding projects to a tenant space
        vars:
            tenant_name: 'Tenant_A'
            projects: 
            - name: 'DEV'
              admins:
                - devAdmin@acme.com
              users:
                - dev1@acme.com
                - dev2@acme.com
            - name: 'TEST'
              admins:
                - testAdmin@acme.com
              users:
                - tester1@acme.com
                - tester2@acme.com
            - name: 'DEMO'
              admins:
                - demoAdmin@acme.com
              users:
                - demo1@acme.com
                - demo2@acme.com
        include_role:
            name: ocp_manage_tenant
            tasks_from: add_projects
```

**Note:** other required variables are in the sample inventory file [here](./inventories/group_vars/control.yml).

<a name="create-token"></a>
### How to create a service account token

```bash
# log in as sys admin
$ oc login -u system:admin

# creates a service account
$ oc create sa automator -n openshift-infra

# adds the account to the admin role
$ oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:openshift-infra:automator

# gets the value of the token using the name
$ oc serviceaccounts get-token automator -n openshift-infra
```

### How to check access rights

```bash
$ oc get sa automator -o yaml
$ oc policy who-can create ProjectRequest
```