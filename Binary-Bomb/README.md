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

Les noms des fonctions étant assez explicite, mais après une rapide vérification, la première compare notre entrée à une autre chaîne de caractère. Si les chaînes ne sont pas égales, le saut amenant à `phase_defused` et `phase_2` est coutourné, et un appel à `explode_bomb` est effecuté. Il faut donc découvrir le contenu à l’adresse où la chaîne est comparée.

En rentrant dans l’appel `strings_not_equal`, je peux vérifier les arguments fournis à la fonction. Dans la convention d'appel de Microsoft x64, l'argument 1 est conservé dans le registre RCX, et l'argument 2 dans le registre RDX. 
Dans WinDBG, on peut afficher la chaine ASCII à ces adresses avec : `da <address>`.

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

`strings_not_equal` compare donc mon input à “I am just a renegade hocky mom.” Comme l'entrée n'est pas égale, voyons ce qui se passe si `explode_bomb` est appelée :

![echec - bombe explose](images/boom1.png)

Je peux donc écrire `I am just a renegade hocky mom.` dans la première ligne de mon fichier `solutions.txt` et relancer le programme avec ce dernier en argument. 
Je définis directement notre breakpoint à `phase_2`, et confirme la réussite de cette première phase:

![succès - bombe diffusé](images/phase1done.png)

# Phase 2
Comme précédement, je parcours la fonction `phase_2` jusqu'à trouvé un appel intéressant. 
```asm
call    bomb!ILT+205(read_six_numbers) (00007ff7`c89710d2)
```
L'appel à la fonction `read_six_numbers` confirme que le programme attend exactement six arguments entiers. Après avoir désassemblé cette fonction (`uf read_six_numbers`), on remarque l'utilisation de `sscanf` avec un format de chaîne spécifique. 
En utilisant la commande `da 00007ff7c897c460` sur l'adresse chargée dans le registre rdx, on peut confirmer le format "%d %d %d %d %d %d".
```
0:000> da 7ff7c897c460
00007ff7`c897c460  "%d %d %d %d %d %d"
```
La fonction vérifie ensuite que la valeur de retour de sscanf (stockée dans eax) est supérieure ou égale à 6 ; dans le cas contraire, la bombe explose immédiatement.

Une fois la lecture validée, le flux d'exécution retourne dans `phase_2` pour la vérification des valeurs. L'analyse pas à pas du désassemblage de `phase_2` (via `uf phase_2`) révèle une structure de contrôle itérative.

Le désassemblage révèle que la validation commence par l'examen du premier élément de la séquence. À l'adresse `00007ff7c89720f0`, l'instruction `cmp dword ptr [rbp+rax+28h], 1` compare la première valeur saisie avec la constante 1. Le registre rax ayant été multiplié par zéro juste avant (`imul rax, rax, 0`), il sert d'index initial pour pointer sur le début de notre tableau de nombres en mémoire. Si ce premier nombre diffère de 1, le programme bifurque vers `explode_bomb`.

![phase_2](images/phase2-1.png)

Ensuite, le programme initialise un compteur à 1 (`mov dword ptr [rbp+4], 1`) et entre dans une boucle de vérification. Pour chaque itération, le code compare l'élément actuel avec l'élément précédent transformé. Le mécanisme de calcul se situe entre les adresses `00007ff7c8972113` et `00007ff7c8972129` : Le programme récupère l'index de l'élément précédent (`dec ecx`).La valeur correspondante est chargée dans le registre `ecx`.L'instruction `shl ecx, 1` est appliquée. En architecture x86-64, décaler les bits vers la gauche d'une position revient mathématiquement à multiplier la valeur par deux. Enfin, le programme compare l'élément actuel (`[rbp+rax*4+28h]`) avec ce résultat. Cette structure implique que chaque nombre de la suite doit être le double exact de son prédécesseur. En débutant la séquence par 1, nous obtenons : `1, 2, 4, 8, 16, 32`

![phase_2-2](images/phase2-2.png)

L'inscription de la suite `1 2 4 8 16 32` dans le fichier `solutions.txt` permet de franchir cette étape. En relançant le binaire avec ce fichier en argument et en configurant le breakpoint à `phase_3`, le programme valide automatiquement les deux premières phases et s'immobilise au point d'arrêt de la phase 3, confirmant l'exactitude de l'analyse.

![phase_2 done](images/phase2done.png)

# Phase 3

# Phase 4

# Phase 5

# Phase 6
