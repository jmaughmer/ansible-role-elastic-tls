# ansible-role-elastic-tls

[![CI](https://github.com/jmaughmer/ansible-role-elastic-tls/actions/workflows/ci.yml/badge.svg)](https://github.com/jmaughmer/ansible-role-elastic-tls/actions/workflows/ci.yml)

This Ansible role creates a private certificate authority with `elasticsearch-certutil`, issues one PEM certificate per host, and configures Elastic Stack components to use them:

- **Elasticsearch**: transport (node-to-node) and HTTP TLS in `elasticsearch.yml`
- **Kibana**: HTTPS for the Kibana server, plus trust of the CA for connections to Elasticsearch, in `kibana.yml`
- **Logstash, Elastic Agent / Fleet Server, Filebeat, Metricbeat**: the role installs the CA, certificate and key, and you reference them from your own pipeline, agent or Beat configuration

## How it works

1. On the CA host (`elastic_tls_ca_host`, by default the first host in the `elasticsearch` group), the role runs `elasticsearch-certutil ca --pem` once. The CA is stored in `elastic_tls_ca_dir` (`/etc/elastic-tls`).
2. For each host with components, the role writes a request spec (name, DNS/IP SANs, validity) on the CA host and runs `elasticsearch-certutil cert --pem`. If the spec changes, for example because a SAN was added, the certificate is reissued.
3. The CA certificate, host certificate and key are copied into each component's `cert_dir` as `elastic-tls-ca.crt`, `elastic-tls.crt` and `elastic-tls.key`. The key is mode `0640`, owned by `root:<component group>`.
4. Elasticsearch and Kibana configuration is updated, and the affected services are restarted.

Back up the CA host's `elastic_tls_ca_dir`. If you lose `ca/ca.key`, you have to reissue every certificate from a new CA.

## Requirements

- ansible-core 2.15 or later. The role uses only `ansible.builtin` modules.
- Targets: RHEL, Rocky Linux or AlmaLinux 8 or 9.
- Install the `elasticsearch` package on the CA host first, because it provides `elasticsearch-certutil`.
- Install each component's package before running the role, so its service group (for example `elasticsearch` or `kibana`) already exists.
- Run the role with `become: true` and gather facts, because the default SANs come from `ansible_facts`.

## Role variables

See [defaults/main.yml](defaults/main.yml) for the full list. The variables you'll most often set are:

| Variable | Default | Description |
| --- | --- | --- |
| `elastic_tls_components` | `[]` | Components on this host: `elasticsearch`, `kibana`, `logstash`, `elastic-agent`, `filebeat`, `metricbeat`. |
| `elastic_tls_ca_host` | first host in `elasticsearch` | Host that keeps the CA and runs certutil. |
| `elastic_tls_ca_dir` | `/etc/elastic-tls` | CA and certificate bundle directory on the CA host. |
| `elastic_tls_ca_key_password` | `""` | Optional passphrase for the CA key. Store it in Vault. |
| `elastic_tls_ca_days` / `elastic_tls_cert_days` | `3650` / `1095` | Validity periods. |
| `elastic_tls_cert_extra_dns` / `elastic_tls_cert_extra_ip` | `[]` | Extra SANs, such as load balancer names or VIPs. Inventory name, FQDN, short hostname and default IPv4 are always included. |
| `elastic_tls_cert_regenerate` | `false` | Set to `true` for one run to reissue every certificate. |
| `elastic_tls_component_settings` | see defaults | Per-component `cert_dir`, `owner`, `group` and `service`. |
| `elastic_tls_restart_services` | `true` | Restart component services when their TLS files or config change. |
| `elastic_tls_fetch_ca_cert` | `""` | Controller path to save the CA certificate to, for clients. |
| `elastic_tls_elasticsearch_transport_verification_mode` | `full` | `full`, `certificate` or `none`. |
| `elastic_tls_elasticsearch_http_ssl` | `true` | Enable HTTPS on the REST API. |
| `elastic_tls_elasticsearch_remove_autoconfig` | `true` | Remove the TLS settings and keystore passwords left by security auto-configuration. |
| `elastic_tls_kibana_server_ssl` | `true` | Serve Kibana over HTTPS. |
| `elastic_tls_kibana_elasticsearch_verification_mode` | `full` | How Kibana verifies Elasticsearch certificates. |

Set `elastic_tls_elasticsearch_manage_config` or `elastic_tls_kibana_manage_config` to `false` if you only want the files deployed and manage the configuration yourself.

### Elasticsearch security auto-configuration

When the Elasticsearch 8+ package is installed, it writes `xpack.security.{http,transport}.ssl` blocks that point at PKCS#12 keystores, and stores their passwords in `elasticsearch.keystore`. With `elastic_tls_elasticsearch_remove_autoconfig: true`, the role removes only those nested SSL blocks and keystore passwords. Other auto-configured settings, such as `xpack.security.enabled`, `http.host` and `cluster.initial_master_nodes`, are left alone. The role's own settings go in a marked block, in flat-key form.

If you have hand-written flat `xpack.security.*.ssl.*` lines outside the managed block, remove them. Otherwise Elasticsearch will refuse to start because of duplicate settings.

### Restarts

The handler restarts every changed host's services at once. Changing transport certificates on a running cluster this way restarts all nodes together. For a rolling change, run the role with `serial: 1`, or set `elastic_tls_restart_services: false` and restart the nodes yourself.

## Example playbook

```yaml
# inventory/group_vars/elasticsearch.yml
elastic_tls_components: [elasticsearch]

# inventory/group_vars/kibana.yml
elastic_tls_components: [kibana]

# inventory/group_vars/logstash.yml
elastic_tls_components: [logstash]

# inventory/group_vars/fleet_server.yml
elastic_tls_components: [elastic-agent]
elastic_tls_cert_extra_dns: [fleet.example.com]
```

```yaml
- name: Configure Elastic Stack TLS
  hosts: elasticsearch:kibana:logstash:fleet_server
  become: true
  roles:
    - role: jmaughmer.elastic_tls
      vars:
        elastic_tls_fetch_ca_cert: "{{ playbook_dir }}/files/elastic-ca.crt"
```

For components the role does not configure, point them at the deployed files. For example, a Logstash output:

```text
output {
  elasticsearch {
    hosts => ["https://es01.example.com:9200"]
    ssl_certificate_authorities => ["/etc/logstash/certs/elastic-tls-ca.crt"]
  }
}
```

A Fleet Server install would use `--fleet-server-cert /etc/elastic-agent/certs/elastic-tls.crt --fleet-server-cert-key /etc/elastic-agent/certs/elastic-tls.key --certificate-authorities /etc/elastic-agent/certs/elastic-tls-ca.crt`.

## Testing

```bash
pip install ansible-core molecule "molecule-plugins[docker]" docker ansible-lint yamllint
yamllint . && ansible-lint
molecule test
```

The Molecule scenario installs Elasticsearch on `es01` (its CA host) and uses a Kibana/Logstash stand-in, `kb01`. It then verifies the certificate chain, SANs, key permissions, the auto-configuration cleanup and the resulting `elasticsearch.yml` and `kibana.yml`. Set `MOLECULE_DISTRO=rockylinux8` to test EL8.

## License

MIT
