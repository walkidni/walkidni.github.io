---
title: "Étude de cas : concevoir un moteur de synchronisation multi-tenant à planification dynamique"
date: 2026-02-27
draft: false
tags:
  - "Architecture logicielle"
  - "Event-Driven"
  - "Planification"
  - "FastAPI"
  - "Celery"
---

## 1. Contexte du problème

Les équipes e-commerce s’appuient souvent sur des tableurs comme source de vérité opérationnelle. De leur côté, les plateformes e-commerce exposent les commandes via des API et des webhooks.

Les exports manuels entraînent :

- Dérive des données  
- Charge opérationnelle  
- Absence de sémantique de retry  
- Absence de garanties de cohérence  

L’objectif n’était pas simplement d’exporter des commandes vers Google Sheets.

Il s’agissait de concevoir un **moteur de synchronisation multi-tenant, tolérant aux pannes, sur base de données partagée, avec une forte isolation par tenant** qui :

- Maintient Google Sheets synchronisé avec les données de commandes  
- Prend en charge des configurations d’export définies par l’utilisateur  
- S’exécute de manière fiable via déclenchement manuel, webhooks ou planification dynamique  
- Se met à l’échelle de façon sûre entre tenants  
- Reste idempotent en cas de retries et de relecture (replay) de webhooks  

---

## 2. Architecture de haut niveau

Le système se compose de trois rôles d’exécution :

### Serveur API (FastAPI)

- Authentifie les installations par tenant  
- Gère la configuration des automations d’export  
- Valide les contraintes  
- Crée des sync runs  
- Met en file des jobs de fond  
- Peut optionnellement exécuter une boucle de planification (scheduler)  

### Worker (Celery + Redis)

- Exécute les sync runs longue durée  
- Streame les données depuis les API amont  
- Écrit dans Google Sheets  
- Met à jour l’état des runs  

### MongoDB

- Persistance multi-tenant sur base partagée  
- Partitionnement et isolation des requêtes au périmètre tenant  
- Définitions d’automations  
- Historique d’exécutions  
- Verrouillage du scheduler  
- Clés d’idempotence  

Cette architecture supporte :

- Une séparation claire des responsabilités  
- Une exécution en arrière-plan plus sûre  
- Un état durable  
- Une scalabilité horizontale  

---

## 3. Modèle d’isolation multi-tenant

Toutes les données sont scellées par un `tenant_id`.

Chaque installation :

- Échange un code d’auth  
- Reçoit un bearer token émis localement  
- A toutes ses données partitionnées par tenant  

Les tokens sont :

- Stockés hachés (SHA-256)  
- Révocables  
- Contrôlés via TTL  

Principe de conception :

> Base partagée, isolation stricte au niveau tenant. Pas de requêtes cross-tenant intentionnelles.

Cela permet une mise à l’échelle horizontale sûre et réduit le risque de fuite de données due à des erreurs au niveau applicatif.

---

## 4. Liaison Google OAuth

Chaque tenant peut lier un ou plusieurs comptes Google.

Mesures de sécurité :

- Nonce d’état OAuth stocké côté serveur  
- Consommation de l’état en une seule fois  
- Enregistrements d’état expirants  
- Comptes liés au périmètre tenant  
- Exactement un compte par défaut par tenant (via index unique partiel)  

Cela aide à assurer :

- Un flux OAuth sûr  
- Pas d’attaques par rejeu (replay)  
- De la flexibilité multi-comptes  

---

## 5. L’automation comme entité de premier ordre

L’abstraction centrale est l’**Automation**.

Une automation définit :

- Quoi exporter  
- Des filtres de périmètre optionnels  
- Le tableur et la feuille de destination  
- Un schéma de colonnes dynamique  
- Une configuration de planification optionnelle (cron)  

Contraintes appliquées :

- Deux automations ne peuvent pas cibler le même onglet de tableur  
- Le cron doit respecter un intervalle minimum d’exécution  
- Seules les automations actives peuvent s’exécuter  

Les exports deviennent une configuration durable et déclarative — plutôt que des jobs ad hoc.

---

## 6. Conception de la planification dynamique (pourquoi pas Celery Beat ?)

La planification est **par automation et définie par l’utilisateur**.

Chaque automation a :

- Sa propre expression cron  
- Un état activé/désactivé  
- Un timestamp de prochaine exécution  
- Un cycle de vie de verrouillage  

Les schedules sont dynamiques et modifiables à chaud.

Celery Beat est conçu pour des tâches périodiques définies centralement et enregistrées au démarrage. Ce n’est pas une solution naturelle pour :

- Des milliers de schedules configurables par l’utilisateur  
- Des mutations de schedule à l’exécution  
- Des jobs périodiques au périmètre tenant  

À la place, le système utilise un **scheduler piloté par les données** :

- Les automations éligibles sont sélectionnées via `next_run_at <= now()`  
- Une mise à jour atomique définit `scheduler_locked_until` pour revendiquer l’exécution  
- Le scheduler met en file un run et calcule la prochaine exécution  
- Le prochain timestamp d’exécution est persisté  

Cela permet :

- Une mise à l’échelle horizontale sûre  
- Une réduction du risque de planifications en double  
- Pas de coordination centralisée du scheduler  
- Une planification entièrement dynamique par tenant  

Le scheduler est stateful et piloté par les données, pas statique ni lié à un seul process.

---

## 7. Les sync runs comme unités de travail durables

Chaque exécution est représentée par un document `sync_run`.

Chaque run :

- A un cycle de vie (pending → running → completed/failed)  
- Stocke des métadonnées  
- Stocke des résumés d’erreurs  
- Est lié à une automation  
- Peut être retenté avec des garde-fous d’idempotence  

Déclencheurs :

- Manuel  
- Via webhook  
- Via scheduler périodique  

La persistance des runs apporte :

- Observabilité  
- Debuggabilité  
- Auditabilité  
- Sécurité en cas de retry  

---

## 8. Ingestion des webhooks & idempotence

Les webhooks sont non fiables et, par nature, livrés au moins une fois (at-least-once).

Le système applique :

- Vérification de signature HMAC-SHA256  
- Résolution du tenant via les headers  
- Clé d’idempotence d’événement stockée dans Mongo  
- Fenêtre TTL pour la déduplication  

Ceci vise à fournir :

- Protection contre le rejeu (replay)  
- Une réduction du risque de sync runs dupliqués  
- De la robustesse lors de tempêtes de retry  

Le système est explicitement conçu pour de l’at-least-once delivery.

---

## 9. Pipeline d’export en streaming

Le principal défi technique était d’éviter une explosion mémoire lors d’exports volumineux.

Approche naïve :

- Matérialiser l’ensemble du dataset  
- Transformer  
- Écrire  

Cela se met à l’échelle linéairement avec la taille de l’export.

Conception finale :

- Streamer les lignes de façon incrémentale  
- Transformer ligne par ligne  
- Regrouper les lignes consécutives par identifiant de commande unique  
- Flusher des batches bornés vers Sheets  

Caps explicites :

- 300 groupes de commandes  
- 1200 lignes  
- 1,2 Mo de payload  

Effets :

- La mémoire reste bornée  
- La taille de payload vers les API externes est contrôlée  
- Les exports volumineux sont gérés plus sereinement  

Le système se comporte de manière prédictible à l’échelle.

---

## 10. Écritures idempotentes dans Sheets

Les écritures Google Sheets utilisent une sémantique d’upsert, indexée par un identifiant de commande unique.

Stratégie d’écriture :

- S’assurer que la feuille existe  
- Réécrire l’en-tête si le schéma change  
- Lire les identités de lignes existantes  
- Mettre à jour les lignes existantes  
- Ajouter les nouvelles lignes  

Cela aide à assurer :

- Des retries sûrs  
- Pas de duplication logique  
- Une cohérence au redémarrage des workers (en supposant des clés d’idempotence stables)  

Objectif de conception :

* L’exécution at-least-once ne doit pas corrompre l’état en aval.

---

## 11. Stabilité de la frontière async ↔ worker

Les services FastAPI sont async.  
Les tâches Celery sont des points d’entrée synchrones.

Pour éviter l’instabilité de la boucle event dans des workers prefork :

- Chaque tâche crée un nouveau client de base de données async  
- `asyncio.run()` est appelé dans la tâche  
- Pas de réutilisation de boucle event entre exécutions  

Cela évite :

- `RuntimeError: Event loop is closed`  
- La contamination de boucle entre tâches  

Un détail de stabilité de production subtil mais important.

---

## 12. Compromis & décisions d’ingénierie

**Pourquoi le streaming plutôt que la matérialisation complète ?**  
Sécurité mémoire et scalabilité.

**Pourquoi un scheduler adossé à la base plutôt que Celery Beat ?**  
Planification dynamique par tenant et sécurité de “claim” atomique.

**Pourquoi l’upsert plutôt qu’un append-only ?**  
Sécurité en cas de retry et idempotence webhooks.

**Pourquoi une auth tenant-scopée ?**  
Des frontières d’isolation plus strictes et un contrôle d’accès plus simple.

**Pourquoi persister les sync runs ?**  
Débogage opérationnel et piste d’audit.

---

## 13. Fiabilité & observabilité

Chaque run enregistre :

- Timestamps de début et de fin  
- Statut  
- Erreurs  
- Métadonnées  

Cela permet :

- Inspection opérationnelle  
- Debug  
- Ré-exécution manuelle  
- Transparence des échecs  
