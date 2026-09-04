# Règles du projet

- **Périmètre des modifications** : Toutes les modifications et créations de fichiers doivent rester strictement cantonnées au répertoire courant (`D:\Dev\ODB`). Ne jamais modifier les répertoires externes ou le profil utilisateur.
- **Intégrité des sous-modules** : Ne jamais modifier le contenu du répertoire [`.agents/skills/easyeda-api`](file:///D:/Dev/ODB/.agents/skills/easyeda-api).
- **Démarrage automatique du serveur EasyEDA Bridge** : Au début de chaque session ou dès la première requête impliquant le projet ou EasyEDA :
  1. Vérifier si le serveur bridge répond sur `http://localhost:49620/health` (ou plage 49620-49629).
  2. S'il n'est pas actif, exécuter `npm install` (si nécessaire) puis lancer en tâche de fond `npm run server` dans [`.agents/skills/easyeda-api`](file:///D:/Dev/ODB/.agents/skills/easyeda-api).
  3. Veiller à ce que le processus tourne en arrière-plan afin de ne pas bloquer les interactions.
