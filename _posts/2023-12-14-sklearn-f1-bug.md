---
title: "Scikit-Learn's F-1 calculator is broken"
author: "Connor Boyle"
---

TL;DR: if you are using scikit-learn 1.3.X and use `f1_score()` or `classification_report()` and the
parameter `zero_division` is set to anything other than `0.0` or `"warn"`, then there's a very good chance that the
output of that function is wrong (possibly by any amount up to 100%, depending on the number of classes in your
dataset). E.g.:

```
>>> sklearn.__version__
'1.3.0'
>>> sklearn.metrics.f1_score(list(range(104)), list(range(100)) + [101, 102, 103, 104], average='macro', zero_division=1.0)
0.9809523809523809
>>> # (^incorrect)
```

compare to (the exact same expression):

```
>>> sklearn.__version__
'1.2.2'
>>> sklearn.metrics.f1_score(list(range(104)), list(range(100)) + [101, 102, 103, 104], average='macro', zero_division=1.0)
0.9523809523809523
>>> # (^correct)
```

The bug also affects the just-introduced `zero_division=np.nan`:

```
>>> sklearn.metrics.f1_score([0, 1], [1, 0], average='macro', zero_division=np.nan)
nan
>>> # (^should be 0.0)
>>> sklearn.metrics.f1_score([0, 1, 2], [1, 0, 2], average='macro', zero_division=np.nan)
1.0
>>> # (^should be ~0.67)
```

Both myself and the Scikit-Learn maintainers consider the behavior in 1.3.X to be incorrect. While a
[pull request](https://github.com/scikit-learn/scikit-learn/pull/27577) to fix this behavior was just merged, the fix
has not yet shipped on any released version of Scikit-Learn. Therefore, the easiest solution to this specific problem is
to revert to Scikit-Learn 1.2.2, or use `zero_division=0.0` if possible, while being careful to understand how this
parameter change will affect precision, recall, & F-1 (see below for an explainer on the purpose and function of
the `zero_division` parameter).

## A classification problem

In case you're wondering what the `zero_division` parameter is supposed to do, we'll go through an imaginary multi-class
classification problem. In an effort to try to keep it simple and accessible, I've made this hypothetical machine
learning project about classifying pictures of Pokémon, a type of fictional creature found in the
[eponymous Japanese media franchise](https://en.wikipedia.org/wiki/Pok%C3%A9mon). The issues in the following example
could come up in any multi-class classification problem.

### F-1, I choose you!

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
a sample that is negative". So, in a very meaningful sense, our classifier is actually perfectly precise at identifying
Dratini; it never tries to tell us that any other Pokémon is Dratini. Let's set the parameter mentioned in the warning
message (I'll trim down the output to just a few rows that we're interested in:

```
>>> print(classification_report(pokemon_dataset["pokemon"], pokemon_dataset["classifier_1_prediction"], zero_division=1.0))
              precision    recall  f1-score   support

     Dratini       1.00      0.00      0.00         3
   Exeggutor       0.00      0.00      0.00         1
      Gengar       0.00      0.00      0.00         1

    accuracy                           0.62       764
   macro avg       0.64      0.63      0.60       764
weighted avg       0.67      0.62      0.62       764
```

Nice, now the classification report correctly shows our classifier's precision on Dratini as 100%. As it happens, this
has brought our classifier's macro average precision up from 63% to 64%. A small difference, but we had to get it right
in the first place to know that.

### Dealing with unrepresented classes

Now that I think about it, that Brock guy never seems to open his eyes. Maybe his photos of Pokémon are out-of-focus,
overexposed, or don't even have the right Pokemon in frame. If that's the case, then our classifier might be doing way
better overall, but is getting dragged down by Brock's low-quality data. Let's break down those classification reports
by photographer:

```
>>> for photographer in set(pokemon_dataset["photographer"]):
...    subset = pokemon_dataset[pokemon_dataset["photographer"] == photographer]
...    print(photographer)
...    print(classification_report(subset["pokemon"], subset["classifier_1_prediction"], zero_division=1.0))
```

Hmmm, something seems wrong with the output. I'll just show the first few rows of each the classification report for
each photographer's subset of the data:

```
Nurse Joy
              precision    recall  f1-score   support

  Aerodactyl       1.00      0.00      0.00         1
    Alakazam       1.00      1.00      1.00         1
       Arbok       1.00      1.00      1.00         2
         ...        ...       ...       ...       ...
Brock
              precision    recall  f1-score   support

        Abra       0.25      1.00      0.40         1
  Aerodactyl       1.00      0.50      0.67         4
    Alakazam       0.00      1.00      0.00         0
         ...        ...       ...       ...       ...
Professor Oak
              precision    recall  f1-score   support

  Aerodactyl       1.00      0.33      0.50         3
    Alakazam       1.00      0.00      0.00         2
       Arbok       1.00      0.00      0.00         1
         ...        ...       ...       ...       ...
Officer Jenny
              precision    recall  f1-score   support

        Abra       0.00      1.00      0.00         0
       Arbok       0.33      1.00      0.50         1
    Beedrill       1.00      1.00      1.00         1
         ...        ...       ...       ...       ...
Ash
              precision    recall  f1-score   support

        Abra       1.00      0.50      0.67         2
    Alakazam       1.00      1.00      1.00         1
       Arbok       0.50      1.00      0.67         1
         ...        ...       ...       ...       ...
Misty
              precision    recall  f1-score   support

        Abra       0.50      1.00      0.67         1
  Aerodactyl       0.00      0.00      0.00         1
       Arbok       0.00      0.00      0.00         2
         ...        ...       ...       ...       ...
```

Hmmm, they all have different Pokémon... but why? Well, for example Nurse Joy didn't take any pictures of Abra, so Abra
doesn't show up on the classification report for her subset of the photo classification data, skipping straight to
Aerodactyl. So, the number of classes for each classification report is equal to the number of classes (unique Pokémon)
photographed by each photographer, right? Wait, no, Brock didn't take any pictures of Alakazam (the `support` is 0), but
it still shows up on the list. That's because the classifier misclassified some other Pokémon.

So, is this bad? Well, it does seem kind of inconsistent. Notice how our classifier misidentified (at least one of)
Officer Jenny's photographs as Abra. That means the classifier's 0.0 precision on the Abra class is now factored into
the average precision for the subset (makes sense). But now the classifier gets considered to have a recall of 1.0 on
that class.

I mean, it did correctly identify 100% of the 0 gold-label Abra photos correctly as Abra (granted, it would be
impossible for a classifier to fail that task). The really weird thing to think about is what would happen to average
recall on this subset if the classifier hadn't misidentified any of our photos as an Abra. The recall on the correct
Pokémon would go up by several points, but we would lose this entry (Abra) with a 1.0 recall. So, if our classifier did
slightly better at classifying that Pokémon, our macro average recall would probably go down! That's weird.

We can fix this by using a fixed set of labels for every classification report:

```
>>> for photographer in set(pokemon_dataset["photographer"]):
...     subset = pokemon_dataset[pokemon_dataset["photographer"] == photographer]
...     print(photographer)
...     print(classification_report(subset["pokemon"], subset["classifier_1_prediction"], labels=sorted(set(pokemon_dataset["pokemon"])), zero_division=1.0))
```

Shortened output:

```
Nurse Joy
              precision    recall  f1-score   support

        Abra       1.00      1.00      1.00         0
  Aerodactyl       1.00      0.00      0.00         1
    Alakazam       1.00      1.00      1.00         1
         ...        ...       ...       ...       ...
Brock
              precision    recall  f1-score   support

        Abra       0.25      1.00      0.40         1
  Aerodactyl       1.00      0.50      0.67         4
    Alakazam       0.00      1.00      0.00         0
         ...        ...       ...       ...       ...
Professor Oak
              precision    recall  f1-score   support

        Abra       1.00      1.00      1.00         0
  Aerodactyl       1.00      0.33      0.50         3
    Alakazam       1.00      0.00      0.00         2
         ...        ...       ...       ...       ...
Officer Jenny
              precision    recall  f1-score   support

        Abra       0.00      1.00      0.00         0
  Aerodactyl       1.00      1.00      1.00         0
    Alakazam       1.00      1.00      1.00         0
         ...        ...       ...       ...       ...
Ash
              precision    recall  f1-score   support

        Abra       1.00      0.50      0.67         2
  Aerodactyl       1.00      1.00      1.00         0
    Alakazam       1.00      1.00      1.00         1
         ...        ...       ...       ...       ...
Misty
              precision    recall  f1-score   support

        Abra       0.50      1.00      0.67         1
  Aerodactyl       0.00      0.00      0.00         1
    Alakazam       1.00      1.00      1.00         0
         ...        ...       ...       ...       ...
```

Oh, that's interesting! Look at the `f1-score` for the classes that used to be left of the reports entirely (e.g. "Abra"
under Nurse Joy). The classifier has 100% precision (it made no positive predictions for the class) and a 100% recall
(the class has no gold-label positives, so it gets that "for free") so therefore the classifier gets a perfect F-1 for
each of those classes. Is this the behavior we want? I think so. If we think of F-1 like a sort of test or trial for a
classifier, we're giving the classifier an opportunity to get every class "right" or "wrong"; in the case of the
unrepresented classes, it can either make zero positive predictions and get 100% F-1, or predict some non-zero number of
the class and get 0% F-1. Notice that just including the full set of labels can affect the macro average F-1 scores:

```
>>> records = []
>>> for photographer in set(pokemon_dataset["photographer"]):
...     subset = pokemon_dataset[pokemon_dataset["photographer"] == photographer]
...     records.append({
...         'photographer': photographer,
...         'f1_no_labels': sklearn.metrics.f1_score(subset["pokemon"], subset["classifier_1_prediction"], zero_division=1.0, average="macro"),
...         'f1_with_labels': sklearn.metrics.f1_score(subset["pokemon"], subset["classifier_1_prediction"], labels=sorted(set(pokemon_dataset["pokemon"])), zero_division=1.0, average="macro"),
...         'total_photographs': len(subset)
...     })
... 
>>> print(pandas.DataFrame.from_records(records))
    photographer  f1_no_labels  f1_with_labels  total_photographs
0      Nurse Joy      0.537607        0.761148                 76
1          Brock      0.580801        0.694623                159
2  Professor Oak      0.510944        0.646973                153
3  Officer Jenny      0.440639        0.729581                 69
4            Ash      0.465969        0.589751                166
5          Misty      0.461817        0.611511                141
```

Alternatively, we could decide that we want unrepresented classes to always give a score of 0.0, by
setting `zero_division=0.0`. Predictably, this will drag down the macro average F-1 by a lot in our subsets (I'll just
show the report for the classifier on Nurse Joy's photographs):

```
>>> subset = pokemon_dataset[pokemon_dataset["photographer"] == "Nurse Joy"]
>>> print(sklearn.metrics.classification_report(subset["pokemon"], subset["classifier_1_prediction"], labels=sorted(set(pokemon_dataset["pokemon"])), zero_division=1.0))
              precision    recall  f1-score   support

        Abra       1.00      1.00      1.00         0
  Aerodactyl       1.00      0.00      0.00         1
    Alakazam       1.00      1.00      1.00         1
         ...        ...       ...       ...       ...

   micro avg       0.70      0.70      0.70        76
   macro avg       0.86      0.88      0.76        76
weighted avg       0.95      0.70      0.71        76

>>> # ^zero_division=1.0
>>> print(sklearn.metrics.classification_report(subset["pokemon"], subset["classifier_1_prediction"], labels=sorted(set(pokemon_dataset["pokemon"])), zero_division=0.0))
              precision    recall  f1-score   support

        Abra       0.00      0.00      0.00         0
  Aerodactyl       0.00      0.00      0.00         1
    Alakazam       1.00      1.00      1.00         1
         ...        ...       ...       ...       ...

   micro avg       0.70      0.70      0.70        76
   macro avg       0.29      0.28      0.28        76
weighted avg       0.77      0.70      0.71        76

>>> # ^zero_division=0.0
```

That seems like a clearly worse way to evaluate the classifier on a subset; its F-1 will have a ceiling equal to the
share of unique Pokémon present in that subset's gold labels. For example, Nurse Joy only photographed 60 unique
Pokémon, so the best possible F-1 the classifier could get on the subset is equal to $$\frac{60}{151} \approx 0.397$$.
Perhaps more importantly, there is now no distinction between an unrepresented class that the classifier successfully
avoids and a class to which the classifier assigns other Pokemon's photos--both will get a 0%.

We started this section to figure out whether one subset of our photographs was dragging down the performance
significantly. Going by the reports generated with a full label set and `zero_division=1.0`, Brock's photos appear to be
fine, with the classifier earning a middle-of-the-pack F-1 of ~69.5%. I guess classifier #1 was just kind of mediocre.

### A few months later...

Months have now passed, machine learning has advanced at the usual breakneck speed. Our team has trained a new
classifier and helpfully joined its predictions onto our existing dataset, next to the predictions from classifier #1:

```
>>> pokemon_dataset
     Unnamed: 0   photographer    pokemon classifier_1_prediction classifier_2_prediction
0             0          Misty    Weezing                 Weezing                 Weezing
1             1          Brock    Kingler                 Kingler                 Kingler
2             2          Brock      Doduo                   Ditto                   Doduo
3             3  Officer Jenny  Growlithe               Growlithe                 Snorlax
4             4          Brock      Hypno                  Horsea                   Hypno
..          ...            ...        ...                     ...                     ...
759         759      Nurse Joy     Staryu                  Staryu                 Venonat
760         760  Officer Jenny      Ditto                 Flareon                   Ditto
761         761            Ash   Cloyster                Cloyster                 Drowzee
762         762            Ash     Golbat                  Golbat                  Kakuna
763         763          Brock  Ninetales               Ninetales               Tentacool

[764 rows x 5 columns]
```

We spent all that time figuring out the intricacies of F-1 score last round, let's use those same settings as last time
around:

```
>>> sklearn.metrics.f1_score(pokemon_dataset["pokemon"], pokemon_dataset["classifier_1_prediction"], zero_division=1.0, average="macro")
0.6105578822051718
>>> sklearn.metrics.f1_score(pokemon_dataset["pokemon"], pokemon_dataset["classifier_2_prediction"], zero_division=1.0, average="macro")
0.7510418886262079
```

Nice! The newer classifier looks like a very strong improvement. Let's rush to deploy it quickly without asking any more
questions. Except... did the F-1 for classifier #1 go up since we last checked? It looks to be about 1 percentage point
higher than what we calculated before at 0.60. The predictions are all the same, so how did that happen? Well, we are
using a new version of Scikit-Learn:

```
>>> sklearn.__version__
'1.3.0'
```

Maybe there's some new implementation detail that changed the order of operations, so it's producing a different
rounding error? That seems like a *really* big difference, though. I'm getting a little nervous now; let's just look at a
full classification report for the new classifier:

```
>>> print(sklearn.metrics.classification_report(pokemon_dataset["pokemon"], pokemon_dataset["classifier_2_prediction"], zero_division=1.0))
              precision    recall  f1-score   support

        Abra       0.00      0.00      1.00         4
  Aerodactyl       0.00      0.00      1.00         9
    Alakazam       0.00      0.00      1.00         4
       Arbok       0.17      0.14      0.15         7
         ...        ...       ...       ...       ...

    accuracy                           0.13       764
   macro avg       0.12      0.13      0.75       764
weighted avg       0.13      0.13      0.74       764
```

**Oh no.** Look at all of those Pokémon where our classifier got 0.0 precision & 0.0 recall, but they all get an F-1 of
1.0! 

What is going on!? Well, let's take a look at the formulae for the metrics in this classification report:

$$ \textrm{precision} = \frac{\textrm{true positive}}{\textrm{true positive} + \textrm{false positive}} $$

$$ \textrm{recall} = \frac{\textrm{true positive}}{\textrm{true positive} + \textrm{false negative}} $$

$$ \textrm{F-1} = \frac{2 \cdot \textrm{precision} \cdot \textrm{recall}}{\textrm{precision} + \textrm{recall}} $$

There are three different places here where a division by zero can occur: 

- in precision, if `true positive + false positive = 0` (the classifier made no positive predictions
  for the class)
- in recall, if `true positive + false negative = 0` (there are no truly positive examples of the class
  in the dataset)
- in F-1, if `precision = recall = 0` (the classifier has made a nonzero number of exclusively incorrect predictions)

In the first 2 cases, there is an interesting discussion to be had about how to interpret the zero-division case. I
argued in this Pokemon classification exercise that it makes sense to treat them as 1.0, but I can definitely understand
treating them as "undefined" or maybe even 0.0. But think about that third case; if precision and recall are both 0.0...
why on earth would F-1 be anything other than 0.0? Especially if `zero_division=1.0`, which means that a) there *were*
gold-labeled positive examples, b) the classifier *did* classify a nonzero number of examples as positive for the class,
and c) there is exactly zero overlap between those two sets.

Essentially, in Scikit-Learn 1.3.0, the `zero_division` parameter was turned into a kind of
[Mankey](https://bulbapedia.bulbagarden.net/wiki/Mankey_(Pok%C3%A9mon))'s
[paw](https://en.wiktionary.org/wiki/monkey%27s_paw) that determines the behavior of *all*
zero-division anywhere in the process of calculating F1. This leads to the totally nonsensical behavior, such as giving
an F1 of 100% to a classifier that gets literally every example wrong:

```
>>> sklearn.metrics.f1_score([0, 1, 0], [1, 0, 1], zero_division=1.0)
1.0
```

but only when classes are represented in both the predictions and the gold-labeled data:

```
>>> print(sklearn.metrics.classification_report([1, 1, 1], [0, 0, 0], zero_division=1.0))
              precision    recall  f1-score   support

           0       0.00      1.00      0.00       0.0
           1       1.00      0.00      0.00       3.0

    accuracy                           1.00       3.0
   macro avg       0.50      0.50      0.00       3.0
weighted avg       1.00      0.00      0.00       3.0
```

because a nonzero recall *or* (=xor) precision means that $$ \textrm{recall} \cdot \textrm{precision} \neq 0 $$, and
therefore the last stage of calculating F-1 involves no division by zero, and so F-1 is just boring old 0.0.
