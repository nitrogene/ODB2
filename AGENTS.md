# Règles du projet

## Périmètre des modifications

- Toutes les modifications et créations de fichiers doivent rester strictement cantonnées au répertoire courant (`D:\Dev\ODB`).
- Ne jamais modifier les répertoires externes ou le profil utilisateur.
- Ne jamais modifier le contenu du répertoire [`.agents/skills/easyeda-api`](file:///D:/Dev/ODB/.agents/skills/easyeda-api).

## Obtention d'informations sur l'état du projet dans EasyEdaPro

- Privilégier les skills easyeda-api, ne pas utiliser directement le serveur http://localhost:49620, sauf pour faire curl http://localhost:49620/health
- Ne jamais utiliser  .\easyeda\ProPrj_ODB2-Scanner.epro2. Ce fichier est exclusivement modifié lors des "Etapes post validation d'une feature"
- Il est possible de demander un export png du schéma dans .\images\Schematic.png, du pcb dans .\images\PCB.png, si c'est plus simple pour obtenir des informations visuelles

## Capitalisation des découvertes API

- Quand une méthode ou un comportement de l'API EasyEDA non documenté dans le skill officiel est découvert (ex. subtilité entre `getState_Name()` et `getState_OtherProperty().Value`), consigner une entrée datée dans `./LEARNINGS.md` (racine du projet) avant de poursuivre.

## Etapes post validation d'une feature

Ces étapes sont séquentielles et bloquantes : si une étape échoue, ne pas exécuter les étapes suivantes et signaler l'échec à l'utilisateur.

0. Voir s'il est nécessaire d'enrichir LEARNINGS.md
1. Utiliser les skills easyeda-api pour exporter le projet dans ./easyeda/, écraser ProPrj_ODB2-Scanner.epro2
2. Utiliser les skills easyeda-api pour exporter un png du schéma à la meilleure résolution dans ./images/Schematic.png
3. Utiliser les skills easyeda-api pour exporter un png du PCB à la meilleure résolution dans ./images/PCB.png
4. Faire un git add ., et proposer un message de commit
5. Une fois le message de commit validé explicitement par l'utilisateur, faire le commit puis le push

**Ne jamais pousser (`git push`) sans confirmation explicite de l'utilisateur sur le message de commit proposé.**
