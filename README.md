Microsoft Azure Cookbook
========================
[![Cookbook Version](https://img.shields.io/cookbook/v/microsoft_azure.svg)](https://supermarket.chef.io/cookbooks/microsoft_azure)

Description
===========

This cookbook provides resources and providers to create an manage
Microsoft Azure components. Currently supported resources are:

* Storage Accounts ('microsoft_azure_storage_account')
* Blob Storage Containers ('microsoft_azure_storage_container')
* SQL Azure Servers ('microsoft_azure_sql_db_server')

**Note** This cookbook uses the `azure` RubyGem to interact with the
  Azure API. This gem requires `nokogiri` which requires compiling
  native extensions, which means build tools are required.

Requirements
============

Requires Chef 0.7.10 or higher for Lightweight Resource and Provider
support. Chef 0.8+ is recommended. While this cookbook can be used in
`chef-solo` mode, to gain the most flexibility, we recommend using
`chef-client` with a Chef Server.

A Microsoft Azure account is required. The Management Certificate and
Subscription ID are used to authenticate with Azure.

Dependent Cookbooks
===================

* None

Azure Credentials
===============

In order to manage Azure components, authentication credentials need
to be available to the node. There are a number of ways to handle
this, such as node attributes or roles. We recommend storing these in
a databag, and loading them in the recipe where the
resources are needed.

DataBag recommendation:

    % knife data bag show microsoft_azure main
    {
      "id": "main",
      "management_certificate": "YOUR PEM FILE CONTENTS",
      "subscription_id": "YOUR SUBSCRIPTION ID"
    }

This can be loaded in a recipe with:

    microsoft_azure = data_bag_item("microsoft_azure", "main")

And to access the values:

    microsoft_azure['management_certificate']
    microsoft_azure['subscription_id']

We'll look at specific usage below.

Recipes
=======

default.rb
----------

The default recipe installs the `azure` RubyGem, which this cookbook
requires in order to work with the Azure API. Make sure that the
microsoft_azure recipe is in the node or role `run_list` before any
resources from this cookbook are used.

    "run_list": [
      "recipe[microsoft_azure]"
    ]

The `gem_package` is created as a Ruby Object and thus installed
during the Compile Phase of the Chef run.

Resources and Providers
=======================

This cookbook provides three resources and corresponding providers.

## microsoft_azure_storage_account


Manage Azure Storage Accounts with this resource.

Actions:

* `create` - create a new storage account
* `delete` - delete the specified storage account

Attribute Parameters:

* `management_certificate` - PEM file contents of Azure management
  certificate, required.
* `subscription_id` - ID of Azure subscription, required.
* `management_endpoint` - Endpoint for Azure API, defaults to
  `management.core.windows.net`.
* `location` - Azure location to create storate account. Either
  location or affinity group are required.
* `affinity_group_name` - Affinity group to create account in. Either
  location or affinity group are required.
* `geo_replication_enabled` - True or false, defaults to true.

## microsoft_azure_storage_container

Manage Azure Blob Containers with this resource

Actions:

* `create` - create a new container
* `delete` - delete the specified container

Attribute Parameters:

* `storage_account` - Account to create container in, required.
* `access_key` - Access key for storage account, required.

## microsoft_azure_sql_db_server

Actions:

* `create` - create a new server. Use the Azure location as the `name`
  of the storage account. The server name is autogenerated.

Attribute Parameters:

* `management_certificate` - PEM file contents of Azure management
  certificate, required.
* `subscription_id` - ID of Azure subscription, required.
* `management_endpoint` - Endpoint for Azure API, defaults to
  `management.database.windows.net`.
* `login` - Desired admin login for db server, required.
* `password` - Desired admin password for db server, required.
* `server_name` - This attribute is set by the provider, and can be
  used by consuming recipies.

## microsoft_azure_protected_file

This resource is a wrapper around the core remote_file resource that will generate an expiring link for you to retrieve your file from protected blob storage.

Actions:

* `create` - create the file
* `create_if_missing` - create the file if it does not already exist. default
* `delete` - delete the file
* `touch` - touch the file

Attribute Parameters:

* `storage_account` - the azure storage account you are accessing
* `access_key` - the access key to this azure storage account
* `path` - where this file will be created on the machine. name attribute
* `remote_path` - the url to the file you are trying to retrieve

The following parameters are inherited from the [remote_file](https://docs.chef.io/resource_remote_file.html) resource.

* `owner`
* `group`
* `mode`
* `checksum`
* `backup`
* `inherits`
* `rights`

Example:

```ruby
microsoft_azure_protected_file '/tmp/secret_file.jpg' do
  storage_account 'secretstorage'
  access_key 'eW91cmtleWluYmFzZTY0.....'
  remote_path 'https://secretstorage.blob.core.windows.net/images/secret_file.jpg'
end
```

Usage
=====

The following examples assume that the recommended data bag item has
been created and that the following has been included at the top of
the recipe where they are used.

    include_recipe "microsoft_azure"
    microsoft_azure = data_bag_item("microsoft_azure", "main")

## microsoft_azure_storage_account

This will create an account named `new-account` in the `West US`
location.

    microsoft_azure_storage_account 'new-account' do
      management_certificate microsoft_azure['management_certificate']
      subscription_id microsoft_azure['subscription_id']
      location 'West US'
      action :create
    end

This will create an account named `new-account` in the existing
`my-ag` affinity group.

    microsoft_azure_storage_account 'new-account' do
      management_certificate microsoft_azure['management_certificate']
      subscription_id microsoft_azure['subscription_id']
      affinity_group_name 'my-ag'
      action :create
    end

## microsoft_azure_storage_container

This will create a container named `my-node` within the storage
account `my-account`.

    microsoft_azure_storage_container 'my-node' do
      storage_account 'my-account'
      access_key microsoft_azure['access_key']
      action :create
    end

## microsoft_azure_sql_db_server

This will create a db server in the location `West US` with the login
`admin` and password `password`.

    microsoft_azure_sql_db_server 'West US' do
      management_certificate microsoft_azure['management_certificate']
      subscription_id microsoft_azure['subscription_id']
      login 'admin'
      password 'password'
      action :create
    end

Here is an example of how you might retrieve the generated server
name.

    file '/etc/db_server_info' do
      content lazy {
        db2 = resources("microsoft_azure_sql_db_server[West US]")
        "Url: https://#{db2.server_name}.database.windows.net"
      }
      mode 0600
      action :create
    end

## vm_extension_removal_policy
This resource can be used to configure the behavior of `chef-extension` on removal. This resource should be run as the root user since it modifies the extension config file.

User needs to set environment variable `EXTENSION_PATH` pointing to the extension root path e.g. `C:/Packages/Plugins/Chef.Bootstrap.WindowsAzure.ChefClient/<extension_version>` for windows or `/var/lib/waagent/Chef.Bootstrap.WindowsAzure.LinuxChefClient-<extension_version>` for linux.

NOTE: `EXTENSION_PATH` may vary from the examples mentioned above.

Actions:

* `set` - sets the configuration in the extension config file.

Attribute Parameters:

* `uninstall_chef_client` - true/false (decides whether chef-client should be uninstalled on extension uninstall). Default value is false
* `delete_chef_config` - true/false (decides whether to delete chef extension configuration file on extension uninstall). Default value is false

Example:

```ruby
azure_cookbook_vm_extension_removal_policy "set policy" do
  uninstall_chef_client false
  delete_chef_config true
end
```

Helpers
=======

# vault_secret

This helper will allow you to retrieve a secret from an azure keyvault.

```ruby
spn = {
  'tenant_id' => '11e34-your-tenant-id-1232',
  'client_id' => '11e34-your-client-id-1232',
  'secret' => 'your-client-secret'
}

super_secret = vault_secret(<vault_name>, <secret_name>, spn)

file '/etc/config_file' do
  content "password = #{super_secret}"
end
```

License and Author
==================

* Author:: Jeff Mendoza (<jemendoz@microsoft.com>)
* Author:: Andre Elizondo (<andre@chef.io>)

Copyright (c) Microsoft Open Technologies, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
