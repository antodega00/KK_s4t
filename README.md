# OpenStack Keystone + Keycloak Integration (Docker)
This project implements a federated authentication architecture (WebSSO) using the OpenID Connect (OIDC) protocol. The objective is to delegate user authentication to Keycloak (Identity Provider - IdP), enabling Keystone (Service Provider - SP) to trust externally provided identities and map them to specific roles and permissions within the OpenStack infrastructure.

## 1. Project Structure

Ensure your working directory strictly follows this structure:

```text
/project-folder/
├── docker-compose.yml
├── Dockerfile                         # Required to install mod_auth_openidc
├── conf_wagent
├── conf_conductor
├── conf_ui
└── keystone-config/
    ├── keystone.conf                  # Keystone logical configuration
    ├── wsgi-keystone.conf             # Apache VirtualHost configuration (OIDC)
    ├── keystone-mapping.json          # User mapping rules
    └── sso_callback.html              # Dummy file for redirect
```

---

## 2. Startup and Configuration Procedure

### 2.1 Phase 1: Infrastructure Startup

**1. Build & Up:**

```bash
docker compose up -d --build
```

**2. Keycloak Setup (Browser):**

*   **Login:** `http://localhost:8081` (credentials: `admin`/`admin`).
*   **Create Realm:** `openstack`.
*   **Create Client:** `keystone`.
    *   **Access Type:** `Auth/On` (Confidential).
    *   **Redirect URI:** `http://localhost:5000/v3/auth/OS-FEDERATION/websso/mapped/redirect`
*   **Important:** Copy the generated **Client Secret** and paste it into the `wsgi-keystone.conf` file (`OIDCClientSecret` parameter).

**3. Restart Keystone:**
Required to apply the secret you just pasted.

```bash
docker compose restart keystone
```

## Environment Preparation

Unlike standard installations, Keystone requires a specific Apache module to handle the OIDC protocol. Since the base image **lucadagati/iotronic-keystone** does not natively include this module, it was necessary to extend it.

<pre>FROM lucadagati/iotronic-keystone

USER root
# Repo update and OIDC module installation for Apache

RUN apt-get update && \
    apt-get install -y libapache2-mod-auth-openidc && \
    rm -rf /var/lib/apt/lists/*

# Enable module in Apache
RUN a2enmod auth_openidc
</pre>


### 2.2 Phase 2: Keystone Initialization (Bootstrap)

After copying the client secret into `wsgi-keystone.conf`, proceed with restarting Keystone and MariaDB.

```bash
docker compose restart keystone mariadb
```

Execute the following commands inside the container:

```bash
docker exec -it keystone bash
```

Once inside the container shell:

```bash
# 1. Populate DB
keystone-manage db_sync

# 2. Setup keys (Fernet and Credential)
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

# 3. Create Admin
keystone-manage bootstrap --bootstrap-password admin \
  --bootstrap-admin-url http://localhost:5000/v3/ \
  --bootstrap-internal-url http://localhost:5000/v3/ \
  --bootstrap-public-url http://localhost:5000/v3/ \
  --bootstrap-region-id RegionOne

# 4. Fix permissions
chown -R keystone:keystone /etc/keystone

# EXIT AND RESTART: Exit the container and restart the service.
exit
docker compose restart keystone
```

### 2.3 Phase 3: Keycloak Configuration (Identity Provider)

In Keycloak, roles are not assigned directly to users, but a **Group-based** approach is used. Groups are subsequently propagated to OpenStack via JWT tokens.

#### 2.3.1 Client Mapper Configuration

For Keycloak to correctly send groups within the JWT token to OpenStack Keystone, a Client Mapper must be configured.

*   **Path:** Clients > keystone > Client Scopes (keystone-dedicated) or Mappers
*   **Mapper Type:** Group Membership

**Mapper Parameters:**
*   **Name:** `groups-mapper`
*   **Token Claim Name:** `groups`
*   **Full group path:** `ON` (generates values like `/KC_IOT_ADMIN`)
*   **Add to ID token:** `ON`
*   **Add to Access token:** `ON`
*   **Add to Userinfo:** `ON`

#### 2.3.2 Group Creation

Three logical groups have been defined:
*   `KC_IOT_USER`: Basic user
*   `KC_IOT_MANAGER`: Project Manager
*   `KC_IOT_ADMIN`: Project Administrator

Each user is associated with one of these groups to inherit the relative permissions.

### 2.4 Phase 4: Federation Configuration (Keystone Side)

Re-enter the container and load environment variables to authenticate as admin (must be done at every access to the container):

```bash
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://localhost:5000/v3
export OS_IDENTITY_API_VERSION=3
```

Execute OpenStack commands to configure federation:

```bash
# 1. Domain, Groups and Roles for federated users
openstack domain create federated_domain
openstack group create --domain federated_domain grp_iot_user
openstack group create --domain federated_domain grp_iot_manager
openstack group create --domain federated_domain grp_iot_admin

# User: resource usage, read-only on Boards
openstack role add --group grp_iot_user \
  --group-domain federated_domain \
  --project admin \
  --project-domain Default user_iot

# Manager: existing Boards management
openstack role add --group grp_iot_manager \
  --group-domain federated_domain \
  --project admin \
  --project-domain Default manager_iot_project

# Admin: new Boards creation
openstack role add --group grp_iot_admin \
  --group-domain federated_domain \
  --project admin \
  --project-domain Default admin_iot_project

# 2. Identity Provider
# Note: The Remote ID must exactly match the Keycloak issuer
openstack identity provider create keycloak \
  --remote-id http://localhost:8081/realms/openstack
```

---

## OIDC Network Architecture in Docker

A critical point in OIDC configuration between containers is the distinction between internal and external endpoints.

**Analysis of `wsgi-keystone.conf` (Difference between Metadata and Issuer):**
In the configuration file, it is fundamental to note the difference between two key parameters:
*   **`OIDCProviderMetadataURL`**: Points to `http://keycloak:8080`. This URL is used for **Server-to-Server** traffic (from Apache to Keycloak) and is only resolvable within the Docker network.
*   **`OIDCProviderIssuer`**: Points to `http://localhost:8081`. This URL represents the public identity of the Issuer and is what the browser (Client) and the user see (**Browser-to-Server**).

**Why it is critical:**
If the `iss` (issuer) inside the JWT token generated by Keycloak does not exactly match what is expected by `mod_auth_openidc`, authentication fails. This configuration ensures that:
1.  Apache can contact Keycloak directly via the Docker network to validate tokens (back-channel).
2.  The user's browser is correctly redirected to Keycloak exposed on the host (front-channel).

---

## Permissions Matrix

| Feature / Resource | User | Manager | Admin |
| :--- | :---: | :---: | :---: |
| View Boards | Yes | Yes | Yes |
| Create Board (registration) | No | No | Yes |
| Edit Board | No | Yes | Yes |
| Delete Board | No | Yes | Yes |
| Create Plugin/Fleet/Service | Yes | Yes | Yes |
| View Plugin/Fleet | Yes | Yes | Yes |


---

## 4. Mapping between Remote and Local Groups

The link between Keycloak groups and Keystone groups is defined in the file `/etc/keystone/keystone-mapping.json`.

### Transformation Mechanism (Claims to Local Groups)

The Keystone mapping engine acts as a translator between federated identities and local resources. The process happens in three steps:

1.  **Input (Environment Variables):**
    The Apache module (`mod_auth_openidc`) validates the JWT token received from Keycloak and extracts the claims, exposing them as environment variables to the Keystone WSGI process (e.g. `OIDC_CLAIM_groups`, `OIDC_CLAIM_preferred_username`).

2.  **Matching (Remote Rules):**
    Keystone analyzes the `remote` section of the rules.
    *   `type`: Specifies which OIDC attribute to check (e.g. `OIDC-groups`).
    *   `any_one_of`: Verifies if the claim value contains at least one of the listed elements. In our case, we look for the exact string of the Keycloak group (e.g. `/KC_IOT_ADMIN`).

3.  **Assignment (Local Mapping):**
    If the match is successful, the directives in the `local` section are applied.
    *   **User**: The federated user is mapped to an ephemeral user in Keystone. The `name` : `{0}` field indicates that the local username will be equal to the first attribute matched in the remote section (in this case `preferred_username`).
    *   **Group**: The user is inserted into a specific OpenStack group (e.g. `grp_iot_admin`). This is the key step that grants permissions, as roles (User, Manager, Admin) are assigned to these local groups.

**Apply configuration:**

```bash
# 3. Mapping
openstack mapping create keycloak_mapping \
  --rules /etc/keystone/keystone-mapping.json

# 4. Protocollo
openstack federation protocol create mapped \
  --identity-provider keycloak \
  --mapping keycloak_mapping
```

**JSON Configuration Example (`keystone-mapping.json`):**

```json
[
  {
    "local": [
      {
        "user": {
          "name": "{0}",
          "domain": { "name": "federated_domain" }
        },
        "group": {
          "name": "grp_iot_admin",
          "domain": { "name": "federated_domain" }
        }
      }
    ],
    "remote": [
      { "type": "OIDC-preferred_username" },
      {
        "type": "OIDC-groups",
        "any_one_of": [
          "/KC_IOT_ADMIN",
          "KC_IOT_ADMIN"
        ]
      }
    ]
  },
  {
    "local": [
      {
        "user": {
          "name": "{0}",
          "domain": { "name": "federated_domain" }
        },
        "group": {
          "name": "grp_iot_manager",
          "domain": { "name": "federated_domain" }
        }
      }
    ],
    "remote": [
      { "type": "OIDC-preferred_username" },
      {
        "type": "OIDC-groups",
        "any_one_of": [
          "/KC_IOT_MANAGER",
          "KC_IOT_MANAGER"
        ]
      }
    ]
  },
  {
    "local": [
      {
        "user": {
          "name": "{0}",
          "domain": { "name": "federated_domain" }
        },
        "group": {
          "name": "grp_iot_user",
          "domain": { "name": "federated_domain" }
        }
      }
    ],
    "remote": [
      { "type": "OIDC-preferred_username" },
      {
        "type": "OIDC-groups",
        "any_one_of": [
          "/KC_IOT_USER",
          "KC_IOT_USER"
        ]
      }
    ]
  }
]
```

---

## 5. Integration with Iotronic (Horizon/Dashboard)

To enable authentication via Keycloak on the Dashboard (IoTronic UI), the `local_settings.py` file (mounted in the `iotronic-ui` container) has been modified.

Key settings to enable WebSSO and correctly handle redirects in a Docker environment are:

```python
# Enable Web Single Sign-On support
WEBSSO_ENABLED = True

# Set default login method (optional)
WEBSSO_INITIAL_CHOICE = "mapped"

# Defines login options visible in the dropdown menu
WEBSSO_CHOICES = (
    ("credentials", "Keystone Credentials"),  # Standard OpenStack Login
    ("mapped", "Keycloak Login"),             # Federated Login
)

# Maps the 'mapped' method to the protocol and IDP configured in Keystone
WEBSSO_IDP_MAPPING = {
    "mapped": ("keycloak", "mapped"), # ("<idp_id>", "<protocol_id>")
}

# CRITICAL FOR DOCKER: Force Keystone URL for WebSSO redirect
# Without this setting, Horizon might try to use the container's internal URL,
# making the redirect impossible for the user's browser.
WEBSSO_KEYSTONE_URL = "http://localhost:5000/v3"
```

#### Technical Note: Patch for custom redirect (WebSSO URL)

In a standard architecture, Horizon calculates the redirect URL using the public Keystone endpoint present in the service catalog (often an internal address like `http://keystone:5000`).
In a Docker environment, this address is **not reachable by the user's browser**, which navigates from localhost or an external IP.

**The Problem:**
The original `get_websso_url` function ignores the `WEBSSO_KEYSTONE_URL` variable defined in `local_settings.py`, always forcing the use of the internal catalog URL. This causes a connection error ("Connection refused") when the user attempts to login, because the browser tries to contact `http://keystone:5000` instead of `http://localhost:5000`.

**The Solution (Patch):**
We modified `openstack_auth/utils.py` to prioritize the `WEBSSO_KEYSTONE_URL` variable if present.

**Modified Code (`openstack_auth/utils.py`):**

```python
def get_websso_url(request, auth_url, websso_auth):
    websso_keystone_url = getattr(settings, 'WEBSSO_KEYSTONE_URL', None)
    
    if websso_keystone_url:
        auth_url = websso_keystone_url

    origin = build_absolute_uri(request, '/auth/websso/')
```


