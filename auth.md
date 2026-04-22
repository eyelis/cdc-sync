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


1. Authentification & Synchronisation (Le "Warm-up")
1. Accès (sans session) : L'utilisateur tente d'accéder au portail via la Gateway. Comme il n'a pas de session, il est intercepté.

2. SSO Flow (OIDC / JWT) : La Gateway redirige l'utilisateur vers uSignOn via le protocole OIDC. Après saisie des identifiants, uSignOn renvoie un ID Token et un Access Token (JWT) à la Gateway.

3. Get Permissions : La Gateway appelle le microservice osm-iam en lui passant l'identifiant utilisateur pour récupérer son profil métier complet.

4. Rôles/Rights : osm-iam renvoie les permissions et les codes spécifiques (ex: utCode, esfCode) nécessaires au Legacy.

5. Store Session Map : La Gateway enregistre dans Redis les infos utilisateur. Cela permet à tous les Pods de la Gateway de partager le contexte.

2. Le Pont entre les Mondes (OIDC vers SAML)
6. POST /session-sync : La Gateway appelle l'endpoint caché du Legacy. Elle lui transmet un "JWT Interne" signé contenant toutes les infos d'identité.

7. Bypass SAML : À l'intérieur du Legacy, le filtre GatewayPreAuthFilter intercepte l'appel. Il voit le JWT, valide sa signature, et shunte le flux SAML habituel.

8. 200 OK (Set-Cookie) : Le Legacy crée une session Tomcat réelle. Il répond à la Gateway en lui donnant un JSESSIONID. La Gateway stocke immédiatement ce cookie dans Redis associé à l'utilisateur.

3. Navigation et Routage (Le "Run")
9. Request /api/catalogue : L'utilisateur appelle une fonctionnalité déjà migrée.

10. Relay OIDC JWT : La Gateway reconnaît une route "moderne". Elle transmet simplement le JWT OIDC au microservice Catalogue.

11. Request /ism-web/home : L'utilisateur appelle une page encore sur le Monolithe.

12. Relay Cookie (JSESSIONID) : La Gateway récupère le JSESSIONID dans Redis et l'injecte dans le header Cookie. Le Legacy croit que l'utilisateur s'est connecté via SAML et affiche la page.

4. Cohérence des Données (L'Arrière-Boutique)
13. CDC Bidirectionnel : L'outil osm-sync (ex: Debezium/Kafka) surveille les logs de la base Legacy. Dès qu'une donnée change, il capture l'événement.

14. Data Update : L'événement est propagé vers le microservice Catalogue (et inversement) pour que les deux bases soient toujours le miroir l'une de l'autre pendant toute la durée de la migration.
    %% -- FLUX DE DONNÉES (CDC) --
    Sync <-->|13. CDC Bidirectionnel| DB_LEG
    Sync <-->|14. Data Update| CAT
