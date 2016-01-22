---
title: "PostgreSQL - CREATE CONSTRAINT TRIGGER: cas d'utilisation"
date: "2012-04-29"
tags: [PostgreSQL, plpgsql, "Base de données", Comptabilité]
---

> **Note**
>
> Le billet ci-dessous n'apportera probablement pas grand chose aux
> personnes qui baignent dans la conception des bases de données ou qui
> connaissent bien [PostgreSQL](http://www.postgresql.org). Elles
> considéreront peut être que ce qui est décrit ci-dessous est un cas
> rencontré et traité de multiples fois. Dans ce cas, si vous prenez la
> peine de lire l'article quand même, merci d'indiquer si vous avez une
> meilleure solution technique que celle proposée.

`CREATE TRIGGER` et `CREATE CONSTRAINT TRIGGER`
-----------------------------------------------

Les
[triggers](http://www.postgresql.org/docs/9.1/interactive/sql-createtrigger.html)
sont des outils formidables qui permettent de vérifier que certaines
contraintes sont bien respectées. Ces vérifications sont généralement
effectuées soit à chaque fois qu'une ligne est insérée, modifiée,
détruite (FOR EACH ROW) soit une seule fois par commande
(FOR EACH STATEMENT).

Cette fonctionnalité est bien adaptée à des contraintes qui doivent être
respectées après ou avant chaque opération élémentaire (INSERT,
UPDATE, DELETE).

Mais quid si la vérification de la contrainte n'a de sens qu'au terme
d'une transaction (ou plus généralement une fois plusieurs opérations
élémentaires menées à leur terme) ?

Prenons l'exemple d'une base de données comptable qui enregistre les
écritures dans une table entries. Les concepteurs, bien avisés et
renseignés, veulent impérativement
s'assurer qu'il soit impossible d'entrer dans la base des écritures
déséquilibrées (rappelons, ou indiquons au lecteur sans notions
comptables, qu'une des règles de base en matière de comptabilité
commerciale (en [partie
double](http://fr.wikipedia.org/wiki/Comptabilité_en_partie_double)) est
l'équilibre
[débit/crédit](http://fr.wikipedia.org/wiki/Comptabilité_en_partie_double#Principes_de_base)).

Dans cette hypothèse il est impossible d'utiliser les
[triggers](http://www.postgresql.org/docs/9.1/interactive/sql-createtrigger.html)
via une simple commande CREATE TRIGGER car la contrainte ne peut être
vérifiée qu'après que l'ensemble des écritures incluses dans la
transaction soit passé.

Même si l'on imposait comme contrainte aux développeurs d'applications
exploitant la base de données d'insérer leurs écritures par bloc dans
une seule commande (INSERT INTO entries(...) VALUES(...), (...),
...) - ce qui permettraient de poser un
[trigger](..%20_CONSTRAINT%20TRIGGER:) FOR EACH
STATEMENT - il resterait le problème des suppressions et des mises à
jour des écritures existantes. En effet, dans la mesure où chaque
écriture doit pouvoir rester indépendante [^1] des autres écritures
insérées en même temps qu'elle, il est impossible d'exécuter des
commandes UPDATE et DELETE concernant plusieurs lignes à la fois avec
des modifications différentes (on se prend à rêver ici d'un
UPDATE table (SET ...  WHERE ....), (SET ... WHERE ...),...).

Et c'est là que la commande CREATE CONSTRAINT TRIGGER [^2] prend tout
son sens: elle seule permet de repousser (via son option DEFERRABLE) la
vérification de la contrainte à la fin de transaction. On va pouvoir
laisser le développeur insérer, modifier et détruire les écritures qu'il
veut dans sa transaction et à la fin de celle-ci l'autoriser ou lui
refuser l'ensemble de ses modifications selon qu'elles préservent ou
détruisent l'équilibre
[débit/crédit](http://fr.wikipedia.org/wiki/Comptabilité_en_partie_double#Principes_de_base).

Équilibre comptable ? Oui, mais lequel ?
----------------------------------------

Avant de décrire la solution mise en oeuvre il reste encore à définir
précisément l'équilibre que l'on cherche à vérifier.

Pour cela vous avez probablement besoin d'en savoir un peu plus sur le
schéma de la base et notamment de sa table des écritures (entries)

Précisons par rapport à cette table que :

-   `book` est la colonne qui identifie une comptabilité (la base
    est multi-comptabilités)
-   `journal` est le code du [journal
    comptable](http://fr.wikipedia.org/wiki/Journal_(comptabilité))
-   `month` est le mois dans lequel est comptabilisée l'écriture
-   `folio` est le numéro du folio dans le journal (pour un mois donné
    un journal peut être composé de multiples folios
    d'écritures comptables)
-   `debit` est le montant débité
-   `credit` est le montant crédité

La contrainte de clef étrangère en ligne 18
`FOREIGN KEY(book, journal, month, folio)` montre que les écritures sont
*regroupées* par comptabilité, par journal, par mois, par folio. C'est à
ce niveau de *détail* que l'équilibre
[débit/crédit](http://fr.wikipedia.org/wiki/Comptabilité_en_partie_double#Principes_de_base)
doit être respecté (il ne doit pas être possible d'enregistrer des
écritures qui bien qu'équilibrées dans leur ensemble ne le seraient pas
au niveau de chaque folio concerné).

Cette nouvelle contrainte explique pourquoi on ne peut se contenter
d'une simple comparaison globale
(`SELECT sum(debit) = sum(credit) FROM entries`). Outre qu'elle ne
permet pas de s'assurer de l'équilibre au niveau de chaque folio (bien
sûr on pourrait résoudre ce point par regroupement -
GROUP BY book, journal, month, folio) elle pose des problèmes de
performances : il paraît difficilement acceptable de faire une sommation
de toutes les écritures de la base à chaque fois qu'une transaction
*touche* à la table des écritures. Dans un mode de fonctionnement multi
comptabilités on peut s'attendre à ce que la table `entries` contiennent
quelques millions d'écritures...

Il faut donc chercher à ne vérifier l'équilibre que des seules écritures
concernées par chaque transaction.

Mise en oeuvre
--------------

Récapitulons les contraintes posées ci-dessus :

-   le contrôle de l'équilibre doit intervenir pour chaque transaction,
    à la fin de celle-ci
-   le contrôle doit se faire au niveau de chaque folio et non
    globalement

Pour y parvenir nous allons :

-   poser des
    [triggers](http://www.postgresql.org/docs/9.1/interactive/sql-createtrigger.html)
    sur la table `entries` qui se contenteront de stocker les variations
    qui interviennent pendant la transaction sur les débits et crédits,
    folio par folio (en fait pour chaque ensemble unique
    `book/jounal/month/folio`). Précisons que chaque transaction doit
    être prise isolément. Pas question évidemment de tenir compte
    d'éventuelles modifications de la table `entries` qui seraient
    faites par des transactions concurrentes (cela doit fonctionner même
    si le développeur a paramétré la transaction en
    `SET TRANSACTION READ UNCOMMITED`).
-   poser un [trigger](..%20_CONSTRAINT%20TRIGGER:) qui se déclenchera
    en fin de transaction pour vérifier que l'équilibre des écritures de
    chaque folio est respecté. Dans le cas contraire une exception sera
    levée qui entraînera l'abandon de la transaction (la table `entries`
    restera dans l'état où elle était avant le début de la transaction).

Pour stocker les variations intervenant pendant une transaction dans les
écritures nous utiliserons la table suivante créée à cet effet dans le
schéma:

{{< highlight plpgsql "linenos=inline" >}}
-- working tables
-- to check balanced entries
CREATE TABLE check_entries(
      txid bigint NOT NULL
    , book varchar(16) NOT NULL
    , journal char(2) NOT NULL
    , month date NOT NULL
    , folio integer NOT NULL
    , delta_debit numeric(16, 2)
    , delta_credit numeric(16, 2)

    , PRIMARY KEY(txid, book, journal, month, folio)
);
{{< /highlight >}}

La clef primaire de la table `check_entries` (ligne 12) montre bien à
quel niveau les variations des débits et crédits sont conservés. Notons
également que `txid` est destiné à stocker le numéro de la transaction
et sera obtenu via la fonction
[txid\_current](http://www.postgresql.org/docs/9.1/static/functions-info.html#FUNCTIONS-TXID-SNAPSHOT)

> **note**
>
> Concernant cette table une alternative, un temps envisagé, aurait pu
> consister à créer la table comme temporaire à la session et la vider à
> chaque fin de transaction via la commande
> `CREATE TEMP TABLE check_entries(book...) ON COMMIT DELETE ROWS` mais
> cela a pour **énorme** inconvénient de reporter partie du contrôle
> d'équilibre sur le développeur d'applications sur qui repose alors la
> responsabilité de créer la table au début de chaque session. Ce que
> l'on veut précisément éviter en imposant à tous de manière
> transparente cet équilibre comptable fondamentale.

### Triggers en charge de stocker les variations des débits et crédits

Cette fois ci allons y, mettons les mains dans le code et commençons par
définir la fonction en charge de récupérer les variations en cours dans
la table \`entries\`:

{{< highlight plpgsql "linenos=inline" >}}
    -- check if entries are balanced
    CREATE OR REPLACE FUNCTION update_balance_entries()
        RETURNS trigger
        LANGUAGE plpgsql
    AS $update_balance_entries$
    <<block>>
    DECLARE
        book varchar(16);
        journal char(2);
        month date;
        folio integer;
        delta_debit numeric(16,2) := 0.0;
        delta_credit numeric(16,2) := 0.0;
    BEGIN
        IF (TG_OP = 'DELETE') THEN
            book = OLD.book;
            journal = OLD.journal;
            month = date_trunc('month', OLD."date");
            folio = OLD.folio;
            delta_debit = -1 * OLD.debit;
            delta_credit = -1 * OLD.credit;
        ELSIF (TG_OP = 'UPDATE') THEN
            book = NEW.book;
            journal = NEW.journal;
            month = date_trunc('month', NEW."date");
            folio = NEW.folio;
            delta_debit = NEW.debit - OLD.debit;
            delta_credit = NEW.credit - OLD.credit;
        ELSE
            book = NEW.book;
            journal = NEW.journal;
            month = date_trunc('month', NEW."date");
            folio = NEW.folio;
            delta_debit = NEW.debit;
            delta_credit = NEW.credit;
        END IF;

        UPDATE check_entries ce 
            SET delta_debit = ce.delta_debit + block.delta_debit,
                delta_credit = ce.delta_credit + block.delta_credit
            WHERE ce.txid = txid_current()
              AND ce.book = block.book
              AND ce.journal = block.journal
              AND ce.month = block.month
              AND ce.folio = block.folio;
        IF NOT FOUND THEN
            INSERT INTO check_entries VALUES
                (txid_current(), book, journal, month, folio,
                delta_debit, delta_credit);
            RAISE DEBUG 'add to check_entries(%, %, %, %, %, %, %)',
                txid_current(), book, journal, month, folio,
                delta_debit, delta_credit;
        ELSE
            RAISE DEBUG 'update check_entries(%, %, %, %, %, %, %)',
                txid_current(), book, journal, month, folio,
                delta_debit, delta_credit;
        END IF;

        RETURN NULL;
    END
    $update_balance_entries$;
{{< /highlight >}}

Des lignes 15 à 36 on récupère les variations des débits et crédits en
fonction de la nature de la modification en cours: INSERT, UPDATE ou
DELETE (via la variable spéciale TG\_OP).

Des lignes 38 à 57 on stocke les variations constatées dans la table
check\_entries.

Il ne reste plus qu'à rattacher la fonction update\_balance\_entries aux
[triggers](http://www.postgresql.org/docs/9.1/interactive/sql-createtrigger.html)
suivants:

{{< highlight plpgsql "linenos=inline" >}}
-- create triggers for checking balanced entries
CREATE TRIGGER a_update_balance_entries AFTER INSERT OR DELETE
    ON entries FOR EACH ROW EXECUTE PROCEDURE update_balance_entries();

CREATE TRIGGER a_update_entry AFTER UPDATE
    ON entries FOR EACH ROW WHEN (OLD.debit <> NEW.debit OR
    OLD.credit <> NEW.credit)
    EXECUTE PROCEDURE update_balance_entries();
{{< /highlight >}}

On distingue le cas des UPDATE de celui des INSERT/DELETE car en cas
d'UPDATE le contrôle ne nous intéresse que si les montants débit ou
crédit sont modifiés (ce qui est toujours vrai sur une insertion ou
destruction de lignes).

### Trigger de fin de transaction de contrôle de l'équilibre comptable

Tout est en place pour procéder au contôle final qui acceptera ou
rejetera la transaction. La fonction est définie comme suit:

{{< highlight plpgsql "linenos=inline" >}}
    CREATE OR REPLACE FUNCTION check_balance_entries()
        RETURNS trigger
        LANGUAGE plpgsql
    AS $check_balance_entries$
    DECLARE
        balanced boolean := true;
        r check_entries%ROWTYPE;
        msg varchar[];
    BEGIN
        FOR r IN SELECT * FROM check_entries 
            WHERE txid = txid_current()
              AND delta_debit - delta_credit <> 0.0 LOOP
            balanced := false;
            msg := msg || ('Ecritures non équilibrées pour la compta ''' 
                            || r.book 
                            || ''' journal ''' || r.journal
                            || ''' mois ' || to_char(r.month, 'MM/YYYY')
                            || ''' folio ' || r.folio
                            || ' écart de ' 
                            || r.delta_debit - r.delta_credit)::varchar;
        END LOOP;
        DELETE FROM check_entries WHERE txid = txid_current();
        if not balanced THEN
            RAISE EXCEPTION '%', array_to_string(msg, E'\n');
        END IF;

        RETURN NULL;
    END
    $check_balance_entries$;
{{< /highlight >}}

On récupère les entrées de la table check\_entries non équilibrées pour
la transaction courante et si il existe au moins une entrée (balanced
est False) on lève une exception (RAISE EXCEPTION) qui aura pour
conséquence l'abandon de la transaction courante.

Notons également qu'un autre effet de bord bénéfique de l'annulation de
la transation est que les modifications faites par les
[triggers](http://www.postgresql.org/docs/9.1/interactive/sql-createtrigger.html)
dans la table check\_entries pendant le déroulement de la transaction
sont elles même annulées. Ce qui a pour conséquence de laisser la table
check\_entries en permanence propre/vide.

Posons pour finir le CONSTRAINT TRIGGER\_ sur lequel tout le mécanisme
repose:

{{< highlight plpgsql "linenos=inline" >}}
-- at the end of transaction check balanced entries
CREATE CONSTRAINT TRIGGER z_check_balance_entries
    AFTER INSERT OR UPDATE OR DELETE
    ON entries 
    DEFERRABLE INITIALLY DEFERRED
    FOR EACH ROW EXECUTE PROCEDURE check_balance_entries();
{{< /highlight >}}

2 choses fondamentales dans cette commande:

-   le CONSTRAINT TRIGGER qui indique que nous avons à faire à une
    variante particulière d'un [trigger](..%20_CONSTRAINT%20TRIGGER:).
-   le DEFERRABLE INITIALLY DEFERRED qui indique à
    [PostgreSQL](http://www.postgresql.org) que la contrainte est non
    seulement différable à la fin de la transaction (DEFERRABLE) mais
    l'est par défaut (INITIALLY DEFERRED).

Voilà, tout est en place, la base est normalement à l'abri d'un
déséquilibre qui si il était possible serait lourd de conséquence
(comptabilité sans valeur, cause de rejet en cas de contrôle fiscal,
etc...).

[^1]: Pendant la rédaction de ce billet est paru [cet
    article](http://philsorber.blogspot.fr/2012/04/denormalizing-real-world-to-normalize.html)
    , en anglais. Bien que son sujet soit la
    normalisation/dénormalisation il prend également pour exemple le cas
    d'une base comptable et évoque le problème de l'équilibre des
    écritures
    [débit/crédit](http://fr.wikipedia.org/wiki/Comptabilité_en_partie_double#Principes_de_base).
    Il y'aurait des commentaires à faire sur cet article sur la manière
    de voir l'équilibre
    [débit/crédit](http://fr.wikipedia.org/wiki/Comptabilité_en_partie_double#Principes_de_base).
    Cela fera, peut être, l'objet d'un autre article (en français, mon
    niveau d'anglais ne m'autorisant pas à répondre dans cette langue).

[^2]: avant PostgreSQL 9.1 `CREATE CONSTRAINT TRIGGER` était une
    commande à part de `CREATE TRIGGER`. Attention donc si vous cherchez
    dans la documentation cette fonctionnalité dans les versions
    antérieures.
