# Règles du projet

## Périmètre des modifications

- Toutes les modifications et créations de fichiers doivent rester strictement cantonnées au répertoire courant (`D:\Dev\ODB`).
- Ne jamais modifier les répertoires externes ou le profil utilisateur.
- Ne jamais modifier le contenu du répertoire [`.agents/skills/easyeda-api`](file:///D:/Dev/ODB/.agents/skills/easyeda-api).

## Intéractions avec EasyEdaPro

- Privilégier les skills easyeda-api, ne pas utiliser directement le serveur http://localhost:49620, sauf pour faire curl http://localhost:49620/health
- Ne jamais utiliser .\easyeda\ProPrj_ODB2-Scanner.epro2. Ce fichier est exclusivement modifié lors des "Étapes post validation d'une feature"
- Demander à l'utilisateur de réaliser l'export haute résolution du Schéma (dans .\images\Schematic.png) et du PCB (dans .\images\PCB.png) via l'interface EasyEDA Pro (Fichier > Exporter) dès que nécessaire pour garantir une lisibilité optimale.
- Ne pas hésiter à me demander de l'aide en cas de difficultés, pour par exempke déplacer des composants

## Documentation du schéma

- Ne pas hésiter à enrichir le schéma d'explications ou de mini-schémas décrivant le rôle de chaque groupe de composants (alimentation, filtrage, interface CAN, etc.) — mieux vaut trop de documentation visuelle que pas assez.

## État de référence du projet ("baseline propre")

Le projet est considéré dans un état de référence valide quand toutes ces conditions sont réunies :

- Le schéma passe l'ERC sans erreur (fonction `easyeda_erc_run` du skill — contrôle électrique : pins non connectées, conflits de type de pin, etc.)
- Le PCB passe le DRC sans erreur (fonction `easyeda_drc_run` du skill — contrôle physique : clearances, largeurs de piste, vias, silk sur pad, etc.)
  *(Dans l'UI EasyEDA Pro, les deux contrôles peuvent apparaître sous le même libellé "DRC" — bien vérifier qu'on lance les deux fonctions distinctes du skill, pas une seule.)*
- `./easyeda/ProPrj_ODB2-Scanner.epro2` reflète fidèlement l'état courant du projet EasyEDA (réexporté via les skills easyeda-api)
- `./images/Schematic.png` et `./images/PCB.png` reflètent fidèlement l'état courant du projet EasyEDA (exportés manuellement par l'utilisateur en haute résolution via l'interface graphique : Fichier > Exporter)
- `git status` ne montre aucune modification en attente (tout est commité et pushé)
- Il ne doit pas y avoir de symbole latex dans le README.md
- Les net labels visibles sur un schéma, le PCB ou dans ce document doivent être lisibles et porteur d'information (pas de $1NXXX ou autre)


Cette checklist est le socle commun utilisé avant de démarrer une feature et pour valider la fin d'une feature (voir sections ci-dessous).

## Avant de commencer une feature

- Vérifier que le projet est dans son état de référence (voir section ci-dessus).
- Si un point de la checklist n'est pas respecté, le signaler à l'utilisateur et ne pas démarrer la feature tant que ce n'est pas résolu.
- Lire LEARNINGS.md afin de ne pas résoudre un problème déjà résolu

## Capitalisation des découvertes API

- Quand une méthode ou un comportement de l'API EasyEDA non documenté dans le skill officiel est découvert (ex. subtilité entre `getState_Name()` et `getState_OtherProperty().Value`), consigner une entrée datée dans `./LEARNINGS.md` (racine du projet) avant de poursuivre.

## Étapes post validation d'une feature

Ces étapes sont séquentielles et bloquantes : si une étape échoue, ne pas exécuter les étapes suivantes et signaler l'échec à l'utilisateur.

- Voir s'il est nécessaire d'enrichir LEARNINGS.md
- Mettre à jour la section "Prochaine Étape Immédiate" du README.md
- Utiliser les skills easyeda-api pour exporter le projet dans ./easyeda/, écraser ProPrj_ODB2-Scanner.epro2
- Demander à l'utilisateur d'effectuer l'export du Schéma dans ./images/Schematic.png et du PCB dans ./images/PCB.png depuis l'interface EasyEDA Pro (Fichier > Exporter > Image en haute résolution / 300 DPI ou 4096 px de large) afin d'assurer une lisibilité parfaite
- Une fois les exports réalisés et confirmés par l'utilisateur, faire un git add ., et proposer un message de commit
- Une fois le message de commit validé explicitement par l'utilisateur, faire le commit puis le push
- Vérifier que le projet est revenu dans son état de référence (ERC schéma + DRC PCB + exports + `git status` vide)

**Ne jamais pousser (`git push`) sans confirmation explicite de l'utilisateur sur le message de commit proposé.**