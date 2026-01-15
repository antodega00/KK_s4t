# Guida Integrazione OpenStack Keystone + Keycloak (Docker)
Questo progetto implementa un'architettura di autenticazione federata (WebSSO) utilizzando il protocollo OpenID Connect (OIDC). L'obiettivo è delegare l'autenticazione degli utenti a Keycloak (Identity Provider - IdP), permettendo a Keystone (Service Provider - SP) di fidarsi delle identità fornite esternamente e mapparle su specifici ruoli e permessi all'interno dell'infrastruttura OpenStack.

## Struttura del progetto

```text
/project-folder/
├── docker-compose.yml
├── conf_wagent
├── conf_conductor
├── conf_ui
└── keystone-config/
    ├── keystone.conf
    ├── wsgi-keystone.conf
    ├── keystone-mapping.json
    └── sso_callback.html
```

## Procedura di Avvio e Configurazione

### Fase 1: Avvio Infrastruttura

**1. Build & Up:**

```bash
docker compose up -d --build
```

**2. Keycloak Setup (Browser):**

*   **Login:** `http://localhost:8081` (credenziali: `admin`/`admin`).
*   **Create Realm:** `openstack`.
*   **Create Client:** `keystone`.
    *   **Access Type:** `Auth/On` (Confidential).
    *   **Redirect URI:** `http://localhost:5000/v3/auth/OS-FEDERATION/websso/mapped/redirect`
*   **Importante:** Copiare il **Client Secret** generato e incollarlo nel file `wsgi-keystone.conf` (parametro `OIDCClientSecret`).

**3. Riavvio Keystone:**
Necessario per applicare il secret appena incollato.

```bash
docker compose restart keystone
```

## Preparazione dell'ambiente

A differenza delle installazioni standard, Keystone richiede un modulo Apache specifico per gestire il protocollo OIDC. Poiché l'immagine base **lucadagati/iotronic-keystone** non include nativamente questo modulo, è stato necessario estenderla.

<pre>FROM lucadagati/iotronic-keystone

USER root
Aggiornamento repo e installazione del modulo OIDC per Apache

RUN apt-get update && \
    apt-get install -y libapache2-mod-auth-openidc && \
    rm -rf /var/lib/apt/lists/*

# Abilitazione del modulo in Apache
RUN a2enmod auth_openidc
</pre>


### Fase 2: Inizializzazione Keystone (Bootstrap)

Dopo aver copiato il client secret dentro `wsgi-keystone.conf`, procedere con il riavvio di Keystone e MariaDB.
```bash
docker compose restart keystone mariadb
```
Eseguire i seguenti comandi all’interno del container:

```bash
docker exec -it keystone bash
```

Una volta dentro la shell del container:

```bash
# 1. Popola DB
keystone-manage db_sync

# 2. Setup chiavi (Fernet e Credential)
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

# 3. Crea Admin
keystone-manage bootstrap --bootstrap-password admin \
  --bootstrap-admin-url http://localhost:5000/v3/ \
  --bootstrap-internal-url http://localhost:5000/v3/ \
  --bootstrap-public-url http://localhost:5000/v3/ \
  --bootstrap-region-id RegionOne

# 4. Fix permessi
chown -R keystone:keystone /etc/keystone

# USCITA E RIAVVIO: Uscire dal container e riavviare il servizio.
exit
docker compose restart keystone
```

### Fase 3: Configurazione lato Keycloak (Identity Provider)

In Keycloak non vengono assegnati ruoli direttamente agli utenti, ma si utilizza un approccio basato sui **Gruppi**. I gruppi vengono successivamente propagati verso OpenStack tramite token JWT.

#### Configurazione del Client Mapper

Affinché Keycloak invii correttamente i gruppi all’interno del token JWT verso OpenStack Keystone, è necessario configurare un Client Mapper.

*   **Percorso:** Clients > keystone > Client Scopes (keystone-dedicated) o Mappers
*   **Tipo Mapper:** Group Membership

**Parametri del Mapper:**
*   **Name:** `groups-mapper`
*   **Token Claim Name:** `groups`
*   **Full group path:** `ON` (genera valori del tipo `/KC_IOT_ADMIN`)
*   **Add to ID token:** `ON`
*   **Add to Access token:** `ON`
*   **Add to Userinfo:** `ON`

#### Creazione dei Gruppi

Sono stati definiti tre gruppi logici:
*   `KC_IOT_USER`: Utente base
*   `KC_IOT_MANAGER`: Gestore del progetto
*   `KC_IOT_ADMIN`: Amministratore del progetto

Ogni utente viene associato a uno di questi gruppi per ereditare i relativi permessi.




### Fase 4: Configurazione Federazione (Lato Keystone)

Rientrare nel container e caricare le variabili d’ambiente per autenticarsi come admin (bisogna farlo ad ogni accesso al container):

```bash
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://localhost:5000/v3
export OS_IDENTITY_API_VERSION=3
```

Eseguire i comandi OpenStack per configurare la federazione:

```bash
# 1. Dominio, Gruppi e Ruoli per utenti federati
openstack domain create federated_domain
openstack group create --domain federated_domain grp_iot_user
openstack group create --domain federated_domain grp_iot_manager
openstack group create --domain federated_domain grp_iot_admin

# User: utilizzo risorse, sola lettura sulle Board
openstack role add --group grp_iot_user \
  --group-domain federated_domain \
  --project admin \
  --project-domain Default user_iot

# Manager: gestione Board esistenti
openstack role add --group grp_iot_manager \
  --group-domain federated_domain \
  --project admin \
  --project-domain Default manager_iot_project

# Admin: creazione nuove Board
openstack role add --group grp_iot_admin \
  --group-domain federated_domain \
  --project admin \
  --project-domain Default admin_iot_project

# 2. Identity Provider
# Nota: Il Remote ID deve coincidere esattamente con l’issuer di Keycloak
openstack identity provider create keycloak \
  --remote-id http://localhost:8081/realms/openstack
```


## Matrice dei Permessi

| Funzionalità / Risorsa | User | Manager | Admin |
| :--- | :---: | :---: | :---: |
| Visualizzare Board | Sì | Sì | Sì |
| Creare Board (registrazione) | No | No | Sì |
| Modificare Board | No | Sì | Sì |
| Cancellare Board | No | Sì | Sì |
| Creare Plugin/Fleet/Service | Sì | Sì | Sì |
| Visualizzare Plugin/Fleet | Sì | Sì | Sì |
| Cancellare Plugin/Fleet altrui | No | No | No* |




## Mapping tra Gruppi Remoti e Locali

Il collegamento tra i gruppi di Keycloak e quelli di Keystone è definito nel file `/etc/keystone/keystone-mapping.json`.

**Logica:**
*   Analisi del claim `OIDC-groups` passato da Apache.
*   Se il valore corrisponde a `/KC_IOT_*`, l'utente viene mappato nel gruppo locale `grp_iot_*`.

**Applica configurazione:**

```bash
# 3. Mapping
openstack mapping create keycloak_mapping \
  --rules /etc/keystone/keystone-mapping.json

# 4. Protocollo
openstack federation protocol create mapped \
  --identity-provider keycloak \
  --mapping keycloak_mapping
```

**Esempio Configurazione JSON (`keystone-mapping.json`):**

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

