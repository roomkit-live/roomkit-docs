# Plan de simplification de la boucle streaming IA

Date : 13 septembre 2026. Statut : terminé et vérifié sur `0ea54725`.

La boucle principale occupe désormais 109 lignes, contre 596 au départ.
Les cinq étapes ont été réalisées dans des commits distincts, avec les correctifs
isolés. Voir le [compte rendu de validation](ai-streaming-validation.md).

Rendre la boucle streaming lisible et facile à faire évoluer, en conservant
ses contrats de livraison, d'exécution des outils et de nettoyage.

**Point de départ**

La méthode `_run_streaming_tool_loop()` de
[`_ai_streaming.py`](../../roomkit/src/roomkit/channels/_ai_streaming.py)
occupait 596 lignes au début de l'intervention, commentaires compris. Elle gérait le contexte du
tour, les fragments du fournisseur, la déduplication, le raisonnement, les outils
externes et locaux, les événements, les budgets, la télémétrie et les sorties.

Les règles communes existent déjà dans
[`_ai_loop_rules.py`](../../roomkit/src/roomkit/channels/_ai_loop_rules.py),
les nouvelles tentatives et le fournisseur de secours dans `_ai_resilience.py`,
et le regroupement des événements dans `_ai_coalescers.py`. Le découpage doit
s'appuyer sur ces responsabilités existantes.

La base incluait les deux correctifs d'hygiène alors locaux : arrêt forcé réellement
terminal en streaming et nettoyage des appels parallèles abandonnés. Leur
validation a donné 9 315 tests réussis, 164 ignorés et 8 tests de stress exclus.
Ces correctifs ont été conservés dans le commit `7ad06651` avant les extractions.

**Architecture visée**

| Emplacement | Responsabilité |
|---|---|
| `_ai_streaming.py` | Cycle de vie du tour et orchestration : préparer un appel au modèle, diffuser ses fragments, décider de terminer ou d'exécuter les outils locaux, passer au suivant, finaliser. |
| Nouveau `_ai_stream_round.py` | Traitement d'un appel au modèle : événements reçus, texte, préfixe déjà affiché, fenêtres de raisonnement et composition des arguments. État du round et petit composant de déduplication définis à côté de leur utilisation. |
| Nouveau `_ai_stream_external_tools.py` | Traitement des appels externes : contrôle préalable des appels encore en attente, observation des appels déjà exécutés, marqueurs et publications correspondants. |
| `_ai_loop_rules.py` | Autorité commune des décisions de boucle, budgets, réponse vide et arrêt forcé pour les modes streaming et non streaming. |
| `_ai_tools.py` | Validation, autorisation, exécution parallèle des outils locaux et attente de leur nettoyage. |
| `_ai_resilience.py`, `_ai_coalescers.py` | Responsabilités actuelles conservées : récupération des erreurs fournisseur et regroupement des événements. |

Privilégier des fonctions et de petites classes internes concrètes. Les nouveaux
noms restent privés. Introduire des paramètres et callbacks typés correspondant
aux besoins de chaque composant ; éviter de transmettre tout `AIChannel` ou une
longue collection de valeurs de type `Any`.

Séparer explicitement les durées de vie : le tour possède ses compteurs et son
transcript ; un round possède ses fragments, outils et raison de fin fournisseur ;
une tentative fournisseur possède ses compteurs de composition. Réutiliser
`_ToolLoopContext` et `_ToolLoopState` pour les données qu'ils portent déjà.
Tout état mutable reste propre à une invocation.

**Contrats à protéger**

- Le texte et les marqueurs arrivent progressivement, dans le même ordre ; aucun
  stockage de toute la réponse avant son premier affichage. La consommation du
  flux continue de contrôler sa progression.
- Le texte brut envoyé au modèle et le texte réellement livré restent distincts.
  Le transcript de `ON_AI_RESPONSE` décrit ce que les participants ont reçu.
- Chaque fenêtre de raisonnement conserve ses propres limites et sa signature.
  Les événements de composition portent des tailles, jamais les arguments.
  Une nouvelle tentative ferme la composition précédente et repart de zéro.
- Les outils locaux passent toujours les contrôles existants. Les arguments
  enregistrés à la fin sont ceux effectivement utilisés après les hooks.
- Un résultat externe déjà présent dans `_result` est observé comme une action
  déjà exécutée. Un appel externe encore en attente passe le contrôle préalable.
  Conserver l'ordre actuel des hooks, marqueurs persistés et événements éphémères.
- Après l'arrêt forcé, une seule génération finale est permise ; elle ne peut
  relancer ni outil ni nouvelle tentative pour réponse vide.
- Les raisons de fin, compteurs de tours et compteurs de tokens restent identiques.
  Un outil supprimé par une limite ne crée pas d'appel orphelin dans l'historique.
- Chaque générateur est fermé par son consommateur direct, qui attend sa
  finalisation. Les tâches des outils restent possédées jusqu'à leur nettoyage.
  Les registres d'activité et le contexte de l'invocation sont libérés même
  lors d'une exception ou d'une annulation pendant le nettoyage.
- Une fin normale produit son marqueur terminal une fois. Une fermeture par le
  consommateur ou une exception respecte le contrat actuel de propagation : ne
  pas essayer de produire un marqueur depuis le nettoyage d'un générateur fermé.
- Les contrats publics et la RFC restent la référence. Tout défaut découvert
  pendant l'extraction reçoit sa reproduction et son correctif distincts.

**Lots d'implémentation, dans cet ordre**

1. **Fixer la référence comportementale.** Conserver les corrections d'hygiène
   et leurs régressions. Recenser les tests existants dans la matrice ci-dessous,
   puis compléter seulement les scénarios manquants : ordre relatif des sorties,
   fermeture à chaque frontière d'extraction, déduplication selon le découpage
   des fragments et isolation entre deux tours concurrents. Les assertions
   portent sur les événements observables, les appels effectués et les ressources
   libérées ; elles restent valides si les méthodes internes changent de nom.

2. **Rendre l'état explicite et extraire la déduplication.** Introduire les petits
   états privés au plus près de leur propriétaire et remplacer les variables
   locales progressivement. Extraire le traitement du préfixe dans un composant
   synchrone avec opérations d'ajout et de fin. Couvrir un préfixe identique,
   partiel, divergent et réparti différemment entre fragments. Critère : mêmes
   sorties, mêmes moments de livraison et même transcript.

3. **Extraire les outils externes.** Déplacer leur traitement dans le module
   dédié, en conservant leurs points d'appel : pendant la réception pour le
   gestionnaire externe, après la réception pour le chemin observé par hooks
   seuls. Préserver les contrôles, réécritures et métadonnées. Critère : aucune
   exécution supplémentaire et aucune modification de l'ordre des événements.

4. **Extraire le traitement d'un round et ses ressources.** Créer un composant
   privé qui consomme le flux issu de `_generate_stream_with_retry`, expose les
   mêmes `StreamDelta` et conserve un état typé consultable à la fin. Il gère
   les fenêtres de raisonnement, les fragments d'outils et leur remise à zéro
   aux frontières de tentative. Conserver une chaîne de possession explicite :
   la boucle ferme le générateur du round ; le round ferme le wrapper de
   récupération ; ce wrapper ferme le flux fournisseur. Utiliser un `finally`
   ou un gestionnaire de contexte async adapté à chaque frontière. Tester les
   finalisations qui attendent réellement une opération async.

5. **Réduire la boucle à l'orchestration et valider.** La méthode principale
   expose clairement : préparer → recevoir et diffuser → décider → exécuter
   les outils locaux → recommencer ou terminer. Les décisions communes restent
   dans `_ai_loop_rules.py`, avec des tests des deux modes. Le cycle de vie du
   tour garde un seul endroit pour ses compteurs, son bilan et la libération
   du registre. Mettre à jour `docs/architecture.md` et, si nécessaire,
   `docs/technical.md` dans le dépôt de documentation pour expliquer la nouvelle
   répartition. Vérifier la livraison progressive, la consommation à la demande
   et le nettoyage, puis lancer `make all`.

Chaque lot constitue un changement relisible et réversible : tests ciblés,
inspection du diff, puis `make all` avant commit. Adapter les tests existants à
leur nouveau propriétaire plutôt que conserver des méthodes relais uniquement
pour satisfaire des tests internes.

**Matrice de validation à réutiliser**

| Comportement | Tests existants dans `roomkit/tests/` |
|---|---|
| Livraison progressive, rounds, marqueurs et tokens | `test_ai_streaming_tool_loop.py` |
| Transcript réellement livré | `test_ai_response_transcript.py` |
| Raisonnement et fenêtres successives | `test_ai_streaming_thinking.py` |
| Composition, fermeture, retry et fournisseur de secours | `test_ai_streaming_tool_events.py` |
| Règles communes et limites | `test_channels/test_ai_loop_rules.py`, `test_channels/test_ai_empty_retry.py`, `test_channels/test_loop_end_reason_nonstreaming.py` |
| Arrêt forcé et appels invalides | `test_tool_repeat_guard.py`, `test_invalid_tool_repeat_guard.py` |
| Outils externes, hooks et arguments réécrits | `test_external_tool_handler.py`, `test_unified_tool_call.py`, `test_tool_arg_fold.py` |
| Erreurs fournisseur et compaction | `test_channels/test_ai_retry.py`, `test_channels/test_ai_streaming_compaction.py` |
| Annulation et fermeture des flux | `test_delivery_cancellation.py`, `test_delivery_cancellation_streams.py`, `test_inbound_stream_cancellation.py`, `test_parallel_tool_cleanup.py` |
| Contexte de l'acteur et du tour | `test_tool_context.py` |
| Comportement du streaming sans outils | `test_ai_response_plain_stream.py` |

**Critères de fin**

- La boucle principale se lit comme une séquence de décisions. Environ 100 à
  150 lignes constitue un repère, commentaires exclus ; le découpage se juge
  surtout à la clarté des responsabilités et des dépendances.
- Chaque état et chaque ressource a un propriétaire identifiable ; aucun état de
  round partagé entre conversations ni nettoyage implicite confié au ramasse-miettes.
- Ajouter un événement fournisseur se traite dans le composant de round ;
  modifier une règle de boucle se traite dans les règles communes.
- Les scénarios de la matrice passent, les deux corrections d'hygiène restent
  couvertes et `make all` réussit sur le résultat final.
- La livraison progressive et la consommation à la demande sont démontrées par
  des tests synchronisés avec des événements async, sans seuils de temps fragiles.

Le plan porte sur la boucle d'outils streaming et ses composants immédiats. Le
streaming texte simple ne recevra que les adaptations nécessaires aux helpers
partagés. Une nouvelle API publique, une nouvelle dépendance ou une modification
de protocole constituerait un changement de périmètre à traiter séparément.
