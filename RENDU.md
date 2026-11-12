
# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :CAPO-CHICHI Samuela
**Groupe :I2-APP-RS2

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > 3 nœuds. `k3d-lab-server-0` est le **control plane** (master) : il héberge l'API Server, le scheduler, le controller manager et la base d'état du cluster. `k3d-lab-agent-0` et `k3d-lab-agent-1` sont les **workers** : ils exécutent les pods des applications.
   > Sur un cluster « vide », des pods tournent déjà dans le namespace `kube-system` : coredns (DNS interne), traefik (ingress controller), metrics-server, local-path-provisioner (stockage) et svclb-traefik.

2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` montre un conteneur par nœud (`k3d-lab-server-0`, `k3d-lab-agent-0`, `k3d-lab-agent-1`, image `rancher/k3s`), plus `k3d-lab-serverlb`, le load balancer qui expose les ports du cluster (ex. 8081). Concrètement, un nœud k3d est un **conteneur Docker qui exécute k3s** et joue le rôle d'une machine du cluster.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > Un **Deployment** `web`, qui crée un **ReplicaSet** `web-xxxx`, qui crée lui-même un **Pod** `web-xxxx-yyyy`. Le **Service** `web` (ClusterIP) a été créé ensuite par `kubectl expose`.

4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Un nouveau pod a été recréé immédiatement, avec un **nom différent**. Le ReplicaSet doit maintenir 1 réplica : il constate l'écart entre l'état désiré (1 pod) et l'état réel (0 pod) et le corrige. C'est la **boucle de réconciliation** (auto-réparation).

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > Non. Le système de fichiers d'un conteneur est **éphémère** : le nouveau pod repart de l'image `nginx` d'origine, avec la page par défaut. Les modifications faites à la main dans un conteneur sont perdues à chaque redémarrage.

6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > Un `port-forward` vers un Service s'attache en réalité à **un seul pod**, choisi au lancement. Quand ce pod est supprimé, le tunnel perd sa cible et s'arrête (erreur visible dans `/tmp/web.log`). Il faut le relancer vers le nouveau pod.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `kubectl create` est **impératif** : il crée une ressource et échoue si elle existe déjà. `kubectl apply -f` est **déclaratif** : il compare le fichier YAML à l'état du cluster et crée ou met à jour la ressource pour qu'elle corresponde. On peut le relancer autant de fois qu'on veut (idempotent), et les fichiers peuvent être versionnés dans Git.

8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` indique au Service quels pods cibler, grâce à leurs **labels** (ici `app: site`). S'il ne correspond à aucun pod, le Service n'a **aucun endpoint** (`kubectl get endpoints site` est vide) : il existe toujours mais ne transmet le trafic à personne, et le site est inaccessible.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   NAME                    READY   STATUS    RESTARTS   AGE   IP          NODE               NOMINATED NODE   READINESS GATES
site-54dbc4f754-6g54t   1/1     Running   0          32m   10.42.1.8   k3d-lab-agent-0    <none>           <none>
site-54dbc4f754-b8nr4   1/1     Running   0          32m   10.42.2.8   k3d-lab-agent-1    <none>           <none>
site-54dbc4f754-vhpd5   1/1     Running   0          31m   10.42.0.8   k3d-lab-server-0   <none>           <none>
   
   > Le scheduler a réparti les 3 réplicas sur les 3 nœuds, un par nœud, pour équilibrer la charge et limiter l'impact de la panne d'un nœud. Un pod tourne même sur le control plane `k3d-lab-server-0` : dans k3s, le nœud server n'a pas de *taint* par défaut et peut donc recevoir des pods applicatifs, contrairement à un cluster Kubernetes classique où le master en est protégé. Chaque pod a sa propre IP, prise dans le sous-réseau de son nœud (10.42.0.x, 10.42.1.x, 10.42.2.x).

10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > Statut `ErrImagePull` puis `ImagePullBackOff`. La commande `kubectl describe pod <nom-du-pod>` donne la cause dans la section **Events** : le tag `nginx:9.99` n'existe pas sur Docker Hub (« not found » / « manifest unknown »).

11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > Oui. Le Deployment utilise une stratégie **RollingUpdate** : il ne supprime les anciens pods que lorsque les nouveaux sont prêts. Le nouveau pod n'ayant jamais démarré, les 3 anciens pods `nginx:1.28` ont continué de tourner, et le Service n'envoie le trafic qu'aux pods prêts. `kubectl rollout undo` a ensuite annulé la mise à jour ratée.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > https://effective-eureka-jrggrw97x7pq3vgr-8585.app.github.dev/
    

13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > `kubectl get deploy mario -o yaml` affiche le Deployment, qui est un objet Kubernetes standard, identique à ceux créés en ligne de commande (Portainer peut seulement ajouter quelques labels ou annotations). Oui, Portainer et `kubectl` parlent au même composant : l'**API Server** du control plane. Portainer tourne dans le cluster et utilise un ServiceAccount avec les droits nécessaires pour appeler cette API.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > **A – Ingress** : Ingress Traefik vers le Service `site`, accessible sur le port 8081. Contrairement au port-forward, le site reste accessible quand on supprime des pods, car l'Ingress passe par le Service qui répartit le trafic sur tous les pods prêts.
    > **B – Probes et ressources** : `readinessProbe` HTTP sur `/` (port 80) et requests/limits 50m/100m CPU, 32Mi/64Mi mémoire. Avec une probe sur `/nope`, nginx répond 404 : les pods restent `0/1` (jamais Ready) et ne reçoivent aucun trafic.
    > **D – Namespaces** : copie du site déployée dans le namespace `prod`, isolée du namespace `default`.

---

**Score obtenu avec `./scripts/check.sh` : 18 / 18