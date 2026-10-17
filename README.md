#  Lab 3.2 : Ingress avec Traefik sur Kubernetes (K3s)

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.36.5-blue?logo=kubernetes)
![K3s](https://img.shields.io/badge/K3s-Lightweight-orange?logo=rancher)
![Traefik](https://img.shields.io/badge/Ingress-Traefik-24A1C1?logo=traefik)

##  Description du projet
Ce dépôt contient le compte rendu complet du **Lab 3.2**. L'objectif est de remplacer l'exposition par ports (NodePort/LoadBalancer) vue au Lab 3.1 par un **point d'entrée unique** via un **Ingress Controller (Traefik)**. 

Nous avons déployé deux applications (Nginx et Podinfo) et configuré le routage par **nom d'hôte** et par **chemin**, tout en testant les erreurs de routage.

---

##  Environnement de travail
- **Cluster** : K3s v1.36.5 (Traefik inclus par défaut)
- **Topologie** : 1 Master (`192.168.56.20`) + 2 Workers (`192.168.56.31`, `192.168.56.32`)
- **Ingress Controller** : Traefik (IngressClass `traefik`)

---

##  Étapes réalisées et Résultats

### Étape 1 : Déployer les deux applications
Création d'un namespace `tp-k8s` et déploiement de deux applications :
- **web** : Nginx (2 réplicas)
- **api** : Podinfo (3 réplicas)

```bash
kubectl apply -f apps.yaml
kubectl get pods
kubectl get endpoints
✅ Résultat : 5 Pods Running (2 web, 3 api). Les endpoints web-svc (2 IPs) et api-svc (3 IPs) sont bien peuplés.

https://./captures_lab32/01_apps_running.png

Étape 2 : Repérer Traefik
Vérification que l'Ingress Controller est opérationnel.

bash
kubectl get pods -n kube-system | grep traefik
kubectl get service traefik -n kube-system
kubectl get ingressclass
✅ Résultat : Le Pod traefik est Running. Le Service traefik est de type LoadBalancer avec les ports 80 et 443 exposés sur toutes les IPs des nœuds. L'IngressClass se nomme traefik.

https://./captures_lab32/02_traefik_ingressclass.png

Étape 3 : Créer un Ingress par nom d'hôte
Création d'un Ingress qui route web.k3s.local vers web-svc et api.k3s.local vers api-svc.

bash
kubectl apply -f ingress-hotes.yaml
kubectl get ingress
kubectl describe ingress apps-ingress
✅ Résultat : L'Ingress apps-ingress est créé. L'ADDRESS est remplie (IP du nœud). Les règles pointent vers les bons backends.

https://./captures_lab32/03_ingress_hotes.png

Étape 4 : Tester avec l'en-tête Host
Test depuis le terminal (VM) en simulant le nom de domaine.

bash
curl -H "Host: web.k3s.local" http://192.168.56.20
curl -H "Host: api.k3s.local" http://192.168.56.20
✅ Résultat : web.k3s.local renvoie la page HTML de Nginx. api.k3s.local renvoie le JSON de Podinfo.

https://./captures_lab32/04_test_host_header.png

Étape 5 : Utiliser de vrais noms d'hôte (Windows)
Configuration du fichier hosts de Windows (C:\Windows\System32\drivers\etc\hosts) pour associer 192.168.56.20 aux noms web.k3s.local et api.k3s.local.

cmd
curl.exe http://web.k3s.local
curl.exe http://api.k3s.local
✅ Résultat : Le navigateur et curl.exe sur Windows affichent bien les applications sans avoir besoin de l'en-tête Host.

https://./captures_lab32/05_test_navigateur.png

Étape 6 : Router par chemin
Création d'un Ingress site-ingress qui utilise le même hôte (site.k3s.local) mais route selon le chemin :

/ → web-svc (Nginx)

/version → api-svc (Podinfo)

bash
kubectl apply -f ingress-chemins.yaml
curl -H "Host: site.k3s.local" http://192.168.56.20/
curl -H "Host: site.k3s.local" http://192.168.56.20/version
✅ Résultat : / renvoie Nginx, /version renvoie le JSON de Podinfo (routage par préfixe le plus long).

https://./captures_lab32/06_routage_chemin.png

Étape 7 : Distinguer les erreurs
Test de 3 scénarios d'erreur :

Hôte inconnu → Erreur de Traefik (404 page not found).

Chemin inexistant sur Nginx → Erreur de Nginx (page HTML 404 Not Found).

Service sans Pod → Erreur de Traefik (503 Service Unavailable).

bash
# a) Hôte inconnu
curl -i -H "Host: inconnu.k3s.local" http://192.168.56.20

# b) Chemin inexistant
curl -i -H "Host: web.k3s.local" http://192.168.56.20/nexiste-pas

# c) Service sans Pod
kubectl scale deployment api --replicas=0
curl -i -H "Host: api.k3s.local" http://192.168.56.20
kubectl scale deployment api --replicas=3
✅ Résultat : On distingue bien les 3 erreurs et on sait si le problème vient du routage (Traefik) ou de l'application (Nginx/API).

https://./captures_lab32/07_erreurs.png

Étape 8 : Nettoyage
bash
kubectl delete namespace tp-k8s
kubectl config set-context --current --namespace=default
✅ Résultat : L'environnement est propre.

https://./captures_lab32/08_nettoyage_lab32.png

🧠 Réponses aux questions de réflexion
1. Pourquoi préfère-t-on un Ingress à deux Services de type NodePort pour des applications web ?

L'Ingress offre un point d'entrée unique (ports 80 et 443) avec des noms d'hôte lisibles et du routage par chemin. Les NodePort multiplient les ports élevés, sans routage par nom ni par chemin.

2. Quel est le rôle de l'objet Ingress, et celui de Traefik ?

L'Ingress décrit les règles de routage. Traefik, l'Ingress controller, les lit et route réellement le trafic. Sans controller, l'Ingress n'a aucun effet.

3. Pourquoi l'application répond-elle sur l'adresse de n'importe quel nœud ?

Traefik est exposé par un Service LoadBalancer (ServiceLB), qui ouvre les ports 80 et 443 sur chaque nœud. Kube-proxy redirige le trafic vers les pods Traefik.

4. Une règle /version et une règle / existent pour le même hôte. Laquelle s'applique à /version ?

La règle /version, car la correspondance la plus spécifique (le préfixe le plus long) l'emporte.

5. Comment reconnaître qu'une réponse 404 vient de Traefik plutôt que de l'application ?

En observant le contenu de la réponse : celui de Traefik est un message court (404 page not found en texte brut), celui de nginx est une page HTML. Si l'erreur est celle de Traefik, aucune règle n'a correspondu : vérifier l'hôte, le chemin et l'Ingress.

👨‍💻 Auteur
Nom : [Dabbech]

Prénom : [Narjess]

Classe : [Ta Classe]

GitHub : @Narjess805

Date : Octobre 2026
