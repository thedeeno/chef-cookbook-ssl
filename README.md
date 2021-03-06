Description
===========

Cookbook to manage deployment of X509 certificates across an
infrastructure.

Keys and CSRs are generated according to "ssl_certificate" resources
during a chef-client run on the hosts which will use them, with the
CSRs pushed to the Chef server. Later, certificates are signed by the
appropriate CAs and pushed to Chef. A subsequent run deploys the
signed certificate.

Temporary certificates are issued to enable services to start while
the CSRs are waiting to be processed.

Generated keys are encrypted to a key-vault key, and pushed to Chef
for backup.

Requirements
============

Chef Server

"gpg" Cookbook

"eassl2", "gpgme" gems

GnuPG 2.x

Attributes
==========

GPG is used to encrypt generated keys for archival purposes.

`node['ssl']['key_vault']` - the email address of the GPG/PGP key.

DN components to use when creating certificate names:

 * `node['ssl']['country']`
 * `node['ssl']['state']`
 * `node['ssl']['city']`
 * `node['ssl']['organization']`
 * `node['ssl']['department']`
 * `node['ssl']['email']`

Usage
=====

    include_recipe "ssl"

Webserver SSL certificate:

    ssl_certificate "www.example.com" do
      ca "MyCA"
      key "/etc/ssl/www.example.com.key"
      certificate "/etc/ssl/www.example.com.cert"
    end

Webserver SSL certificate specifying key size and validity period:

    ssl_certificate "www.example.com" do
      ca "MyCA"
      key "/etc/ssl/www.example.com.key"
      certificate "/etc/ssl/www.example.com.cert"
      bits 1024
      days 365
    end

REST API Server Certificate, with CA Certificate:

    ssl_certificate "service.example.com" do
      ca "Service-CA"
      key "/etc/ssl/service.example.com.key"
      certificate "/etc/ssl/service.example.com.cert"
      cacertificate "/etc/ssl/service_cacert"
    end

REST API Client Certificate:

    ssl_certificate "service-#{node['fqdn']}" do
      ca "Service-CA"
      type "client"
      key "/etc/service/client_key.pem"
      certificate "/etc/service/client_cert.pem"
    end

CA Certificate only, for verification:

    ssl_ca_certificate "My-CA" do
      cacertificate "/etc/myca.pem"
    end

Signing Client
==============

A client is provided which allows a user to search Chef for
outstanding CSRs, sign them, and create data bag items containing the
new certificates.

The client is available in this repository, in the `client-gem`
directory, and may be packaged as a gem for installation.

A number of gems are required, and a Gemfile is provided for Bundler's
use.

It also needs access to a Chef admin client key, which may be your own
client, and to your ~/.chef/knife.rb which configures the server, key
path and client name.

There are a number of  modes of operation:

 * Find CSRs to be signed by a specific CA, and sign them with that CA.

 * Issue an adhoc certificate to a specific DN.

 * Create a new CA.

 * Find all CSRs awaiting action.

 * Find a specific CSR and provide its signed cert - intended for externally-signed CSRs,
   such as the public SSL certificate providers.

See the chef-ssl program's embedded help text for options:

    $ chef-ssl
    chef-ssl

    Chef-automated SSL certificate signing tool

    Commands:
      autosign             Search for CSRs and sign them with the given CA
      help                 Display global or [command] help documentation.
      issue                Issue an ad hoc certificate
      makeca               Creates a new CA
      search               Searches for outstanding CSRs
      sign                 Search for the given CSR by name and provide a signed certificate

    Global Options:
      -h, --help           Display help documentation
      -v, --version        Display version information
      -t, --trace          Display backtrace when an error occurs


Workflow
========

1) Use the `ssl_certificate` resource in a recipe, and run chef-client
on the node.  The first converge of the resource does the following:

 * Creates a new key, with no passphrase.
 * Generates and installs a certificate, signed by an ephemeral CA.
 * Creates a CSR, which is placed in `node[:csr_outbox]`.

2) Use the `chef-ssl` tool to find and process pending CSRs.  The
signed certificate is placed into a databag item.

3) Run chef-client on the node again.  This converge does the
following:

 * Retrieves the certificate databag item.
 * Removes the corresponding entry from `node[:csr_outbox]`.
 * Installs the signed certificate from the databag.


FAQ
===

Q) Can I get my CSR signed by a commercial Certificate Authority?

A) Yes - use the `chef-ssl sign` command to retrieve the CSR, and to
supply the text of the signed certificate.

Q) My certificate is about to expire - how can I generate a new CSR?

A) Remove the databag item for the certificate.  The next time
chef-client is run on the node, a new CSR will be placed in
node[:csr_outbox].  The existing key and certificate will not be
touched.


TODO
====



Licence and Author
==================

Author:: Chris Andrews (<candrews@venda.com>)
Author:: Zac Stevens (<zts@cryptocracy.com>)

Copyright 2011-2012 Venda Ltd

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
