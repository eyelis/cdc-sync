graph TD
    %% -- ACTEURS & INFRA --
    User((Utilisateur<br/>Navigateur))
    
    subgraph "Infrastructure Kubernetes"
        
        %% -- GATEWAY LAYER --
        subgraph "namespace: gateway"
            GW[<B>osm-gateway</B><br/>(Spring Cloud Gateway)]
            Redis[(Cache Redis<br/>Sessions)]
            GW -.->|Active/Désactive<br/>Migration| FF[Feature Flip<br/>(Flag)]
        end

        %% -- SERVICES COMMUNS --
        subgraph "namespace: common"
            IAM[<B>osm-iam</B><br/>(Permissions/Droits)]
            Sync[<B>osm-sync</B><br/>(CDC Data Sync)]
        end

        %% -- LEGACY LAYER --
        subgraph "namespace: legacy"
            LEG[<B>Monolithe Legacy</B><br/>(SP SAML)]
            DB_LEG[(DB Legacy)]
            LEG <--> DB_LEG
        end

        %% -- NEW SERVICES LAYER --
        subgraph "namespace: microservices"
            CAT[<B>ms-catalogue</B>]
            DB_CAT[(DB Catalogue)]
            CAT <--> DB_CAT
        end
    end

    %% -- IDP LAYER --
    subgraph "uSignOn (IdP)"
        OIDC[Endpoint OIDC]
        SAML[Endpoint SAML]
    end

    %% -- FLUX D'AUTHENTIFICATION & CONVERSIONS --
    
    %% 1. Auth Initiale (OIDC)
    User == "1. Accès (sans session)" ==> GW
    GW <-->|2. SSO Flow (OIDC / JWT)| OIDC

    %% 2. Enrichissement & Cache
    GW -->|3. Get Permissions| IAM
    IAM -->>|4. Rôles/Rights| GW
    GW -->|5. Store Session Map| Redis

    %% 3. Conversion de Protocole (OIDC -> SAML Bypass)
    GW -.->|6. POST /session-sync<br/>(JWT Interne)| LEG
    LEG -->|7. Bypass SAML (Header/JWT)| LEG
    Note right of LEG: Crée Session<br/>Mock SAMLUser
    LEG -->>|8. 200 OK<br/>(Set-Cookie: JSESSIONID)| GW

    %% -- FLUX DE REQUÊTES --
    
    %% 4. Requête vers BC (JWT Relay)
    User -->|9. Request /api/catalogue| GW
    GW -->|10. Relay OIDC JWT| CAT

    %% 5. Requête vers Legacy (Cookie Relay)
    User -->|11. Request /ism-web/home| GW
    GW -->|12. Relay Cookie:<br/>JSESSIONID| LEG

    %% -- FLUX DE DONNÉES (CDC) --
    Sync <-->|13. CDC Bidirectionnel| DB_LEG
    Sync <-->|14. Data Update| CAT
