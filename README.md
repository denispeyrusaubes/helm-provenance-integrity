# helm-provenance-integrity




## Objectif

Cette issue a pour objectif d'expliquer la mise en place de la signature d'un chart Helm signé et son déploiement dans Kubernetes :
- A la main
- Via argoCD




## Signature d'un chart

L'objectif de la signature d'un chart est double et permet d'assurer :
- L'intégrité du chart : s'assurer ainsi qu'il n'a pas été modifié depuis qu'il a été signé
- Vérifier que l'on connait bien celui qui l'a créé car nous avons pris le soin d'obtenir puis installer sa clé publique

Dans ce document, nous allons travaillé sur un chart basique, spécifiquement créé pour illustrer nos propos :

```
helm create test-chart
```

> Le chart créé est basique et se limite à déployer un pod nginx et quelques ressources complémentaires telles qu'un service, etc

Une fois le chart Helm créé, nous souhaitons le signer. Pour cela, nous utilisons 
Nous avons scrupuleusement suivi la documentation officielle de [Helm](https://helm.sh/docs/topics/provenance/), avec succès.

Après ce processus de signature, nous obtenons deux fichiers :

- `test-chart-0.1.0.tgz` : le chart
- `test-chart-0.1.0.tgz.prov` : Ce fichier contient les informations de signature du chart ci dessus



## Delivery du chart

Les deux fichiers d'un chart signé doivent-être délivrés dans ChartMuseum :

```
curl -u admin:PASSWORD -F "chart=@test-chart-0.1.0.tgz" -F "prov=@test-chart-0.1.0.tgz.prov" https://helm.opt.nc/api/charts
```

> Par convention, les deux fichiers doivent avoir le même codebase.


## Utilisation d'un chart signé

> Attention, pour remplir l'objectif de cette documentation, le code ci-dessous est exécuté sur une machine différente de celle utilisée pour signer l'application (i.e. la clé publique n'est pas présente pour le moment). Nous considérons néanmoins que cette machine est paramétrée pour accéder à un cluster Kubernetes

Nous téléchargeons le chart signé ainsi que son proficher provenance :

```
helm repo add opt https://helm.opt.nc
helm pull opt/test-chart --prov
```

Une fois téléchargés, nous allons utiliser helm afin d'en vérifier l'intégrité et l'auteur.

```
helm verify ./test-chart-0.1.0.tgz
Error: failed to load keyring: open /home/retengr/.gnupg/pubring.gpg: no such file or directory
```


Importez la clé publique associée au chart (Par mesure de simplicité elle est fournie dans ce repo github):

```
gpg --import public.key
```


> Warning: the GnuPG v2 store your secret keyring using a new format kbx on the default location ~/.gnupg/pubring.kbx. Please use the following command to convert your keyring to the legacy gpg format:
```
gpg --export >~/.gnupg/pubring.gpg
```

Et finalement :

```
helm verify ./test-chart-0.1.0.tgz
Signed by: Denis Peyrusaubes (test OPT) <denis.peyrusaubes@retengr.nc>
Using Key With Fingerprint: E4AF0C705574BCA3CFE391BFC8D427BDBEC27A77
Chart Hash Verified: sha256:52aa2e76d9b91547fa903cdbcdef5eb1d5b307c5b3b65dcad45b9448a46a5398
```

Pour refaire le test à l'envers :

``` 
gpg --list-keys
gpg --delete-key Denis
gpg --export >~/.gnupg/pubring.gpg
helm verify ./test-chart-0.1.0.tgz
Error: openpgp: signature made by unknown entity
```

> On supprime la clé de keychain gpg (que l'on converti ensuite dans le format adapté), Le chart ne peut alors plus être vérifié !


Il est aussi possible de procéder à la vérification au moment de l'installation :

```
helm install --generate-name --verify test-chart-0.1.0.tgz
```


## ArgoCD et Kubernetes

[Documentation Officielle](https://argo-cd.readthedocs.io/en/stable/user-guide/gpg-verification/)

Il est mentionnée que :

`Verification of GnuPG signatures is only supported with Git repositories. It is not possible using Helm repositories.`

Réflexion : est-il possible de modifier la ligne de commande d'installation helm qui est utilisé par ArgoCD, afin de rajouter l'option --verify ? Il resterait ensuite la question de savoir comment il pourrait trouver la clé publique...


```
helm install --generate-name --verify test-chart-0.1.0.tgz
```
