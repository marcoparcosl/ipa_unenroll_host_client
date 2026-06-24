# ipa_unenroll_host_client

# IPA Unenroll — Playbook Ansible

Playbook genérico para desenrolar uno o varios hosts de **FreeIPA / Red Hat IDM** de forma automatizada. Realiza el proceso completo en dos fases: primero actúa sobre el cliente para desinstalarlo, y luego elimina el host del servidor IPA.

---

## Requisitos

- Ansible 2.9 o superior
- Acceso SSH con usuario `root` a los hosts cliente y al servidor IPA
- Usuario `admin` con privilegios en el servidor IPA
- `ipa-client` instalado en los hosts a desenrolar

---

## Estructura del proyecto

```
ipa_unenroll/
├── ansible.cfg                         # Configuración de Ansible
├── .gitignore                          # Archivos excluidos del repositorio
├── .vault_pass                         # Contraseña del vault (no se versiona)
├── ipa_unenroll.yml                    # Playbook principal
├── inventory/
│   └── hosts.ini                       # Inventario de hosts
├── vars/
│   ├── ipa_clients.yml                 # Variables para los clientes
│   └── ipa_servers.yml                 # Variables para el servidor IPA (vault)
└── roles/
    ├── ipa_client_unenroll/            # Rol que actúa en el cliente
    │   ├── handlers/main.yml           # Reinicio de sssd y oddjobd
    │   ├── tasks/main.yml              # Lógica de desinstalación
    │   └── templates/krb5.conf.j2     # krb5.conf limpio post-unenroll
    └── ipa_server_cleanup/             # Rol que actúa en el servidor IPA
        ├── handlers/main.yml           # Destrucción del ticket Kerberos
        ├── tasks/
        │   ├── main.yml                # Kinit + loop de hosts + reporte
        │   └── remove_host.yml         # Lógica de eliminación por host
        └── templates/
            └── unenroll_report.j2      # Reporte generado en /var/log/
```

---

## Configuración inicial

### 1. Inventario

Edita `inventory/hosts.ini` y agrega los hosts que deseas desenrolar en la sección `[ipa_clients]`:

```ini
[ipa_clients]
app-jboss-01.local    ansible_user=root
app-jboss-02.local    ansible_user=root

[ipa_servers]
idm01.idm.daytwo.cloud  ansible_user=root
```

### 2. Contraseña del vault

Crea el archivo `.vault_pass` con la contraseña que usarás para cifrar secretos:

```bash
echo "MiClaveVault" > .vault_pass
chmod 600 .vault_pass
```

### 3. Cifrar la contraseña del admin IPA

Edita `vars/ipa_servers.yml` y agrega la contraseña cifrada:

```bash
ansible-vault encrypt_string 'PasswordDelAdmin' --name 'vault_ipa_admin_password'
```

Pega el resultado en `vars/ipa_servers.yml`:

```yaml
ipa_admin_user: "admin"
ipa_admin_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
ipa_remove_dns: true
```

---

## Variables principales

| Variable              | Archivo              | Descripción                                                       |
| --------------------- | -------------------- | ----------------------------------------------------------------- |
| `ipa_admin_user`      | vars/ipa_servers.yml | Usuario administrador de IPA                                      |
| `ipa_admin_password`  | vars/ipa_servers.yml | Contraseña admin (cifrada con vault)                              |
| `ipa_remove_dns`      | vars/ipa_servers.yml | Eliminar registro DNS al borrar el host                           |
| `ipa_hosts_to_remove` | vars/ipa_servers.yml | Lista de hosts a eliminar (por defecto toma el grupo ipa_clients) |
| `ipa_cleanup_files`   | vars/ipa_clients.yml | Limpiar archivos residuales en el cliente                         |
| `ipa_force_unenroll`  | vars/ipa_clients.yml | Forzar unenroll aunque haya errores parciales                     |
| `ipa_cleanup_paths`   | vars/ipa_clients.yml | Rutas a eliminar del cliente tras el unenroll                     |

---

## Uso

### Desenrolar todos los hosts del inventario

```bash
ansible-playbook ipa_unenroll.yml
```

### Desenrolar un host específico

```bash
ansible-playbook ipa_unenroll.yml --limit app-jboss-01.local
```

### Sin eliminar el registro DNS

```bash
ansible-playbook ipa_unenroll.yml -e "ipa_remove_dns=false"
```

### Forzar unenroll aunque el cliente tenga errores

```bash
ansible-playbook ipa_unenroll.yml -e "ipa_force_unenroll=true"
```

### Modo dry-run (sin aplicar cambios)

```bash
ansible-playbook ipa_unenroll.yml --check
```

---

## Fases del playbook

### FASE 1 — Cliente (`ipa_client_unenroll`)

Actúa sobre cada host del grupo `[ipa_clients]`:

1. Verifica si el host está enrollado en IPA (`/etc/ipa/default.conf`)
2. Ejecuta `ipa-client-uninstall --unattended`
3. Elimina archivos residuales (`/etc/ipa`, `/etc/krb5.keytab`, etc.)
4. Despliega un `krb5.conf` limpio desde el template
5. Notifica a los handlers para reiniciar `sssd` y `oddjobd`

### FASE 2 — Servidor IPA (`ipa_server_cleanup`)

Actúa sobre el servidor IPA:

1. Obtiene ticket Kerberos con `kinit admin`
2. Por cada host a eliminar:
   - Verifica que exista en IPA
   - Ejecuta `ipa host-disable` (revoca keytab y servicios asociados)
   - Ejecuta `ipa host-del --updatedns` (elimina el host y su registro DNS)
3. Genera un reporte en `/var/log/ipa_unenroll_report.txt`
4. Destruye el ticket Kerberos con `kdestroy`

---

## Archivos ignorados por Git

Los siguientes archivos **no se versionan** por contener datos sensibles o ser generados automáticamente:

- `ansible.cfg`
- `.vault_pass`
- `vars/ipa_servers.yml`
- `inventory/hosts.ini`
- `*.log`, `*.retry`
  Desenrolar un host cliente del Re Hat Identity Manager
