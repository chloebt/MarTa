MarTa
=======

Discourse Marker Tagger : identify the discourse markers in a text, the relations they trigger and the arguments.


# Code : répertoire MarTa/py/

Le répertoire py/ contient le code nécessaire pour lire un fichier ou un répertoire de fichiers de texte brut, et éventuellement le ou les fichiers correspondants contenant une analyse syntaxique (constituants) du texte.

### Description

Le programme tokenize et POS tag le fichier brut en entrée puis identifie les formes correspondant à un connecteur, identifie les connecteurs (désambiguïsation en emploi), les relations qu'ils déclenchent (désambiguïsation en relation) et la position des arguments (inter ou intraphrastiques). Si un fichier de parse est fourni, on identifie les arguments des connecteurs, ensuite les phrases non liées par une relation explicite sont considérées comme des exemples de relation implicite (pour l'instant, pas de modèle pour discriminer relation implicite/altlex/entrel/norel et pour identifier la relation). Pour un exemple explicite, il peut arriver que le modèle prédise des arguments interphrastiques alors que le connecteur est dans la première phrase du document : dans ce cas, la position prédite est modifiée en intraphrastique et les scores sont modifiés avec 100 pour intra et 0 pour inter.

Environ 3s. pour lire la section 00/ du PDTB.

Le code est divisé en 8 fichiers :

* **stream.py** :
  *  classe *Document* représentant l'ensemble de mentions correspondant à un fichier raw. (Note : c'est aussi dans cette classe qu'on définit les répertoires en sortie, ie annot/, pdtb/),
  *  classe abstraite *Stream* représentant l'entrée du programme. La méthode principale est **read()** qui lit l'entrée et retourne une instance de *Document* par fichier raw,
  *  classe *FileSrc* dérivée de *Stream* : cas où l'entrée est un fichier,
  *  classe *PathSrc* dérivée de *Stream* : cas où l'entrée est un répertoire.
* **mention.py** : 
  *  classe abstraite *Mention*,
  *  classe *ExplicitMention*, avec la méthode **feater()** qui calcule les traits pour la mention,
  *  classe *ImplicitMention* (non utilisée),
  *  classe *Connective* pour représenter un connecteur (non utilisée),
  *  classe *Argument* : représente un argument.
* **tokenizer.py** :
  *  classe abstraite *Tokenizer*. La méthode principale est la méthode **tokenize( Document document )** qui tokenize le texte brut, ie le champ *Document.doc_tokenized_sents = liste de tuple (token, start, end)* avec *start..end* l'empan dans le texte brut,
  *  classe *NLTKTokenizer* qui tokenize le texte  brut en utilisant *nltk.tokenize.TreebankWordTokenizer*. N'est utilisé que si aucun de fichier d'analyse syntaxique n'est fourni.
* **posTagger.py** :
  *  classe abstraite *POSTagger*. La méthode principale est la méthode **postag( Document document )** qui récupère les POS des tokens, ie le champ *Document.doc_postagged_sents = liste de tuple (token, POS, start, end)* avec *start..end* l'empan dans le texte brut,
  *  classe *NLTKPosTagger* : POS tag en utilisant *nltk.tag*,
  *  classe *PTBPosTagger* : utilise le fichier contenant le format parenthésé pour tokeniser et POS tagger le texte brut.
* **connTagger.py** : 
  *  méthode **main**,
  *  classe abstraite *ConnTagger*. La méthode principale est la méthode **conntag( Document document )** qui récupère les exemples négatifs, explicites positifs et les exemples implicites, ie remplit des champs de *Document document*,
  *  classe *ConnTaggerFromModel*. La méthode **conntag( Document document )** applique les différents modèles pour identifier les connecteurs, les relations et la position des arguments (inter ou intraphrastique),
* **segmenter.py** :
  *  classe abstraite *Segmenter*. La méthode principale est la méthode **segment( Document document )** qui calcule les arguments pour les exemples explicites (positifs) et implicites si un fichier de parse est fourni. Récupère les informations suivantes : empan, texte et éventuellement adresses de gorn des arguments (selon la position inter/intra prédite). Les classes implémentant cette classe doivent uniquement redéfinir la méthode **get_span_intra( Mention mention )**, puisque ces informations se récupèrent toujours de la même façon si un argument est une phrase,
  *  classe *HeuristicSegmenter* : récupère les informations sur les arguments en utilisant une heuristique simpliste basée sur la position du connecteur.
* **utils.py** : contient essentiellement des méthodes pour lire et écrire des fichiers.
* **tree.py** : version (ancienne et/ou modifiée ?) de *nltk.tree*, on utilise la classe *ParentedTree* (permet de calculer des adresse de gorn, similaires à celles du PDTB).


### Options
* **-r** --raw (required) : un texte brut, au format une phrase par ligne, ou un répertoire de fichiers au même format ;
* **-p** --parsed (not required) : un fichier ou un répertoire de fichiers contenant une analyse syntaxique du ou des textes bruts (format parenthésé). Si on donne des répertoires en entrée, les noms des fichiers doivent être les mêmes que ceux des fichiers bruts avec l'extension **.mrg**, avec éventuellement la même hiérarchie de répertoires (ie. si on donne en options *-r raw/ -p ptb/*, avec la hiérarchie *raw/00/wsj_0001*, alors on doit avoir *ptb/00/wsj_0001.mrg*). Si on dispose de l'analyse syntaxique, elle est utilisée pour tokeniser et POStagger le texte brut. Sinon, on utilise NLTK pour tokeniser et POStagger. Si l'analyse syntaxique n'est pas fournie, le programme ne produit que les exemples explicites (pas d'implicites), ne cherche pas les arguments et pas de format PDTB (nécessite un calcul d'adresses de Gorn) ;
* **-o** --out : le répertoire dans lequel seront écrits les fichiers en sortie. Si aucun répertoire n'est donné, le répertoire *markerTagger_out/* est créé ;
* **-l** --labels : jeu de relations, fichier avec correspondance relation et label numérique. Doivent correspondre aux modèles, donc pour le PDTB, utiliser l'un des fichiers disponibles dans *data/ressources/relations/*, décrit infra ;
* **-e** --model_emploi : le modèle pour la désambiguïsation en emploi (discourse reading), tel que produit par un code scikit-learn (utilisation de *joblib* pour loader le modèle). Cf description plus bas ;
* **-m** --model_relation : le modèle permettant d'identifier les relations déclenchées par les connecteurs, idem précédent ;
* **-s** --model_position : le modèle permettant de discriminer connecteurs déclanchant une relation inter ou intra-phrastique, est utilisé pour identifier les arguments (mais si pas de parse fourni, pas d'argument. L'information apparaît quand même des fichiers *.annot*), idem précédent ;
* **-c** --connectives : lexique de connecteurs. Pour l'instant, ne considère qu'un simple fichier texte ;
* **-f** --lexfeats : l'alphabet de traits tel qu'utilisé pour construire les modèles. Cf description plus bas ;

### Sortie 
En sortie, le programme fournit minimalement les fichiers suivants, dans le répertoire *out/annot/* (en reproduisant éventuellement la hiérarchie de sous-répertoires en entrée), *file_name* correspond au basename du fichier raw en entrée :

* *file_name.annot* : format colonne, la première ligne correspond aux catégories puis une ligne par exemple. Ce fichier contient les exemples positifs et négatifs pour chaque connecteur, avec les informations suivantes : 
  * id de l'exemple (id), 
  * path du fichier raw (base_file), 
  * forme du connecteur (form), 
  * le type prédit, entre negative et positive (discourse_use), 
  * scores obtenus pour la désambiguïsation en emploi (score_emploi), au format (score_negative, score_positive),
  * la position prédite, entre -1_intra et 1_inter (position), 
  * scores obtenus pour le classifieur inter/intra (score_position), au format (score_intra, score_inter), 
  * la relation prédite, au format label_relation (relation),
  * scores obtenus pour la désambiguïsation en relation (score_relation), au format label1:score_label1;label2:score_label2 ...,
  * empan du connecteur dans le fichier raw (span_conn). 

* *file_name.annot.raw* : le fichier raw d'origine avec annotation des connecteurs (ie exemples positifs), au format 
<\connective form=because text=because id=wsj_0004_18 relation=Contingency\>

* *file_name.feats* : le fichier de traits produit,

* *file_name.postag* : un fichier format colonne, au format token\tPOS.

Si une analyse syntaxique est fournie, le programme crée en plus un répertoire out/pdtb_root/ contenant des fichiers de la forme file_name.pdtb :

* ces fichiers sont au format PDTB, donc on a les mêmes informations que dans les fichiers PDTB : adresses de gorn et empan du connecteur (gorn de la seconde phrase et start du premier token de la second phrase pour les implicites), adresses de gorn, empan et texte des arguments. Les informations de type "features" ne sont pas calculés, on a donc toujours les mêmes valeurs. Les relations des exemples implicites ne sont pas identifiées, on a donc toujours le connecteur "and" et la relation "Expansion" pour ces exemples ;
* ils contiennent les exemples explicites (positifs) et les exemples implicites.
* normalement donc, ils peuvent être lus par l'API PDTB, en lien avec le raw et le ptb.

###### TODO

* Dans le PDTB, il semble que le point final d'une phrase ne soit pas compris dans l'argument. Vérifier que c'est le cas pour les intra et les inter-phrastiques, et modifier le code pour faire la même chose.
* Classe *DependencySegmenter*, basée sur une analyse en dépendance pour récupérer les arguments intra-phrastiques (en utilisant nltk.parse.malt).
* Refaire entièrement la classe *Mention*, pour l'instant tout est une mention explicite (parce que la plupart des informations sont les mêmes, dans le PDTB les implicites on un connecteur qui a un empan et une adresse de gorn) ...
* Vérifier ce qui n'est pas fait quand il n'y a pas de parse fourni (Pour l'instant ne segmente pas, normalement, ne devrait juste pas écrire de format PDTB.)
* Ajouter les implicites dans les fichiers .annot.
* Ajouter les arguments dans les fichiers .annot.
* A vérifier, je pense que pour l'instant seul un MaxEnt fonctionne, à cause de la façon dont on récupère les scores.
* Faire d'autres modèles.
* Optimisation des paramètres des modèles.
* Finir de mettre les tableaux infra en format markdown.
* Regrouper la tonne de TODO qui est dans le code ...
* Ajouter un parser (et extraire des traits syntaxiques).
* Modifier dans certaines parties du code, confusion entre stream et document pour les noms de variables.
* Ajouter des fichier de labels pour position et emploi.
* Et bien sûr, ajouter lecture d'Annodis.


# Données
Le programme a besoin d'un certain nombre de ressources, toutes disponibles dans data/ressources/ :
* **Modèles** *data/ressources/models/* :
  * **models_pdtb_connective/** : contient les modèles calculés en prenant uniquement comme trait la forme du connecteur (cf scores plus bas) :
    * *model_discourse_reading_connective* : modèle (binaire) de désambiguïsation en emploi ;
    * *model_explicit_lvl1_connective* : modèle (multiclasse) de désambiguïsation en relation de niveau 1 (4 classes) ;
    * *model_explicit_lvl2_allrel_connective* : modèle (multiclasse) de désambiguïsation en relation de niveau 2 en prenant toutes les relations disponibles (16 classes) ;
    * *model_explicit_lvl2_linrel_connective* : modèle (multiclasse) de désambiguïsation en relation de niveau 2 en prenant les relations conservées par Lin09 (11 classes) ;
    * *model_position_connective* : modèle (binaire) classant les exemples entre inter et intraphrastiques.
  * **models_pdtb_poswdcontext** : contient les modèles calculés en prenant comme trait les traits de type POS, c'est-à-dire des traits qui ne nécessitent pas d'analyse syntaxique (forme du connecteur (C), POS du connecteur (C-POS), POS du mot précédent (prev-POS), POS du mot suivant (next-POS) et combinaison : prev+C (prev=le mot précédent), C+next (next=le mot suivant), prev-POS+C-POS, C-POS+next-POS) :
    * *model_discourse_reading_poswdcontext* : modèle (binaire) de désambiguïsation en emploi ;
    * *model_explicit_lvl1_poswdcontext* : modèle (multiclasse) de désambiguïsation en relation de niveau 1 (4 classes) ;
    * *model_explicit_lvl2_allrel_poswdcontext* : modèle (multiclasse) de désambiguïsation en relation de niveau 2 en prenant toutes les relations disponibles (16 classes) ;
    * *model_explicit_lvl2_linrel_poswdcontext* : modèle (multiclasse) de désambiguïsation en relation de niveau 2 en prenant les relations conservées par Lin09 (11 classes) ;
    * *model_position_poswdcontext* : modèle (binaire) classant les exemples entre inter et intraphrastiques.

* **Alphabet de traits** *data/ressources/models/* :
  * *lexfeats.txt* : contient l'alphabet de traits (ie : correspondance entre noms de traits et identifiant numérique tel qu'utilisé pour construire les modèles) ;

* **Liste des relations** *data/ressources/relations/* :
  * *pdtb_hierarchie.txt* : toutes les relations annotées dans le PDTB, correspondance entre relation et label numérique ;
  * *pdtb_hierarchie_lin.txt* : les relations utilisées par Lin09 pour les implicites (ie 5 relations supprimées).

* **Liste de connecteurs** *data/ressources/markers/* :
  * *lexconn_pdtb.txt* : un connecteur par ligne. Les connecteurs discontinus sont de la forme part1..part2.


# Exemples

##### Données pour tester le programme

Dans *data/data2test/* se trouvent des données pour tester le programme, des répertoires *raw/* et *ptb/*.

##### Lancement du programme

Exemple, pas de fichier contenant le parse (tokenisation, pos tagging avec nltk), ne sort pas d'exemples implicites ni de format PDTB :

```
python MarTa/py/markerTagger.py -r MarTa/data/data2test/raw/00/wsj_0004 -e MarTa/data/ressources/models/model_pdtb_postagFeats/model_emploi -m MarTa/data/ressources/models/model_pdtb_postagFeats/model_relation -s MarTa/data/ressources/models/model_pdtb_postagFeats/model_position -f MarTa/data/ressources/models/lexfeats.txt -l MarTa/data/ressources/relations/labels_pdtb_level1.txt -c MarTa/data/ressources/markers/lexconn_pdtb.txt -o MarTa/data/data2test/out/
```

Exemple, avec fichier contenant le parse :

```
python MarTa/py/connTagger.py -r MarTa/data/data2test/raw/00/wsj_0004 -p MarTa/data/data2test/ptb/00/wsj_0004.mrg -e MarTa/data/ressources/models/model_pdtb_postagFeats/model_emploi -m MarTa/data/ressources/models/model_pdtb_postagFeats/model_relation -s MarTa/data/ressources/models/model_pdtb_postagFeats/model_position -f MarTa/data/ressources/models/lexfeats.txt -l MarTa/data/ressources/relations/labels_pdtb_level1.txt -c MarTa/data/ressources/markers/lexconn_pdtb.txt -o MarTa/data/data2test/out/
```

La plupart des options ont une valeur raisonnable par défaut (en utilisant les données 
dans *data/ressources/*), notamment, si on veut utiliser les modèles basés sur le jeu de trait 
POS-Word-Context avec des relations de niveau 1, il suffit de lancer :

```
python MarTa/py/markerTagger.py -r MarTa/data/data2test/raw/ -p MarTa/data/data2test/ptb/
```

# Scores obtenus par les différents modèles

Scores obtenus pour les différents modèles sur le test :

* Split Lin, train = sections 2-21, test = section 23 du PDTB ;
* On ne conserve que la première relation annotée ;
* Pour le niveau 2, on donne des résultats en conservant les memes 11 relations que Lin (pour les implicites) et en conservant les 16 relations ;
* Tous les modèles sont obtenus avec un MaxEnt (Scikit), L2, C=1.

### Modèle désambiguïsation en emploi


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



###### traits = POS Word Contxt


             |      prec.|	rec. |	f1   |	support
-------------|-----------|-------|-------|------------
negative (-1)|      93.6 |	96.3 |	95.0 |	2075.0
positive (1) |      91.2 |	85.3 |	88.1 |	923.0
Micro-Avg    |	    92.9 |	92.9 |	92.9 |	None
Macro-Avg    |	    92.4 |	90.8 |	91.5 |	None
Weighted-Avg |	    92.9 |	92.9 |	92.9 |	None



is/as 	      |  negative (-1) | positive (1)
--------------|----------------|----------------------
negative (-1) |	    1999  	   |     76
positive (1)  |	    136   	   |     787



### Modèle désambiguïsation en position 


###### traits = connective


         	    |    prec. |	rec.  |	f1    |	support
----------------|----------|----------|-------|-------
intra (-1)      |	 89.7  |	81.7  |	85.5  |	546.0
inter (1)       |	 76.5  |	86.5  |	81.2  |	377.0
Micro-Avg   	|    83.6  |	83.6  |	83.6  |	None
Macro-Avg       |	 83.1  |	84.1  |	83.4  |	None
Weighted-Avg	|    84.3  |    83.6  |	83.8  |	None


is/as 	       | intra (-1)	  |  inter (1)
---------------|--------------|-----------------------
intra (-1)	   |     446   	  |  100
inter (1) 	   |     51    	  |  326


###### traits = POS Word Contxt

                |	prec.  |	rec.  |	f1    |	support
----------------|----------|----------|-------|-------
intra (-1)      |	97.4   |	96.0  |	96.7  |	546.0
inter (1)       |	94.3   |	96.3  |	95.3  |	377.0
Micro-Avg   	|    96.1  |	96.1  |	96.1  |	None
Macro-Avg   	|    95.8  |	96.1  |	96.0  |	None
Weighted-Avg	|    96.1  |	96.1  |	96.1  |	None




is/as 	      | intra (-1)	   |  inter (1)
--------------|----------------|----------------------
intra (-1)	  |   524     	   | 22
inter (1) 	  |  14    	       | 363





### Modèle désambiguïsation en relation niveau 1


###### traits = connective

                |	prec.  |	rec.  |	f1    |	support
----------------|----------|----------|-------|-------
Temporal-1   	|83.6 	   |89.5	  |86.4	  |171.0
Contingency-6	|95.5 	   |85.1	  |90.0	  |175.0
Comparison-22	|95.7 	   |97.3	  |96.5	  |299.0
Expansion-31 	|98.2 	   |98.9	  |98.6	  |278.0
Micro-Avg    	|94.0 	   |94.0	  |94.0	  |None
Macro-Avg    	|93.3 	   |92.7	  |92.9	  |None
Weighted-Avg 	|94.2 	   |94.0	  |94.0	  |None



is/as        	|Temporal-1	  |Contingency-6|	Comparison-22 |	Expansion-31
----------------|-------------|-------------|-----------------|-------------
Temporal-1   	|153       	|7            	|11           	|0
Contingency-6	|25        	|149          	|0            	|1
Comparison-22	|4         	|0            	|291          	|4
Expansion-31 	|1         	|0            	|2            	|275




###### traits = POS Word Contxt


                |	prec.  |	rec.  |	f1    |	support
----------------|----------|----------|-------|-------
Temporal-1   	|84.9 	|95.3	|89.8	|171.0
Contingency-6	|98.7 	|85.7	|91.7	|175.0
Comparison-22	|97.0 	|96.7	|96.8	|299.0
Expansion-31 	|98.2 	|99.3	|98.7	|278.0
Micro-Avg    	|95.1 	|95.1	|95.1	|None
Macro-Avg    	|94.7 	|94.2	|94.3	|None
Weighted-Avg 	|95.4 	|95.1	|95.1	|None



is/as        	|Temporal-1	  |Contingency-6 |	Comparison-22 |	Expansion-31
----------------|-------------|--------------|----------------|-------------
Temporal-1   	|163       	|1            	|7            	|0
Contingency-6	|23        	|150          	|0            	|2
Comparison-22	|6         	|1            	|289          	|3
Expansion-31 	|0         	|0            	|2            	|276



### Modèle désambiguïsation en relation niveau 2 Lin Rel


###### traits = connective


                |	prec.  |	rec.  |	f1    |	support
----------------|----------|----------|-------|-------
Asynchronous-2    	|97.4 	|76.0 	|85.4 	|100.0
Synchronous-5     	|61.9 	|84.5 	|71.4 	|71.0
Cause-7           	|93.2 	|86.5 	|89.7 	|111.0
Pragmatic Cause-10	|0.0  	|0.0  	|0.0  	|1.0
Contrast-23       	|92.2 	|91.1 	|91.7 	|271.0
juxtaposition-24  	|0.0  	|0.0  	|0.0  	|0.0
Concession-27     	|42.4 	|60.9 	|50.0 	|23.0
Conjunction-32    	|85.7 	|98.1 	|91.5 	|213.0
Instantiation-33  	|100.0	|100.0	|100.0	|21.0
Restatement-34    	|66.7 	|28.6 	|40.0 	|7.0
Alternative-38    	|87.5 	|87.5 	|87.5 	|8.0
List-43           	|0.0  	|0.0  	|0.0  	|29.0
Micro-Avg         	|85.6 	|85.6 	|85.6 	|None
Macro-Avg         	|60.6 	|59.4 	|58.9 	|None
Weighted-Avg      	|84.1 	|85.6 	|84.4 	|None



is/as              |2	|5	 |7	 |10   |23  |24	 |27 |32 |33   |34 |38 |43
-------------------|----|----|---|-----|----|----|---|---|-----|---|---|-------
Asynchronous-2    	|76  |17  |7   |0   |0   |0   |0   |0   |0   |0   |0   |0
Synchronous-5     	|0   |60  |0   |0   |11  |0   |0   |0   |0   |0   |0   |0
Cause-7           	|0   |15  |96  |0   |0   |0   |0   |0   |0   |0   |0   |0
Pragmatic Cause-10	|0   |1   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0
Contrast-23       	|0   |4   |0   |0  |247  |0   |19  |1   |0   |0   |0   |0
juxtaposition-24  	|0   |0   |0   |0  | 0   |0   |0   |0   |0   |0   |0   |0
Concession-27     	|0   |0   |0   |0   |8   |0   |14  |0   |0   |0   |1   |0
Conjunction-32    	|2   |0   |0   |0   |2   |0   |0  |209  |0   |0   |0   |0
Instantiation-33  	|0   |0   |0   |0   |0   |0   |0   |0   |21  |0   |0   |0
Restatement-34    	|0   |0   |0   |0   |0   |0   |0   |5   |0   |2   |0   |0
Alternative-38    	|0   |0   |0   |0   |0   |0   |0   |0   |0   |1   |7   |0
List-43           	|0   |0   |0   |0   |0   |0   |0   |29  |0   |0   |0   |0




###### traits = POS Word Contxt


                |	prec.  |	rec.  |	f1    |	support
----------------|----------|----------|-------|-------
Asynchronous-2    	|94.3 	|83.0 	|88.3 	|100.0
Synchronous-5     	|63.0 	|88.7 	|73.7 	|71.0
Cause-7           	|98.0 	|88.3 	|92.9 	|111.0
Pragmatic Cause-10	|0.0  	|0.0  	|0.0  	|1.0
Contrast-23       	|93.1 	|90.0 	|91.6 	|271.0
juxtaposition-24  	|0.0  	|0.0  	|0.0  	|0.0
Concession-27     	|34.5 	|43.5 	|38.5 	|23.0
Conjunction-32    	|85.6 	|97.7 	|91.2 	|213.0
Instantiation-33  	|100.0	|100.0	|100.0	|21.0
Restatement-34    	|100.0	|14.3 	|25.0 	|7.0
Alternative-38    	|88.9 	|100.0	|94.1 	|8.0
List-43           	|50.0 	|3.4  	|6.5  	|29.0
Micro-Avg         	|86.2 	|86.2 	|86.2 	|None
Macro-Avg         	|67.3 	|59.1 	|58.5 	|None
Weighted-Avg      	|86.6 	|86.2 	|85.0 	|None




is/as             	|2	|5	|7	|10	|23	|24	  |27	|32	  |33  | 34	|38	|43
--------------------|---|---|---|----|----|----|----|-----|----|----|---|-----
Asynchronous-2    	|83 |17 | 0  | 0  | 0 |  0 |  0 |  0  | 0  | 0  | 0 |  0
Synchronous-5     	|1  |63 | 1  | 0  | 6 |  0 |  0 |  0  | 0  | 0  | 0 |  0
Cause-7           	|1   |12|  98|  0 |  0|   0|   0|   0 |  0 |  0 |  0|   0
Pragmatic Cause-10	|0   |1 |  0 |  0 |  0|   0|   0 |  0 |  0 |  0 |  0|   0
Contrast-23       	|0   |7 |  0 |  0 | 244|  0|   19|  1 |  0 |  0 |  0|   0
juxtaposition-24  	|0   |0 |  0 |  0 |  0 | 0 |  0   |0  | 0  | 0  | 0 |  0
Concession-27     	|2   |0 |  1 |  0 |  10|  0|   10 | 0 |  0 |  0 |  0|   0
Conjunction-32    	|1   |0 |  0 |  0 |  2 |  0|   0  |208|  0 |  0 |  1|   1
Instantiation-33  	|0   |0 |  0 |  0 |  0 |  0|   0  | 0 |  21|  0 |  0|   0
Restatement-34    	|0   |0 |  0 |  0 |  0 |  0|   0  | 6 |  0 |  1 |  0|   0
Alternative-38    	|0   |0 |  0 |  0 |  0 |  0|   0  | 0 |  0 |  0 |  8|   0
List-43           	|0   |0 |  0 |  0 |  0 |  0|  0   |28 | 0  | 0  | 0 |  1


 
### Modèle désambiguïsation en relation niveau 2 All Rel

###### traits = connective

                |	prec.  |	rec.  |	f1    |	support
----------------|----------|----------|-------|-------
Asynchronous-2     	|97.4 	|76.0 	|85.4 	|100.0
Synchronous-5      	|56.6 	|84.5 	|67.8 	|71.0
Cause-7            	|93.2 	|86.5 	|89.7 	|111.0
Pragmatic Cause-10 	|0.0  	|0.0  	|0.0  	|1.0
Condition-12       	|90.6 	|85.7 	|88.1 	|56.0
Pragmatic Condition	|0.0  	|0.0  	|0.0  	|7.0
Contrast-23        	|90.8 	|91.1 	|91.0 	|271.0
Pragmatic Contrast-	|0.0  	|0.0  	|0.0  	|0.0
Concession-27      	|42.4 	|60.9 	|50.0 	|23.0
Pragmatic Concessio	|0.0  	|0.0  	|0.0  	|4.0
Conjunction-32     	|85.7 	|98.1 	|91.5 	|213.0
Instantiation-33   	|100.0	|100.0	|100.0	|21.0
Restatement-34     	|66.7 	|28.6 	|40.0 	|7.0
Alternative-38     	|77.8 	|87.5 	|82.4 	|8.0
Exception-42       	|0.0  	|0.0  	|0.0  	|0.0
List-43            	|0.0  	|0.0  	|0.0  	|29.0
Micro-Avg          	|84.6 	|84.6 	|84.6 	|None
Macro-Avg          	|50.1 	|49.9 	|49.1 	|None
Weighted-Avg       	|82.6 	|84.6 	|83.0 	|None



is/as              	    |2	|5	|7	|10	|12	|19	|23	|26	|27	|30	|32	|33	|34	|38	|42	|43
------------------------|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
Asynchronous-2     	    |76  |17  |7   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0
Synchronous-5      	    |0   |60  |0   |0   |0   |0   |11  |0   |0   |0   |0   |0   |0   |0   |0   |0
Cause-7            	    |0   |15  |96  |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0
Pragmatic Cause-10 	    |0   |1   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0
Condition-12       	    |0   |8   |0   |0   |48  |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0
Pragmatic Condition-19	|0   |1   |0   |0   |5   |0   |0   |0   |0   |0   |0   |0   |0   |1   |0   |0
Contrast-23        	    |0   |4   |0   |0   |0   |0  |247  |0   |19  |0   |1   |0   |0   |0   |0   |0
Pragmatic Contrast-26	|0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0
Concession-27      	    |0   |0   |0   |0   |0   |0   |8   |0   |14  |0   |0   |0   |0   |1   |0   |0
Pragmatic Concession-30	|0   |0   |0   |0   |0   |0   |4   |0   |0   |0   |0   |0   |0   |0   |0   |0
Conjunction-32     	    |2   |0   |0   |0   |0   |0   |2   |0   |0   |0  |209  |0   |0   |0   |0   |0
Instantiation-33   	    |0   |0   |0   |0   |0   |0   |0   |0   |0   |0  | 0   |21  |0   |0   |0   |0
Restatement-34     	    |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |5   |0   |2   |0   |0   |0
Alternative-38     	    |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |1   |7   |0   |0
Exception-42       	    |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |0
List-43            	    |0   |0   |0   |0   |0   |0   |0   |0   |0   |0   |29  |0   |0   |0   |0   |0




###### traits = POS Word Contxt

                |	prec.  |	rec.  |	f1    |	support
----------------|----------|----------|-------|-------
Asynchronous-2     	|94.3 	|83.0 	|88.3 	|100.0
Synchronous-5      	|57.4 	|87.3 	|69.3 	|71.0
Cause-7            	|98.0 	|88.3 	|92.9 	|111.0
Pragmatic Cause-10 	|0.0  	|0.0  	|0.0  	|1.0
Condition-12       	|90.4 	|83.9 	|87.0 	|56.0
Pragmatic Condition	|0.0  	|0.0  	|0.0  	|7.0
Contrast-23        	|92.1 	|90.0 	|91.0 	|271.0
Pragmatic Contrast-	|0.0  	|0.0  	|0.0  	|0.0
Concession-27      	|33.3 	|43.5 	|37.7 	|23.0
Pragmatic Concessio	|0.0  	|0.0  	|0.0  	|4.0
Conjunction-32     	|85.7 	|98.1 	|91.5 	|213.0
Instantiation-33   	|100.0	|100.0	|100.0	|21.0
Restatement-34     	|100.0	|14.3 	|25.0 	|7.0
Alternative-38     	|80.0 	|100.0	|88.9 	|8.0
Exception-42       	|0.0  	|0.0  	|0.0  	|0.0
List-43            	|50.0 	|3.4  	|6.5  	|29.0
Micro-Avg          	|85.0 	|85.0 	|85.0 	|None
Macro-Avg          	|55.1 	|49.5 	|48.6 	|None
Weighted-Avg       	|84.9 	|85.0 	|83.7 	|None



is/as              	    |2	|5	|7	|10	  |12  |19	|23	 |26  |27  |30	|32	 |33   |34 |38	|42	 |43
------------------------|---|---|---|-----|----|----|----|----|----|----|----|-----|---|----|----|----
Asynchronous-2     	    |83 | 17|  0|   0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0  | 0 |  0 |  0 |  0
Synchronous-5      	    |2  | 62|  1|   0 |  0 |  0 |  6 |  0 |  0 |  0 |  0 |  0  | 0 |  0 |  0 |  0
Cause-7             	|1  | 12|  98|  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0  | 0 |  0 |  0 |  0
Pragmatic Cause-10 	    |0  | 1 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0  |0  | 0  | 0  | 0
Condition-12       	    |0  | 8 |  0 |  0 |  47|  1 |  0 |  0 |  0 |  0 |  0 |  0  | 0 |  0 |  0 |  0
Pragmatic Condition-19	|0  | 1 |  0 |  0 |  5 |  0 |  0 |  0 |  0 |  0 |  0 |  0  | 0 |  1 |  0 |  0
Contrast-23        	    |0  | 7 |  0 |  0 |  0 |  0 | 244|  0 |  19|  0 |  1 |  0  | 0 |  0 |  0 |  0
Pragmatic Contrast-26	|0  | 0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0  | 0 |  0 |  0 |  0
Concession-27      	    |2  | 0 |  1 |  0 |  0 |  0 |  10|  0 |  10|  0 |  0 |  0  | 0 |  0 |  0 |  0
Pragmatic Concession-30	|0  | 0 |  0 |  0 |  0 |  0 |  3 |  0 |  1 |  0 |  0 |  0  | 0 |  0 |  0 |  0
Conjunction-32     	    |0  | 0 |  0 |  0 |  0 |  0 |  2 |  0 |  0 |  0 | 209|  0  | 0 |  1 |  0 |  1
Instantiation-33   	    |0  | 0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  21 | 0 |  0 |  0 |  0
Restatement-34     	    |0  | 0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  6 |  0  | 1 |  0 |  0 |  0
Alternative-38     	    |0  | 0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0  | 0 |  8 |  0 |  0
Exception-42       	    |0  | 0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0  | 0 |  0 |  0 |  0
List-43            	    |0  | 0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  28|  0  | 0 |  0 |  0 |  1

