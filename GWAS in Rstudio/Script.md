
Purpose: Genetic determinism of one trait through GWAS approach


#Clean memory
rm(list=ls())

```{r}
source("fonctions_EPO.r")##create at the begining
library(lattice)
library(QTLRel)
library(qqman)
library(igraph)
```

## Importation data


```{r}
# Importation data

GRAINS<-read.table(file, header=TRUE, sep=";", dec=".")

dim(GRAINS)
names(GRAINS)
head(GRAINS,1)

```

## Preparation data
Modify the name of the first row in order 

on modifie le nom de la colonne listant les noms de lignées pour qu'elle soit compatible avec celle du fichier de données de génotypage pour l'analyse GWAS.
Puis surtout, on "choisit" quelles traits/variables on veut étudier.
En outre, on réalise les moyennes des mesures qui ont été répétées (NIRS)

```{r}
# On renomme la première colonne pour qu'elle ait le meme nom que sur le fichier de genotypes
names(GRAINS)[1]<-"Code.unique.2010"

# Juste pour voir, on liste les noms des lignées présents dans la colonne code.unique.2010
GRAINS[,1]

# Création d'une base de données (DATA) qui ne contient qu'un sous-ensemble des colonnes (donc des données de phénoytpage) du tableau complet initial
DATA<-GRAINS[,c(1:12, 15:16, 19:20, 23, 26, 25, 30, 116)]

# réalisation des moyennes entre les deux mesures de protéine mesurées à partir d'un lot de grains abondants (GA) d'une part, et du lot de grains réduit à partir duquel les broyages ont été faits (GB) d'autre part. Réalisation des moyennes entre les deux mesures de jaune, de mitadin, puis de PS. Les valeurs sont créées dans des colonnes nouvelles situées à la suite de la dernière colonne.
DATA$ProteineGA<-(DATA$proteines_1 + DATA$proteines_2 )/2
DATA$ProteineGR<-(DATA$proteines_3 + DATA$proteines_4 )/2
DATA$Jaune <-(DATA[,15]+ DATA[,16] )/2
DATA$Mitadin <-(DATA[,13]+ DATA[,14] )/2
DATA$PS <-(DATA[,11]+ DATA[,12] )/2

#suppression des données "brutes" initiales pour ne conserver que les moyennes
DATA<-DATA[,c(1:6, 17:ncol(DATA))]
names(DATA)
```

# Exploration des données phenotypiques

## Distribution des caractères

```{r}
# pour regarder tous les histogrammes des distributions des variables
for ( i in 2:ncol(DATA)) { hist(DATA[,i], main=names(DATA)[i]) }

#diagrammes 2 à 2 
pairs(DATA[,c(2:ncol(DATA) )])

```

## Quels sont les caractères corrélés au Fer et au Zinc ?

```{r}
cor(DATA[,-1], DATA[,"Zn"], use="pairwise.complete.obs")
cor(DATA[,-1], DATA[,"Fe"], use="pairwise.complete.obs")

```

## Recherche du point bizarre sur Zn/Fe 

```{r}
DATA$ZN_FE<- DATA$Zn / DATA$Fe
hist(DATA$Zn.Fe)

DATA[which(DATA$Zn.Fe > 4 ),]

hist(DATA$Zn)
hist(DATA$Fe)

plot(DATA$Zn, DATA$Fe)
plot(DATA$Zn.Fe, DATA$ZN_FE)

# Remplacer une valeur aberrante
DATA$Fe[which(DATA$Zn.Fe > 4 )]<-NA


# Recalcul de la variable
DATA$Zn.Fe<- DATA$Zn / DATA$Fe

hist(DATA$Zn.Fe)


cor(DATA[,-1], DATA[,"Zn"], use="pairwise.complete.obs")
cor(DATA[,-1], DATA[,"Fe"], use="pairwise.complete.obs")
cor(DATA[,"Fe"], DATA[,"Fe"]/DATA[,"Zn"])

```

## Recherche de variables synthétiques

```{r}

# quelles sont les données manquantes
 

DATA_sub<-DATA[which(!is.na(DATA$Fe)),]
dim(DATA)
dim(DATA_sub)

DATA_sub<-DATA_sub[which(!is.na(DATA_sub$Zn)),]
DATA_sub<-DATA_sub[-1,]

head(DATA_sub, 50)

names(DATA)
for (i in 2:ncol(DATA_sub)) { 
  
   DATA_sub<-DATA_sub[which(!is.na(DATA_sub[,i])),]
   
  }


cor(DATA_sub[,-1], DATA_sub[,"Zn"], use="pairwise.complete.obs")
cor(DATA_sub[,-1], DATA_sub[,"Fe"], use="pairwise.complete.obs")
cor(DATA_sub[,"Fe"], DATA_sub[,"Fe"]/DATA_sub[,"Zn"])
cor(DATA_sub[,"Zn"], DATA_sub[,"Fe"]/DATA_sub[,"Zn"])

plot(DATA_sub[,"Cu"], DATA_sub[,"Zn"])
plot(log10(DATA_sub[,"Zn"]), log10(DATA_sub[,"Fe"]))
plot(log10(DATA_sub[,"Cu"]), log10(DATA_sub[,"Fe"]))
plot(log10(DATA_sub[,"Cu"]), log10(DATA_sub[,"Zn"]))


m1<-lm(DATA_sub[,"Cu"] ~ DATA_sub[,"Fe"] )

A<-DATA_sub[,"Fe"]^2

m2<-lm(DATA_sub[,"Cu"] ~ DATA_sub[,"Fe"] + A )

anova(m1,m2)


# Pricipal Components Analysis
# entering raw data and extracting PCs
# from the correlation matrix

fit <- princomp(DATA_sub[,2:ncol(DATA_sub)], cor=TRUE)

summary(fit) # print variance accounted for
loadings(fit) # pc loadings
plot(fit,type="lines") # scree plot
fit$scores # the principal components
biplot(fit) 


# comment se faire une idée sur des données simulées
Zn<-rnorm(1000, mean=10, sd=1)
Fe<-0.7*Zn + rnorm(1000, mean=0, sd=0.01)

hist(Zn)
hist(Fe)
plot(Zn, Fe)

cor(Zn, Fe)

cor(Zn, Fe/Zn)

```



## Fichier de génotypes prêt à l'emploi
Le fichier contenantles données Axiom pour 168 K SNP a été préparé pour cette manipe spécifique.
L'objet généré s'appelle G_EPO.

```{r}

load("G_EPO.Rdata")
dim(G_EPO)

```

# Elimination des MAFS
## Calcul des fréquences alleliques

Nous pouvons calculer les fréquences alléliques sur la matrice de lignées non redondantes et observer leur distribution.

```{r}
Freq_EPO<-freqall(G_EPO)
hist(Freq_EPO, main="Distribution des fréquences alléliques")

```

Les minor alleles frequencies sont supprimées pour un seuil de 5% ce qui fait à peu près 10 individus minimum 

```{r}
MAF<-which(Freq_EPO<0.05|Freq_EPO>0.95)
G_EPO<-G_EPO[,-MAF]
dim(G_EPO)

```

# remplacement des données manquantes
```{r}

nom_row<-rownames(G_EPO)
nom_col<-colnames(G_EPO)

G_EPO<-apply(G_EPO,MARGIN=2,FUN=tirage)
dim(G_EPO)

rownames(G_EPO)<-nom_row
colnames(G_EPO)<-nom_col


```

Nous pouvons re calculer les fréquences alléliques et observer leur distribution.

```{r}
Freq_EPO<-freqall(G_EPO)
hist(Freq_EPO)


```


## La matrice G est la matrice des génotypes
```{r}

# QTL Rel demande des formats AA, BB et AB
GA<-G_EPO
GA<-as.matrix(GA)

GA[which(GA==0)]<-"AA"
GA[which(GA==1)]<-"AB"
GA[which(GA==2)]<-"BB"

```


## Fusion des fichiers des  valeurs phénotypiques et des génotypes

Les deux fichiers sont fusionnés ensemble par la fonction merge et le nom EPO utilisé comme clef de tri.

```{r}
names(DATA)
head(rownames(G_EPO),1)

# le rang de rownames dans l'ordre des colonnes est 0, d ou le by.y=0
# le rang de la variable nom dans GRAINS est 1 d'ou le by.x=1
PG<-merge(DATA, GA, by.x=1, by.y=0)
dim(PG)

PG[1:3, 1:10]


```

La matrice PG comporte à ce stade 180 lignées et 97 494 marqueurs.


# GWAS & le calcul des pvalues

## Choix de la variable à analyser et préparation du fichier.

```{r}
head(names(PG), 20)

```

On voit que le premier marqueur commence en 18.
Revenir à cette étape pour changer de variable. Changer la valeur de i.


```{r}
# mettre ici le numero de la variable
i<-4

Y<-PG[,i]

liste<- which(!(is.na(Y)))
Y<-Y[liste]
  
hist(Y, main=names(PG)[i])

# les marqueurs commencent en 18
G<-PG[liste, 18:dim(PG)[2]]

length(Y)


```

## Calcul de la matrice de Kinship

```{r}

# avec 10000 marqueurs tirés au hasard ca passe en 2-3 minutes 
# la foncton genMatrix de QTLRel fait le calcul des matrices 
# ici il n'y en a que 1000
liste <- sample(colnames(G),1000,replace=FALSE)
K<-genMatrix(G[,liste])
I<-diag(length(Y))

hist(K$AA)

```


## Calcul des composantes de la variance et estimation des covariances dues à la kinship

Le modèle estime une variance génétique additive et une variance environnementale.

```{r}

# le premier modele sert a calculer la variance
mod1<-estVC(y=Y,v=list(AA=K$AA,DD=NULL,HH=NULL,AD=NULL,MH=NULL,EE=I))
mod1


```

## Boucle sur tous les marqueurs retenus

La fonction scanOne va faire une analyse par marqueur en utilisant les composantes de la variance calculées précédemment.

```{r}

print(date())
GWAS.mod1<-scanOne(y=Y,gdat=G,vc=mod1,test="Chisq")
print(date())

# il faut 25 min pour les 93000 marqueurs

```

## Stockage des pvalues pour une étude ultérieure
```{r}

pval.mod1<-data.frame(marqueurs=colnames(G),pvalue=GWAS.mod1$p)

# stocker les valeurs 
varnom<-names(PG)[i]

file<-paste("GRAINS_GWAS.mod1_",varnom,".Rdata", sep="") 

save(GWAS.mod1, file=file, compress= TRUE)

file<-paste("GRAINS.pval.mod1_",varnom,".Rdata", sep="")
save(pval.mod1, file=file, compress= TRUE)

```


# Post traitement des données

Cette étape peut repartir des fichiers de pvalues stockées.

```{r}
library("data.table")
library(ggplot2)

```


## Charger les packages pour l'étude des qvalues 

Pas indispensables ici mais utile de réfléchir au nombre de faux positifs
```{r}
#http://www.bioconductor.org/packages/release/bioc/vignettes/qvalue/inst/doc/qvalue.pdf
#https://github.com/jdstorey/qvalue

#install.packages("devtools")

library("devtools")

#install_github("jdstorey/qvalue")
library(qvalue)

```


## Données de pvalues
Il s'agit là de récuperer pour l instant simplement les noms des variables étudiées


```{r}
varnom <- "moy_prot"

file<-paste("GRAINS_GWAS.mod1_",varnom,".Rdata", sep="")

load(file)

pvalues<-GWAS.mod1$p
length(GWAS.mod1$p)

pval.mod1<-data.frame(Marker_ID=names(GWAS.mod1$p),pvalue=pvalues)


```

## La valeur de la pvalue du SNP le plus significatif
```{r}

min(pval.mod1$pvalue)


```

## La distribution des pvalues
```{r}
hist(pval.mod1$pvalue,nclass=20, main=paste("Pvalues on ", varnom, sep=""))



```

## les qvalues
```{r}
pvalues<-pval.mod1$pvalue

qobj<-qvalue(p=pvalues)

summary(qobj)

# controle du taux de fdr
#qobj<-qvalue(p=pvalues, fdr.level=0.1)
# nom des marqueurs a taux de fdr controle
#as.character(pval.mod1[which(qobj$significant==TRUE),1])

#  qvalues and pvalues
plot(qobj)

# histogrammes des q values
hist(qobj)


# qq plot
qq(pval.mod1$pvalue,main=paste("QQplot on", varnom, sep=""))
```

Conclusions 
En ce qui concerne le caractère moy_prot il ne semble pas y avoir de SNP lié au caractère...


## Exploration des valeurs intéressantes
```{r}

# seuil à 4
seuil.mod1<--log10(0.0001)

assoc.mod1<-pval.mod1[-log10(pval.mod1$pvalue)>=seuil.mod1,]
dim(assoc.mod1)

# histrogramme des pvalues positives
hist(-log10(assoc.mod1[,2]), 
     main = paste("Distribution du -log de la pvalue pour ",varnom,sep=""),
     xlab= " logarithme de la pvalue" ,
     ylab= " Effectif" ,
     col="blue", border="white")


```


## Les données de positions des marqueurs sur le blé dur

```{r}
file_Phys_DRW<-"BREEDWHEAT_on_durum_physic.Rdata"

# le fichier s'appelle BLAST
load(file_Phys_DRW)

names(BLAST)
SNP_PHYS<-BLAST
SNP_PHYS<-as.data.frame(SNP_PHYS)
names(SNP_PHYS)

```


## Les positions des marqueurs intéressants sont elle connues ?

```{r}

Positif_280K<-merge(SNP_PHYS,assoc.mod1, by.x=1, by.y=1 )
names(Positif_280K)

Positif_280K
```

Chr le chromosome ou se trouve le SNP candidat
Start sa position physique sur le génome de référence
ensuite suivent les paramètres de BLAST
et enfin la pvalue est redonnée



