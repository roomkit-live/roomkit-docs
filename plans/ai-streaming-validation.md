# Validation du refactoring streaming IA

Intervention du 13 septembre 2026, bibliothèque vérifiée sur le commit `0ea54725`.

Le plan est implémenté. `_run_streaming_tool_loop()` passe de 596 à 109 lignes
(103 lignes non vides hors commentaires). Les états du tour et du round,
les fenêtres de composition et le traitement des outils externes ont des
propriétaires explicites. Les générateurs intermédiaires exposent leur capacité
de fermeture dans leur type et sont fermés par leur consommateur direct.

**Changements livrés en commits locaux**

| Commit | Objet |
|---|---|
| `7ad06651` | Corrections d'hygiène : arrêt forcé terminal et nettoyage des outils parallèles. |
| `b689ea5f` | Tests de contrat sur le texte livré, le transcript, l'ordre des événements et la fermeture async. |
| `0ef91c63` | État du round et filtrage du préfixe répété. |
| `090ab5dc` | Traitement séparé des outils externes. |
| `805bb667` | Correctif découvert pendant l'extraction : fermeture de la composition après annulation directe de la tâche. |
| `adf4649c` | Composant de round responsable des fragments et des ressources ; test de deux conversations entrelacées. |
| `0ea54725` | Orchestrateur simplifié, bilan du tour centralisé et essais Cerebras de bout en bout. |

**Vérifications automatiques**

| Vérification | Résultat |
|---|---|
| `make all`, résultat final | 9 327 réussis, 168 ignorés, 8 tests de stress exclus ; lint, formatage, typage et Bandit réussis. |
| Suite ciblée finale | 206 tests réussis : streaming, raisonnement, outils, reprises, compaction, contexte et annulations. |
| Chaque étape de refactoring | Suite ciblée, inspection du diff et `make all` avant commit. |
| Documentation | Construction MkDocs en mode strict réussie. |
| Distribution Python | Wheel et sdist construits ; présence des nouveaux modules vérifiée ; import effectué depuis la wheel. |
| Diff | `git diff --check` réussi. |

Les paquets construits portent la version de développement existante
`0.73.0.dev0`. Cette validation n'a ni changé la version ni publié de paquet.

**Essais réels avec Cerebras**

Les tests ont chargé la clé fournie depuis `/home/quintana/.secrets/cerebras`
dans l'environnement du processus de test. Elle n'a été ni ajoutée au dépôt,
ni passée en argument de commande, ni affichée. Les requêtes utilisent des
prompts et des outils synthétiques.

Les six tests fournisseur existants sont passés avant le refactoring. La
validation finale comporte dix tests, tous réussis, répartis sur
`gpt-oss-120b` et `qwen-3.8-27b` :

- réponse simple et streaming de texte ;
- appel d'outil au niveau du fournisseur ;
- tour complet d'`AIChannel` : émission d'un code synthétique par un outil,
  lecture de ce résultat par le modèle, vérification par un second outil,
  puis réponse finale `VERIFIED` ;
- fermeture anticipée du flux du canal, retour de `active_turns` à zéro,
  puis nouvelle requête réussie avec le même fournisseur.

| Modèle | Outils exécutés dans le scénario dépendant | Rounds d'outils | État final |
|---|---|---|---|
| `gpt-oss-120b` | 2 | 2 | `completed` |
| `qwen-3.8-27b` | 2 | 2 | `completed` |

Une première exécution du nouveau scénario a montré une erreur de copie d'un
code synthétique long par GPT-OSS. L'erreur a été renvoyée au modèle, qui a
corrigé son appel. Le test live accepte donc les reprises dans la limite de
quatre rounds, exige une vérification réellement réussie et compare les
compteurs aux appels observés. Les tests déterministes gardent les assertions
exactes sur le nombre d'appels et l'ordre des événements.

**Preuves de comportement conservé**

- Huit découpages du préfixe vérifient le texte livré et le transcript :
  préfixe complet, partiel, divergent, ou réparti entre plusieurs fragments.
- Un fournisseur piloté par le consommateur prouve qu'un premier fragment
  est livré avant que le suivant soit produit. Sa finalisation attend un
  événement async ; la fermeture attend effectivement cette finalisation.
- Une trace vérifie l'ordre raisonnement → composition → marqueur de début
  → événement de début → outil → événement de fin.
- Deux rooms entrelacées sur le même canal gardent des transcripts,
  contextes d'exécution et compteurs de tokens distincts.
- Les tests existants couvrent aussi les fenêtres de raisonnement successives,
  les compteurs de cache, le plafonnement des appels, les erreurs fournisseur,
  les reprises de composition et les annulations pendant le nettoyage.
- Le nouveau cas d'annulation directe a d'abord échoué : une seule publication
  de composition, sans trame terminale. Après correction, la trame terminale
  vide est publiée et le canal ne conserve aucun tour actif.

**Reproduction**

Depuis le dépôt `roomkit` :

```bash
make all
uv run pytest tests/test_ai_stream_contract.py -q
uv run --frozen --extra docs mkdocs build --strict -f ../roomkit-docs/mkdocs.yml
uv build
```

Pour les essais réels, charger `CEREBRAS_API_KEY` dans l'environnement depuis
le stockage de secrets, puis lancer :

```bash
ROOMKIT_RUN_CEREBRAS_LIVE=1 uv run --frozen pytest tests/test_integration/test_cerebras_live.py -q
```

Les résultats détaillés de cette exécution sont conservés localement dans
`/tmp/roomkit-stream-final-checks.log`, `/tmp/roomkit-stream-final-targeted.log`,
`/tmp/roomkit-cerebras-final-verified.xml` et `/tmp/roomkit-stream-docs-build.log`.
Ces fichiers temporaires ne sont pas nécessaires aux tests et peuvent disparaître.

Les scénarios live valident les modèles et chemins indiqués sur cette exécution.
Les suites normalement ignorées ou exclues restent hors de cette validation ;
ces résultats ne constituent pas une preuve d'absence de tout bug possible.
