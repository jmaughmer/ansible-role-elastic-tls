# ansible-role-elastic-tls

This Ansible role issues TLS certificates for Elastic Stack hosts, from a private CA it creates and keeps or from Let's Encrypt, and publishes them as the host fact `elastic_tls_ssl_certificates`. The [elasticsearch](https://github.com/jmaughmer/ansible-role-elasticsearch), [kibana](https://github.com/jmaughmer/ansible-role-kibana), [logstash](https://github.com/jmaughmer/ansible-role-logstash) and [elastic_fleet](https://github.com/jmaughmer/ansible-role-elastic-fleet) roles read that fact, install the files and configure TLS.

The role only writes certificates and keys below `elastic_tls_dir` (`/etc/pki/elastic-tls`), plus the CA on the CA host. It never edits `elasticsearch.yml`, `kibana.yml` or any other Elastic configuration, and never restarts a service: the consumer roles do that, in health-checked batches.

## Quick start

```yaml
- name: Elasticsearch
  hosts: elasticsearch
  become: true
  roles:
    - role: elastic_tls        # first, in the same play
    - role: elasticsearch
```

```yaml
# group_vars/all/elastic-tls.yml
elastic_tls_provider: ca
elastic_tls_ca_common_name: "Example Corp Internal CA"
elastic_tls_ca_passphrase: "{{ vault_elastic_tls_ca_passphrase }}"
```

With nothing else set, every host gets one certificate named after its inventory hostname, with its inventory name, FQDN, short hostname and default IPv4 address as subjectAltNames, for both server and client authentication. That is what the consumer roles look for by default.

Add names, for example a load balancer or virtual address, with `elastic_tls_certificates`:

```yaml
# group_vars/fleet_server/main.yml
elastic_tls_certificates:
  - name: "{{ inventory_hostname }}"
    dns:
      - "{{ inventory_hostname }}"
      - fleet.example.com
    usage: [server, client]
```

It works on a bare host. Nothing Elastic needs to be installed first, so the role can run before the elasticsearch role on a new cluster.

## What gets published

After the role, each host has:

```yaml
elastic_tls_ssl_certificates:
  - name: es-data-1.example.com
    certificate: {path: /etc/pki/elastic-tls/certs/es-data-1.example.com.crt}
    key: {path: /etc/pki/elastic-tls/private/es-data-1.example.com.key}
    ca: {path: /etc/pki/elastic-tls/certs/ca.crt}
```

one entry per item in `elastic_tls_certificates`, and, with the private CA, `elastic_tls_ca_certificate` (the CA certificate as PEM text). The consumer roles read `ssl_certificates` first and `elastic_tls_ssl_certificates` after it, so a manually supplied entry with the same name always wins.

| File | Owner, mode | Content |
| --- | --- | --- |
| `certs/<name>.crt` | `root:root 0644` | The certificate (with Let's Encrypt, the full chain) |
| `private/<name>.key` | `root:root 0600` | Unencrypted PKCS#8 private key, made on the host |
| `certs/ca.crt` | `root:root 0644` | The private CA's certificate. Also installed on hosts without certificates of their own unless `elastic_tls_install_ca: false` |

## How it works

### Private CA (`elastic_tls_provider: ca`)

1. On `elastic_tls_ca_host` (by default the first host in the `elasticsearch` group; it does not have to be in the play), the role creates the CA key and a self-signed CA certificate once, in `elastic_tls_ca_dir` (`/etc/elastic-tls`). The key is PKCS#8, encrypted with `elastic_tls_ca_passphrase` when one is set. An existing CA is never replaced.
2. Each host creates its own key and a certificate request. The CA host signs the request and only the certificate is copied back, so host keys never leave their host.
3. A certificate is reissued when its names or usages change, when it was not signed by the current CA, when its key changes, or when it expires within `elastic_tls_renew_before_days` (30). Otherwise nothing changes.

Back up `elastic_tls_ca_dir` on the CA host. Without `ca/ca.key`, every certificate has to be reissued from a new CA.

Node certificates have `serverAuth` and `clientAuth`, `CA:FALSE`, and `digitalSignature, keyEncipherment` key usage, which Elasticsearch transport TLS needs.

### Let's Encrypt (`elastic_tls_provider: letsencrypt`)

The role requests certificates with `community.crypto.acme_certificate` and an account key per host:

- **http-01**: challenge files are written to `elastic_tls_acme_webroot` on the host; a web server there must serve them on port 80 for every name in the certificate.
- **dns-01**: set `elastic_tls_acme_dns_tasks` to a tasks file of yours. It is included with `elastic_tls_acme_dns_records` (`{fqdn: [txt values]}`) and `elastic_tls_acme_dns_state: present` before validation, and again with `absent` afterwards.

You have to set `elastic_tls_acme_terms_agreed: true`. Let's Encrypt certificates are for servers only: they carry no `clientAuth` and no IP addresses, so the role refuses `usage: [client]` and `ip` entries with this provider. Elasticsearch node (transport) certificates have to come from the private CA; Let's Encrypt suits the Kibana or Fleet Server address that browsers and agents connect to. Each entry's `ca` is the system trust store, `elastic_tls_acme_ca_path` (`/etc/pki/tls/certs/ca-bundle.crt` on Enterprise Linux, `/etc/ssl/certs/ca-certificates.crt` on Ubuntu).

The Let's Encrypt path is not exercised by the Molecule tests.

## Requirements

- ansible-core 2.15 or later and the `community.crypto` collection 2.10 or later (`ansible-galaxy collection install -r requirements.yml`).
- Targets and the CA host: RHEL, Rocky Linux or AlmaLinux 9 or 10, or Ubuntu 24.04 or 26.04. The role installs `python3-cryptography` on them (`elastic_tls_packages`) with dnf or apt; change that list if `ansible_python_interpreter` is not the platform Python.
- Run with `become: true`, and gather facts unless every certificate lists its own `dns` and `ip`.

## Role variables

See [defaults/main.yml](defaults/main.yml) for all of them.

| Variable | Default | Description |
| --- | --- | --- |
| `elastic_tls_provider` | `ca` | `ca` or `letsencrypt`. |
| `elastic_tls_certificates` | one entry named after the host | Certificates for this host: `name`, `dns`, `ip`, `usage` (`server`, `client`), `days`. Missing keys take the defaults below. `[]` installs only the CA certificate. |
| `elastic_tls_default_dns` / `elastic_tls_default_ip` | inventory name, FQDN, short hostname / default IPv4 | SANs for entries without `dns` / `ip`. |
| `elastic_tls_dir` | `/etc/pki/elastic-tls` | Where certificates and keys are written on each host. |
| `elastic_tls_key_type` | `RSA` | `RSA` (`elastic_tls_key_size`, 2048) or `ECC` (`elastic_tls_key_curve`, `secp256r1`). |
| `elastic_tls_cert_days` | `1095` | Validity of private-CA certificates. |
| `elastic_tls_renew_before_days` | `30` | Reissue certificates that expire sooner than this. |
| `elastic_tls_ca_host` | first host in `elasticsearch` | Host that keeps the CA key and signs. |
| `elastic_tls_ca_dir` | `/etc/elastic-tls` | CA directory on the CA host (`ca/ca.crt`, `ca/ca.key`); see also `elastic_tls_ca_certificate_path` and `elastic_tls_ca_privatekey_path`. |
| `elastic_tls_ca_common_name` | `Elastic Stack CA` | Subject of a new CA. |
| `elastic_tls_ca_passphrase` | `""` | Passphrase of the CA key. Store it in Vault. |
| `elastic_tls_ca_days` | `3650` | Validity of a new CA certificate. |
| `elastic_tls_fetch_ca_cert` | `""` | Controller path to save the CA certificate to, for clients. |
| `elastic_tls_acme_*` | see defaults | Directory URL, contact e-mail, terms, challenge type, webroot, DNS tasks and CA bundle for Let's Encrypt. |

## Using the certificates elsewhere

For software without a consumer role, point it at the published paths, for example a Logstash pipeline output:

```text
output {
  elasticsearch {
    hosts => ["https://es01.example.com:9200"]
    ssl_certificate_authorities => ["/etc/pki/elastic-tls/certs/ca.crt"]
  }
}
```

The key is readable by root only. A service running as another user needs its own copy with suitable ownership; the consumer roles make one.

## Migrating from the certutil-based role

Earlier versions of this role ran `elasticsearch-certutil`, copied files into each component's `certs` directory and edited `elasticsearch.yml` and `kibana.yml`. That conflicted with the elasticsearch and kibana roles, which template those files completely (and remove `/etc/elasticsearch/certs`), and its handler restarted every node at once. This version fails with a message naming any of the old variables that is still set, so an unmigrated inventory cannot silently stop getting certificates.

1. **Keep the CA.** The CA paths are unchanged (`/etc/elastic-tls/ca/ca.crt` and `ca.key` on `elastic_tls_ca_host`, which still defaults to the first `elasticsearch` host), and an existing CA is never replaced, so existing trust keeps working. Set `elastic_tls_ca_passphrase` to the old `elastic_tls_ca_key_password`, or leave it empty if there was none. `elastic_tls_ca_common_name` only applies to a new CA.
2. **Rename or remove the old variables:**

   | Old | New |
   | --- | --- |
   | `elastic_tls_ca_key_password` | `elastic_tls_ca_passphrase` |
   | `elastic_tls_cert_name` | `name` in `elastic_tls_certificates` |
   | `elastic_tls_cert_extra_dns`, `elastic_tls_cert_extra_ip`, `elastic_tls_cert_dns`, `elastic_tls_cert_ip` | `dns` and `ip` in `elastic_tls_certificates`. A list you give replaces the defaults, so include the host's own names. |
   | `elastic_tls_cert_regenerate` | Not needed: certificates are reissued when they change or near expiry. Delete a certificate file to force it. |
   | `elastic_tls_components`, `elastic_tls_component_settings`, `elastic_tls_*_filename`, `elastic_tls_certutil` | Remove. Each consumer role installs the files itself. |
   | `elastic_tls_restart_services` | Remove. The consumer roles restart in batches. |
   | `elastic_tls_elasticsearch_*`, `elastic_tls_kibana_*` | Remove, and set TLS in the elasticsearch and kibana roles' own variables. Their templates replace the blocks this role wrote. |

   `elastic_tls_ca_host`, `elastic_tls_ca_dir`, `elastic_tls_ca_days`, `elastic_tls_cert_days` and `elastic_tls_fetch_ca_cert` keep their meaning.
3. **Run `elastic_tls` before the consumer role in the same play**, as in the quick start. Every host gets a new key and certificate from the same CA, and the consumer role rolls them out with its usual batched restart.
4. **Clean up.** Files under `/etc/<component>/certs/elastic-tls*` and `/etc/elastic-tls/specs` and `/etc/elastic-tls/certs` on the CA host are no longer used or renewed. Remove them once nothing points at them; Logstash pipelines, Beats or agents configured by hand need their paths changed to `/etc/pki/elastic-tls/...` first.

## Testing

```bash
pip install -r molecule/requirements.txt
yamllint . && ansible-lint
molecule test                              # MOLECULE_DISTRO=rockylinux10 for EL 10
```

[`molecule/default/`](molecule/default/) runs on three bare containers with rootless Podman: `es01` (in `elasticsearch`, so the CA host), `kb01` with two certificates, one of them server-only with its own names, and `client01` with none. Nothing Elastic is installed; [`prepare.yml`](molecule/default/prepare.yml) only adds the `openssl` command for verify. The sequence is converge, idempotence, check mode, then [`verify.yml`](molecule/default/verify.yml), which checks:

- the published `elastic_tls_ssl_certificates` per host: names, and certificate, key and CA paths, and that it passes the elasticsearch role's preflight checks;
- every certificate chains to the CA, its key is unencrypted PKCS#8, matches it and is `root:root 0600`;
- extended key usage (server and client by default, server only where asked), `CA:FALSE`, default and explicit SANs;
- the CA certificate is installed on every host, and only the CA host has the CA key, encrypted;
- nothing was created under `/etc/elasticsearch`, `/etc/kibana` or `/etc/logstash`;
- changing a certificate's names reissues it;
- a variable of the certutil-based role makes the role fail.

### On KVM virtual machines

[`molecule/kvm/`](molecule/kvm/) runs the default scenario on real AlmaLinux 10 virtual machines
instead of containers: the same hosts, groups, playbooks and inventory, one VM per host, on local
KVM/libvirt (`qemu:///system`). Each VM has its own kernel, so SELinux is enforcing and settings such
as `vm.max_map_count` are the VM's own. Overrides for the VMs go in
`molecule/kvm/inventory/group_vars/<group>/kvm.yml`.

[`molecule/kvm-ubuntu2404/`](molecule/kvm-ubuntu2404/) and
[`molecule/kvm-ubuntu2604/`](molecule/kvm-ubuntu2604/) run the same thing on Ubuntu 24.04 and 26.04
VMs. They only swap the cloud image, and share `molecule/kvm/`'s playbooks and inventory.

```bash
for s in kvm kvm-ubuntu2404 kvm-ubuntu2604; do molecule test -s "$s" || break; done
```

The machine running Molecule needs libvirt, `virt-install`, `qemu-img` and `xorriso`, and its user
must be able to manage `qemu:///system`. Each cloud image downloads once, checksum verified, to
`/var/lib/libvirt/images/molecule`; each VM gets a throwaway disk on top of it. VM names include the
scenario, so the three scenarios can also run side by side if the machine has the memory.

## License

MIT
