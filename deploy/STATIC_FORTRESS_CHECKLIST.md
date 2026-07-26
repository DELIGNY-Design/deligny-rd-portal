# STATIC_FORTRESS — checklist deligny-rd.fr

Préparé le 26/07/2026, suite à la décision de Baptiste de débloquer t661 avec
un profil de sécurité **scopé au risque réel d'un portail statique**, pas la
checklist Atlas Fortress complète (Gardien/Watchdog/TOTP/chiffrement DB sont
inutiles ici : pas de base de données, pas d'auth, pas de paiement).

## 1. HTTPS, HSTS, CSP, en-têtes — PRÊT

`nginx-static-fortress.conf` dans ce dossier. À adapter (chemin racine,
certificat) et appliquer sur l'hébergement une fois choisi. Reprend le style
des en-têtes déjà en prod sur atlas-studio.pro (X-Frame-Options,
X-Content-Type-Options, Referrer-Policy, Permissions-Policy) + HSTS et une CSP
resserrée (site 100% statique, aucun JS/CDN externe).

## 2. Contrôle d'intégrité du déploiement — MÉCANISME DÉJÀ EXISTANT

`HUB/lib/hub/factory.rb` a déjà `staged_checksum` (SHA256 des fichiers au
moment du stage → détection de dérive) — même principe applicable ici :
avant chaque déploiement, comparer le SHA256 du contenu local à celui
réellement servi en ligne (`curl | sha256sum` vs le hash du commit déployé).
Pas de nouveau mécanisme à inventer, juste à appliquer à ce déploiement précis
une fois la méthode d'hébergement connue (FTP/SFTP/git push).

## 3. Surveillance disponibilité/défacement — À BRANCHER UNE FOIS LIVE

Pas construit maintenant (rien à surveiller tant que le site n'est pas en
ligne). Une fois live : réutiliser le pattern Vigie déjà prouvé sur Atlas
(`atlas_vigie_agent.py`, cron */15min) — deux contrôles suffisent pour ce
périmètre :
- disponibilité : `curl -sf https://deligny-rd.fr` → code 200 attendu.
- défacement : hash SHA256 du contenu attendu vs contenu réellement servi ;
  écart → alerte (même canal email déjà câblé, `health_monitor.py`/mailer).

## 4. Rollback reproductible — PROCÉDURE (pas d'outillage neuf nécessaire)

Le contenu est déjà versionné (`DELIGNY_R_D_PORTAL` est un repo git). Rollback
= redéployer le commit précédent :
1. `git log --oneline` pour identifier le dernier commit sain.
2. Re-publier ce commit exact vers l'hébergement (même mécanisme que le
   déploiement normal — FTP/SFTP/git push selon ce qui sera choisi).
3. Vérifier via le contrôle d'intégrité (§2) que le hash publié correspond.

## 5. MFA obligatoire — GESTE HUMAIN, PAS AUTOMATISABLE

À activer par Baptiste lui-même sur les 3 comptes concernés avant mise en
ligne : hébergeur (OVH), registrar (OVH aussi si domaine acheté là), GitHub
(déjà utilisé pour les 4 repos publiés — print-checker, gaia-live,
deligny-store, nano-worlds). Je ne peux ni vérifier ni activer ça depuis ici.

## 6. Aucun secret ni donnée client dans le dépôt/logs — DÉJÀ VRAI

`DELIGNY_R_D_PORTAL` est un site 100% statique (HTML/CSS), aucun backend,
aucun log applicatif, aucune donnée utilisateur collectée. Rien à durcir ici
— déjà conforme par construction.

## Ce qui reste bloquant

Uniquement l'accès hébergement/DNS OVH (aucun credential sur cette machine) —
cf t661. Une fois cet accès obtenu : appliquer §1 (config nginx), exécuter le
déploiement, puis §2/§3 pour vérifier et surveiller.
