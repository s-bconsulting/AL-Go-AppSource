# Signature de l'app et publication sur AppSource

Documentation des deux contraintes spécifiques à ce template AppSource (en plus des affixes obligatoires `appSourceCopMandatoryAffixes`) : la signature du package et la soumission à Microsoft AppSource.

## 1. Signature de l'app

Ce n'est **pas** un workflow séparé : c'est une étape intégrée au build lui-même.

- **Où** : job `Sign` dans `.github/workflows/_BuildALGoProject.yaml` (action `microsoft/AL-Go-Actions/Sign`).
- **Déclenchement** : automatique, à **chaque build**. `CICD.yaml` passe `signArtifacts: true` en dur, donc ça s'exécute sur chaque push/PR/CI, ainsi qu'à chaque release via `CreateReleaseWithDeploy.yaml`. Ce n'est pas une étape manuelle ni réservée à la release.
- **Condition d'exécution réelle** : l'étape ne signe effectivement que si un certificat est configuré — `keyVaultCodesignCertificateName` (Key Vault) ou `trustedSigning` (Azure Trusted Signing) rempli dans les settings/secrets — et que `doNotSignApps` n'est pas activé. Si rien n'est configuré, l'étape est **silencieusement sautée**, sans faire échouer le build.
- **État actuel de ce repo** : aucun certificat n'est configuré → aucune signature effective pour l'instant. C'est un prérequis Microsoft à mettre en place avant une vraie soumission AppSource (le package doit être signé).

## 2. Soumission à AppSource

Workflow dédié : `.github/workflows/PublishToAppSource.yaml`.

- **Déclenchement** : 100% **manuel** (`workflow_dispatch` uniquement). Jamais automatique — même avec `autoReleaseOnMerge` activé, une release GitHub ne pousse jamais toute seule vers AppSource. C'est toujours un geste volontaire séparé.
- **Fonctionnement** : prend un build/version cible (`appVersion` : `current`, `prerelease`, `draft`, `latest` ou une version précise) et appelle l'action `Deliver` avec `deliveryTarget: AppSource`, authentifiée via le secret `appSourceContext` (credentials Partner Center).
- **Paramètre `GoLive`** :
  - `false` (défaut) : la soumission passe la validation technique Microsoft mais reste en attente — mise en ligne manuelle ensuite dans Partner Center.
  - `true` : passage live automatique dès que la validation technique réussit.

## 3. Retour des erreurs de certification Microsoft

Le retour se fait **uniquement via les logs du run GitHub Actions** : l'étape `Deliver` appelle l'API Partner Center, et si Microsoft rejette la soumission (échec de la validation technique), l'étape échoue et le job apparaît en rouge dans l'onglet Actions, avec le détail de l'erreur de validation dans les logs.

Il n'y a **pas de canal séparé** (pas d'issue GitHub auto-créée, pas d'email dédié) au-delà de ça — il faut soit consulter le run manuellement, soit avoir des notifications GitHub Actions configurées sur échec de workflow pour être alerté.

## 4. Ce qu'il faut pour publier (pas un "environnement" AL-Go)

Contrairement aux déploiements BC (`DeployTo<NomEnv>`), la publication AppSource **n'utilise pas de GitHub Environment** — le job `Deliver` de `PublishToAppSource.yaml` ne déclare aucun bloc `environment:`, donc pas de règle de protection/approbation à ce niveau. Ça repose sur trois éléments :

### a. Un Produit déjà créé dans Partner Center (préalable, hors AL-Go)

L'offre AppSource doit déjà exister côté Microsoft Partner Center. C'est de là que vient le **Product ID**.

### b. Le setting `deliverToAppSource` dans `.AL-Go/settings.json`

Pas encore présent dans ce repo. Bloc JSON à ajouter dans `.AL-Go/settings.json` :

```json
"deliverToAppSource": {
  "productId": "<id du produit Partner Center>",
  "mainAppFolder": "AppsourceSample",
  "continuousDelivery": false,
  "includeDependencies": []
}
```

Définition de chaque paramètre :

| Paramètre | Où | Définition |
|---|---|---|
| `productId` | `.AL-Go/settings.json` | **Obligatoire.** L'ID du produit récupéré dans Partner Center (l'offre AppSource doit déjà exister côté Microsoft). |
| `mainAppFolder` | `.AL-Go/settings.json` | Utile seulement si plusieurs apps dans le même projet — désigne laquelle est l'app principale à soumettre. |
| `continuousDelivery` | `.AL-Go/settings.json` | `true`/`false` (défaut `false`). Si `true`, **chaque build CI/CD réussi** est automatiquement soumis à la validation technique AppSource, sans action manuelle. L'app reste en "preview" — il faut ensuite cliquer "Go Live" manuellement dans Partner Center pour la rendre publique (`continuousDelivery` ne rend jamais rien public tout seul). |
| `includeDependencies` | `.AL-Go/settings.json` | Tableau de noms de fichiers (wildcards acceptés) de dépendances à inclure dans la soumission. Nécessite `generateDependencyArtifact: true` ailleurs dans les settings. |
| `GoLive` | **Input du workflow manuel** `PublishToAppSource.yaml` (pas un setting) | `true`/`false` (défaut `false` sur le workflow). Passé au moment de lancer `workflow_dispatch` : `false` = soumission qui reste en attente après validation technique (mise en ligne manuelle ensuite dans Partner Center) ; `true` = passage live automatique dès que la validation technique réussit. C'est l'équivalent manuel du "Go Live" mentionné pour `continuousDelivery`. |

En résumé : `continuousDelivery` (setting) automatise l'**envoi** vers la validation AppSource à chaque build ; `GoLive` (paramètre du workflow manuel `PublishToAppSource.yaml` uniquement, n'existe pas comme setting) contrôle si la **mise en ligne publique** se fait automatiquement une fois la validation technique réussie. Les deux sont indépendants l'un de l'autre.

### c. Le secret GitHub `appSourceContext`

Un JSON compressé contenant le contexte d'authentification Partner Center, lu par `PublishToAppSource.yaml` (`getSecrets: 'appSourceContext'`). Il se génère en deux étapes avec BcContainerHelper :

**Étape 1 — `New-BcAuthContext`** (construit le contexte d'authentification) :

- `clientID` : ID de l'app registration Azure AD à utiliser.
- `tenantID` : le tenant Azure AD (défaut `Common`).
- `scopes` : **doit être `https://api.partner.microsoft.com/.default`** (ou `https://api.partner.microsoft.com/user_impersonation offline_access`) — pas le scope BC par défaut.
- Un seul mode d'authentification parmi : `clientSecret` (client_credentials, non-interactif — adapté à un usage CI/CD répété), `credential` (login/mot de passe), `refreshToken`, ou `-includeDeviceLogin` (login interactif par code, pratique pour générer le contexte une fois à la main).

**Étape 2 — `New-ALGoAppSourceContext`** :

```powershell
$authContext = New-BcAuthContext -clientID <id> -clientSecret <secret> -tenantID <tenant> -scopes "https://api.partner.microsoft.com/.default"
New-ALGoAppSourceContext -authContext $authContext
```

- `-authContext` (obligatoire) : le hashtable produit par `New-BcAuthContext`.
- `-skipTest` (optionnel) : sans lui, la fonction valide réellement le contexte en appelant `Get-AppSourceProduct` contre l'API Partner Center — erreur immédiate si le contexte est invalide.

Le résultat est un JSON compressé (clientID/clientSecret/tenantID/scopes, ou refreshToken/tenantID/scopes) à coller tel quel dans le secret GitHub `appSourceContext`.

**Prérequis en amont** : l'app registration Azure AD utilisée doit avoir les permissions API vers Partner Center accordées (consentement admin), et le compte/l'app doit être ajouté comme utilisateur avec le rôle **Manager** directement dans Partner Center (indépendant d'Azure AD).

### Secret d'organisation (`appSourceContext` partagé entre plusieurs repos)

SB Consulting publie 5 repos sur AppSource sous le même compte Partner Center. Le secret `appSourceContext` est donc configuré **au niveau de l'organisation GitHub** (`Organization Settings → Secrets and variables → Actions`), avec la visibilité restreinte à **"Selected repositories"** (les 5 repos AppSource uniquement, pas les repos PTE). Chaque repo continue de lire un secret nommé `appSourceContext` sans rien changer côté workflow — GitHub résout automatiquement vers le secret d'organisation quand aucun secret de même nom n'existe au niveau du repo.

Avantages : un seul secret à générer/roter au lieu de 5, et un futur repo AppSource sous le même compte Partner Center n'a qu'à être ajouté à la liste "Selected repositories" plutôt que de refaire toute la procédure `New-BcAuthContext`/`New-ALGoAppSourceContext`. Si un repo AppSource utilise un compte Partner Center différent, il doit définir son propre secret au niveau repo (qui prend priorité sur celui de l'organisation).
