# OpenSecurityTraining2 Arch1001: Binary Bomb Lab

Ce write-up explore l'exercice Binary Bomb, conçu à l'origine pour le cours d'architecture de CMU par Bryant et O'Hallaron et adapté à l'architecture Intel x86-64 par Xeno Kovah dans le cadre du cours Architecture 1001 : x86-64 Assembly de Xeno chez [OpenSecurityTraining2](https://ost2.fyi/).

Le projet Binary Bomb constitue une immersion dans l'analyse de fichiers binaires et l'ingénierie inverse sous architecture x86-64. L'objectif consiste à désamorcer six phases successives, chacune agissant comme une serrure logique dont la clé est une chaîne de caractères ou une suite de nombres. Toute erreur d'entrée déclenche une fonction d'explosion, ce qui impose une analyse statique et dynamique rigoureuse.
