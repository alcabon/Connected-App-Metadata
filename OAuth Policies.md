Je vais rechercher les informations les plus récentes sur les OAuth Policies dans Salesforce pour vous donner une réponse précise.Bonjour ! Excellente question sur les OAuth Policies dans Salesforce. Voici la réponse détaillée :

## Réponse courte
**Oui, vous pouvez requêter les OAuth Policies**, mais uniquement via **le Metadata API**, pas via SOQL.

## Détails techniques

### 1. **Via Metadata API** ✅

Les OAuth Policies sont accessibles via le type de métadonnée **`ConnectedApp`** qui contient :
- **`ConnectedAppOauthConfig`** : Configuration OAuth de base (scopes, callback URL, etc.)
- **`ConnectedAppOauthPolicy`** : Politiques OAuth (IP Relaxation, Refresh Token Policy, etc.)

**Exemple de récupération avec Metadata API :**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>*</members>
        <name>ConnectedApp</name>
    </types>
    <version>64.0</version>
</Package>
```

### 2. **Via SOQL** ❌

Les informations OAuth ne sont pas disponibles via SOQL - vous ne pouvez pas faire de requête sur l'objet `ConnectedApplication` pour obtenir les scopes ou politiques OAuth détaillées.

### 3. **Types de métadonnées disponibles**

- **`ConnectedApp`** : Application connectée principale
- **`OauthCustomScope`** : Scopes OAuth personnalisés
- **`SessionPolicy`** (depuis Summer '20) : Politiques de session
- **`ConnectedAppOauthPolicy`** (depuis Summer '20) : Les champs oauthPolicy et sessionPolicy sont devenus visibles et éditables depuis Summer '20

### 4. **Méthodes pratiques**

**Avec SFDX/CLI :**
```bash
sf project retrieve start -m ConnectedApp
```

**Avec Apex (via apex-mdapi) :**
```apex
MetadataService.MetadataPort service = createService();
MetadataService.ConnectedApp connectedApp = 
    (MetadataService.ConnectedApp)service.readMetadata('ConnectedApp', 
    new String[] {'NomDeLApp'}).getRecords()[0];
```

### 5. **Limitations importantes**

- Le `consumerSecret` n'est **jamais retourné** par le Metadata API pour des raisons de sécurité
- Certaines politiques avancées nécessitent des versions API récentes (v46.0+)
- Les permission sets associés aux Connected Apps sont récupérables via le champ `permissionSetName`

**Besoin d'aide pour une implémentation spécifique ?** Je peux vous guider davantage !

---

Excellente question ! Les **managed connected apps** (applications connectées dans des packages gérés) ont plusieurs restrictions et comportements spécifiques. Voici un tour d'horizon complet :

## 🔐 Restrictions principales des Managed Connected Apps

### 1. **Architecture "Shadow Copy"**
La Connected App réside dans l'org d'origine (packaging org), tandis que l'instance installée dans l'org subscriber n'est qu'une copie virtuelle ("shadow") de l'originale.

**Implications :**
- Les modifications (certificats, scopes, etc.) dans le packaging org se propagent automatiquement à **tous les orgs subscribers**
- Les subscribers ne peuvent **pas modifier** la définition de base de l'app (consumerKey, consumerSecret, scopes OAuth)
- Seules les **politiques de gestion** peuvent être modifiées localement dans chaque subscriber org

### 2. **Récupération via Metadata API** ⚠️

**Limitations :**
- Vous **ne pouvez PAS** récupérer une connected app d'un managed package via Metadata API dans un org de développement
- Le `consumerSecret` n'est **jamais retourné** par le Metadata API pour des raisons de sécurité
- Sans accès direct au packaging org, vous ne pouvez pas obtenir les métadonnées d'une Connected App en dehors d'un managed package

### 3. **Consumer Key/Secret Globaux** 🌐

Le principal avantage d'inclure une Connected App dans un managed package est de préserver les Consumer Key/Secret à travers plusieurs orgs.

**Cela signifie :**
- Un seul set de key/secret pour communiquer avec un nombre illimité d'orgs
- Pas besoin de reconfigurer votre système externe pour chaque nouveau client
- **MAIS** : Chaque autorisation OAuth reste liée à un org et un user spécifique

### 4. **Restrictions OAuth spécifiques**

#### **Client Credentials Flow** ❌
Le flow "Client Credentials" ne fonctionne pas avec les managed packages car il nécessite un utilisateur "Run As" spécifique qui n'existe pas dans les subscriber orgs.

**Solutions alternatives :**
- Utiliser JWT Bearer Flow
- Utiliser Web Server Flow (Authorization Code)
- Envisager External Client Apps (2GP)

#### **Politiques OAuth modifiables localement** ✅
Les subscribers peuvent configurer :
- Permitted Users (All users / Admin approved)
- IP Relaxation
- Refresh Token Policy
- Session Timeout

Les restrictions IP ne sont PAS propagées depuis le packaging org - elles doivent être configurées par l'admin du subscriber.

### 5. **Gestion des certificats** 🔄

Lors de la mise à jour d'un certificat expirant dans le packaging org, la Connected App fonctionne de manière transparente avec le nouveau certificat dans tous les subscriber orgs - aucune mise à jour de package n'est requise.

### 6. **Installation et duplication** 🔄

**Problème connu :**
- Lors d'upgrades de package, des **duplications** de Connected App peuvent apparaître
- L'utilisateur doit parfois cliquer sur "Install" après l'upgrade, bien que l'app avec le même client_id reste comme duplicate
- Le comportement peut être **incohérent** entre différents orgs

### 7. **Nouvelles restrictions de sécurité (2025)** 🔒

**Depuis septembre 2025 :**
- Les "uninstalled connected apps" ne sont plus accessibles pour la plupart des utilisateurs - si une app n'est pas installée dans l'org, elle est bloquée
- Deux nouvelles permissions permettent des exceptions :
  - `Approve Uninstalled Connected Apps`
  - `Use Any API Client`

**Impact pour les managed packages :**
- Les Connected Apps dans les managed packages sont considérées comme "installed"
- Mais attention aux flows OAuth Device qui sont bloqués même pour les apps déjà autorisées

### 8. **Limites de création** 📊

Il n'y a pas de limite sur le nombre de Connected Apps créées manuellement ou via Metadata API, mais la limite pour la registration dynamique OAuth 2.0 est de 100 (extensible à 3000).

## 📋 Recommandations pratiques

### Pour les ISV / Package Providers :

```apex
// ❌ NE PAS FAIRE : Tenter de modifier la définition de base
// Les subscribers ne peuvent pas modifier les scopes, certificats, etc.

// ✅ FAIRE : Fournir un post-install script pour configurer les politiques
public class PostInstallScript implements InstallHandler {
    public void onInstall(InstallContext context) {
        // Configurer SetupEntityAccess pour pré-autoriser des profils
        // Documenter les étapes de configuration nécessaires
    }
}
```

### Pour la distribution :

1. **Créer la Connected App dans un Production org** (pas DE org qui peuvent expirer)
2. **Inclure dans le package** pour faciliter la gestion des permissions
3. **Documenter clairement** les étapes de configuration post-installation
4. **Éviter Client Credentials Flow** - préférer JWT ou Web Server Flow

### Pour le développement :

Ne mettez pas les métadonnées d'une Connected App dans un repository et n'essayez pas de la déployer - maintenez la définition directement dans le production org.

## 🎯 Cas d'usage idéaux

**Quand inclure une Connected App dans un managed package :**
- Application externe nécessitant l'accès à plusieurs orgs clients
- Besoin de consumer key/secret unifié
- Nécessité pour l'admin de contrôler finement les accès (vs. simple block/allow)

**Quand l'éviter :**
- OAuth Client Credentials requis
- Besoin de modifications fréquentes de la définition
- Workflow de développement collaboratif complexe

Besoin de clarifications sur un point spécifique ?

---

Excellente observation ! C'est une nuance **très importante**. Voici la réalité complète :

## 🔍 Ce qui est VISIBLE vs RÉCUPÉRABLE

### 1. **Profils & Permission Sets** ✅ REQUÊTABLES via SOQL

**Dans un subscriber org, vous POUVEZ requêter :**

```sql
-- Récupérer les profils/permission sets ayant accès à une Connected App
SELECT Id, ParentId, Parent.Name, Parent.Profile.Name, 
       Parent.IsOwnedByProfile, SetupEntityId
FROM SetupEntityAccess 
WHERE SetupEntityType = 'ConnectedApplication'
AND SetupEntityId = '0H4...' -- ID de votre Connected App
```

Les permissions de connected application sont stockées dans l'objet SetupEntityAccess, où SetupEntityId représente l'ID de la connected application.

**Exemple pour lister toutes les Connected Apps et leurs autorisations :**

```sql
-- Récupérer les Connected Apps
SELECT Id, Name, OptionsAllowAdminApprovedUsersOnly
FROM ConnectedApplication

-- Puis pour chaque app, récupérer les autorisations
SELECT Id, Parent.Name, Parent.Profile.Name, 
       Parent.IsCustom, Parent.IsOwnedByProfile
FROM SetupEntityAccess 
WHERE SetupEntityType = 'ConnectedApplication'
ORDER BY SetupEntityId
```

### 2. **OAuth Policies** ⚠️ PARTIELLEMENT requêtables

**Champs disponibles dans `ConnectedApplication` via SOQL :**

```sql
SELECT Id, Name, 
       OptionsAllowAdminApprovedUsersOnly,  -- "Permitted Users"
       OptionsIPRestrictions,                -- IP Relaxation (boolean limité)
       OptionsRefreshTokenValidityMetric,    -- Refresh Token Policy
       RefreshTokenValidityPeriod,
       StartUrl,
       MobileStartUrl
FROM ConnectedApplication
WHERE Name = 'YourAppName'
```

**MAIS attention :** Le champ OptionsIPRestrictions n'est qu'un booléen alors qu'il y a 4 valeurs dans l'UI, et SessionPolicyAction n'est apparemment pas accessible via l'API.

### 3. **Metadata API** ⚠️ Plus complet MAIS avec restrictions

**Via Metadata API, vous avez accès à :**

```xml
<ConnectedApp>
    <oauthConfig>
        <scopes>api</scopes>
        <scopes>refresh_token</scopes>
        <!-- consumerSecret jamais retourné -->
    </oauthConfig>
    
    <oauthPolicy>
        <ipRelaxation>ENFORCE</ipRelaxation>
        <refreshTokenPolicy>infinite</refreshTokenPolicy>
    </oauthPolicy>
    
    <permissionSetName>MyPermissionSet</permissionSetName>
</ConnectedApp>
```

**Mais pour les managed apps :**
- Vous ne pouvez PAS récupérer la définition complète depuis un subscriber org
- Le `consumerSecret` n'est **JAMAIS** retourné
- Les politiques OAuth peuvent être lues mais pas toujours modifiables

## 📊 Tableau comparatif : Ce qui est accessible

| Information | Visible UI | SOQL | Metadata API (Subscriber) | Metadata API (Packaging) |
|-------------|-----------|------|---------------------------|-------------------------|
| **Profils/Permission Sets** | ✅ | ✅ | ✅ | ✅ |
| **Permitted Users** | ✅ | ✅ (OptionsAllowAdminApprovedUsersOnly) | ✅ (isAdminApproved) | ✅ |
| **IP Relaxation (détail)** | ✅ (4 valeurs) | ⚠️ (boolean seulement) | ✅ (ipRelaxation) | ✅ |
| **Refresh Token Policy** | ✅ | ✅ | ✅ | ✅ |
| **Session Policies** | ✅ | ❌ | ⚠️ (limité) | ✅ |
| **OAuth Scopes** | ✅ | ❌ | ✅ | ✅ |
| **Consumer Key** | ✅ | ❌ | ✅ | ✅ |
| **Consumer Secret** | ✅ (UI only) | ❌ | ❌ | ❌ |
| **Certificats** | ✅ | ❌ | ✅ | ✅ |

## 💡 Solutions pratiques pour un subscriber org

### **Approche 1 : SOQL pour les autorisations** ✅

```apex
// Récupérer les détails d'une Connected App managed
ConnectedApplication ca = [
    SELECT Id, Name, 
           OptionsAllowAdminApprovedUsersOnly,
           OptionsIPRestrictions
    FROM ConnectedApplication 
    WHERE Name = 'MyManagedApp'
    LIMIT 1
];

// Récupérer les profils/permission sets autorisés
List<SetupEntityAccess> accessList = [
    SELECT Parent.Name, Parent.Profile.Name, 
           Parent.IsOwnedByProfile
    FROM SetupEntityAccess 
    WHERE SetupEntityType = 'ConnectedApplication'
    AND SetupEntityId = :ca.Id
];

System.debug('App autorisée pour admin seulement: ' + 
             ca.OptionsAllowAdminApprovedUsersOnly);

for(SetupEntityAccess sea : accessList) {
    System.debug('Accès via: ' + sea.Parent.Name);
}
```

### **Approche 2 : Metadata API pour les policies détaillées** ⚠️

**Dans un subscriber org, vous pouvez LIRE mais pas toujours MODIFIER :**

```apex
// Via apex-mdapi
MetadataService.MetadataPort service = new MetadataService.MetadataPort();
service.SessionHeader = new MetadataService.SessionHeader_element();
service.SessionHeader.sessionId = UserInfo.getSessionId();

// Tenter de lire (peut fonctionner selon les permissions)
MetadataService.ConnectedApp app = 
    (MetadataService.ConnectedApp)service.readMetadata(
        'ConnectedApp', 
        new String[] {'MyManagedApp'}
    ).getRecords()[0];

// Les policies peuvent être lues MAIS...
// Pour une managed app, vous ne pourrez pas tout modifier
```

### **Approche 3 : Combinaison des deux** 🎯

```apex
// 1. Utiliser SOQL pour ce qui est facilement accessible
Map<Id, ConnectedApplication> apps = new Map<Id, ConnectedApplication>(
    [SELECT Id, Name, OptionsAllowAdminApprovedUsersOnly 
     FROM ConnectedApplication]
);

// 2. Utiliser SetupEntityAccess pour les permissions
Map<Id, List<SetupEntityAccess>> appPermissions = new Map<Id, List<SetupEntityAccess>>();
for(SetupEntityAccess sea : [
    SELECT SetupEntityId, Parent.Name, Parent.Profile.Name
    FROM SetupEntityAccess 
    WHERE SetupEntityType = 'ConnectedApplication'
]) {
    if(!appPermissions.containsKey(sea.SetupEntityId)) {
        appPermissions.put(sea.SetupEntityId, new List<SetupEntityAccess>());
    }
    appPermissions.get(sea.SetupEntityId).add(sea);
}

// 3. Pour les détails OAuth complexes, utiliser Metadata API si nécessaire
```

## ⚠️ Limitations importantes pour managed apps

### **Dans un subscriber org :**

1. **Vous NE POUVEZ PAS :**
   - Modifier les scopes OAuth (définis dans le packaging org)
   - Modifier le certificat
   - Récupérer le consumer secret
   - Modifier certaines policies de base

2. **Vous POUVEZ :**
   - Voir et modifier les profils/permission sets autorisés (via UI ou SetupEntityAccess)
   - Voir les policies OAuth actuelles (lecture seule via SOQL/Metadata)
   - Modifier les policies de gestion locale (IP Relaxation, etc.)

### **Exemple de tentative de modification (échouera) :**

L'objet PermissionSetAssignment ne sert PAS de junction entre PermissionSet et ConnectedApp - la requête ne retourne aucun résultat.

## 🎯 Recommandations

### Pour auditer vos Connected Apps managed :

```apex
// Script d'audit complet
public class ConnectedAppAudit {
    public static void auditManagedApps() {
        // Récupérer toutes les Connected Apps
        List<ConnectedApplication> apps = [
            SELECT Id, Name, OptionsAllowAdminApprovedUsersOnly
            FROM ConnectedApplication
        ];
        
        for(ConnectedApplication app : apps) {
            System.debug('=== ' + app.Name + ' ===');
            
            // Permissions
            List<SetupEntityAccess> permissions = [
                SELECT Parent.Name, Parent.IsOwnedByProfile
                FROM SetupEntityAccess 
                WHERE SetupEntityType = 'ConnectedApplication'
                AND SetupEntityId = :app.Id
            ];
            
            System.debug('Nombre de profils/permission sets: ' + 
                        permissions.size());
            System.debug('Mode admin approved: ' + 
                        app.OptionsAllowAdminApprovedUsersOnly);
        }
    }
}
```

### Pour automatiser la gestion :

Vous pouvez créer des enregistrements SetupEntityAccess pour pré-autoriser des profils avant un basculement vers "Admin approved users".

```apex
// Exemple d'insertion de permissions
SetupEntityAccess sea = new SetupEntityAccess();
sea.SetupEntityId = connectedAppId; // ID de la Connected App
sea.ParentId = permissionSetId;     // ID du PermissionSet ou Profile
insert sea;
```

## 📝 En résumé

**Oui, vous pouvez visualiser ET requêter :**
- ✅ Les profils/permission sets associés (SOQL + SetupEntityAccess)
- ✅ Certaines policies OAuth basiques (SOQL + ConnectedApplication)
- ⚠️ Les policies détaillées (Metadata API en lecture seule dans subscriber)

**Mais NON, vous ne pouvez PAS :**
- ❌ Récupérer toutes les policies complexes via SOQL pur
- ❌ Modifier la définition de base d'une managed app
- ❌ Accéder au consumer secret une fois créé

**Besoin d'un use case spécifique ?** Je peux vous aider à construire la requête ou le script exact !

---

Excellente question ! C'est un point **critique** et frustrant. Voici la réalité précise :

## 🔐 Les Scopes des Managed Connected Apps : La Situation

### 1. **Dans le Packaging Org** ✅

```xml
<!-- Metadata API fonctionne normalement -->
<ConnectedApp>
    <oauthConfig>
        <scopes>api</scopes>
        <scopes>web</scopes>
        <scopes>refresh_token</scopes>
        <scopes>full</scopes>
    </oauthConfig>
</ConnectedApp>
```

**Accessible via :**
- ✅ Metadata API retrieve
- ✅ SFDX CLI
- ✅ Interface UI

### 2. **Dans le Subscriber Org** ❌ BLOQUÉ

**Problème : Les scopes NE SONT PAS accessibles via :**

```sql
-- ❌ Pas de champ "scopes" dans ConnectedApplication
SELECT Id, Name 
FROM ConnectedApplication
WHERE Name = 'ManagedApp'
-- Aucun champ pour les scopes OAuth !
```

```apex
// ❌ Metadata API ne retourne PAS la définition complète
MetadataService.ConnectedApp app = 
    (MetadataService.ConnectedApp)service.readMetadata(
        'ConnectedApp', 
        new String[] {'namespace__ManagedApp'}
    ).getRecords()[0];
// app sera NULL ou incomplet pour une managed app
```

Sans accès au packaging org, vous ne pouvez pas obtenir les métadonnées d'une Connected App en dehors d'un managed package.

## 🚫 Pourquoi c'est bloqué ?

### Architecture "Shadow Copy"

La Connected App dans le subscriber org est une **copie virtuelle** :

```
Packaging Org (Source)              Subscriber Org (Shadow)
├── Définition complète             ├── Référence seulement
│   ├── Consumer Key/Secret         │   ├── Voir les policies
│   ├── Scopes OAuth ✅             │   ├── Gérer permissions
│   ├── Certificats                 │   └── Configurer localement
│   └── Configuration de base       │
└── Modifiable                      └── Lecture limitée
```

## 💡 Solutions de contournement

### **Solution 1 : Via l'API OAuth Metadata** ⚠️ (Limité)

Il existe un endpoint REST peu documenté :

```apex
// Endpoint REST pour les métadonnées OAuth (peut fonctionner)
String endpoint = URL.getSalesforceBaseUrl().toExternalForm() + 
    '/services/oauth2/metadata';

HttpRequest req = new HttpRequest();
req.setEndpoint(endpoint);
req.setMethod('GET');
req.setHeader('Authorization', 'Bearer ' + UserInfo.getSessionId());

HttpResponse res = new Http().send(req);
System.debug(res.getBody());
```

**Résultat attendu :**
```json
{
  "issuer": "https://login.salesforce.com",
  "authorization_endpoint": "https://login.salesforce.com/services/oauth2/authorize",
  "token_endpoint": "https://login.salesforce.com/services/oauth2/token",
  "scopes_supported": ["api", "web", "full", "chatter_api", ...]
}
```

**Mais ⚠️** : Cela donne les scopes **globaux** de Salesforce, pas ceux spécifiques à votre app managed.

### **Solution 2 : Via l'Identity URL** 🎯 (Meilleure approche)

Quand un utilisateur s'authentifie, l'Identity URL contient les scopes :

```apex
// Après authentification OAuth
// L'Identity URL est fournie avec l'access token
String identityUrl = 'https://login.salesforce.com/id/00Dxx0000001gEREAY/005xx000001SwiUAAS';

HttpRequest req = new HttpRequest();
req.setEndpoint(identityUrl);
req.setMethod('GET');
req.setHeader('Authorization', 'Bearer ' + accessToken);

HttpResponse res = new Http().send(req);
Map<String, Object> identity = 
    (Map<String, Object>)JSON.deserializeUntyped(res.getBody());

// Les scopes accordés sont dans la réponse
List<Object> scopes = (List<Object>)identity.get('urls');
System.debug('Scopes actifs: ' + scopes);
```

**Exemple de réponse Identity :**
```json
{
  "id": "https://login.salesforce.com/id/00D.../005...",
  "asserted_user": true,
  "user_id": "005xx000001SwiUAAS",
  "organization_id": "00Dxx0000001gEREAY",
  "username": "user@example.com",
  "nick_name": "User",
  "display_name": "User Name",
  "email": "user@example.com",
  "urls": {
    "enterprise": "https://.../services/Soap/c/{version}/00D...",
    "metadata": "https://.../services/Soap/m/{version}/00D...",
    "partner": "https://.../services/Soap/u/{version}/00D...",
    "rest": "https://.../services/data/v{version}/",
    "sobjects": "https://.../services/data/v{version}/sobjects/",
    "search": "https://.../services/data/v{version}/search/",
    "query": "https://.../services/data/v{version}/query/",
    "recent": "https://.../services/data/v{version}/recent/",
    "tooling_soap": "https://.../services/Soap/T/{version}/00D...",
    "tooling_rest": "https://.../services/data/v{version}/tooling/",
    "profile": "https://...--c.na1.content.force.com/...",
    "feeds": "https://.../services/data/v{version}/chatter/feeds",
    "groups": "https://.../services/data/v{version}/chatter/groups",
    "users": "https://.../services/data/v{version}/chatter/users",
    "feed_items": "https://.../services/data/v{version}/chatter/feed-items",
    "custom_domain": "https://mydomain.my.salesforce.com"
  }
}
```

**MAIS** : Cela nécessite qu'un utilisateur se soit déjà authentifié.

### **Solution 3 : Interroger les tokens actifs** 💪 (Pratique)

```apex
// Voir quels scopes sont effectivement utilisés
List<OAuthToken> tokens = [
    SELECT Id, AppName, User.Username, 
           LastUsedDate, CreatedDate
    FROM OAuthToken
    WHERE AppName = 'YourManagedApp'
    ORDER BY LastUsedDate DESC
];

// Malheureusement, OAuthToken ne contient PAS les scopes directement
// Mais on peut voir qui utilise l'app
```

**Limitation** : L'objet `OAuthToken` ne contient **pas** les scopes accordés.

### **Solution 4 : Documentation contractuelle** 📋 (Recommandé)

Puisque c'est techniquement impossible, l'approche pragmatique :

```apex
/**
 * Managed Package: MyPackage
 * Connected App: MyPackage__MyApp
 * 
 * OAuth Scopes requis (documentés) :
 * - api : Accès API Salesforce
 * - web : Accès web-server flow
 * - refresh_token : Tokens de rafraîchissement
 * - full : Accès complet (pour fonctionnalités avancées)
 * 
 * Ces scopes sont configurés dans le packaging org et 
 * ne sont PAS modifiables dans les subscriber orgs.
 */
public class MyPackageConnectedApp {
    // Constantes documentant les scopes
    public static final Set<String> REQUIRED_SCOPES = new Set<String>{
        'api', 'web', 'refresh_token', 'full'
    };
    
    // Méthode pour vérifier si l'utilisateur a accordé l'accès
    public static Boolean isUserAuthorized(Id userId) {
        List<OAuthToken> tokens = [
            SELECT Id 
            FROM OAuthToken 
            WHERE UserId = :userId 
            AND AppName = 'namespace__MyApp'
            LIMIT 1
        ];
        return !tokens.isEmpty();
    }
}
```

### **Solution 5 : Via l'UI Salesforce (Manuel)** 🖱️

Dans le subscriber org, un admin **peut voir** les scopes dans l'interface :

```
Setup → Manage Connected Apps → [Votre App] 
→ Section "OAuth Policies"
→ Voir "Selected OAuth Scopes"
```

**Mais** : C'est en **lecture seule** et **manuel** - pas programmatique.

## 🔍 Vérification pratique dans le Subscriber Org

```apex
// Script pour diagnostiquer ce qui est accessible
public class ConnectedAppScopeChecker {
    
    public static void checkAvailableInfo(String appName) {
        // 1. Vérifier l'objet ConnectedApplication
        List<ConnectedApplication> apps = [
            SELECT Id, Name, OptionsAllowAdminApprovedUsersOnly,
                   OptionsRefreshTokenValidityMetric
            FROM ConnectedApplication 
            WHERE Name = :appName
        ];
        
        if(apps.isEmpty()) {
            System.debug('❌ App non trouvée dans ConnectedApplication');
            return;
        }
        
        ConnectedApplication app = apps[0];
        System.debug('✅ App trouvée: ' + app.Id);
        
        // 2. Utiliser Describe pour voir tous les champs
        Map<String, Schema.SObjectField> fields = 
            Schema.SObjectType.ConnectedApplication.fields.getMap();
        
        System.debug('Champs disponibles:');
        for(String fieldName : fields.keySet()) {
            if(fieldName.containsIgnoreCase('oauth') || 
               fieldName.containsIgnoreCase('scope')) {
                System.debug('  - ' + fieldName);
            }
        }
        
        // 3. Vérifier les tokens actifs
        List<OAuthToken> tokens = [
            SELECT Id, AppName, User.Username, LastUsedDate
            FROM OAuthToken 
            WHERE AppName = :appName
            LIMIT 5
        ];
        
        System.debug('Tokens actifs: ' + tokens.size());
        
        // 4. Tenter Metadata API (si apex-mdapi est disponible)
        try {
            // Cette partie nécessite apex-mdapi
            System.debug('⚠️ Metadata API: Tentative de lecture...');
            // MetadataService.ConnectedApp metaApp = ...
            // Probablement NULL ou erreur pour une managed app
        } catch(Exception e) {
            System.debug('❌ Metadata API: ' + e.getMessage());
        }
    }
}

// Exécuter
ConnectedAppScopeChecker.checkAvailableInfo('YourManagedApp');
```

**Résultat typique :**
```
✅ App trouvée: 0H4xx0000000001
Champs disponibles:
  - OptionsAllowAdminApprovedUsersOnly
  - OptionsRefreshTokenValidityMetric
  (PAS de champ pour les scopes)
Tokens actifs: 3
⚠️ Metadata API: Entity not found / Access denied
```

## 📊 Tableau récapitulatif : Accès aux Scopes

| Contexte | Méthode | Scopes accessibles ? | Notes |
|----------|---------|---------------------|-------|
| **Packaging Org** | Metadata API | ✅ Oui | Accès complet |
| **Packaging Org** | SOQL | ❌ Non | Pas de champ dédié |
| **Subscriber Org** | Metadata API | ❌ Non | Shadow copy - accès refusé |
| **Subscriber Org** | SOQL | ❌ Non | Pas de champ dédié |
| **Subscriber Org** | UI (manuel) | ✅ Oui (lecture) | Visible mais pas programmable |
| **Subscriber Org** | Identity URL | ⚠️ Partiel | Après auth, scopes actifs |
| **Subscriber Org** | OAuth Metadata | ⚠️ Non | Scopes globaux SF, pas app-specific |

## 🎯 Recommandations pragmatiques

### **Pour les ISV / Package Providers :**

1. **Documenter clairement les scopes** dans la documentation du package
2. **Fournir une classe helper** qui liste les scopes requis
3. **Créer un custom setting** ou custom metadata pour tracker la configuration

```apex
// Exemple : Custom Metadata Type
// MyPackage__ConnectedAppConfig__mdt
public class ConnectedAppConfig {
    public static List<String> getRequiredScopes() {
        MyPackage__ConnectedAppConfig__mdt config = 
            MyPackage__ConnectedAppConfig__mdt.getInstance('Default');
        
        return config.RequiredScopes__c.split(',');
        // Exemple: "api,web,refresh_token,full"
    }
}
```

4. **Post-install script** pour vérifier la configuration

```apex
global class PostInstallScript implements InstallHandler {
    global void onInstall(InstallContext context) {
        // Vérifier que la Connected App est installée
        List<ConnectedApplication> apps = [
            SELECT Id, Name 
            FROM ConnectedApplication 
            WHERE Name = 'namespace__MyApp'
        ];
        
        if(apps.isEmpty()) {
            // Envoyer email à l'admin avec instructions
            sendSetupInstructions(context.installerID());
        }
    }
    
    private void sendSetupInstructions(Id adminId) {
        Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();
        email.setTargetObjectId(adminId);
        email.setSubject('Configuration requise: Connected App');
        email.setPlainTextBody(
            'La Connected App requiert les scopes OAuth suivants:\n' +
            '- api\n' +
            '- web\n' +
            '- refresh_token\n' +
            '\n' +
            'Ces scopes sont automatiquement configurés. ' +
            'Veuillez autoriser les profils appropriés dans Setup.'
        );
        Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{email});
    }
}
```

### **Pour les Admins / Subscribers :**

Si vous devez auditer les scopes d'une managed app :

1. **Via l'UI** : Setup → Manage Connected Apps → Voir les scopes
2. **Contacter l'ISV** pour la documentation officielle
3. **Tester l'authentification** et consulter l'Identity URL
4. **Monitorer les OAuthTokens** pour voir l'utilisation effective

## ✅ Conclusion

**Oui, vous êtes effectivement bloqué** pour récupérer programmatiquement les scopes d'une managed connected app depuis un subscriber org via Metadata API ou SOQL.

**Mais** :
- ✅ Les scopes sont visibles manuellement dans l'UI
- ✅ Les scopes sont récupérables depuis le packaging org
- ✅ Vous pouvez documenter/hardcoder les scopes dans votre code
- ✅ Vous pouvez vérifier les scopes actifs via Identity URL après auth

**La meilleure pratique** : Documenter explicitement les scopes requis dans votre package et fournir des helpers pour que les admins sachent ce qui est configuré.

Avez-vous un cas d'usage spécifique qui nécessite l'accès programmatique aux scopes dans le subscriber org ?
