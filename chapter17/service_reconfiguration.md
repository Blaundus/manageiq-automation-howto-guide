## Service Reconfiguration

So far when we have created our Service Catalog Items, we have specified a Provisioning Entry Point State Machine to handle the provisioning workflow for the new Service. There are two other Entry Points that we can optionally hook into, a Retirement Entry Point (see [Retirement](../chapter18/retirement.md)), and a Reconfigure Entry Point.
<br> <br>

![screenshot](images/screenshot63.png?)

If we create a Service Catalog Item to have a Reconfigure Entry Point State Machine, then any Service created from that Catalog Item will have a **Reconfigure this Service** option available under its **Configuration** menu:
<br> <br>

![screenshot](images/screenshot64.png?)

If we select this option, we are presented with the original Service Dialog once more. Entering new values and clicking the Submit button will create a _ServiceReconfigureRequest_ to peform the reconfiguration action, based on the revised values that we have have entered into the dialog.

This duel-use of an initial configuration dialog as a reconfiguration dialog, works well if we are using a configuration management tool such as Foreman (or Satellite 6), and Puppet. In this case we can specify Puppet  _Smart Class Parameters_ in our Service Dialog that can be passed to Foreman and used to override the statically defined Puppet class parameters.

### Service Design

When we create a Service that can be reconfigured in this way, we need to put extra thought into our service design and provisioning workflow. We need to make some of our Service Dialog Elements _Reconfigurable_ so that we can enter new values when re-presented with the Dialog on a Service reconfiguration request (elements not marked as Reconfigurable will be greyed-out). We need to create a _set\_configuration_ method that can be called from either a VM Provisioning or Service Reconfiguration State Machine, and retrieve dialog values from the correct location in each case. This method should detect whether or not the VM Provision was initiated from a Service that passed the correct dialog values, rather than an interactive VM Provision Request for example.

### Adding a Configuration Management Provider

We can add our Satellite 6 server as a Configuration Management Provider in CloudForms. This imports the Foreman Host Groups as CloudForms _Configuration Profiles_, saving us from having to make a REST call to the Satellite server to list them.

![screenshot](images/screenshot89.png)


### Creating the Automation Framework

Even though a Service Reconfiguration capability is provided for us by CloudForms, we still need to add several datastore components if we wish to use it.

#### Creating the Namespaces and State Machines

In our own Domain, we'll create a `/Service/Reconfiguration/StateMachines` Namespace:
<br> <br>

![screenshot](images/screenshot65.png)

We'll create a simple State Machine Class called `ServiceReconfigure`, with 7 state/stages:
<br> <br>
![screenshot](images/screenshot66.png)

**pre{1-3}** and **post{1-3}** are future-proofing placeholders in case we wish to enhance the functionality in future. For now we'll just be using the **reconfigure** state/stage.

We'll copy the `ManageIQ/Service/Provisioning/StateMachines/ServiceProvision_Template/update_serviceprovision_status` Method into our domain, and rename it to `update_servicereconfigure_status`. We change line 6 from:

```prov = $evm.root['service_template_provision_task']```

to:

```reconfigure_task = $evm.root['service_reconfigure_task']```

<br>
We'll edit the **On Entry**, **On Exit** and **On Error** columns in the State Machine Class Schema to refer to the new `update_servicereconfigure_status` Method:
<br> <br>

![screenshot](images/screenshot68.png)

We create a `Default` Instance of the `ServiceReconfiguration` State Machine Class, and we'll point the **reconfigure** stage to the `SetConfiguration` Instance that we'll create:
<br> <br>

![screenshot](images/screenshot67.png)

#### Email Classes

We need to create two new email Instances with associated Methods, to send emails when a Service Reconfigure is Approved and Completed. For convenience we'll just copy, rename and edit the `ManageIQ/Service/Provisioning/Email` Instances and Methods:
<br> <br>

![screenshot](images/screenshot80.png)

#### Policies

We need to generate Policy Instances for two ServiceReconfigure events, _ServiceReconfigureRequest\_created_ and _ServiceReconfigureRequest\_approved_.

We copy `ManageIQ/System/Policy/ServiceTemplateProvisionRequest_created` into our Domain as `System/Policy/ServiceReconfigureRequest_created`. We can leave the schema contents as they are because we'll use the same auto-approval state machine as when the service was originally provisioned.

We copy `ManageIQ/System/Policy/ServiceTemplateProvisionRequest_approved` into our Domain as `/System/Policy/ServiceReconfigureRequest_approved`, and we edit the **rel5** state to point to our new `/Service/Reconfiguration/Email/ServiceReconfigurationRequestApproved` email Instance:
<br> <br>

![screenshot](images/screenshot81.png)

### Modifying the VM Provision Workflow

We need to change our VM Provision workflow to add a state/stage to perform the initial configuration using the values input from the Service Dialog. We'll take the State Machine that we used in [Integrating with Satellite 6 During Provisioning](../chapter16/integrating_with_satellite_6.md) and add a **SetConfiguration** stage after **RegisterSatellite**. **SetConfiguration** points to the same Instance as our new `ServiceReconfiguration` State Machine's **reconfigure** stage:
<br> <br>
 
![screenshot](images/screenshot82.png)

### Service Dialog

We're going to create a completely dynamic service dialog, interacting with Satellite to retrieve information. The dialog will search the VMDB for Configuration Profiles (Host Groups), and present them in a drop-down list. For the Host Group selected, Satellite will be queried for the configured Activation Keys and Puppet Classes, and these will be presented in drop-down lists. For the Puppet Class selected, Satellite will be queried for the available Smart Class Parameters and these will be presented in a drop-down list. Finnaly a text area box will be presented to optionally input an override parameter.

The **Puppet Class**, **Smart Class Parameter** and **New Parameter Value** elements will be marked as **Reconfigurable** in the Service Dialog:

![screenshot](images/screenshot84.png)

When ordered the Dialog will look as follows:
<br> <br>
 
![screenshot](images/screenshot83.png)

### Instances and Methods

#### Dynamic Dialogs

The dynamic dialog Instances and Methods are defined under an `/Integration/Satellite/DynamicDialogs` Namespace in our Domain:
<br> <br>

![screenshot](images/screenshot85.png)

The Schema for the `Methods` class holds variables containing the credentials to connect to our Satellite server.

##### Common Functionality

Each of the dynamic Methods has a simple `rest_action` method to perform the REST call to Satellite:

```ruby
def rest_action(uri, verb, payload=nil)
  headers = {
    :content_type  => 'application/json',
    :accept        => 'application/json;version=2',
    :authorization => "Basic #{Base64.strict_encode64("#{@username}:#{@password}")}"
  }
  response = RestClient::Request.new(
    :method      => verb,
    :url         => uri,
    :headers     => headers,
    :payload     => payload,
    verify_ssl: false
  ).execute
  return JSON.parse(response.to_str)
end
```

They each pull the credentials from the Instance schema, define the base URI and an empty values_hash

```ruby
servername = $evm.object['servername']
@username  = $evm.object['username']
@password  = $evm.object.decrypt('password')

uri_base = "https://#{servername}/api/v2"
values_hash = {}
```

##### ListHostGroups

The `list_hostgroups` Method does not need to connect to the Satellite RESTful API. It performs a simple VMDB lookup of all Configuration Profiles:

```ruby
  hostgroups = $evm.vmdb(:configuration_profile).all
  
  if hostgroups.length > 0
    if hostgroups.length > 1
      values_hash['!'] = '-- select from list --'
    end
    hostgroups.each do |hostgroup|
      $evm.log(:info, "Found Host Group '#{hostgroup.name}' with ID: #{hostgroup.manager_ref}")
      values_hash[hostgroup.manager_ref] = hostgroup.name
    end
  else
    values_hash['!'] = 'No hostgroups are available'
  end
```
##### ListActivationKeys

The `list_activationkeys` Method retrieves the hostgroup\_id from the **Host Group** element, and makes a Satellite API call to get the hostgroup parameters:

```ruby
  hostgroup_id = $evm.object['dialog_hostgroup_id']

  if hostgroup_id.nil?
    values_hash['!'] = "Select a Host Group and click 'Refresh'"
  else
    rest_return = rest_action("#{uri_base}/hostgroups/#{hostgroup_id}/parameters", :get)
    rest_return['results'].each do |hostgroup_parameter|
      if hostgroup_parameter['name'].to_s == "kt_activation_keys"
        hostgroup_parameter['value'].split(',').each do |activationkey|
          values_hash[activationkey] = activationkey
        end
      end
    end
    if values_hash.length > 0
      if values_hash.length > 1
        values_hash['!'] = '-- select from list --'
      end
    else
      values_hash['!'] = 'This Host Group has no Activation Keys'
    end
  end
```

##### ListPuppetClasses

The `list_puppetclasses` Method retrieves the hostgroup\_id from the **Host Group** element, and makes a Satellite API call to get the Puppet classes associated with the host group:

```ruby
hostgroup_id = $evm.object['dialog_hostgroup_id']

if hostgroup_id.nil?
  values_hash['!'] = "Select a Host Group and click 'Refresh'"
else
  rest_return = rest_action("#{uri_base}/hostgroups/#{hostgroup_id}/puppetclasses", :get)
  if rest_return['total'] > 0
    if rest_return['total'] > 1
      values_hash['!'] = '-- select from list --'
    end
    rest_return['results'].each do |classname, classinfo|
      values_hash[classinfo[0]['id'].to_s] = classname
    end
  else
    values_hash['!'] = 'No Puppet Classes are defined for this Hostgroup'
  end
end
```

##### ListSmartClassParameters

The `list_smart_class_parameters` Method retrieves the hostgroup\_id and puppetclass_id from previous elements, and makes a Satellite API call to get the Puppet smart class parameters associated with the host group. For each parameter returned it then makes a further Satellite API call to cross-reference against the requested Puppet class (this operation will be slow if many Puppet classes with smart class variables are defined in our host group, but is suitable for our example):

```ruby
hostgroup_id    = $evm.object['dialog_hostgroup_id']
puppet_class_id = $evm.object['dialog_puppet_class_id']

if puppet_class_id.nil?
  values_hash['!'] = "Select a Puppet Class and click 'Refresh'"
else
  call_string = "#{uri_base}/hostgroups/#{hostgroup_id}/smart_class_parameters"
  rest_return = rest_action(call_string, :get)
  rest_return['results'].each do |parameter|
    #
    # Retrieve the details of this smart class parameter 
    # to find out which puppet class it's associated with
    #
    call_string = "#{uri_base}/hostgroups/#{hostgroup_id}/smart_class_parameters/#{parameter['id']}"
    parameter_details = rest_action(call_string, :get)
    if parameter_details['puppetclass']['id'].to_s == puppet_class_id
      values_hash[parameter['id'].to_s] = parameter_details['parameter']
    end
  end
  if values_hash.length > 0
    if values_hash.length > 1
      values_hash['!'] = '-- select from list --'
    end
  else
    values_hash['!'] = 'This Puppet class has no Smart Class Parameters'
  end
end
```


#### Configuration Workflow

##### RegisterSatellite

We edit the `register_satellite` Method from [Integrating with Satellite 6 During Provisioning](../chapter16/integrating_with_satellite_6.md) to take out the hard-coded selection of Host Group. We also bypass Satellite registration entirely if we don't find the hostgroup_id:

```ruby
#
# Only register if the provisioning template is linux
#
if template.platform == "linux"
  #
  # Only register with Satellite if we've been passed a 
  # hostgroup ID from a service dialog
  #
  hostgroup_id = $evm.root['miq_provision'].get_option(:dialog_hostgroup_id)
  unless hostgroup_id.nil?
    ...
```

##### ActivateSatellite

We edit the `activate_satellite` Method from [Integrating with Satellite 6 During Provisioning](../chapter16/integrating_with_satellite_6.md) to take out the hard-coded selection of Activation Key. We also bypass Satellite activation entirely if we don't find the Activation Key name:

```ruby
#
# Only register if the provisioning template is linux
#
if template.platform == "linux"
  #
  # Only register and activate with Satellite if we've been passed an 
  # activation key from a service dialog
  #
  activationkey = $evm.root['miq_provision'].get_option(:dialog_activationkey_name)
  unless activationkey.nil?
    ...
```
##### SetConfiguration

The `set_configuration` Method will be called from two completely different State Machines, once to perform an initial configuration during provisioning, and possibly again during a service reconfigure request. The Method must retrieve the Service Dialog values from either of two different places:


```ruby
  if $evm.root['vmdb_object_type'] == 'miq_provision'
    parameter_id    = $evm.root['miq_provision'].get_option(:dialog_parameter_id)
    parameter_value = $evm.root['miq_provision'].get_option(:dialog_parameter_value)
    hostgroup_id    = $evm.root['miq_provision'].get_option(:dialog_hostgroup_id)
    hostname        = $evm.root['miq_provision'].get_option(:dialog_vm_name)
  elsif $evm.root['vmdb_object_type'] == 'service_reconfigure_task'
    parameter_id    = $evm.root['dialog_parameter_id']
    parameter_value = $evm.root['dialog_parameter_value']
    hostgroup_id    = $evm.root['dialog_hostgroup_id']
    hostname        = $evm.root['dialog_vm_name']  
  end
```
If a Smart Class Parameter override value has not been input the Method simply exits:

```ruby
  #
  # Only set the smart class parameter if we've been passed a 
  # parameter value from a service dialog
  #
  unless parameter_value.nil?
     ...
```
The Method must fetch the default domain name from the Host Group to assemble the correct FQDN for the match:

```ruby
rest_return = rest_action("#{uri_base}/hostgroups/#{hostgroup_id}", :get)
domain_name = rest_return['domain_name']
match = "fqdn=#{hostname}.#{domain_name}"
```

The Method must determine whether the override match already exists. If it doesn't exist it must be created with a POST action; if it does exist is must be updated with a PUT action:

```ruby
call_string = "#{uri_base}/smart_class_parameters/#{parameter_id}/override_values"
rest_return = rest_action(call_string, :get)
override_value_id = 0
if rest_return['total'] > 0
  rest_return['results'].each do |override_value|
    if override_value['match'] == match
      override_value_id = override_value['id']
    end
  end
end
if override_value_id.zero?
  payload = {
    :match => match,
    :value => parameter_value
  }
  call_string = "#{uri_base}/smart_class_parameters/#{parameter_id}/override_values"
  rest_return = rest_action(call_string, :post, JSON.generate(payload))
else
  payload = {
    :value => parameter_value
  }
  call_string = "#{uri_base}/smart_class_parameters/#{parameter_id}/override_values/#{override_value_id}"
  rest_return = rest_action(call_string, :put, JSON.generate(payload))
end
```

The full code for the Methods is [here](https://github.com/pemcg/cloudforms-automation-howto-guide/tree/master/chapter17/scripts)

### Testing

We'll order a new service, and select appropriate Host Group and Activation Keys from the drop-downs. We'll select the _motd_ Puppet Class, and override the _content_ Smart Class Parameter:
<br> <br>
 
![screenshot](images/screenshot86.png)

We click **Submit**, and wait for our newly provisioned Service.

Logging in to the newly provisioned server confirms that the motd has been set:

```
root@192.168.1.193's password:
Last login: Sat Nov  7 09:48:50 2015 from cloudforms05.bit63.net
Power Shutdown Saturday 7th Nov 12pm
#[root@s3-host01 ~]#
```
If we look at the details of our new Service in **My Services** and select **Configuration -> Reconfigure This Service**, we are again presented with the service dialog, but the elements not marked as **Reconfigurable** are greyed out:
<br> <br>

![screenshot](images/screenshot87.png)

We can select the _motd_ Puppet Class again, enter a new value for the _content_ Smart Class Parameter, and click **Submit**.

We receive an email informing us that the reconfiguration request has been approved:

```
Hello, 
Your Service reconfiguration request was approved. If Service reconfiguration 
is successful you will be notified via email when the Service is available.

Approvers notes: Auto-Approved

To view this Request go to: https://cloudforms05/miq_request/show/1000000000109

Thank you,
Virtualization Infrastructure Team
```

We can login to the Satellite 6 User Interface to confirm that the "Override value for specific hosts" contains our updated value against the match filter:
<br> <br>

![screenshot](images/screenshot88.png)

Once the Puppet agent has run on the client again, we can login and see the new message:

```
root@192.168.1.193's password:
Last login: Sat Nov  7 10:02:29 2015 from my-macbook-pro.bit63.net
Power Shutdown Cancelled
#[root@s3-host01 ~]#
```

Success!

