# A01 - Boarding Pass Mix-Up

## Vulnerability

Un client connecté pouvait consulter la réservation d'un autre en remplaçant l'identifiant dans la requête.

## Exploitation

Connection en tant qu'Alice -> Ouverture d'un reservation et observation de la requête GET dans l'onglet network -> Copie de la requête au format cURL
Remplacement de l'identifiant en conservant le jeton d'Alice -> Execution de la requête en conservant le jeton d'Alice.

## Root Cause

Le serveur verifiait que le client était connecté mais il ne verifiait pas si il était propriétaire de la reservation.

## Remediation

Rajout d'un controle supplémentaire qui check l'user_id en plus de b.id:

```HERE b.id=$1 AND b.user_id=$2```

## Verification

Les screenshots sont dans le dossier evidence, les teste ont été modifié pour verifier le nouveau filtre.
