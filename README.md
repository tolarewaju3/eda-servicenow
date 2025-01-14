# Autoamted Password Reset (ServiceNow + Ansible)

## The Problem

**40% of IT help desk time go towards password related tasks.** 

A key example of this is resetting employee's forgotten passwords. But while employees wait, productivity stalls, costing money on both ends—IT resolving the issue and employees stuck in limbo. 

**Enter Event-Driven Ansible.** It automates responses to events, like password reset tickets, saving time and money.

## The Architecture

Our archiecture will capture ServiceNow password reset tickets, reset the password on a RHEL host, and notify the user. The architecture has three main parts.

* **SerivceNow Instance** - IT Service management for password reset tickets
* **Ansible EDA** - Event Driven Automation to respond to password reset tickets
* **RHEL Host** - Password reset target

![ServiceNow home](img/arch_diagram.png)

## What You'll Need

1. Ansible Automation Platform 2.5
1. Red Hat Enterprise Linux Host (or similar Linux distro)
1. ServiceNow instance (or setup a developer account below)

*AAP needs connectivity to the RHEL Host, and ServiceNow needs connectivity to AAP.*

## Setup ServiceNow Envrionment

We'll use ServiceNow as our incident management system for password request tickets. If you already have a ServiceNow instance in your environment, you can skip to the business rule step.

**First, sign up for a ServiceNow developer account.** Navigate to [developer.servicenow.com](https://developer.servicenow.com) and select `Sign Up and Start Building`.

![ServiceNow home](img/servicenow_home.png)

Fill out the relevant information and hit `Sign Up`. Once you're signed in, you should see this screen.

![ServiceNow sign in](img/servicenow_signin.png)

**Request your ServiceNow instance**. In the top right corner, select `Request Instance`. This may take a while, but eventually you should be able open your ServiceNow developer instance.

Next, we'll send events to Ansible each time we open a ticket (or incident). For entierprise instances of ServiceNow, there's an Ansible EDA add-on from the store. But if you're using the developer instance, we'll create a business rule to send events.

**Create a Business Rule**. In your instance, go to the top left and select `All`. Type `Business Rule` into the search bar. Select the one under `System Definition`.

![ServiceNow select business rule](img/select_business_rule.png)

In the top right, select `New` and enter the following:

* **Name:** Send Incident to Ansible EDA
* **Table:** Incident [Incident]
* **Advanced:** Selected
* **When to run:** async
* **Insert:** Selected

![ServiceNow select business rule](img/business_rule.png)

**Switch over to the advanced tab** and replace the code with this.

```js
(function executeRule(current, previous /*null when async*/) {
    // Webhook URL
    var webhookUrl = 'https://aap-aap.apps.cluster-tw277-1.dynamic.redhatworkshops.io/eda-event-streams/api/eda/v1/external_event_stream/7c22ea0b-cec1-4594-8266-7e76d8b8536e/post/';

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
	restMessage.setRequestHeader('Authorization', 'Bearer emkAoV-uFd0AFu5PSMgcd-m0QdIu6uRP7l5F5rG2XxE');
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
1. **Send the REST message** to the Ansible webhook url

Click Submit to create the rule.

## Setup Ansible Envrionment

We'll create two projects: an execution project and a decision project. 

The execution project contains our playbook that resets the password, while our decision project contains rulebooks to capture ServiceNow events.

**Create a New Execution Project.** Sign in to your ansible instance and select `Automation Execution --> Projects --> Create project`. Use the following details.

* Name: password-reset
* Organization: Default
* Execution Environment: Default execution environment
* Source control type: Git
* Source control URL: https://github.com/tolarewaju3/eda-servicenow.git

You should see the `Last job status` as Success.

**Create a New Decision Project.** In your ansible instance, select `Automation Decisions --> Projects --> Create project`. Use the following details.

* Name: password-reset
* Organization: Default
* Source control URL: https://github.com/tolarewaju3/eda-servicenow.git

Again, make sure you see the `Last job status` as Success.

**Create a New Decision Environment.** On the left panel, select Decision Environment under `Automation Decisions`.

[This should actually be the default decision environment]

