# OpenSecurityTraining2 Arch1001: Binary Bomb Lab

Ce write-up explore l'exercice Binary Bomb, conçu à l'origine par l'Université Carnegie Mellon pour pour le cours d'architecture de CMU et adapté à l'architecture Intel x86-64 par Xeno Kovah dans le cadre du cours [Architecture 1001 : x86-64 Assembly](https://p.ost2.fyi/courses/course-v1:OpenSecurityTraining2+Arch1001_x86-64_Asm+2021_v1/about) chez [OpenSecurityTraining2](https://ost2.fyi/).
 
Le laboratoire Binary Bomb constitue une immersion dans l'analyse de fichiers binaires et la rétro-ingénierie sous architecture x86-64. L'objectif consiste à désamorcer six phases successives, chacune agissant comme une serrure logique dont la clé est une chaîne de caractères ou une suite de nombres. Toute erreur d'entrée déclenche une fonction d'explosion.

# Sommaire 
- [Configuration](#configuration)
- [Phase 1](#phase-1)
- [Phase 2](#phase-2)
- [Phase 3](#phase-3)
- [Phase 4](#phase-4)
- [Phase 5](#phase-5)
- [Phase 6](#phase-6)

# Configuration
\> Cette analyse est réalisée sous Windows (WinDbg & Visual Studio).

\> `bomb.pdb` permet de charger les symboles, transformant les adresses mémoires brutes en noms de fonctions et de variables explicites.

\> Pour fluidifier le désamorçage, les réponses de chaque phase sont enregistrées dans un fichier texte à raison d'une solution par ligne. En passant ce fichier en argument au binaire (bomb.exe solutions.txt), le programme valide automatiquement les phases déjà résolues.


# Phase 1

Après avoir ouvert `bomb.exe` dans WinDbg et chargé les symboles (`.reload /f`), je défini un breakpoint sur la fonction `main`, puis je parcours les instructions pas à pas jusqu'à ce que le programme me demande de rentrer un input. Je rentre `test` pour observer le comportement, et nous pouvons voir que la suite est un appel à une fonction `phase_1`. 

\> `uf phase_1`pour désassembler la fonction : 
![Fonction phase_1()](images/ufphase1.png)

Deux appels de fonction nous intéressent : 

```asm
call    bomb!ILT+810(strings_not_equal) (00007ff7`c897132f)
```
&
```asm
call    bomb!ILT+945(explode_bomb) (00007ff7`c89713b6)
```

Les noms des fonctions étant assez explicite, mais après une rapide vérification, la première compare notre entrée à une autre chaîne de caractère. Si les chaînes ne sont pas égales, le saut amenant à `phase_defused` et `phase_2` est coutourné, et un appel à `explode_bomb` est effecuté. Nous devons donc découvrir le contenu à l’adresse où la chaîne est comparée.

Si nous rentrons dans l’appel `strings_not_equal`, nous pouvons vérifier les arguments fournis à la fonction. Dans la convention d'appel de Microsoft x64, l'argument 1 est conservé dans le registre RCX, et l'argument 2 dans le registre RDX. 
Dans WinDBG, nous pouvons afficher la chaine ASCII à ces adresses avec : `da <address>`.

<table style="width: 100%;">
  <tr>
    <td style="width: 50%;">
      <p align="center">
        <img src="images/registre1.png" width="100%" alt="registres">
      </p>
    </td>
    <td style="width: 50%;">
      <p align="center">
        <img src="images/dardxrcx.png" width="100%" alt="da_rcx_rdx">
      </p>
    </td>
  </tr>
</table>

`strings_not_equal` compare donc notre input à “I am just a renegade hocky mom.” Comme notre entrée n'est pas égale, voyons ce qui se passe si explode_bomb s'appelle:

![echec - bombe explose](images/boom1.png)

Nous pouvons donc écrire `I am just a renegade hocky mom.` dans la première ligne de notre fichier `solutions.txt` et relancer le programme avec ce dernier en argument. 
Nous définissons directement notre breakpoint à `phase_2`, et confirmons la réussite de cette première phase:

![succès - bombe diffusé](images/phase1done.png)


# Phase 2

# Phase 3

# Phase 4

# Phase 5

# Phase 6
