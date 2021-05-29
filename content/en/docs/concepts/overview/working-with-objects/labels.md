---
critiques:
- mikedanese
titre: Étiquettes et sélecteurs
type de contenu: concept
poids: 40
---

<!-- overview -->

Les étiquettes sont des paires clé / valeur attachées à des objets, tels que des pods. Les étiquettes sont destinées à être utilisées pour spécifier les attributs d'identification des objets qui sont significatifs et pertinents pour les utilisateurs, mais n'impliquent pas directement la sémantique du système central. Les étiquettes peuvent être utilisées pour organiser et sélectionner des sous-ensembles d'objets. Les étiquettes peuvent être attachées aux objets au moment de la création, puis ajoutées et modifiées à tout moment. Chaque objet peut avoir un ensemble d'étiquettes clé / valeur définies. Chaque clé doit être unique pour un objet donné.
```json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

Les étiquettes permettent des requêtes et des contrôles efficaces et sont idéales pour une utilisation dans les interfaces utilisateur et les CLI. Les informations non identifiantes doivent être enregistrées en utilisant
[annotations](/docs/concepts/overview/working-with-objects/annotations/).

<!-- body -->

## Motivation

Les libellés permettent aux utilisateurs de mapper leurs propres structures organisationnelles sur des objets système de manière faiblement couplée, sans exiger des clients qu'ils stockent ces mappages.

Les déploiements de services et les pipelines de traitement par lots sont souvent des entités multidimensionnelles (par exemple, plusieurs partitions ou déploiements, plusieurs pistes de publication, plusieurs niveaux, plusieurs micro-services par niveau). La gestion nécessite souvent des opérations transversales, ce qui rompt l'encapsulation des représentations strictement hiérarchiques, en particulier des hiérarchies rigides déterminées par l'infrastructure plutôt que par les utilisateurs.

Exemples d'étiquettes:

   * `"release" : "stable"`, `"release" : "canary"`
   * `"environment" : "dev"`, `"environment" : "qa"`, `"environment" : "production"`
   * `"tier" : "frontend"`, `"tier" : "backend"`, `"tier" : "cache"`
   * `"partition" : "customerA"`, `"partition" : "customerB"`
   * `"track" : "daily"`, `"track" : "weekly"`

Ce sont des exemples d'étiquettes couramment utilisées; vous êtes libre de développer vos propres conventions. Gardez à l'esprit que l'étiquette Key doit être unique pour un objet donné.

## Syntaxe et jeu de caractères

Les libellés sont des paires clé / valeur. Les clés d'étiquette valides ont deux segments: un préfixe et un nom facultatifs, séparés par une barre oblique (`/`). Le segment de nom est obligatoire et ne doit pas comporter plus de 63 caractères, commençant et se terminant par un caractère alphanumérique (`[a-z0-9A-Z]`) avec des tirets (`-`), souligne (`_`), points (`.`), et alphanumériques entre. Le préfixe est facultatif. S'il est spécifié, le préfixe doit être un sous-domaine DNS: une série d'étiquettes DNS séparées par des points (`.`), pas plus de 253 caractères au total, suivis d'une barre oblique (`/`).

Si le préfixe est omis, l'étiquette Clé est présumée privée pour l'utilisateur. Composants du système automatisé (e.g. `kube-scheduler`, `kube-controller-manager`, `kube-apiserver`, `kubectl`, ou autre automatisation tierce) qui ajoutent des étiquettes aux objets de l'utilisateur final doivent spécifier un préfixe.

Le `kubernetes.io/` et `k8s.io/` les préfixes sont réservés aux composants principaux de Kubernetes.

Valeur d'étiquette valide:
* doit contenir 63 caractères ou moins (peut être vide),
* sauf s'il est vide, doit commencer et se terminer par un caractère alphanumérique (`[a-z0-9A-Z]`),
* pourrait contenir des tirets (`-`), souligne (`_`), points (`.`), et entre alphanumériques.

Par exemple, voici le fichier de configuration d'un pod qui a deux étiquettes `environment: production` et `app: nginx` :

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80

```

## Sélecteurs d'étiquettes

Contrairement à [noms et UIDs](/docs/concepts/overview/working-with-objects/names/), les étiquettes n'offrent pas de caractère unique. En général, nous nous attendons à ce que de nombreux objets portent la même étiquette(s).

Via un sélecteur d'étiquettes, le client / utilisateur peut identifier un ensemble d'objets. Le sélecteur d'étiquettes est la primitive de regroupement de base dans Kubernetes.

L'API prend actuellement en charge deux types de sélecteurs: _equality-based_ and _set-based_.
Un sélecteur d'étiquette peut être constitué de plusieurs _exigences_ séparées par des virgules. Dans le cas d'exigences multiples, toutes doivent être satisfaites pour que le séparateur par virgule agisse comme un opérateur logique _AND_ (`&&`) operateur.

La sémantique des sélecteurs vides ou non spécifiés dépend du contexte,
et les types d'API qui utilisent des sélecteurs doivent documenter la validité et la signification de
eux.

{{< Remarque >}}
Pour certains types d'API, tels que les ReplicaSets, les sélecteurs d'étiquettes de deux instances ne doivent pas se chevaucher dans un espace de noms, ou le contrôleur peut voir cela comme des instructions contradictoires et ne pas déterminer le nombre de réplicas qui doivent être présents.
{{< /Remarque >}}

{{< mise en garde >}}
For both equality-based and set-based conditions there is no logical _OR_ (`||`) operator. Ensure your filter statements are structured accordingly.
{{< /mise en garde >}}

### Exigeance _Equality-based_

_Equality-_ ou _inequality-based_ les exigences permettent le filtrage par clés d'étiquette et valeurs. Les objets correspondants doivent satisfaire toutes les contraintes d'étiquette spécifiées, bien qu'ils puissent également avoir des étiquettes supplémentaires.
Trois types d'opérateurs sont admis `=`,`==`,`!=`. Les deux premiers représentent _equality_ (et sont des synonymes), tandis que ce dernier représente l'inégalité.
<br>
Par exemple:

```
environment = production
tier != frontend
```

Le premier sélectionne toutes les ressources dont la clé est égale à `environment` et valeur égale à `production`.
Ce dernier sélectionne toutes les ressources dont la clé est égale à `tier` et une valeur distincte de `frontend`, et toutes les ressources sans étiquette avec la clé `tier`.
On pourrait filtrer les ressources dans `production` excluant `frontend` en utilisant l'opérateur virgule: `environment=production,tier!=frontend`

Un scénario d'utilisation pour l'exigence d'étiquette basée sur l'égalité est que les pods spécifient
critères de sélection des nœuds. Par exemple, l'exemple de pod ci-dessous sélectionne des nœuds avec
l'étiquette "`accelerator=nvidia-tesla-p100`".

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-test
spec:
  containers:
    - name: cuda-test
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
  nodeSelector:
    accelerator: nvidia-tesla-p100
```

### Exigence basée sur un ensemble Set-based

Les exigences d'étiquette _Set-based_ permettent de filtrer les clés en fonction d'un ensemble de valeurs. Trois types d'opérateurs sont pris en charge: `in`,`notin` et `exists` (uniquement l'identifiant de la clé). Par exemple:

```
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```

* Le premier exemple sélectionne toutes les ressources dont la clé est égale à `environment` et valeur égale à `production` ou `qa`.
* Le deuxième exemple sélectionne toutes les ressources dont la clé est égale à `tier` et des valeurs autres que `frontend` et `backend`, et toutes les ressources sans étiquette avec la clé `tier`.
* Le troisième exemple sélectionne toutes les ressources, y compris une étiquette avec clé `partition`; aucune valeur n'est vérifiée.
* Le quatrième exemple sélectionne toutes les ressources sans étiquette avec clé `partition`; aucune valeur n'est vérifiée.

De même, le séparateur par virgule agit comme un opérateur _AND_. Donc, filtrer les ressources avec une clé `partition`(peu importe la valeure) et avec `environnement` différent de `qa` peut être réalisé en utilisant `partition,environment notin (qa)`.
Le sélecteur d'étiquettes basé sur un ensemble(Set-based) est une forme générale d'égalité puisque `environment=production` il est équivalent à `environment in (production)`; de même pour `!=` et `notin`.

Les exigences _set-based_ peuvent être combinées avec des exigences _equality-based_. Par exemple:<br> `partition in (customerA, customerB),environment!=qa`.


## API

### Filtrage LISTE et visuel (LIST AND WATCH)

Les opérations LIST et WATCH peuvent spécifier des sélecteurs d'étiquettes pour filtrer les ensembles d'objets renvoyés à l'aide d'un paramètre de requête. Les deux exigences sont autorisées (présentées ici telles qu'elles apparaissent dans une chaîne de requête URL):
  * exigences fondées sur l'égalité (_equality-based_): `?labelSelector=environment%3Dproduction,tier%3Dfrontend`
  * exigences basées sur les ensembles (_set-based_): `?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29`

Les deux styles de sélecteur d'étiquettes peuvent être utilisés pour répertorier ou surveiller des ressources via un client REST. Par exemple, le ciblage `apiserver` avec `kubectl` et l'utilisation de _equality-based_ on peut écrire:

```shell
kubectl get pods -l environment=production,tier=frontend
```

ou utiliser l'exigence basée sur les ensembles (_set-based_) :

```shell
kubectl get pods -l 'environment in (production),tier in (frontend)'
```

Comme déjà mentionné, les exigences basées sur les ensembles sont plus expressives.  Par exemple, ils peuvent implémenter l'opérateur _OR_ sur les valeurs:

```shell
kubectl get pods -l 'environment in (production, qa)'
```

ou restreindre la correspondance négative via l'opérateur _exists_:

```shell
kubectl get pods -l 'environment,environment notin (frontend)'
```

### Définir des références dans les objets API

Certains objets Kubernetes, tels que [`services`](/docs/concepts/services-networking/service/)
et [`replicationcontrollers`](/docs/concepts/workloads/controllers/replicationcontroller/),
utilisez également des sélecteurs d'étiquettes pour spécifier des ensembles d'autres ressources, telles que
[pods](/docs/concepts/workloads/pods/).

#### Service et ReplicationController

L'ensemble des pods qu'un `service` Les cibles sont définies avec un sélecteur d'étiquettes. De même, la population de gousses qui a `replicationcontroller` should manage est également défini avec un sélecteur d'étiquettes.

Les sélecteurs d'étiquettes pour les deux objets sont définis dans `json` ou `yaml` les fichiers utilisant des cartes, et seuls les sélecteurs d'exigences _basés sur l'égalité(_equality-based_) sont pris en charge:

```json
"selector": {
    "component" : "redis",
}
```
ou

```yaml
selector:
    component: redis
```

ce sélecteur (respectivement dans `json` ou `yaml` format) est équivalent à `component=redis` ou `component in (redis)`.

#### Ressources prenant en charge les exigences basées sur les ensembles

Des ressources plus récentes, telles que [`Job`](/docs/concepts/workloads/controllers/job/),
[`Deployment`](/docs/concepts/workloads/controllers/deployment/),
[`ReplicaSet`](/docs/concepts/workloads/controllers/replicaset/), et
[`DaemonSet`](/docs/concepts/workloads/controllers/daemonset/),
prend également en charge les exigences basées sur les ensembles(_set-based_).

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```

`matchLabels` est une carte de `{key,value}` paires. Un seul `{key,value}` dans la carte `matchLabels` équivaut à un élément de `matchExpressions`, dont le champ `key` est "key", l'opérateur `operator` est "In", et le tableau `values` contient uniquement "value". `matchExpressions`est une liste des exigences du sélecteur de pod. Les opérateurs valides incluent In, NotIn, Exists, and DoesNotExist.Les valeurs définies doivent être non vides dans le cas de In et NotIn.Toutes les exigences, des deux `matchLabels`et `matchExpressions` sont ANDed together -- ils doivent tous être satisfaits pour correspondre.

#### Sélection d'ensembles de nœuds.

Un cas d'utilisation pour la sélection sur des étiquettes est de contraindre l'ensemble de nœuds sur lequel un pod peut planifier.
Voir la documentation sur [node selection](/docs/concepts/scheduling-eviction/assign-pod-node/) pour plus d'information.
