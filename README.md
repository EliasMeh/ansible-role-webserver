# ansible-role-webserver

Rôle Ansible qui déploie automatiquement **WordPress** avec **MariaDB** et **Apache/httpd** sur **Ubuntu** (famille Debian) et **Rocky Linux** (famille RedHat), y compris dans des environnements conteneurisés où systemd n'est pas disponible.

---

## Prérequis

| Dépendance | Objectif |
|---|---|
| Ansible ≥ 2.14 | Moteur du rôle |
| Collection `community.mysql` | Modules MariaDB |

Installez la collection avant d'exécuter le rôle :

```bash
ansible-galaxy collection install community.mysql
```

---

## Variables du rôle

### `defaults/main.yml` – modifiables par l'utilisateur

| Variable | Défaut | Description |
|---|---|---|
| `wp_db_name` | `wordpress` | Nom de la base de données WordPress |
| `wp_db_user` | `wpuser` | Utilisateur de la base de données WordPress |
| `wp_db_password` | `wppassword` | Mot de passe de la base de données WordPress |
| `wp_db_host` | `localhost` | Hôte de la base de données |
| `mysql_root_password` | `rootpassword` | Mot de passe root MariaDB |
| `wp_download_url` | `https://wordpress.org/latest.zip` | URL de téléchargement WordPress |
| `wp_install_dir` | `/var/www/html` | Chemin de la racine web |
| `apache_server_admin` | `admin@localhost` | ServerAdmin du VirtualHost |
| `apache_server_name` | `localhost` | ServerName du VirtualHost |

### `vars/Debian.yml` / `vars/RedHat.yml` – spécifiques à l'OS (ne pas modifier)

Chargées automatiquement via `include_vars` selon `ansible_os_family`.

| Variable | Valeur Debian | Valeur RedHat |
|---|---|---|
| `apache_service` | `apache2` | `httpd` |
| `http_packages` | liste de paquets apt | liste de paquets dnf |
| `apache_conf_dir` | `/etc/apache2/sites-available` | `/etc/httpd/conf.d` |
| `apache_user` / `apache_group` | `www-data` | `apache` |
| `mysql_socket` | `/var/run/mysqld/mysqld.sock` | `/var/lib/mysql/mysql.sock` |
| `use_a2ensite` | `true` | `false` |

---

## Structure du rôle

```
ansible-role-webserver/
├── defaults/main.yml          # Valeurs par défaut modifiables
├── vars/
│   ├── main.yml               # Placeholder / documentation
│   ├── Debian.yml             # Variables OS Ubuntu/Debian (auto-chargées)
│   └── RedHat.yml             # Variables OS Rocky/RHEL (auto-chargées)
├── tasks/
│   ├── main.yml               # Orchestrateur – charge les vars + appelle les sous-tâches
│   ├── install.yml            # Installation des paquets (apt / dnf)
│   ├── mariadb.yml            # Démarrage MariaDB + création DB/utilisateur
│   ├── wordpress.yml          # Téléchargement WP, extraction, wp-config
│   └── apache.yml             # Déploiement VirtualHost + démarrage serveur web
├── handlers/main.yml          # Gestionnaires Restart / Reload (Debian & RedHat)
├── templates/
│   ├── wp-config.php.j2       # Fichier de configuration WordPress
│   └── wordpress.conf.j2      # VirtualHost Apache/httpd
├── tests/
│   ├── inventory              # Inventaire des conteneurs Docker
│   └── test.yml               # Playbook de test
└── meta/main.yml              # Métadonnées Galaxy
```

---

## Gestionnaires (Handlers)

| Nom d'écoute | Action |
|---|---|
| `Restart webserver` | `service apache2 restart` (Debian) / `pkill -HUP httpd` (RedHat) |
| `Reload webserver` | `service apache2 reload` (Debian) / `httpd -k graceful` (RedHat) |

Les gestionnaires sont déclenchés lorsque la configuration VirtualHost est déployée ou que `a2enmod`/`a2ensite` s'exécute.

---

## Idempotence

- L'installation des paquets utilise `state: present` (apt/dnf).
- MariaDB n'est démarrée que si son socket n'existe pas déjà (garde `creates:`).
- Les fichiers WordPress ne sont téléchargés/copiés que si `wp-login.php` est absent de la racine web.
- `a2enmod`/`a2ensite` utilisent `creates:` pour détecter si le lien symbolique existe déjà.
- `wp-config.php` est déployé via template (Ansible détecte automatiquement les changements).

---

## Exemple de playbook

```yaml
- name: Déployer la stack WordPress
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

## Tests avec l'environnement Docker

Démarrez les conteneurs depuis `evaluation-1/` :

```bash
docker compose up -d
```

Puis exécutez le playbook de test depuis le conteneur `ansible` (ou directement si vos clés SSH sont configurées) :

```bash
ansible-playbook -i ansible-role-webserver/tests/inventory \
                    ansible-role-webserver/tests/test.yml
```

WordPress sera accessible à :
- Ubuntu client1 → http://localhost:8083
- Ubuntu client2 → http://localhost:8084
- Rocky  client3 → http://localhost:8085
- Rocky  client4 → http://localhost:8086

---

## Publication sur Ansible Galaxy

1. Poussez le rôle vers un **dépôt GitHub** nommé `ansible-role-webserver`.
2. Connectez-vous sur [galaxy.ansible.com](https://galaxy.ansible.com) avec votre compte GitHub.
3. Allez dans **My Content → Add Content → Import Role from GitHub**.
4. Sélectionnez votre dépôt et cliquez sur **Import**.
5. Le rôle sera disponible sous `EliasMeh.ansible-role-webserver`.

---

## Licence

MIT

## Auteur

EliasMeh
