# deregister-node Razor Hook

This hook uses the `node-delete` and `node-unbound-from-policy` events to revoke/delete the Puppet Agent certificates
from the Puppet CA using the REST API.

It also tries to disable the node in PuppetDB so that any exported resources can be purged outmatically without having
to wait for the `node_ttl` setting to kick-in.

This hook requires that the Razor server certificate is in the Puppet CA and PuppetDB white lists. To do this, add
these lines to the relevant Hiera data file (e.g., `common.yaml`) replacing 'razor-server' with the certificate name(s)
of your Razor server(s).

```yaml
puppet_enterprise::profile::certificate_authority::client_whitelist: ["razor-server"]
puppet_enterprise::profile::puppetdb::whitelisted_certnames: ["razor-server"]
```

Since Razor runs as a non-privileged user, it is also necessary to copy the agent certificates to a place the Razor
service can access them. You can use the following Puppet code to automate it:

```puppet
$razor_ssldir = '/etc/puppetlabs/razor-server/ssl'

File {
  owner  => 'pe-razor',
  group  => 'pe-razor',
  mode   => '0644',
}

file { $razor_ssldir:
  ensure => directory,
}

file { "${razor_ssldir}/ca.pem":
  ensure => file,
  source => "${settings::localcacert}",
}

file { "${razor_ssldir}/client_cert.pem":
  ensure => file,
  source => "${settings::hostcert}",
}

file { "${razor_ssldir}/client_key.pem":
  ensure => file,
  source => "${settings::hostprivkey}",
  mode   => '0600',
}
```

To use the hook, install it to `/etc/puppetlabs/razor-server/hooks` and then create a new instance using the `razor`
command line tool.

```sh
razor create-hook --type deregister-node --name deregister-node \
  --config ca_server=$YOUR_CA_HOSTNAME \
  --config puppetdb_server $YOUR_PUPPETDB_HOSTNAME
```

The hook supports several configuration options, see `configuration.yaml` for the list of configuration parameters
available and their default values.

