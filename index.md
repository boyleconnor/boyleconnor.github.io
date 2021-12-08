![Photo of me at the Summer Palace (颐和园) in Beijing](https://user-images.githubusercontent.com/6520892/144486930-ed6d1318-b5ec-4423-8f67-c4bdb7421fa8.jpg)

I am Connor Boyle, an M.S. student in Computational Linguistics at the University of Washington. I currently conduct research with [Tao Yu](https://taoyds.github.io/) as part of [Noah's Ark](https://noahs-ark.github.io/) and work with [Thomas Schaffter](https://cd2h.org/index.php/node/124) on [NLP Sandbox][1].

## Current Projects

### UniSKG

I am currently researching methods of encoding structured knowledge for NLP
tasks, under the supervision of Tao Yu. More details about the project will be
publicized as our work progresses.

### NLP Sandbox

[NLP Sandbox][1] is an NLP tool benchmarking project for medical notes. NLP
Sandbox allows data sites (hospitals and universities) to securely run and
evaluate containerized NLP tools on their private datasets. NLP Sandbox
currently supports two named-entity recognition tasks: named-entity recognition
(NER) for patient personal information ([PHI][2]) & COVID-19 symptoms.

I helped design the [API schemas][3] and a web client frontend ([live demo][4];
[source][5]) for NLP Sandbox PHI annotator modules. I trained and containerized
a BERT-based PHI annotator ([container source][7]; [model demo][8]) on
the [2014 I2B2 PHI dataset][6] to be benchmarked on NLP Sandbox.

## Past Projects

### NL-Augmenter

[NL-Augmenter][9] is a collaborative project to create transformations and
filters for augmenting and processing natural language datasets; it was created
as an [ACL 2021 workshop][13]. I contributed [two][10] [transformations][11]
and [one filter][12], as well as multiple bugfixes to the core codebase of the
project. A co-authored paper is forthcoming.

### Cartograph

[Cartograph][14] ([source][15]) is an interactive semantic-relatedness map of
Wikipedia articles. The maps are created by generating high-dimensional vectors
representing hyperlinks between Wikipedia articles, then projecting these
vectors into 2D space using T-SNE. I contributed to the development of
Cartograph as an NSF grant-funded undergraduate research assistant to its
creator, [Shilad Sen][16].

[1]: https://nlpsandbox.io/
[2]: https://www.hhs.gov/answers/hipaa/what-is-phi/index.html
[3]: https://github.com/nlpsandbox/nlpsandbox-schemas
[4]: https://phi-deidentifier.nlpsandbox.io/
[5]: https://github.com/nlpsandbox/phi-deidentifier-app
[6]: https://portal.dbmi.hms.harvard.edu/projects/n2c2-nlp/
[7]: https://github.com/cascadianblue/bert-phi-annotator
[8]: https://huggingface.co/connorboyle/bert-ner-i2b2
[9]: https://gem-benchmark.com/nl_augmenter
[10]: https://github.com/GEM-benchmark/NL-Augmenter/tree/main/transformations/yes_no_question
[11]: https://github.com/GEM-benchmark/NL-Augmenter/tree/main/transformations/pinyin
[12]: https://github.com/GEM-benchmark/NL-Augmenter/tree/main/filters/code_mixing
[13]: https://www.aclweb.org/portal/content/nl-augmenter
[14]: http://cartograph.info/
[15]: https://github.com/shilad/cartograph
[16]: https://www.macalester.edu/mscs/facultystaff/shiladsen/
