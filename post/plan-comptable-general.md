---
title: "Le Plan Comptable Général sous format exploitable"
date: "2016-01-22"
tags: [plan-comptable-general, pcg, comptabilité, tableur, csv, libreoffice, excel]
---

Après avoir tenté vainement de trouver sur le Net le plan comptable
général français sous forme de fichier texte brut (ou un autre format
exploitable: csv, etc...)  j'ai fini par me rendre à l'évidence:
impossible de trouver cette ressource aussi incroyable que cela puisse
paraître ![^1]

Il y a bien quelques fichiers mais ceux que j'ai trouvé ont tous des
limitations: soit que le fichier est protégé, soit qu'il est incomplet,
pas mis à jour, etc...

Bref, je me suis résolu à constituer ce fichier moi-même à partir de la
source officielle située sur le site de
l'[ANC](http://www.anc.gouv.fr/cms/sites/anc/accueil.html) et plus
exactement à partir du [réglement n°2014-03 du 5 juin
2014](https://frama.link/Plan-Comptable-General) dans lequel figure la
liste officielle des comptes (pages 124 et suivantes).

Hélas, ce document est en pdf. Après avoir appliqué les traitements ci-dessous
j'ai enfin pu aboutir à ce que je cherchais depuis le début à savoir un
document tableur ('_ods_' cela va sans dire)  comportant les 3 colonnes de
base: le numéro de compte, son libellé et à quel système il est rattaché
(abrégé, base ou développé).

**Vous pouvez récupérer ce fichier <a href='/files/pcg.ods'>ici</a>.**
Vous pouvez l'utiliser comme bon vous semble. Enjoy !

À titre de documentation personnelle et pour les personnes qui voudraient à
l'avenir recommencer le processus voici comment je m'y suis pris (**sous Linux
évidemment**, pour les autres plates-formes je vous laisse adapter):

* Récupération du [pdf](https://frama.link/Plan-Comptable-General)
* Conversion en texte:

~~~
# de la page 124 à 143 où se situe le plan comptable
$ pdftotext -f 124 -l 143 Plan\ comptable\ general.pdf
~~~

À ce stade le fichier texte ressemble à cela:

~~~
Section 2 – Plan de comptes général
Art. 932-1
Le plan de comptes, visé à l'article 911-5 et présenté ci-après, est commun au système de base, au système
abrégé et au système développé. Les comptes utilisés dans chaque système sont distingués de la façon
suivante :
 système de base : comptes imprimés en caractères normaux,
 système abrégé : comptes imprimés en caractères gras exclusivement,
 système développé : comptes du système de base et comptes imprimés en caractères italiques.
Classe 1 : Comptes de capitaux
10 - Capital et réserves
101 – Capital
1011 - Capital souscrit - non appelé
1012 - Capital souscrit - appelé, non versé
1013 - Capital souscrit - appelé, versé
10131 - Capital non amorti
10132 - Capital amorti
...
~~~

* Manipulations sous [vim](http://www.vim.org/):

~~~~
$ vim Plan\ comptable\ general.txt
~~~~

1) Suppression des ^L intempestifs en début de ligne

~~~
:%s/^L/
~~~

2) Suppression de l'ensemble des lignes ne commençant pas par un chiffre:

~~~
:%s/^\D.*$/
~~~

3) Suppression des lignes vides

~~~
:g/^$/d
~~~

Après ces manipulations les lignes restantes ressemblent à ceci:

~~~
10 - Capital et réserves
101 – Capital
1011 - Capital souscrit - non appelé
1012 - Capital souscrit - appelé, non versé
1013 - Capital souscrit - appelé, versé
...
~~~

4) Remplacement des caractères situés entre les numéros de compte et les
libellés par une tabulation (cela facilitera l'import dans le tableur):

~~~
:%s/^\(\d\+\)\W\+\(\w.\+\)$/\1\t\2/
~~~

À ce stade, aprés enregistrement, vous pouvez récupérer le fichier dans un
tableur en suivant sa procédure d'import: les données sont séparées par des
tabulations.

Vous pouvez déjà exploiter le fichier en l'état mais il manque
encore l'information du type de système auquel est rattaché chaque
compte. Cette information figure dans le fichier pdf d'origine mais sous
forme de police de caractère (les comptes relevant du système abrégé
sont en gras, ceux du système de base en caractères normaux et ceux du
système développé en italiques). Où l'on voit le manque de culture
informatique du normalisateur... ;-)

Pour cette dernière étape je n'ai pas trouvé mieux comme solution
hélas que de saisir l'information moi même dans une colonne
dédiée... Ce fut le gros du boulot...


[^1]: Mesdames, Messieurs les normalisateurs, si vous me lisez un jour,
      pitié mettez à disposition une liste des comptes officielle sous un
      format exploitable, libre et documenté ! Un simple fichier csv fera
      l'affaire. Merci d'avance.
