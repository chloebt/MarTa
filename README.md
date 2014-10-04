MarTa
=======

Discourse Marker Tagger : identify the discourse markers in a text, the relations they trigger and the arguments.


# Code : répertoire connTagger/py/

Le répertoire py/ contient le code nécessaire pour lire un fichier ou un répertoire de fichiers de texte brut, et éventuellement le ou les fichiers correspodants contenant une analyse syntaxique du texte.

### Description

Le programme tokenize et POS tag le fichier brut en entrée puis identifie les formes correspondant à un connecteur, identifie les connecteurs (désambiguïsation en emploi), les relations qu'ils déclenchent (désambiguïsation en relation) et la position des arguments (inter ou intraphrastiques) utilisée pour identifier les arguments des connecteurs. Ensuite les phrases non liées par une relation explicite sont considérées comme des exemples de relation implicite (pour l'instant, pas de modèle pour discriminer relation implicite/altlex/entrel/norel et pour identifier la relation).


Le code est divisé en 8 fichiers :

* stream.py :
  * Contient la classe Document représentant l'ensemble d'instances correspondant à un fichier raw. (Note : c'est aussi dans cette classe qu'on définit les répertoires en sortie, ie annot/, pdtb/),
  * Contient la classe abstraite Stream représentant l'entrée du programme. La méthode principale est **read()** qui lit l'entrée et retourne une instance de Document par fichier raw,
  * Contient la classe FileSrc : cas où l'entrée est un fichier,
  * Contient la classe PathSrc : cas où l'entrée est un répertoire.
* mention.py : **TODO** A refaire entièrement, pour l'instant tout est une instance explicite ...
  * Contient la classe abstraite Mention,
  * Contient la classe ExplicitMention, avec la méthode **feater()** qui calcule les traits pour la mention,
  * Contient la classe ImplicitMention (non utilisée),
  * Contient la classe Connective pour représenter un connecteur (non utilisée),
  * Contient la classe Argument : représente un argument.
* tokenizer.py :
  * Contient la classe abstraite Tokenizer. La méthode principale est la méthode **tokenize( Stream document )** qui tokenize le texte brut, ie le champ Stream.doc_tokenized_sents = liste de tuple (token, start, end) avec start..end l'empan dans le texte brut,
  * Contient la classe NLTKTokenizer qui tokenize le texte  brut en utilisant nltk.tokenize.TreebankWordTokenizer. N'est utilisé que si aucun de fichier d'analyse syntaxique n'est fourni.
* posTagger.py :
  * Contient la classe abstraite POSTagger. La méthode principale est la méthode **postag( Stream document )** qui récupère les POS des tokens, ie le champ Stream.doc_postagged_sents = liste de tuple (token, POS, start, end) avec start..end l'empan dans le texte brut,
  * Contient la classe NLTKPosTagger : POS tag en utilisant nltk.tag,
  * Contient la classe PTBPosTagger : utilise le fichier contenant le format parenthésé pour tokeniser et POS tagger le texte brut.
* connTagger.py : 
  * Contient la méthode **main**,
  * Contient la classe abstraite ConnTagger. La méthode principale est la méthode **conntag(Stream document)** qui récupère les exemples négatifs, explicites positifs et les exemples implicites, ie remplit des champs de Stream document,
  * Contient la classe dérivée ConnTaggerFromModel. La méthode **conntag(Stream document)** applique les différents modèles pour identifier les connecteurs, les relations et la position des arguments (inter ou intraphrastique) (**TODO** ne devrait être fait que s'il y a un parse, à vérifier),
* segmenter.py :
  * Contient la classe abstraite Segmenter. La méthode principale est la méthode **segment( Stream document )** qui calcule les arguments pour les exemples explicites (positifs) et implicites si un fichier de parse est fourni (**TODO** : pourquoi en fait ? On ne peut pas faire le format PDTB mais on peut chercher les arguments, à modifier). Récupère les informations suivantes : empan, texte et éventuellement adresses de gorn des arguments (selon la position inter/intra prédite). Les classes implémentant cette classe doivent uniquement redéfinir la méthode **get_span_intra(Mention mention)**, puisque ces informations se récupèrent toujours de la même façon si un argument est une phrase,
  * Contient la classe HeuristicSegmenter : récupère les informations sur les arguments en utilisant une heuristique simpliste basée sur la position du connecteur.
  * **TODO** : classe DependencySegmenter, basée sur une analyse en dépendance pour récupérer les arguments intra-phrastiques (en utilisant nltk.parse.malt, modèle malt déjà présent dans les ressources).
* utils.py : contient essentiellement des méthodes pour lire et écrire des fichiers.
* tree.py : version (ancienne et/ou modifiée ?) de nltk.tree, on utilise la classe ParentedTree (permet de calculer des adresse de gorn, similaires à celles du PDTB).


### Options
* **-r** --raw (required) : un texte brut, au format une phrase par ligne, ou un répertoire de fichiers au même format ;
* **-o** --out : le répertoire dans lequel seront écrits les fichiers en sortie. Si aucun répertoire n'est donné, le répertoire connTagger_out/ est créé ;
* **-p** --parsed (not required) : un fichier ou un répertoire de fichiers contenant une analyse syntaxique du ou des textes bruts (format parenthésé). Si on donne des répertoires en entrée, les noms des fichiers doivent être les mêmes que ceux des fichiers bruts avec l'extension **.mrg**, avec éventuellement la même hiérarchie de répertoires (ie. si on donne en options -r raw/ -p ptb/, avec la hiérarchie raw/00/wsj_0001, alors on doit avoir ptb/00/wsj_0001.mrg). Si on dispose de l'analyse syntaxique, elle est utilisée pour tokeniser et POStagger le texte brut. Sinon, on utilise NLTK pour tokeniser et POStagger. Si l'analyse syntaxique n'est pas fournie, le programme ne produit que les exemples explicites (pas d'implicites) et pas de format PDTB (nécessite un calcul d'adresses de Gorn) ;
* **-l** --labels : jeu de relations, fichier avec correspondance relation et label numérique. Doivent correspondre aux modèles, donc pour le PDTB, utiliser le fichier disponible dans data/, décrit infra ;
* **-e** --model_emploi : le modèle pour la désambiguïsation en emploi (discourse reading), tel que produit par un code scikit-learn (utilisation de joblib pour loader le modèle). TODO : à vérifier, je pense que pour l'instant seul un MaxEnt fonctionne, à cause de la façon dont on récupère les scores. Cf description plus bas ;
* **-m** --model_relation : le modèle permettant d'identifier les relations déclenchées par les connecteurs, idem précédent ;
* **-s** --model_position : le modèle permettant de discriminer connecteurs déclanchant une relation inter ou intra-phrastique, est utilisé pour identifier les arguments, idem précédent ;
* **-c** --connectives : lexique de connecteurs. Pour l'instant, ne considère qu'un simple fichier texte ;
* **-f** --lexfeats : l'alphabet de traits tel qu'utilisé pour construire les modèles. Cf description plus bas ;

### Sortie 
En sortie, le programme fournit minimalement les fichiers suivants, dans le répertoire out/annot/ (en reproduisant éventuellement la hiérarchie de sous-répertoires en entrée), file_name correspond au basename du fichier raw en entrée :

* file_name.annot : format colonne, la première ligne correspond aux catégories puis une ligne par exemple. Ce fichier contient les exemples positifs et négatifs pour chaque connecteur, avec les informations suivantes : 
  ** id de l'exemple (id), 
  ** path du fichier raw (base_file), 
  ** forme du connecteur (form), 
  ** le type prédit, entre negative et positive (discourse_use), 
  ** scores obtenus pour la désambiguïsation en emploi (score_emploi), au format (score_negative, score_positive),
  ** la position prédite, entre -1_intra et 1_inter (position), 
  ** scores obtenus pour le classifieur inter/intra (score_position), au format (score_intra, score_inter), 
  ** la relation prédite, au format label_relation (relation),
  ** scores obtenus pour la désambiguïsation en relation (score_relation), au format label1:score_label1;label2:score_label2 ...,
  ** empan du connecteur dans le fichier raw (span_conn). 

**TODO** : ajouter les arguments.

* file_name.annot.raw : le fichier raw d'origine avec annotation des connecteurs (ie exemples positifs), au format 
<\connective form=because text=because id=wsj_0004_18 relation=Contingency\>

* file_name.feats : le fichier de traits produit,

* file_name.postag : un fichier format colonne, au format token\tPOS.

Si une analyse syntaxique est fournie, le programme crée en plus un répertoire out/pdtb_root/ contenant des fichiers de la forme file_name.pdtb :

* ces fichiers sont au format PDTB, donc on a les mêmes informations que dans les fichiers PDTB : adresses de gorn et empan du connecteur (gorn de la seconde phrase et start du premier token de la second phrase pour les implicites), adresses de gorn, empan et texte des arguments (cf description ifnra). Les informations de type "features" ne sont pas calculés, on a donc toujours les mêmes valeurs. Les relations des exemples implicites ne sont pas identifiées, on a donc toujours le connecteur "and" et la relation "Expansion" pour ces exemples ;
* ils contiennent les exemples explicites (positifs) et les exemples implicites (cf description infra).
* Normalement donc, ils peuvent être lus par l'API PDTB, en lien avec le raw et le ptb.

# Données
Le programme a besoin d'un certain nombre de ressources, toutes disponibles dans data/ :
* **Modèles** :
  * **models_pdtb_connective/** : contient les modèles calculés en prenant uniquement comme trait la forme du connecteur (cf scores plus bas) :
    * model_discourse_reading_connective : modèle (binaire) de désambiguïsation en emploi ;
    * model_explicit_lvl1_connective : modèle (multiclasse) de désambiguïsation en relation de niveau 1 (4 classes) ;
    * model_explicit_lvl2_allrel_connective : modèle (multiclasse) de désambiguïsation en relation de niveau 2 en prenant toutes les relations disponibles (16 classes) ;
    * model_explicit_lvl2_linrel_connective : modèle (multiclasse) de désambiguïsation en relation de niveau 2 en prenant les relations conservées par Lin09 (11 classes) ;
    * model_position_connective : modèle (binaire) classant les exemples entre inter et intraphrastiques.
  * **models_pdtb_poswdcontext** : contient les modèles calculés en prenant comme trait les traits de type POS, c'est-à-dire des traits qui ne nécessitent pas d'analyse syntaxique (forme du connecteur (C), POS du connecteur (C-POS), POS du mot précédent (prev-POS), POS du mot suivant (next-POS) et combinaison : prev+C (prev=le mot précédent), C+next (next=le mot suivant), prev-POS+C-POS, C-POS+next-POS) :
    * model_discourse_reading_poswdcontext : modèle (binaire) de désambiguïsation en emploi ;
    * model_explicit_lvl1_poswdcontext : modèle (multiclasse) de désambiguïsation en relation de niveau 1 (4 classes) ;
    * model_explicit_lvl2_allrel_poswdcontext : modèle (multiclasse) de désambiguïsation en relation de niveau 2 en prenant toutes les relations disponibles (16 classes) ;
    * model_explicit_lvl2_linrel_poswdcontext : modèle (multiclasse) de désambiguïsation en relation de niveau 2 en prenant les relations conservées par Lin09 (11 classes) ;
    * model_position_poswdcontext : modèle (binaire) classant les exemples entre inter et intraphrastiques.

**TODO** : faire d'autres modèles.

* **lexfeats.txt** : contient l'alphabet de traits (ie : correspondance entre noms de traits et identifiant numérique tel qu'utilisé pour construire les modèles) ;

* **Liste des relations** :
  * **pdtb_hierarchie.txt** : toutes les relations annotées dans le PDTB, correspondance entre relation et label numérique ;
  * **pdtb_hierarchie_lin.txt** : les relations utilisées par Lin09 pour les implicites (ie 5 relations supprimées).

* **Liste de connecteurs** :
  * **lexconn_pdtb.txt** : un connecteur par ligne. Les connecteurs discontinus sont de la forme part1..part2.


# Exemples

##### Données pour tester le programme

Dans data/data2test/ se trouvent des données pour tester le programme, des répertoires raw/ et ptb/.

##### Lancement du programme

Exemple, pas de fichier contenant le parse (tokenisation, pos tagging avec nltk), ne sort pas d'exemples implicites ni de format PDTB :

python connectiveTagger/py/connTagger.py -r connectiveTagger/data/data2test/raw/00/wsj_0004 -e connectiveTagger/data/model_pdtb_postagFeats/model_emploi -f connectiveTagger/data/model_pdtb_postagFeats/correspondance_valnum2feat.txt -l connectiveTagger/data/labels_pdtb_level1.txt -c connectiveTagger/data/lex_conn_pdtb.txt -m connectiveTagger/data/model_pdtb_postagFeats/model_relation -s connectiveTagger/data/model_pdtb_postagFeats/model_position -o connectiveTagger/data/data2test/out/

Exemple, avec fichier contenant le parse :

python connectiveTagger/py/connTagger.py -r connectiveTagger/data/data2test/raw/00/wsj_0004 -p connectiveTagger/data/data2test/ptb/00/wsj_0004.mrg -e connectiveTagger/data/model_pdtb_postagFeats/model_emploi -m connectiveTagger/data/model_pdtb_postagFeats/model_relation -s connectiveTagger/data/model_pdtb_postagFeats/model_position -f connectiveTagger/data/model_pdtb_postagFeats/correspondance_valnum2feat.txt -l connectiveTagger/data/labels_pdtb_level1.txt -c connectiveTagger/data/lex_conn_pdtb.txt -o connectiveTagger/data/data2test/out/
 

# Scores obtenus par les différents modèles

Scores obtenus pour les différents modèles sur le test :

* Split Lin, test = section 23 du PDTB ;
* On ne conserve que la première relation annotée ;
* Pour le niveau 2, on donne des résultats en conservant les memes 11 relations que Lin (pour les implicites) et en conservant les 16 relations ;
* Tous les modèles sont obtenus avec un MaxEnt (Scikit), L2, C=1
       
**TODO** : optimisation des paramètres

##### Modèle désambiguïsation en emploi


###### traits = connective


             |      prec.|	rec. |	f1   |	support
-------------|-----------|-------|-------|------------
negative (-1)|   	86.7 |	94.1 |	90.3 |	2075.0
positive (1) |      83.6 |	67.6 |	74.8 |	923.0
Micro-Avg    |   	86.0 |	86.0 |	86.0 |	None
Macro-Avg    |	    85.2 |	80.9 |	82.5 |	None
Weighted-Avg |	    85.8 |	86.0 |	85.5 |	None




is/as 	       |  negative (-1)	| positive (1)
---------------|----------------|--------------
negative (-1)  | 	1953  	    |   122
positive (1)   | 	299   	    |   624
