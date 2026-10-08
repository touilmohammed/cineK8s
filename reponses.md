# Examen CinéK8s — NOM Prénom

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
## Partie 3
## Partie 4
## Partie 5
## Partie 6
## Partie 7
