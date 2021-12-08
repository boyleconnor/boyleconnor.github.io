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
a BERT-based PHI annotator ([container GitHub][7]; [HuggingFace demo][8]) on
the [2014 I2B2 PHI dataset][6] to be benchmarked on NLP Sandbox.

## Past Projects

To be continued

[1]: https://nlpsandbox.io/
[2]: https://www.hhs.gov/answers/hipaa/what-is-phi/index.html
[3]: https://github.com/nlpsandbox/nlpsandbox-schemas
[4]: https://phi-deidentifier.nlpsandbox.io/
[5]: https://github.com/nlpsandbox/phi-deidentifier-app
[6]: https://portal.dbmi.hms.harvard.edu/projects/n2c2-nlp/
[7]: https://github.com/cascadianblue/bert-phi-annotator
[8]: https://huggingface.co/connorboyle/bert-ner-i2b2
