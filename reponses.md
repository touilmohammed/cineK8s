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
## Partie 4
## Partie 5
## Partie 6
## Partie 7
