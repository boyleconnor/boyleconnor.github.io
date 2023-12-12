# Scikit-Learn's F-1 calculator is broken

TL;DR: if you are using scikit-learn 1.3.X and use `f1_score()` or `classification_report()` and the
parameter `zero_division` is set to anything other than `0.0` or `"warn"`, then there's a very good chance that the
output of that function is wrong (possibly by any amount up to 100%, depending on the number of classes in your
dataset). E.g.:

```
>>> sklearn.__version__
'1.2.2'
>>> sklearn.metrics.f1_score(list(range(104)), list(range(100)) + [101, 102, 103, 104], average='macro', zero_division=1.0)
0.9523809523809523
```

vs. (the exact same expression)

```
>>> sklearn.__version__
'1.3.0'
>>> sklearn.metrics.f1_score(list(range(104)), list(range(100)) + [101, 102, 103, 104], average='macro', zero_division=1.0)
0.9809523809523809
```

(!!!)

## A classification problem

Our friends Ash, Misty, Brock, and others have photographed all 151 original Pokémon for us, and we're going to use that
nice gold-labeled dataset to train and evaluate some classifiers. Actually, we're going to skip ahead to after we've
already trained some classifiers, and just evaluate them.

Let's load up our dataset and take a peek:

```
>>> dataset = pandas.read_csv("pokemon_dataset.csv", index_col=0)
>>> dataset
      photographer    pokemon classifier_1_prediction
0            Misty    Weezing                 Weezing
1            Brock    Kingler                 Kingler
2            Brock      Doduo                   Ditto
3    Officer Jenny  Growlithe               Growlithe
4            Brock      Hypno                  Horsea
..             ...        ...                     ...
759      Nurse Joy     Staryu                  Staryu
760  Officer Jenny      Ditto                 Flareon
761            Ash   Cloyster                Cloyster
762            Ash     Golbat                  Golbat
763          Brock  Ninetales               Ninetales

[764 rows x 4 columns]
```

let's see, we have the human-annotated gold label Pokemon name under the `pokemon` column, then some predictions from
a classifier (let's say it's a vision transformer). We also recorded who took what picture in the `photographer` column;
we'll look into using that information later.

First, let's just try to get some idea of our performance. Since it's not a balanced dataset, accuracy is no good; we
should use F-score. And, rather than foolishly attempt to implement it ourselves, let's just use the implementation from
good ol' Scikit-Learn version 1.2.2 (start warming up your scrolling finger):

```
>>> print(classification_report(pokemon_dataset["pokemon"], pokemon_dataset["classifier_1_prediction"]))

              precision    recall  f1-score   support

        Abra       0.38      0.75      0.50         4
  Aerodactyl       0.75      0.33      0.46         9
    Alakazam       0.50      0.50      0.50         4
       Arbok       0.50      0.57      0.53         7
    Arcanine       0.75      0.75      0.75         8
    Articuno       1.00      0.67      0.80         3
    Beedrill       0.67      0.73      0.70        11
  Bellsprout       0.43      0.43      0.43         7
   Blastoise       0.50      0.33      0.40         6
   Bulbasaur       0.33      0.33      0.33         3
  Butterfree       0.60      1.00      0.75         6
    Caterpie       0.60      0.75      0.67         4
     Chansey       0.67      0.50      0.57         4
   Charizard       0.62      0.83      0.71         6
  Charmander       0.40      0.40      0.40         5
  Charmeleon       0.67      1.00      0.80         4
    Clefable       0.50      1.00      0.67         1
    Clefairy       0.78      1.00      0.88         7
    Cloyster       0.57      0.57      0.57         7
      Cubone       0.88      0.88      0.88         8
     Dewgong       0.50      0.80      0.62         5
     Diglett       0.80      0.50      0.62         8
       Ditto       0.43      0.75      0.55         4
      Dodrio       1.00      0.25      0.40         4
       Doduo       0.67      0.67      0.67         6
   Dragonair       0.33      0.50      0.40         6
   Dragonite       0.56      0.71      0.63         7
     Dratini       0.00      0.00      0.00         3
     Drowzee       0.67      0.67      0.67         6
     Dugtrio       0.67      0.67      0.67         3
       Eevee       0.50      0.50      0.50         2
       Ekans       0.83      0.71      0.77         7
  Electabuzz       1.00      1.00      1.00         7
   Electrode       0.56      0.62      0.59         8
   Exeggcute       0.50      0.20      0.29         5
   Exeggutor       0.00      0.00      0.00         1
  Farfetch'd       0.25      0.50      0.33         2
      Fearow       0.50      1.00      0.67         2
     Flareon       0.80      0.57      0.67         7
      Gastly       0.25      0.50      0.33         4
      Gengar       0.00      0.00      0.00         1
     Geodude       1.00      0.67      0.80         3
       Gloom       0.57      0.80      0.67         5
      Golbat       0.50      0.33      0.40         3
     Goldeen       1.00      0.33      0.50         3
     Golduck       0.50      1.00      0.67         1
       Golem       0.33      1.00      0.50         2
    Graveler       0.33      0.25      0.29         4
      Grimer       0.67      0.33      0.44         6
   Growlithe       1.00      0.83      0.91         6
    Gyarados       0.75      0.38      0.50         8
     Haunter       0.50      1.00      0.67         1
  Hitmonchan       0.60      0.60      0.60         5
   Hitmonlee       0.50      0.25      0.33         4
      Horsea       0.80      0.80      0.80         5
       Hypno       0.50      0.50      0.50         4
     Ivysaur       0.20      0.20      0.20         5
  Jigglypuff       0.83      0.83      0.83         6
     Jolteon       0.67      1.00      0.80         2
        Jynx       0.44      0.80      0.57         5
      Kabuto       0.67      0.75      0.71         8
    Kabutops       0.38      0.75      0.50         4
     Kadabra       0.60      1.00      0.75         3
      Kakuna       0.20      0.33      0.25         3
  Kangaskhan       1.00      0.67      0.80         3
     Kingler       0.75      1.00      0.86         6
     Koffing       0.67      0.40      0.50         5
      Krabby       0.33      0.25      0.29         4
      Lapras       0.64      0.78      0.70         9
   Lickitung       0.14      0.50      0.22         2
     Machamp       1.00      0.50      0.67         2
     Machoke       1.00      1.00      1.00         5
      Machop       0.67      0.86      0.75         7
    Magikarp       1.00      0.33      0.50         6
      Magmar       0.50      0.67      0.57         3
   Magnemite       0.75      1.00      0.86         3
    Magneton       0.60      0.75      0.67         4
      Mankey       1.00      0.67      0.80         6
     Marowak       0.25      0.50      0.33         2
      Meowth       0.50      1.00      0.67         4
     Metapod       1.00      0.67      0.80         3
         Mew       0.40      1.00      0.57         2
      Mewtwo       0.75      1.00      0.86         3
     Moltres       0.71      0.83      0.77         6
    Mr. Mime       0.83      0.50      0.62        10
         Muk       0.67      1.00      0.80         4
    Nidoking       1.00      0.50      0.67         2
   Nidoqueen       0.60      0.50      0.55         6
    Nidoran♀       0.67      0.75      0.71         8
    Nidoran♂       0.43      0.75      0.55         4
    Nidorina       0.83      0.56      0.67         9
    Nidorino       0.25      0.14      0.18         7
   Ninetales       1.00      1.00      1.00         1
      Oddish       1.00      0.75      0.86         4
     Omanyte       1.00      0.75      0.86         4
     Omastar       0.60      1.00      0.75         3
        Onix       0.25      0.50      0.33         2
       Paras       1.00      0.50      0.67         6
    Parasect       0.50      0.75      0.60         4
     Persian       0.50      0.60      0.55         5
     Pidgeot       0.50      0.67      0.57         6
   Pidgeotto       0.33      0.20      0.25         5
      Pidgey       1.00      0.33      0.50         3
     Pikachu       0.57      0.80      0.67         5
      Pinsir       0.80      0.57      0.67         7
     Poliwag       0.83      0.71      0.77         7
   Poliwhirl       1.00      1.00      1.00         3
   Poliwrath       0.50      0.33      0.40         3
      Ponyta       0.50      0.40      0.44         5
     Porygon       0.60      0.50      0.55         6
    Primeape       0.50      0.14      0.22         7
     Psyduck       0.67      1.00      0.80         6
      Raichu       1.00      0.83      0.91         6
    Rapidash       1.00      0.50      0.67         6
    Raticate       0.50      0.50      0.50         6
     Rattata       0.50      0.40      0.44         5
      Rhydon       0.80      0.80      0.80         5
     Rhyhorn       0.67      0.67      0.67         6
   Sandshrew       0.33      0.33      0.33         3
   Sandslash       0.86      0.55      0.67        11
     Scyther       0.57      0.67      0.62         6
      Seadra       0.33      0.25      0.29         4
     Seaking       0.70      0.78      0.74         9
        Seel       0.75      0.60      0.67         5
    Shellder       0.75      0.60      0.67         5
     Slowbro       0.80      0.80      0.80         5
    Slowpoke       0.83      0.62      0.71         8
     Snorlax       0.50      0.50      0.50         4
     Spearow       0.25      0.33      0.29         3
    Squirtle       1.00      0.50      0.67         8
     Starmie       0.83      0.83      0.83         6
      Staryu       1.00      0.62      0.77         8
     Tangela       0.62      0.62      0.62         8
      Tauros       0.50      0.60      0.55         5
   Tentacool       0.67      0.67      0.67         3
  Tentacruel       0.50      0.50      0.50         4
    Vaporeon       0.67      0.57      0.62         7
    Venomoth       0.50      0.67      0.57         3
     Venonat       0.86      0.86      0.86         7
    Venusaur       0.80      0.67      0.73        12
  Victreebel       0.50      0.60      0.55         5
   Vileplume       0.83      0.83      0.83         6
     Voltorb       1.00      0.44      0.62         9
      Vulpix       0.25      1.00      0.40         1
   Wartortle       0.33      0.25      0.29         4
      Weedle       0.50      0.33      0.40         3
  Weepinbell       0.71      0.71      0.71         7
     Weezing       0.60      0.86      0.71         7
  Wigglytuff       1.00      0.50      0.67         8
      Zapdos       0.71      0.83      0.77         6
       Zubat       0.50      0.40      0.44         5

    accuracy                           0.62       764
   macro avg       0.63      0.63      0.60       764
weighted avg       0.67      0.62      0.62       764

/usr/local/lib/python3.10/dist-packages/sklearn/metrics/_classification.py:1344: UndefinedMetricWarning: Precision and F-score are ill-defined and being set to 0.0 in labels with no predicted samples. Use `zero_division` parameter to control this behavior.
  _warn_prf(average, modifier, msg_start, len(result))
/usr/local/lib/python3.10/dist-packages/sklearn/metrics/_classification.py:1344: UndefinedMetricWarning: Precision and F-score are ill-defined and being set to 0.0 in labels with no predicted samples. Use `zero_division` parameter to control this behavior.
  _warn_prf(average, modifier, msg_start, len(result))
/usr/local/lib/python3.10/dist-packages/sklearn/metrics/_classification.py:1344: UndefinedMetricWarning: Precision and F-score are ill-defined and being set to 0.0 in labels with no predicted samples. Use `zero_division` parameter to control this behavior.
  _warn_prf(average, modifier, msg_start, len(result))
```

Wow, that's a lot of Pokémon (and just think, kids these days have even more!). Overall, our classifier looks... okay, I
suppose? I mean, it's certainly a lot better than random but there's quite a bit of room for improvement.

Anyway, what are those weird warnings at the bottom? It says something about precision and F-score being undefined due
to a division by zero. Hmmm... looking at the classification report, it looks like our classifier completely failed on
gold examples of Dratini, Exeggutor, and Gengar:

```
              precision    recall  f1-score   support
     Dratini       0.00      0.00      0.00         3
   Exeggutor       0.00      0.00      0.00         1
      Gengar       0.00      0.00      0.00         1
```

But when we look at the gold labels and predictions we notice that our classifier actually _never_ labels anything as
"Dratini":

```
>>> (pokemon_dataset["classifier_1_prediction"] == "Exeggutor").any()
True
>>> (pokemon_dataset["classifier_1_prediction"] == "Gengar").any()
True
>>> (pokemon_dataset["classifier_1_prediction"] == "Dratini").any()
False
```

A wise documentation page once told me, "precision is, intuitively, the ability of a classifier to not label as positive
a sample that is negative". So, in a very meaningful sense, our classifier is actually perfectly precise.

TO BE CONTINUED


<!--
    Joke: "The Mankey's paw has granted our wish"
-->