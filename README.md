# ansible-role-webserver

Ansible role that automatically deploys **WordPress** with **MariaDB** and **Apache/httpd** on **Ubuntu** (Debian family) and **Rocky Linux** (RedHat family), including containerised environments where systemd is not available.

---

## Requirements

| Dependency | Purpose |
|---|---|
| Ansible ≥ 2.14 | Role engine |
| `community.mysql` collection | MariaDB modules |

Install the collection before running the role:

```bash
ansible-galaxy collection install community.mysql
```

---

## Role variables

### `defaults/main.yml` – overridable by the user

| Variable | Default | Description |
|---|---|---|
| `wp_db_name` | `wordpress` | WordPress database name |
| `wp_db_user` | `wpuser` | WordPress database user |
| `wp_db_password` | `wppassword` | WordPress database password |
| `wp_db_host` | `localhost` | Database host |
| `mysql_root_password` | `rootpassword` | MariaDB root password |
| `wp_download_url` | `https://wordpress.org/latest.zip` | WordPress download URL |
| `wp_install_dir` | `/var/www/html` | Web root path |
| `apache_server_admin` | `admin@localhost` | VirtualHost ServerAdmin |
| `apache_server_name` | `localhost` | VirtualHost ServerName |

### `vars/Debian.yml` / `vars/RedHat.yml` – OS-specific (do not override)

Loaded automatically via `include_vars` based on `ansible_os_family`.

| Variable | Debian value | RedHat value |
|---|---|---|
| `apache_service` | `apache2` | `httpd` |
| `http_packages` | apt package list | dnf package list |
| `apache_conf_dir` | `/etc/apache2/sites-available` | `/etc/httpd/conf.d` |
| `apache_user` / `apache_group` | `www-data` | `apache` |
| `mysql_socket` | `/var/run/mysqld/mysqld.sock` | `/var/lib/mysql/mysql.sock` |
| `use_a2ensite` | `true` | `false` |

---

## Role structure

```
ansible-role-webserver/
├── defaults/main.yml          # User-overridable defaults
├── vars/
│   ├── main.yml               # Placeholder / documentation
│   ├── Debian.yml             # Ubuntu/Debian OS vars (auto-loaded)
│   └── RedHat.yml             # Rocky/RHEL OS vars (auto-loaded)
├── tasks/
│   ├── main.yml               # Orchestrator – loads vars + calls sub-tasks
│   ├── install.yml            # Package installation (apt / dnf)
│   ├── mariadb.yml            # MariaDB start + DB/user creation
│   ├── wordpress.yml          # WP download, extract, wp-config
│   └── apache.yml             # VirtualHost deployment + web-server start
├── handlers/main.yml          # Restart / Reload handlers (Debian & RedHat)
├── templates/
│   ├── wp-config.php.j2       # WordPress configuration file
│   └── wordpress.conf.j2      # Apache/httpd VirtualHost
├── tests/
│   ├── inventory              # Docker container inventory
│   └── test.yml               # Test playbook
└── meta/main.yml              # Galaxy metadata
```

---

## Handlers

| Listen name | Action |
|---|---|
| `Restart webserver` | `service apache2 restart` (Debian) / `pkill -HUP httpd` (RedHat) |
| `Reload webserver` | `service apache2 reload` (Debian) / `httpd -k graceful` (RedHat) |

Handlers are triggered when the VirtualHost config is deployed or `a2enmod`/`a2ensite` runs.

---

## Idempotence

- Package installation uses `state: present` (apt/dnf).
- MariaDB is only started if its socket does not already exist (`creates:` guard).
- WordPress files are only downloaded/copied if `wp-login.php` is absent from the web root.
- `a2enmod`/`a2ensite` use `creates:` to detect if the symlink already exists.
- `wp-config.php` is deployed via template (Ansible detects changes automatically).

---

## Example playbook

```yaml
- name: Deploy WordPress stack
  hosts: webservers
  become: true
  vars:
    mysql_root_password: "S3cur3R00t!"
    wp_db_password: "S3cur3WP!"
    wp_db_user: "wpuser"
    wp_db_name: "wordpress"
    apache_server_name: "mysite.example.com"
  roles:
    - role: EliasMeh.ansible-role-webserver
```

---

## Testing with the Docker environment

Start the containers from `evaluation-1/`:

```bash
docker compose up -d
```

Then run the test playbook from the `ansible` container (or directly if your SSH keys are configured):

```bash
ansible-playbook -i ansible-role-webserver/tests/inventory \
                    ansible-role-webserver/tests/test.yml
```

WordPress will be reachable at:
- Ubuntu client1 → http://localhost:8083
- Ubuntu client2 → http://localhost:8084
- Rocky  client3 → http://localhost:8085
- Rocky  client4 → http://localhost:8086

---

## Publishing to Ansible Galaxy

1. Push the role to a **GitHub repository** named `ansible-role-webserver`.
2. Log in at [galaxy.ansible.com](https://galaxy.ansible.com) with your GitHub account.
3. Go to **My Content → Add Content → Import Role from GitHub**.
4. Select your repository and click **Import**.
5. The role will be available as `EliasMeh.ansible-role-webserver`.

---

## License

MIT

## Author

EliasMeh
