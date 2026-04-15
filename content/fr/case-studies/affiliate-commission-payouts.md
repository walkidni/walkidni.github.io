---
title: "Système de commissions et de paiements d’affiliation"
date: 2026-04-15
draft: false
tags:
  - "Architecture logicielle"
  - "Flux financiers"
  - "Intégrité des données"
  - "Concurrence"
  - "Laravel"
description: ""
toc:

---

## 1. Contexte du problème

Les systèmes d’affiliation sont faciles à décrire au niveau de l’interface.

Les partenaires reçoivent un code de parrainage, voient les utilisateurs référés, puis finissent par demander un paiement.

Ce n’est pas la partie difficile.

La vraie difficulté est la **correction financière** :

- À quel moment un parrainage devient-il un gain réel ?
- À quel moment ce gain devient-il retirable ?
- Comment éviter les commissions dupliquées en cas de retries ou d’événements de facturation rejoués ?
- Comment éviter les demandes de paiement en double ou trop tôt ?
- Comment déplacer l’argent sans mises à jour partielles de l’état ?

L’objectif n’était pas simplement d’ajouter un dashboard d’affiliation.

Il s’agissait de concevoir un système de commissions et de paiements qui **sépare l’attribution, le gain, l’approbation et le décaissement** afin que la plateforme puisse prendre en charge l’affiliation sans fragiliser la confiance dans le flux financier.

---

## 2. Surface produit en bref

La surface produit comportait deux côtés :

### Côté affilié

- Code de parrainage et flow d’onboarding
- Visibilité sur les utilisateurs référés
- Résumé des gains
- Historique des commissions
- Flow de demande de paiement
- Gestion des méthodes de paiement

### Côté admin

- Résumé affilié et visibilité sur les utilisateurs référés
- Revue des commissions
- Actions d’approbation et de paiement des payouts
- Mise à jour des taux de commission
- Activation / désactivation des affiliés

Cette couche produit comptait, mais elle restait surtout une enveloppe autour du vrai problème :

**modéliser le cycle de vie de l’argent avec des états clairs et des transitions sûres.**

---

## 3. Modèle d’attribution des parrainages

La première décision a été de traiter l’attribution comme un événement d’identité unique.

Lorsqu’un utilisateur référé s’inscrit, la plateforme résout le référent puis stocke cette relation une seule fois. La logique de commission lit ensuite cette attribution persistée plus tard, au lieu d’essayer de recalculer l’identité du référent pendant le traitement des factures.

Cette séparation est importante parce qu’elle :

- Réduit l’ambiguïté d’attribution
- Garde la logique de gains indépendante de la logique d’inscription
- Empêche les calculs de paiement de dépendre de lookups fragiles au moment de l’exécution

Principe de conception :

> Résoudre l’identité du référent une seule fois. Calculer ensuite l’argent à partir de l’attribution stockée.

Cela rend le système plus simple à raisonner et les événements financiers aval plus déterministes.

---

## 4. Modèle de création des commissions

Les commissions sont créées à partir de l’activité de facturation payée.

Une décision d’ingénierie importante a été de différer la création des commissions jusqu’à ce que le travail de facturation associé soit correctement validé en base. En pratique, cela signifie que les effets de bord liés aux commissions ne se produisent qu’une fois l’état de la facture durable.

Cela protège le système contre une classe classique de bugs :

- l’état de la facture est rollback
- la commission a déjà été créée
- les gains divergent de la réalité de facturation

Le calcul des commissions a lui aussi été modélisé explicitement :

- L’argent est représenté en centimes entiers
- Les taux sont représentés en basis points
- Les lignes de facture éligibles sont filtrées selon la base de commission

Le système supporte deux modèles de gains :

- `SUBSCRIPTION`
- `TRANSACTION_FEE`

Cette séparation est importante car elle rend la politique de revenus explicite dans le code. Au lieu d’un vague “taux de commission”, le système peut décider quels composants de la facture participent réellement au calcul des gains.

Cela donne au moteur de commissions un contrat clair :

- lire les événements de facturation
- sélectionner les revenus éligibles
- calculer la commission de façon déterministe
- persister un enregistrement de gain par facture

---

## 5. Couche d’approbation et d’éligibilité

La décision suivante a été de **ne pas** rendre tous les gains immédiatement retirables.

Le système introduit une couche d’approbation distincte entre la création de la commission et sa disponibilité pour payout.

Comportementalement, le modèle est le suivant :

- les commissions peuvent être créées
- les commissions peuvent rester en attente
- les commissions approuvées deviennent éligibles au payout
- seule la valeur approuvée contribue au solde disponible pour payout

C’est important parce que “gagné” et “retirable” ne veulent pas dire la même chose.

Pour les commissions de type abonnement, le système applique une période d’attente avant que les gains ne soient approuvés. Un chemin d’approbation planifié promeut les commissions pending éligibles après le délai configuré.

Cela crée un modèle financier plus sûr :

- les événements de gain bruts sont enregistrés tôt
- l’approbation est volontairement différée
- la disponibilité au payout est dérivée uniquement des gains approuvés

Le résultat est une frontière de settlement beaucoup plus propre.

Au lieu d’un nombre de type “wallet” qui dérive indépendamment, la plateforme peut expliquer le montant disponible au payout comme :

> commissions approuvées moins historique déjà payé

C’est plus simple, plus défendable et plus facile à expliquer.

---

## 6. Idempotence et prévention des doublons

Les systèmes pilotés par les factures sont naturellement exposés aux retries, aux mises à jour répétées et aux événements rejoués.

Pour cette raison, la prévention des doublons devait faire partie de la conception, et non être ajoutée après coup.

Le garde-fou le plus fort est la frontière d’idempotence au niveau de la facture :

- le service vérifie si une facture a déjà produit une commission
- la base de données impose aussi l’unicité au niveau de la facture

Cette combinaison est importante.

Les vérifications applicatives sont utiles, mais les garanties au niveau du schéma sont ce qui protège réellement le système lorsque la concurrence ou les comportements de retry deviennent désordonnés.

Cela transforme la création de commissions en contrat plus fiable :

- une facture payée
- une commission
- pas de duplication silencieuse en cas de retraitement

Pour une fonctionnalité liée à l’argent, c’est l’une des décisions d’ingénierie les plus utiles de tout le système.

---

## 7. Sécurité des demandes de payout

Les demandes de payout ont été conçues comme un chemin transactionnel, pas comme un simple insert.

Lorsqu’un affilié demande un payout, le système :

- verrouille la ligne de l’affilié
- calcule le solde disponible à partir des gains approuvés
- vérifie le montant minimum de payout
- rejette les demandes supérieures au solde disponible
- vérifie qu’il n’existe aucun payout pending
- crée la demande de payout seulement si tous les contrôles passent

C’est important parce que les demandes de payout sont précisément l’endroit où les bugs de concurrence deviennent coûteux.

Sans verrouillage, deux demandes soumises presque en même temps peuvent voir le même solde et créer des demandes de cashout en double.

Les protections au niveau du service sont renforcées par un invariant au niveau du schéma :

- un seul payout pending est autorisé par affilié

C’est un choix de conception solide parce qu’il combine :

- l’application des règles métier dans le service
- la protection contre les races dans la base de données

Le système ne se contente pas d’espérer que les demandes de payout dupliquées seront rares. Il les rend structurellement plus difficiles à créer.

---

## 8. Gestion des méthodes de paiement et des données sensibles

La gestion des méthodes de paiement est facile à sous-concevoir.

Si les détails de paiement sont traités comme de simples données de profil, l’historique des demandes de payout devient difficile à fiabiliser et les informations financières sensibles peuvent fuiter trop largement.

L’implémentation a évité cela en séparant :

- les méthodes de paiement courantes
- les demandes de payout historiques

Chaque demande de payout stocke un **snapshot** de la méthode de paiement sélectionnée au moment de la demande.

Cela signifie que des modifications ultérieures des coordonnées de paiement ne réécrivent pas l’historique.

C’est une décision discrète mais importante.

Elle améliore :

- la précision historique
- la confiance des opérateurs pendant la revue des payouts
- la résilience face aux modifications de profil après création d’une demande

Les données sensibles de paiement sont également gérées avec précaution :

- les données des méthodes de paiement sont stockées chiffrées
- les vues côté affilié masquent les informations bancaires
- le système maintient exactement une méthode de paiement par défaut par affilié

Ensemble, ces décisions améliorent à la fois la confidentialité et la clarté opérationnelle.

---

## 9. Flux final de décaissement

Le garde-fou technique le plus fort du système se trouve dans l’étape finale de paiement.

Lorsqu’un payout approuvé est décaissé, la plateforme effectue la mise à jour de l’état financier de manière atomique :

- la demande de payout est verrouillée
- l’agrégat affilié est verrouillé
- le total historique déjà payé est incrémenté
- le statut du payout passe à `PAID`
- la transition d’état se produit dans une seule transaction

C’est le cœur du design qui préserve la confiance.

Si le process ne met à jour qu’un seul côté de cet état, le système peut dériver vers des situations impossibles :

- payout marqué payé mais agrégat non mis à jour
- agrégat mis à jour mais payout toujours visible comme impayé
- tentatives concurrentes de paiement qui se chevauchent

En traitant le paiement comme une seule unité de travail atomique, le système réduit le risque de mises à jour partielles et de décaissements dupliqués.

C’est la décision d’ingénierie centrale de l’étude de cas.

---

## 10. Contrôle privilégié et flux opérationnel

Le déplacement d’argent nécessite malgré tout une surface de contrôle opérationnelle.

La plateforme inclut un chemin admin privilégié pour :

- revoir les payouts affiliés
- approuver ou rejeter les demandes de payout
- payer les payouts approuvés
- rejeter les commissions pending
- mettre à jour les taux de commission

Cette couche est intentionnellement séparée de la surface self-service des affiliés.

Les actions admin sont protégées par un chemin de contrôle signé, et les demandes de payout génèrent des notifications opérationnelles afin que le cycle de vie du payout soit visible au-delà du simple dashboard affilié.

Cela crée une caractéristique produit importante :

- les affiliés peuvent demander
- les admins peuvent revoir
- le paiement reste une action privilégiée et délibérée

Cette séparation fait souvent la différence entre une fonctionnalité d’affiliation utile et un système de payout digne de confiance.

---

## 11. Décisions d’ingénierie et compromis

Plusieurs choix de conception ont façonné le système :

**Pourquoi utiliser des centimes entiers et des basis points plutôt que des floats pour l’argent ?**  
Pour réduire les erreurs de précision et rendre les calculs financiers déterministes.

**Pourquoi créer les commissions à partir des événements de facturation ?**  
Parce que les événements de billing sont le déclencheur le plus défendable pour reconnaître un gain.

**Pourquoi séparer gain, approbation et décaissement ?**  
Parce que tous les gains ne doivent pas devenir immédiatement retirables.

**Pourquoi utiliser à la fois des vérifications service et des contraintes de schéma ?**  
Parce qu’un système financier doit être protégé à la fois contre les erreurs de validation ordinaires et contre les conditions de concurrence.

**Pourquoi snapshotter les méthodes de paiement ?**  
Parce que l’historique des demandes de payout ne doit pas changer lorsqu’un utilisateur modifie plus tard ses coordonnées bancaires.

**Pourquoi garder le paiement comme étape admin explicite ?**  
Parce que le décaissement est la transition d’état la plus sensible du flow.

Ces choix ajoutent un peu de complexité, mais ils produisent un système beaucoup plus fiable qu’une simple implémentation plate de type “solde de commission”.

---

## 12. Fiabilité et pistes d’amélioration

Le système possède déjà plusieurs propriétés orientées fiabilité :

- séparation des états entre gains et payouts
- création transactionnelle des demandes de payout
- prévention des doublons soutenue par le schéma
- approbation différée avant retrait
- gestion chiffrée des données de paiement
- mises à jour atomiques au moment du décaissement final

Les pistes d’amélioration porteraient surtout sur :

- une auditabilité plus poussée pour les actions privilégiées sur l’argent
- des règles de transition d’état encore plus strictes autour de la revue des payouts
- une visibilité opérationnelle plus riche autour des notifications de payout et des workflows de reconciliation

Ce sont des extensions utiles, mais le système démontre déjà l’objectif principal :

> l’argent doit traverser des états explicites, avec des frontières d’approbation délibérées et de solides garde-fous contre les doublons.

---

## 13. Résultat

Le résultat n’a pas été seulement une fonctionnalité d’affiliation.

Il s’agit d’un système structuré de commissions et de payouts avec :

- une attribution durable des parrainages
- une création déterministe des commissions
- un timing d’approbation explicite
- des règles sûres d’éligibilité au payout
- des demandes de payout conscientes de la concurrence
- des mises à jour atomiques au moment du décaissement

Cette combinaison a transformé la fonctionnalité en plus qu’un dashboard.

Elle en a fait une capacité plateforme capable de supporter l’affiliation tout en préservant la correction du flux financier.
