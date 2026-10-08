# Examen CinéK8s — Touil Mohammed

## Partie 1

**Q1.1**

 MovieClient lit la propriété `movie.url` (`@Value("${movie.url}")`).
On la surcharge sans toucher au code avec la variable d'environnement `MOVIE_URL`
(relaxed binding : majuscules, le point devient `_`).
**Q1.2** 
(a) Film inexistant : 422 Unprocessable Entity (movie répond 404, ticket le convertit en 422).
(b) Pas assez de places : 409 Conflict.
(c) movie-service injoignable : 503 Service Unavailable.
**Q1.3** 
Ligne complétée : `include: readinessState,movie`
La dépendance à movie doit être dans la readiness : si movie tombe, les Pods ticket
ne peuvent plus servir de réservations, donc le kubelet les retire des endpoints du
Service (plus de trafic) sans les tuer, et ils reviennent seuls quand movie revient.
Dans la liveness, l'échec ferait redémarrer tous les Pods ticket pour rien : redémarrer
ticket ne répare pas movie, et cela provoquerait une cascade de redémarrages.
**Q1.4** 
| Endpoint | Probe(s) Kubernetes | Conséquence d'un échec |
|---|---|---|
| /actuator/health/liveness | startupProbe et livenessProbe | Le kubelet redémarre le conteneur (RESTARTS augmente) |
| /actuator/health/readiness | readinessProbe | Le Pod est retiré des endpoints du Service, sans redémarrage |

`server.shutdown: graceful` : lors d'un rolling update, le Pod qui s'arrête termine
ses requêtes en cours avant de s'éteindre au lieu de les couper, ce qui évite des erreurs.

## Partie 2

**2.1**

```
simot@isco:~$ curl -s localhost:8085/api/movies | jq '.[].title'
"Pod Fiction"
"Le Seigneur des Pods"
"Docker Wars"
"Rollback to the Future"
simot@isco:~$ curl -s localhost:8085/api/movies/whoami
{"environment":"local","hostname":"isco"}
simot@isco:~$ curl -s -X POST localhost:8086/api/tickets -H 'Content-Type: application/json' -d '{"movieId":2,"seats":3}' | jq
{
  "id": 1,
  "movieId": 2,
  "movieTitle": "Le Seigneur des Pods",
  "seats": 3,
  "total": 36.00,
  "createdAt": "2026-10-08T10:09:49.897511509Z"
}
simot@isco:~$ curl -s localhost:8086/actuator/health/readiness | jq
{
  "status": "UP",
  "components": {
    "movie": {
      "status": "UP"
    },
    "readinessState": {
      "status": "UP"
    }
  }
}
```

**2.2**

```
simot@isco:~$ curl -s localhost:8086/actuator/health/readiness | jq
{
  "status": "DOWN",
  "components": {
    "movie": {
      "status": "DOWN",
      "details": {
        "error": "I/O error on GET request for \"http://localhost:8085/actuator/health/liveness\": null"
      }
    },
    "readinessState": {
      "status": "UP"
    }
  }
}
simot@isco:~$ curl -s localhost:8086/actuator/health/liveness | jq .status
"UP"
simot@isco:~$ curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8086/api/tickets -H 'Content-Type: application/json' -d '{"movieId":2,"seats":3}'
503
```

**Q2.1**

On lance ticket-service avec des variables d'environnement (MOVIE_URL, ou SERVER_PORT comme dans l'énoncé) plutôt que de modifier application.yaml, car le même jar doit
fonctionner dans tous les environnements sans être reconstruit. Le mécanisme est la
configuration externalisée de Spring Boot : les variables d'environnement ont une priorité
supérieure à application.yaml, et le relaxed binding convertit MOVIE_URL en movie.url. C'est ce qui permettra de
configurer l'application depuis une ConfigMap dans Kubernetes.

**Q2.2**

C'est exactement le comportement voulu : si movie-service tombe, redémarrer
ticket-service ne servirait à rien, car la panne n'est pas dans ticket. La liveness
répond « le processus est vivant », donc elle reste UP et le kubelet ne redémarre pas le
conteneur. La readiness répond « je peux traiter du trafic », donc elle passe DOWN :
dans Kubernetes, le Pod est alors retiré des endpoints du Service. Les clients ne
reçoivent plus de requêtes vouées à l'échec, et le Pod revient automatiquement quand
movie redevient disponible, sans redémarrage.
## Partie 3

**3.1**

```
simot@isco:~/cineK8s$ docker images | grep -E 'movie-service|ticket-service'
movie-service:1.0.0     4bb8dd2b6ac2     331MB     95.2MB
ticket-service:1.0.0    edfe9ac22d2e     331MB     95.2MB
simot@isco:~/cineK8s$ docker run --rm --entrypoint id movie-service:1.0.0
uid=10001(spring) gid=101(spring) groups=101(spring)
```
Les deux images pèsent 331 Mo sur disque (95,2 Mo de contenu compressé), bien moins qu'une
image JDK+Maven (~700 Mo), car l'étape finale ne contient que le JRE Alpine et le jar.
Le conteneur tourne avec l'utilisateur `spring` (UID 10001), pas en root.
Le Dockerfile de ticket-service est identique, sauf `EXPOSE 8086` (port réel du service).

**3.2**

```
simot@isco:~/cineK8s$ docker compose ps
NAME               IMAGE                  SERVICE   STATUS                    PORTS
cinek8s-movie-1    movie-service:1.0.0    movie     Up 13 seconds (healthy)   0.0.0.0:8085->8085/tcp
cinek8s-ticket-1   ticket-service:1.0.0   ticket    Up 7 seconds              0.0.0.0:8086->8086/tcp
simot@isco:~/cineK8s$ curl -s localhost:8085/api/movies/whoami
{"environment":"compose","hostname":"76b972426643"}
simot@isco:~/cineK8s$ curl -s -X POST localhost:8086/api/tickets -H 'Content-Type: application/json' -d '{"movieId":1,"seats":2}' | jq
{
  "id": 1,
  "movieId": 1,
  "movieTitle": "Pod Fiction",
  "seats": 2,
  "total": 21.00,
  "createdAt": "2026-10-08T11:06:02.383768343Z"
}
```

**Q3.1**

On copie `pom.xml` avant `src/` pour exploiter le cache des couches Docker. Le téléchargement
des dépendances (`dependency:go-offline`, 89 s au premier build) ne dépend que du pom : tant
que le pom ne change pas, cette couche est réutilisée. Quand on ne modifie qu'une ligne de
Java, seules les couches à partir de `COPY src` sont rejouées (compilation, ~12 s) : on ne
retélécharge pas les dépendances à chaque build.

**Q3.2**

`-XX:MaxRAMPercentage=75` dimensionne le heap en pourcentage de la mémoire allouée au
conteneur (sa limite), alors que `-Xmx512m` fige une valeur. Avec `-Xmx512m`, si la limite
du Pod est 256 Mi la JVM dépasse et le Pod est tué (OOMKilled) ; si elle est de 2 Gi, on
gaspille la mémoire. Avec le pourcentage, la même image s'adapte à la limite définie dans
Kubernetes, sans rebuild, et les 25 % restants couvrent la mémoire hors heap (metaspace,
threads, mémoire native).

**Q3.3**

Kubernetes ne garantit aucun ordre de démarrage. Si les Pods ticket démarrent avant movie,
leur readiness passe DOWN (le bean `movie` échoue) : ils ne sont pas ajoutés aux endpoints
du Service, donc ne reçoivent aucun trafic. Leur liveness reste UP, donc ils ne sont pas
redémarrés. Dès que movie devient disponible, la readiness repasse UP et les Pods entrent
dans le Service automatiquement. La résilience vient des probes, pas d'un ordre de démarrage.
## Partie 4

**4.0**

```
simot@isco:~/cineK8s$ minikube -p wsl image ls | grep -E 'movie|ticket'
docker.io/library/ticket-service:1.0.0
docker.io/library/movie-service:1.0.0
```

**4.1**

Fichiers `k8s/00-namespace.yaml` (Namespace `cinema-exam`) et `k8s/10-config.yaml`
(ConfigMaps `movie-config` avec `MOVIE_ENVIRONMENT: kubernetes` et `ticket-config` avec
`MOVIE_URL: http://movie:8085`, le port 8085 étant celui de movie-service dans mon dépôt).

**4.2**

Fichier `k8s/20-movie.yaml` : Deployment `movie` (2 réplicas, label `app: movie`, image
`movie-service:1.0.0`, `imagePullPolicy: IfNotPresent`, port 8085 nommé `http`, `envFrom`
sur `movie-config`, requests cpu 100m / memory 256Mi, limit memory 512Mi, startupProbe
(2 s, 30 échecs), livenessProbe (10 s), readinessProbe (5 s)) et Service `movie`
(port 8085 vers le port nommé `http`).

**4.3**

Fichier `k8s/30-ticket.yaml` : même structure que movie, avec le nom `ticket`, le label
`app: ticket`, l'image `ticket-service:1.0.0`, `envFrom` sur `ticket-config`, le port 8086
et `timeoutSeconds: 3` sur la readinessProbe, car celle-ci appelle movie en HTTP et le
timeout par défaut d'une probe (1 s) est trop court.
```
simot@isco:~/cineK8s$ kubectl apply -f k8s/
namespace/cinema-exam created
configmap/movie-config created
configmap/ticket-config created
deployment.apps/movie created
service/movie created
deployment.apps/ticket created
service/ticket created
```

**4.4**

```
simot@isco:~/cineK8s$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
movie-568bd7bc65-gqrpk    1/1     Running   0          49s
movie-568bd7bc65-nshcx    1/1     Running   0          49s
ticket-7bc5799688-fkcjw   1/1     Running   0          49s
ticket-7bc5799688-qx2k6   1/1     Running   0          49s
simot@isco:~/cineK8s$ kubectl get endpoints movie ticket
NAME     ENDPOINTS                           AGE
movie    10.244.0.31:8085,10.244.0.32:8085   76s
ticket   10.244.0.33:8086,10.244.0.34:8086   76s
simot@isco:~/cineK8s$ kubectl exec deploy/ticket -- wget -qO- http://movie:8085/api/movies/whoami
{"environment":"kubernetes","hostname":"movie-568bd7bc65-nshcx"}
simot@isco:~/cineK8s$ kubectl exec deploy/ticket -- wget -qO- http://localhost:8086/actuator/health/readiness
{"status":"UP","components":{"movie":{"status":"UP"},"readinessState":{"status":"UP"}}}
simot@isco:~/cineK8s$ kubectl port-forward svc/ticket 8086:8086 &
simot@isco:~/cineK8s$ curl -s -X POST localhost:8086/api/tickets -H 'Content-Type: application/json' -d '{"movieId":2,"seats":2}' | jq
{
  "id": 1,
  "movieId": 2,
  "movieTitle": "Le Seigneur des Pods",
  "seats": 2,
  "total": 24.00,
  "createdAt": "2026-10-08T11:19:13.998553743Z"
}
```
L'appel inter-services par le nom du Service (`http://movie:8085`) fonctionne et la variable
`MOVIE_ENVIRONMENT` de la ConfigMap est bien prise en compte (`"environment":"kubernetes"`).
Les ports sont ceux de mon dépôt (8085 pour movie, 8086 pour ticket), donc `MOVIE_URL`
vaut `http://movie:8085`.

**4.5**

**Q4.1**

`kubectl apply -f k8s/` traite les fichiers du dossier dans l'ordre alphabétique de leurs
noms, et les ressources d'un même fichier dans l'ordre où elles sont écrites. Les préfixes
numériques fixent donc un ordre logique : le Namespace (00) doit exister avant toute
ressource qui l'utilise, et les ConfigMaps (10) doivent exister avant les Deployments (20,
30) qui les référencent dans `envFrom`. Sinon, les Pods échoueraient à démarrer (erreur de
namespace inexistant ou `CreateContainerConfigError`).

**Q4.2**

C'est la `startupProbe` qui est responsable. Tant qu'elle n'a pas réussi, les autres probes
sont suspendues et le Pod n'est pas Ready, d'où `0/1`. Ce n'est pas une anomalie : Spring
Boot met plusieurs secondes à démarrer, et la startupProbe laisse jusqu'à 60 s (30 échecs ×
2 s) sans redémarrer le conteneur ni imposer un `initialDelaySeconds` fixe sur la liveness.
Dans mon cas, les images étaient déjà sur le nœud et la JVM a démarré vite : les Pods
étaient déjà `1/1` à 14 s.

**Q4.3**

Avec `imagePullPolicy: Always`, le kubelet interroge le registre à chaque création de Pod.
Or `movie-service:1.0.0` et `ticket-service:1.0.0` n'existent que dans le nœud Minikube
(chargées avec `minikube image load`), pas sur un registre public. Les Pods resteraient en
`ErrImagePull` puis `ImagePullBackOff` et ne démarreraient jamais. `IfNotPresent` utilise
l'image locale si elle est déjà présente, ce qui est le bon choix ici.
## Partie 5
## Partie 6
## Partie 7
