# Règles du projet

## Périmètre des modifications

- Toutes les modifications et créations de fichiers doivent rester strictement cantonnées au répertoire courant (`D:\Dev\ODB`).
- Ne jamais modifier les répertoires externes ou le profil utilisateur.
- Ne jamais modifier le contenu du répertoire [`.agents/skills/easyeda-api`](file:///D:/Dev/ODB/.agents/skills/easyeda-api).

## Obtention d'informations sur l'état du projet dans EasyEdaPro

- Privilégier les skills easyeda-api, ne pas utiliser directement le serveur http://localhost:49620, sauf pour faire curl http://localhost:49620/health
- Ne jamais utiliser .\easyeda\ProPrj_ODB2-Scanner.epro2. Ce fichier est exclusivement modifié lors des "Étapes post validation d'une feature"
- Il est possible de demander un export png du schéma dans .\images\Schematic.png, du pcb dans .\images\PCB.png, si c'est plus simple pour obtenir des informations visuelles

## État de référence du projet ("baseline propre")

Le projet est considéré dans un état de référence valide quand toutes ces conditions sont réunies :

- Le schéma et le PCB passent le DRC sans erreur
- `./easyeda/ProPrj_ODB2-Scanner.epro2`, `./images/Schematic.png` et `./images/PCB.png` reflètent fidèlement l'état courant du projet EasyEDA (réexportés via les skills easyeda-api)
- `git status` ne montre aucune modification en attente (tout est commité et pushé)

Cette checklist est le socle commun utilisé avant de démarrer une feature et pour valider la fin d'une feature (voir sections ci-dessous).

## Avant de commencer une feature

- Vérifier que le projet est dans son état de référence (voir section ci-dessus).
- Si un point de la checklist n'est pas respecté, le signaler à l'utilisateur et ne pas démarrer la feature tant que ce n'est pas résolu.

## Capitalisation des découvertes API

- Quand une méthode ou un comportement de l'API EasyEDA non documenté dans le skill officiel est découvert (ex. subtilité entre `getState_Name()` et `getState_OtherProperty().Value`), consigner une entrée datée dans `./LEARNINGS.md` (racine du projet) avant de poursuivre.

## Étapes post validation d'une feature

Ces étapes sont séquentielles et bloquantes : si une étape échoue, ne pas exécuter les étapes suivantes et signaler l'échec à l'utilisateur.

- Le PCB doit être valide (check DRC)
- Voir s'il est nécessaire d'enrichir LEARNINGS.md
- Utiliser les skills easyeda-api pour exporter le projet dans ./easyeda/, écraser ProPrj_ODB2-Scanner.epro2
- Utiliser les skills easyeda-api pour exporter un png du schéma à la meilleure résolution dans ./images/Schematic.png
- Utiliser les skills easyeda-api pour exporter un png du PCB à la meilleure résolution dans ./images/PCB.png
- Faire un git add ., et proposer un message de commit
- Une fois le message de commit validé explicitement par l'utilisateur, faire le commit puis le push
- Vérifier que le projet est revenu dans son état de référence (DRC + exports + `git status` vide)

**Ne jamais pousser (`git push`) sans confirmation explicite de l'utilisateur sur le message de commit proposé.**