------------------------------------------------------------------------------------------------------
ATELIER PRA/PCA
------------------------------------------------------------------------------------------------------
L’idée en 30 secondes : Cet atelier met en œuvre un **mini-PRA** sur **Kubernetes** en déployant une **application Flask** avec une **base SQLite** stockée sur un **volume persistant (PVC pra-data)** et des **sauvegardes automatiques réalisées chaque minute vers un second volume (PVC pra-backup)** via un **CronJob**. L’**image applicative est construite avec Packer** et le **déploiement orchestré avec Ansible**, tandis que Kubernetes assure la gestion des pods et de la disponibilité applicative. Nous observerons la différence entre **disponibilité** (recréation automatique des pods sans perte de données) et **reprise après sinistre** (perte volontaire du volume de données puis restauration depuis les backups), nous mesurerons concrètement les RTO et RPO, et comprendrons les limites d’un PRA local non répliqué. Cet atelier illustre de manière pratique les principes de continuité et de reprise d’activité, ainsi que le rôle respectif des conteneurs, du stockage persistant et des mécanismes de sauvegarde.
  
**Architecture cible :** Ci-dessous, voici l'architecture cible souhaitée.   
  
![Screenshot Actions](Architecture_cible.png)  
  
-------------------------------------------------------------------------------------------------------
Séquence 1 : Codespace de Github
-------------------------------------------------------------------------------------------------------
Objectif : Création d'un Codespace Github  
Difficulté : Très facile (~5 minutes)
-------------------------------------------------------------------------------------------------------
**Faites un Fork de ce projet**. Si besoin, voici une vidéo d'accompagnement pour vous aider à "Forker" un Repository Github : [Forker ce projet](https://youtu.be/p33-7XQ29zQ) 
  
Ensuite depuis l'onglet **[CODE]** de votre nouveau Repository, **ouvrez un Codespace Github**.
  
---------------------------------------------------
Séquence 2 : Création du votre environnement de travail
---------------------------------------------------
Objectif : Créer votre environnement de travail  
Difficulté : Simple (~10 minutes)
---------------------------------------------------
Vous allez dans cette séquence mettre en place un cluster Kubernetes K3d contenant un master et 2 workers, installer les logiciels Packer et Ansible. Depuis le terminal de votre Codespace copier/coller les codes ci-dessous étape par étape :  

**Création du cluster K3d**  
```
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```
```
k3d cluster create pra \
  --servers 1 \
  --agents 2
```
**vérification de la création de votre cluster Kubernetes**  
```
kubectl get nodes
```
**Installation du logiciel Packer (création d'images Docker)**  
```
PACKER_VERSION=1.11.2
curl -fsSL -o /tmp/packer.zip \
  "https://releases.hashicorp.com/packer/${PACKER_VERSION}/packer_${PACKER_VERSION}_linux_amd64.zip"
sudo unzip -o /tmp/packer.zip -d /usr/local/bin
rm -f /tmp/packer.zip
```
**Installation du logiciel Ansible**  
```
python3 -m pip install --user ansible kubernetes PyYAML jinja2
export PATH="$HOME/.local/bin:$PATH"
ansible-galaxy collection install kubernetes.core
```
  
---------------------------------------------------
Séquence 3 : Déploiement de l'infrastructure
---------------------------------------------------
Objectif : Déployer l'infrastructure sur le cluster Kubernetes
Difficulté : Facile (~15 minutes)
---------------------------------------------------  
Nous allons à présent déployer notre infrastructure sur Kubernetes. C'est à dire, créér l'image Docker de notre application Flask avec Packer, déposer l'image dans le cluster Kubernetes et enfin déployer l'infratructure avec Ansible (Création du pod, création des PVC et les scripts des sauvegardes aututomatiques).  

**Création de l'image Docker avec Packer**  
```
packer init .
packer build -var "image_tag=1.0" .
docker images | head
```
  
**Import de l'image Docker dans le cluster Kubernetes**  
```
k3d image import pra/flask-sqlite:1.0 -c pra
```
  
**Déploiment de l'infrastructure dans Kubernetes**  
```
ansible-playbook ansible/playbook.yml
```
  
**Forward du port 8080 qui est le port d'exposition de votre application Flask**  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
  
---------------------------------------------------  
**Réccupération de l'URL de votre application Flask**. Votre application Flask est déployée sur le cluster K3d. Pour obtenir votre URL cliquez sur l'onglet **[PORTS]** dans votre Codespace (à coté de Terminal) et rendez public votre port 8080 (Visibilité du port). Ouvrez l'URL dans votre navigateur et c'est terminé.  

**Les routes** à votre disposition sont les suivantes :  
1. https://...**/** affichera dans votre navigateur "Bonjour tout le monde !".
2. https://...**/health** pour voir l'état de santé de votre application.
3. https://...**/add?message=test** pour ajouter un message dans votre base de données SQLite.
4. https://...**/count** pour afficher le nombre de messages stockés dans votre base de données SQLite.
5. https://...**/consultation** pour afficher les messages stockés dans votre base de données.
  
---------------------------------------------------  
### Processus de sauvegarde de la BDD SQLite

Grâce à une tâche CRON déployée par Ansible sur le cluster Kubernetes (un CronJob), toutes les minutes une sauvegarde de la BDD SQLite est faite depuis le PVC pra-data vers le PCV pra-backup dans Kubernetes.  

Pour visualiser les sauvegardes périodiques déposées dans le PVC pra-backup, coller les commandes suivantes dans votre terminal Codespace :  

```
kubectl -n pra run debug-backup \
  --rm -it \
  --image=alpine \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "debug",
      "image": "alpine",
      "command": ["sh"],
      "stdin": true,
      "tty": true,
      "volumeMounts": [{
        "name": "backup",
        "mountPath": "/backup"
      }]
    }],
    "volumes": [{
      "name": "backup",
      "persistentVolumeClaim": {
        "claimName": "pra-backup"
      }
    }]
  }
}'
```
```
ls -lh /backup
```
**Pour sortir du cluster et revenir dans le terminal**
```
exit
```

---------------------------------------------------
Séquence 4 : 💥 Scénarios de crash possibles  
Difficulté : Facile (~30 minutes)
---------------------------------------------------
### 🎬 **Scénario 1 : PCA — Crash du pod**  
Nous allons dans ce scénario **détruire notre Pod Kubernetes**. Ceci simulera par exemple la supression d'un pod accidentellement, ou un pod qui crash, ou un pod redémarré, etc..

**Destruction du pod :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario1.png)  

Nous perdons donc ici notre application mais pas notre base de données puisque celle-ci est déposée dans le PVC pra-data hors du pod.  

Copier/coller le code suivant dans votre terminal Codespace pour détruire votre pod :
```
kubectl -n pra get pods
```
Notez le nom de votre pod qui est différent pour tout le monde.  
Supprimez votre pod (pensez à remplacer <nom-du-pod-flask> par le nom de votre pod).  
Exemple : kubectl -n pra delete pod flask-7c4fd76955-abcde  
```
kubectl -n pra delete pod <nom-du-pod-flask>
```
**Vérification de la suppression de votre pod**
```
kubectl -n pra get pods
```
👉 **Le pod a été reconstruit sous un autre identifiant**.  
Forward du port 8080 du nouveau service  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
Observez le résultat en ligne  
https://...**/consultation** -> Vous n'avez perdu aucun message.
  
👉 Kubernetes gère tout seul : Aucun impact sur les données ou sur votre service (PVC conserve la DB et le pod est reconstruit automatiquement) -> **C'est du PCA**. Tout est automatique et il n'y a aucune rupture de service.
  
---------------------------------------------------
### 🎬 **Scénario 2 : PRA - Perte du PVC pra-data** 
Nous allons dans ce scénario **détruire notre PVC pra-data**. C'est à dire nous allons suprimer la base de données en production. Ceci simulera par exemple la corruption de la BDD SQLite, le disque du node perdu, une erreur humaine, etc. 💥 Impact : IL s'agit ici d'un impact important puisque **la BDD est perdue**.  

**Destruction du PVC pra-data :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario2.png)  

🔥 **PHASE 1 — Simuler le sinistre (perte de la BDD de production)**  
Copier/coller le code suivant dans votre terminal Codespace pour détruire votre base de données :
```
kubectl -n pra scale deployment flask --replicas=0
```
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":true}}'
```
```
kubectl -n pra delete job --all
```
```
kubectl -n pra delete pvc pra-data
```
👉 Vous pouvez vérifier votre application en ligne, la base de données est détruite et la service n'est plus accéssible.  

✅ **PHASE 2 — Procédure de restauration**  
Recréer l’infrastructure avec un PVC pra-data vide.  
```
kubectl apply -f k8s/
```
Vérification de votre application en ligne.  
Forward du port 8080 du service pour tester l'application en ligne.  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
https://...**/count** -> =0.  
https://...**/consultation** Vous avez perdu tous vos messages.  

Retaurez votre BDD depuis le PVC Backup.  
```
kubectl apply -f pra/50-job-restore.yaml
```
👉 Vous pouvez vérifier votre application en ligne, **votre base de données a été restaureé** et tous vos messages sont bien présents.  

Relance des CRON de sauvgardes.  
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":false}}'
```
👉 Nous n'avons pas perdu de données mais Kubernetes ne gère pas la restauration tout seul. Nous avons du protéger nos données via des sauvegardes régulières (du PVC pra-data vers le PVC pra-backup). -> **C'est du PRA**. Il s'agit d'une stratégie de sauvegarde avec une procédure de restauration.  

---------------------------------------------------
Séquence 5 : Exercices  
Difficulté : Moyenne (~45 minutes)
---------------------------------------------------
**Complétez et documentez ce fichier README.md** pour répondre aux questions des exercices.  
Faites preuve de pédagogie et soyez clair dans vos explications et procedures de travail.  

Exercice 1 : Quels sont les composants dont la perte entraîne une perte de données ?

Dans l'architecture actuelle de cet atelier, la perte simultanée des deux composants suivants entraîne une perte de données définitive :

Le PVC pra-data : Il s'agit du volume de production qui contient le fichier de la base de données SQLite active. Sa perte seule constitue un sinistre, mais pas une perte définitive tant que le backup existe.
Le PVC pra-backup : Il contient l'historique des sauvegardes générées chaque minute par le CronJob. 

Scénarios de perte définitive :
1. Destruction des deux PVC (pra-data ET pra-backup) : Si le stockage physique sous-jacent (le disque du nœud Kubernetes ou le cluster K3d lui-même) est supprimé ou corrompu, la production et les sauvegardes disparaissent en même temps.
2. Panne du nœud hébergeant le cluster K3d : Comme les deux PVC partagent le même stockage local (LocalPath Provisioner par défaut sur K3d), la perte de la machine hôte détruit instantanément les données actives et les sauvegardes.


Exercice 2 : Expliquez-nous pourquoi nous n'avons pas perdu les données lors de la suppression du PVC pra-data.

La préservation des données repose sur le principe de la séparation des cycles de vie du stockage et sur l'indépendance des volumes.

Voici le déroulement précis :
1. Sauvegarde en amont : Avant la simulation du sinistre, le CronJob Kubernetes s'est exécuté avec succès et a couplé le fichier SQLite depuis le dossier de production (monté sur pra-data) vers le dossier de sauvegarde (monté sur pra-backup).
2. Isolation des volumes : Le PVC pra-backup est un objet Kubernetes totalement indépendant du PVC pra-data. Supprimer le premier n'affecte en rien l'intégrité du second.
3. Restauration explicite : Lors de la Phase 2, l'exécution du Job "pra/50-job-restore.yaml" a monté simultanément le volume pra-backup (contenant la copie saine) et le nouveau volume pra-data (qui était vide). Le conteneur du Job a ensuite transféré le fichier SQLite restauré vers le volume de production avant que l'application Flask ne redémarre.


Exercice 3 : Quels sont les RTO et RPO de cette solution ?

1. RPO (Recovery Point Objective - Perte de données maximale admissible)
* Valeur : 1 minute.
* Justification : Le CronJob effectue une copie de la base de données toutes les minutes. Dans le pire des scénarios (un crash survenant juste avant l'exécution de la minute suivante), les données enregistrées durant les 59 dernières secondes sont perdues.

2. RTO (Recovery Time Objective - Durée maximale d'interruption admissible)
* Valeur : Quelques minutes (variable selon la réactivité humaine).
* Justification : La reconstruction n'est pas automatisée (il s'agit d'un PRA, pas d'un PCA). Le RTO correspond au temps nécessaire pour détecter l'anomalie, exécuter manuellement les commandes kubectl pour recréer l'infrastructure, lancer le Job de restauration, et attendre que le conteneur applicatif soit opérationnel.


Exercice 4 : Pourquoi cette solution (cet atelier) ne peut pas être utilisée dans un vrai environnement de production ? Que manque-t-il ?

Bien que l'exercice valide les concepts théoriques, il présente des failles critiques pour de la production :

* Point de défaillance unique (SPOF) local : Les volumes pra-data et pra-backup sont stockés sur le même cluster, souvent sur la même machine physique. Si le serveur physique crash ou brûle, le PRA est totalement inopérant.
* Verrous SQLite : SQLite est un fichier unique. Effectuer une copie directe du fichier via un script (cp) pendant que l'application Flask écrit dedans peut corrompre la sauvegarde (problème de concurrence).
* Intervention humaine requise : La procédure de restauration est entièrement manuelle. En production, un RTO élevé dû à des actions manuelles complexes représente une perte financière ou de service critique.
* Absence de cycle de vie des sauvegardes : Le script actuel écrase ou accumule les données sans gestion de rétention (pas de versioning, pas de rotation des sauvegardes, pas d'externalisation).


Exercice 5 : Proposez une architecture plus robuste.

Pour transformer ce prototype en une architecture hautement disponible et résiliente aux sinistres majeurs, voici les améliorations à implémenter :

1. Stockage et Sauvegarde :
* Externalisation des Backups : Remplacer le PVC local pra-backup par un stockage objet distant, managé et géo-redondant (ex: AWS S3 ou Backblaze B2) avec chiffrement et immuabilité (Object Locking) pour contrer les ransomwares.
* Remplacement de SQLite : Utiliser un vrai système de gestion de base de données relationnelle (PostgreSQL ou MySQL) configuré en haute disponibilité (Cluster avec réplication Master/Slave).

2. Infrastructure et Orchestration :
* Multi-AZ (Zones de Disponibilité) : Déployer le cluster Kubernetes sur plusieurs zones physiques. Le stockage de production doit utiliser des CSI (Container Storage Interface) répliqués de manière synchrone entre les zones (ex: Ceph/Rook ou Longhorn).
Outil de PRA Kubernetes dédié : Intégrer un outil comme Velero. Velero automatise la sauvegarde complète des volumes persistants et des définitions d'objets Kubernetes directement vers un stockage externe, permettant une restauration complète de l'ensemble du cluster de manière automatisée ou via GitOps.



---------------------------------------------------
Séquence 6 : Ateliers  
Difficulté : Moyenne (~2 heures)
---------------------------------------------------
### **Atelier 1 : Ajoutez une fonctionnalité à votre application**  
**Ajouter une route GET /status** dans votre application qui affiche en JSON :
* count : nombre d’événements en base
* last_backup_file : nom du dernier backup présent dans /backup
* backup_age_seconds : âge du dernier backup

*..**Déposez ici une copie d'écran** de votre réussite..*

---------------------------------------------------
### **Atelier 2 : Choisir notre point de restauration**  
Aujourd’hui nous restaurobs “le dernier backup”. Nous souhaitons **ajouter la capacité de choisir un point de restauration**.

Étape 1 : Lister les sauvegardes disponibles
Avant toute manipulation, il faut identifier le nom précis du fichier de sauvegarde que l'on souhaite restaurer.

1. Lancer un pod de diagnostic temporaire pour inspecter le contenu du PVC "pra-backup" :
kubectl -n pra run debug-backup --rm -it --image=alpine --overrides='{"spec": {"containers": [{"name": "debug", "image": "alpine", "command": ["sh"], "stdin": true, "tty": true, "volumeMounts": [{"name": "backup", "mountPath": "/backup"}]}], "volumes": [{"name": "backup", "persistentVolumeClaim": {"claimName": "pra-backup"}}]}}'

2. Une fois dans le conteneur, lister les fichiers avec leur horodatage :
ls -la /backup

3. Noter le nom exact du fichier cible (Exemple : "production_20261105_143200.db" si le CronJob a été configuré pour dater les fichiers, ou repérer le fichier exact via sa date de modification).

4. Quitter le pod de diagnostic :
exit


Étape 2 : Préparer l'environnement (Mise en pause de la production)
Pour éviter les écritures concurrentes et les conflits pendant la restauration, il faut couper l'accès à la base de données active.

1. Arrêter temporairement l'application Flask (réduire le nombre de pods à 0) :
kubectl -n pra scale deployment flask --replicas=0

2. Suspendre le CronJob de sauvegarde pour éviter qu'il ne s'exécute pendant la restauration ou qu'il ne crée un backup corrompu :
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":true}}'


Étape 3 : Configurer et exécuter le Job de restauration ciblée

Pour choisir le point de restauration, nous allons modifier le Job de restauration afin de lui passer le nom du fichier ciblé en variable d'environnement ou en argument de commande.

1. Modifier le fichier "pra/50-job-restore.yaml" ou créer une version modifiée (ex: "pra/50-job-restore-custom.yaml"). La section "command" du conteneur doit être adaptée pour accepter une variable.

Exemple de structure du Job à appliquer :
apiVersion: batch/v1
kind: Job
metadata:
  name: sqlite-restore-custom
  namespace: pra
spec:
  template:
    spec:
      containers:
      - name: restore
        image: alpine
        command: ["sh", "-c"]
        args:
        - |
          echo "Début de la restauration ciblée..."
          # Vérification de l'existence du fichier demandé
          if [ ! -f "/backup/${TARGET_BACKUP_FILE}" ]; then
            echo "Erreur : Le fichier /backup/${TARGET_BACKUP_FILE} n'existe pas."
            exit 1
          fi
          # Nettoyage de l'ancienne base de production
          rm -f /data/app.db
          # Copie et renommage de la sauvegarde choisie vers l'emplacement de production
          cp /backup/${TARGET_BACKUP_FILE} /data/app.db
          echo "Restauration réussie du fichier ${TARGET_BACKUP_FILE}."
        env:
        - name: TARGET_BACKUP_FILE
          value: "NOM_DU_FICHIER_CHOISI_A_L_ETAPE_1"
        volumeMounts:
        - name: data-vol
          mountPath: /data
        - name: backup-vol
          mountPath: /backup
      restartPolicy: OnFailure
      volumes:
      - name: data-vol
        persistentVolumeClaim:
          claimName: pra-data
      - name: backup-vol
        persistentVolumeClaim:
          claimName: pra-backup

2. Remplacer "NOM_DU_FICHIER_CHOISI_A_L_ETAPE_1" dans le fichier YAML par le nom réel du fichier noté précédemment.

3. Nettoyer les anciens Jobs terminés pour éviter les conflits de nommage :
kubectl -n pra delete job --all

4. Exécuter le Job de restauration personnalisé :
kubectl apply -f pra/50-job-restore-custom.yaml

5. Vérifier la bonne exécution et consulter les logs du Job :
kubectl -n pra get jobs
kubectl -n pra logs job/sqlite-restore-custom


Étape 4 : Relancer les services de production

Une fois que les logs du Job confirment le message "Restauration réussie", l'infrastructure peut être relancée.

1. Redémarrer l'application Flask (remonter le replica à 1) :
kubectl -n pra scale deployment flask --replicas=1

2. Réactiver le CronJob de sauvegarde automatique :
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":false}}'

3. Vérifier le bon fonctionnement de l'application en consultant la route de consultation pour s'assurer que les données correspondent bien au point de restauration choisi.
  
---------------------------------------------------
Evaluation
---------------------------------------------------
Cet atelier PRA PCA, **noté sur 20 points**, est évalué sur la base du barème suivant :  
- Série d'exerices (5 points)
- Atelier N°1 - Ajout d'un fonctionnalité (4 points)
- Atelier N°2 - Choisir son point de restauration (4 points)
- Qualité du Readme (lisibilité, erreur, ...) (3 points)
- Processus travail (quantité de commits, cohérence globale, interventions externes, ...) (4 points) 

