# Déploiement du site CEPJS sur AWS EC2 (Docker + GitHub + SSH par clé)

## Structure
- `site/index.html` : ton site (hy.html renommé)
- `site/manifest.webmanifest` : manifeste PWA (le site le référençait mais il manquait)
- `Dockerfile`, `nginx.conf`, `.dockerignore`

## 1. Tester en local
    docker build -t cepjs-site .
    docker run -d --name cepjs -p 8080:80 cepjs-site
    # ouvrir http://localhost:8080

## 2. Pousser sur GitHub
    git init && git add . && git commit -m "Site conteneurisé"
    git branch -M main
    git remote add origin https://github.com/TON_COMPTE/cepjs-site.git
    git push -u origin main

## 3. Créer l'instance EC2 (console AWS)
- AMI : Ubuntu 24.04 LTS, type t2.micro / t3.micro (offre gratuite)
- Key pair : créer `cepjs-key` (RSA ou ED25519, format .pem) et la télécharger
- Security Group :
  - SSH (22) : source = **Mon IP** uniquement
  - HTTP (80) : 0.0.0.0/0
  - HTTPS (443) : 0.0.0.0/0 (plus tard, pour le certificat)

## 4. Se connecter (clé SSH uniquement)
    chmod 400 cepjs-key.pem
    ssh -i cepjs-key.pem ubuntu@IP_PUBLIQUE

## 5. Installer Docker et Git sur le serveur
    sudo apt update && sudo apt install -y docker.io git
    sudo systemctl enable --now docker
    sudo usermod -aG docker ubuntu     # puis se déconnecter/reconnecter

## 6. Déployer
    git clone https://github.com/TON_COMPTE/cepjs-site.git
    cd cepjs-site
    docker build -t cepjs-site .
    docker run -d --name cepjs --restart unless-stopped -p 80:80 cepjs-site
    # ouvrir http://IP_PUBLIQUE

## 7. Mise à jour du site
    cd ~/cepjs-site && git pull
    docker build -t cepjs-site . 
    docker rm -f cepjs
    docker run -d --name cepjs --restart unless-stopped -p 80:80 cepjs-site

## 8. Verrouiller SSH (mot de passe interdit)
    sudo nano /etc/ssh/sshd_config.d/99-hardening.conf
Contenu :

    PasswordAuthentication no
    KbdInteractiveAuthentication no
    PermitRootLogin no
    PubkeyAuthentication yes

Puis :

    sudo sshd -t && sudo systemctl restart ssh
**Garde une 2e session ouverte pendant le test**, pour ne pas t'enfermer dehors.
Vérification depuis ton PC : `ssh -o PubkeyAuthentication=no ubuntu@IP` doit répondre `Permission denied (publickey)`.

## Points d'attention pour CE site
- Il utilise **Firebase** (Auth + Firestore). Dans la console Firebase > Authentication > Settings > *Authorized domains*,
  ajoute le domaine du site. **Une adresse IP n'est pas acceptée** : il faut un nom de domaine (ex. cepjs.com)
  pointé vers l'IP publique (utilise une **Elastic IP** pour qu'elle ne change pas).
- La clé `apiKey` Firebase dans le HTML est publique par conception ; la vraie protection vient des **règles Firestore**. Vérifie-les.
- Pas de HTTPS tant qu'il n'y a pas de domaine. Ensuite : `certbot` (Let's Encrypt) ou un reverse proxy Caddy.
