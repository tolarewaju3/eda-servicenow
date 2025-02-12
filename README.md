# Automated Password Reset (ServiceNow + Ansible)

## The Problem

**40% of IT help desk time go towards password related tickets.** 

A key example of this is resetting employee's forgotten passwords. But while employees wait, productivity stalls, costing money on both ends—IT resolving the issue and employees stuck in limbo. 

**Enter Event-Driven Ansible.** It can automates responses to events, like password reset requests, saving a ton of time and money.

## The Architecture

Our architecture will capture ServiceNow password reset tickets and reset the password on a RHEL host. The architecture has three main parts.

* **SerivceNow** - IT Service management where we'll submit password reset tickets
* **Ansible EDA** - Event Driven Automation to respond to password reset tickets
* **RHEL Host** - Password reset target

![ServiceNow home](img/arch_diagram.png)

## What You'll Need

1. Ansible Automation Platform 2.5
1. Red Hat Enterprise Linux Host (or similar Linux distro)
1. ServiceNow instance (or setup a developer account below)

*AAP needs connectivity to the RHEL Host, and ServiceNow needs connectivity to AAP.*

## Setup Ansible Envrionment

We'll create an ansible inventory, a job template and a rulebook.

The Ansible inventory specifies **the systems** where passwords will be reset, the job template outlines **how the reset** will be performed, and the rulebook determines **which events trigger a reset.**

### Create an Inventory

We'll define our target system in the invetory. If you already have an inventory with your target system, skip to the next section.

**First, create a new inventory.** Under the *Automation Execution* menu, in the *Infrastructure* section, select *Inventories* and **choose Create Inventory**. Use the following details.

```
Name: RHEL Inventory
Organization: Default
```

![Create inventory](img/create_inventory.png)

**Next, create the target host.** On the Inventories page, select the *Hosts* tab and choose **Create Host.** Enter the publicly accessible hostname of your host (Ex. `ec2-24-191-132-171.us-east-2.compute.amazonaws.com`)

![Create host](img/create_host.png)

**Finally, create credentials** to access the host. Under the *Automation Execution* menu, in the *Infrastructure* section, select *Credentials* and **choose Create Credentials**.

```
Name: RHEL Host
Organization: Default
Credential Type: Machine
Username: <your_username>
Private Key or Password: <your_key>
```
![Create credentials](img/create_credential.png)

### Create a Job Template

**First, create an execution project.** Under the *Automation Execution* menu, select *Projects* and **choose Create project**. Use the following details.

```
Name: password-reset
Organization: Default
Execution Environment: Default execution environment
Source control type: Git
Source control URL: https://github.com/tolarewaju3/eda-servicenow.git
```

![Exeuciton project](img/execution_project.png)

Create the project. You should see the `Last job status` as Success.

**Next, create a job template.** Under the *Automation Execution* menu, select *Templates* and **choose Create Template**. Use the following details.

```
Name: password-reset
Job type: Run
Inventory: RHEL Inventory
Project: password-reset
Playbook: playbooks/playbook.yml
Execution Environment: Default execution environment
Credentials: rhel-creds
Extra vars: Prompt on launch
```
![Job template](img/job_template.png)

**Create the job template.** This template runs a playbook to reset the password of user on a RHEL host. It's NOT production-ready (as the password is in plain text), but it'll work for us. Here's the playbook.

```yml
---
- name: Reset password on RHEL host
  hosts: all  # Specify your target host group
  become: yes  # Use 'become' if root privileges are needed
  tasks:

    - name: Print Vars
      debug:
        msg: Resetting password for {{ username }}

    - name: Set password for user
      user:
        name: "{{ username }}"
        password: "{{ 'NewSecurePassword123!' | password_hash('sha512') }}"
```

### Create a Rulebook Activation

We'll create a rulebook activation. Rulebook activations **detect events from a source and trigger automations.**

First, **create a decision project.** Under the *Automation Decisions* menu, select *Projects* and **choose Create a new project**. Use the following details.

```
Name: password-reset
Organization: Default
Source control URL: https://github.com/tolarewaju3/eda-servicenow.git
```

![Decision project](img/decision_project.png)

Create the project. Make sure the Status shows `Completed`.

Next, we'll create an Event Stream. Event Streams are a **simple way to capture events from external systems.** This serves as the server-side webhook where ServiceNow will send events.

**Create an Event Stream token.** Under the *Automation Decisions* menu, in the *Infrastructure* section, select *Credentials* and **choose Create a Credential**. Generate a random token [here](https://it-tools.tech/token-generator?length=21) and use the details below.

```
Name: servicenow-credential
Organization: Default
Credential type: ServiceNow Event Stream
Token: Your token. Make sure to save it for later!
```
![Credential](img/credential.png)

Click Create Credential. This token will be used in our webhook and sent with our ServiceNow REST calls. 

**Next, create the event stream.** Under the *Automation Decisions* menu, select *Event Streams* and **choose Create Event Stream**. Use the following details.

```
Name: servicenow
Organization: Default
Event stream type: ServiceNow Event Stream
Credential: servicenow-credential
```

![Event stream](img/event_stream.png)

Click **Create event stream**. After it finishes, **copy the webhook URL** as we'll use this later in ServiceNow.

**Next, we'll create an AAP credential.** Under the *Automation Decisions* menu, in the *Infrastructure* section, select *Credentials* and **choose Create a Credential**.

```
Name: aap
Organization: Default
Credential type: Red Hat Ansible Automation Platform
Red Hat Ansible Automation Platform: https://<<your_base_url>>/api/controller/
Username: <your_aap_admin_username>
Password: <your_aap_admin_password>
```

![AAP Credentials](img/aap_creds.png)

**Create the credential.** This credential will allow our rulebook to call the password reset job on our Ansible controller. You can find your base url under *Settings* in the *System* section.

**Finally, create a rulebook activation.** Under the *Automation Decisions* menu, in the *Rulebook Activations* section, choose **Create Rulebook Activation.** 

```
Name: password-reset
Organization: Default
Project: password-reset
Credential: aap
Rulebook: servicenow-rulebook.yml
Decision Environment: Default decision environment
Event streams: servicenow
```
![Rulebook activation](img/rulebook_activation.png)

**Create the rulebook activation.** This rulebook will trigger our password reset job when it receives events from ServiceNow. ServiceNow event. Here's the rulebook we're using.

```yml
---
- name: Listen for events from service now
  hosts: all
  sources:
   - ansible.eda.webhook:
      host: 0.0.0.0
      port: 5005
  rules:
   - name: Password reset request
     condition: event.payload.short_description == "Reset my password"
     actions:
       - debug:
         msg:
          - "{{ event.payload }}"
       - run_job_template:
           name: password-reset
           organization: Default
           job_args:
             extra_vars:
               username: "{{ event.payload.caller.id }}"
```

After the rulebook activation starts, the Activation status should be `Running`.

## Setup ServiceNow Envrionment

We'll use ServiceNow as our incident management system for password request tickets. If you already have a ServiceNow instance in your environment, you can skip to the "Create a Business Rule" step.

First, **sign up for a ServiceNow developer account.** Navigate to [developer.servicenow.com](https://developer.servicenow.com) and select **Sign Up and Start Building**.

![ServiceNow home](img/servicenow_home.png)

Fill out the relevant information and hit **Sign Up**. Once you're signed in, you should see this screen.

![ServiceNow sign in](img/servicenow_signin.png)

**Request your ServiceNow instance**. In the top right corner, select **Request Instance**. This may take a while, but eventually you should be able open your ServiceNow developer instance.

We'll send events to Ansible each time we open a ticket (or incident). For entierprise instances of ServiceNow, there's an Ansible EDA add-on from the store. But if you're using the developer instance, we'll create a business rule to send events.

**Create a Business Rule**. In your instance, go to the top left and select **All**. Type **Business Rule** into the search bar. Select the one under **System Definition**.

![ServiceNow select business rule](img/select_business_rule.png)

In the top right, select **New** and enter the following:

```
Name: Send Incident to Ansible EDA
Table: Incident [Incident]
Advanced: Selected
When to run: async
Insert: Selected
```

![ServiceNow select business rule](img/business_rule.png)

**Switch to the advanced tab** and replace the code with this. Don't forget to insert your webhook url and token.

```js
(function executeRule(current, previous /*null when async*/) {
    // Webhook URL
    var webhookUrl = '<your_ansible_event_stream_url>';

    var userName = gs.getUser().getName(); // Full name of the user
    var userID = gs.getUser().getID();    // User sys_id
    var userEmail = gs.getUser().getEmail(); // User email address

    // Fetch caller details from the 'caller_id' field
    var caller = current.caller_id.getRefRecord(); // Get the full caller record
    var callerName = caller.getValue('name') || 'Unknown'; // Caller's full name
    var callerEmail = caller.getValue('email') || 'Unknown'; // Caller's email
    var callerID = caller.getUniqueValue(); // Caller's sys_id

    // Payload to send
    var payload = {
        incident_number: current.number.toString(),
        short_description: current.short_description.toString(),
        priority: current.priority.toString(),
        status: current.state.toString(),
        updated_by: current.sys_updated_by.toString(), // User who updated the record
        user: {
            name: userName,
            id: userID,
            email: userEmail
        },
        caller: {
            name: callerName,
            id: callerID,
            email: callerEmail
        }
    };

    // Create the RESTMessageV2 object
    var restMessage = new sn_ws.RESTMessageV2();
    restMessage.setEndpoint(webhookUrl);
	restMessage.setRequestHeader('Authorization', 'Bearer YOUR_TOKEN_HERE');
    restMessage.setHttpMethod('POST');

    // Set request headers
    restMessage.setRequestHeader('Content-Type', 'application/json');

    // Set the request body
    restMessage.setRequestBody(JSON.stringify(payload));

    try {
        // Execute the request
        var response = restMessage.execute();
        var httpStatus = response.getStatusCode();
        var responseBody = response.getBody();

        gs.info('Webhook sent successfully. Status: ' + httpStatus + ', Response: ' + responseBody);
    } catch (ex) {
        gs.error('Error sending webhook: ' + ex.message);
    }
})(current, previous);
```

There's a lot going on here. Here are the important steps.

1. **Build a payload** with variables from our incident (username, userID, incident number)
1. **Create a REST message** with the payload and our authorization headers
1. **Send the REST message** to the Ansible webhook url with our token

**Click Submit** to create the rule.

## Create a ServiceNow ticket

First, we'll create a user with the same username as the one on our RHEL host. We'll submit the ticket with this user and it'll change their password.

Go to the top left and select **All**. Type `Users` into the search bar and select the one under **Organization**. Click the **New** button in the top right.

```
User ID: test-user
Firstname: Test
Lastname: User
```

![New User](img/new_servicenow_user.png)

Hit Submit.

**Create an Incident**. Go to the top left and select **All**. Type `Incidents` into the search bar and choose the one under **Self Service**. Click the **New** button in the top right.

![New Ticket](img/new_servicenow_ticket.png)

Change the caller to `test-user`. For the short description, select the lightbulb on the right and **choose Reset my password.** Our rulebook will only fire for events that contain the description "Password reset".

**Submit the incident.** If all went well, you should see the job template run in AAP and your user should have a new password.

*Note that if there is no user present, the playbook will create one*